# `loop` — STOP 로직 (termination)

> 보편 종료 판정 레퍼런스. DESIGN §0(invariant #4 "결정론적 종료")·§7(STOP 블록)의 산문 확장.
> `score`(references/scoring.md)가 점수를 내고, `run`(references/cycle.md / driver.js)이 라운드를 돌면,
> **이 문서가 "이제 멈출까/사람을 부를까/계속할까"를 결정**한다.
>
> 4갈래 결정 — `STOP` / `ESCALATE` / `BLOCKED` / `CONTINUE`. 그뿐이다. 모호한 "거의 다 됐다"로 멈추지 않는다.
> 전부 `loop.yaml`의 `stop{target, consecutive, full_audit_every}`로 파라미터화된다 — 하드코딩 금지.

---

## 0. 왜 종료가 결정론이어야 하나

루프의 5 invariants(DESIGN §0) 중 #4가 "결정론적 종료"다. 종료가 사람의 직감("이 정도면 됐지")에 맡겨지면
두 가지가 동시에 깨진다:

- **거짓 STOP** — 국소 회귀를 못 본 채 "done" 선언 → comprehension-debt.
- **무한 루프** — 무엇이 "끝"인지 숫자로 안 박혀서 영원히 다음 deficit만 처리.

그래서 STOP은 **숫자 임계(`target`) × 연속 횟수(`consecutive`) × 권위 점수(full audit)** 세 조건의 곱으로만
선언한다. 이 세 가지가 동시에 참이 아니면 멈추지 않는다. 반대로 참이면 *반드시* 멈춘다(주관 개입 없음).

---

## 1. 결정 4갈래 (decision table)

매 라운드 끝(references/cycle.md [5단계] / driver.js의 stop-check)에서 아래 표를 위→아래 순서로 평가한다.
**첫 번째로 매칭되는 행이 그 라운드의 verdict**다(짧은 순환, short-circuit).

| 우선순위 | 조건 | verdict | 동작 |
|---------|------|---------|------|
| 1 | `blocker` 신규 발견 (룰/스펙/cert/외부의존 부재) | **BLOCKED** | 해당 dimension BLOCKED 표기 → 분자·분모 제외(§4) → surface → 그 dimension은 triage 풀에서 제외하고 **다음 deficit으로 CONTINUE** |
| 2 | **full audit** 점수 `active_total ≥ target` 가 직전 full과 **연속 `consecutive`회** | **STOP** | done 선언 → 브랜치 핸드오프(auto-merge 금지) → 루프 종료 |
| 3 | 동일 deficit이 `consecutive` 라운드(기본 2) 미해결 (no_progress 누적) | **ESCALATE** | 사람 호출 → 그 deficit surface → 루프 정지(인간 재트리거 대기) |
| 4 | 그 외 | **CONTINUE** | triage 다음 deficit으로 진행 |

> 우선순위가 중요하다. blocker가 STOP보다 위인 이유: blocker를 분모에서 빼지 않으면 `active_total`이
> `target`에 영원히 못 닿아(상한이 `target` 미만으로 눌림) STOP 조건 자체가 거짓이 된다 → §4의 재정규화가
> 먼저 일어나야 STOP 평가가 정상 작동한다. ESCALATE가 STOP보다 아래인 이유: 점수가 목표에 닿았다면
> 그 라운드의 막판 deficit이 미해결이어도 "done"이 우선(이미 목표 달성).

---

## 2. STOP — full 점수로만, per-round 금지 ⚠️

**STOP은 `full` 모드(전 dimension 재채점) 점수로만 선언한다.** per-round(터치한 dimension 1개만 채점한)
부분 점수로는 절대 STOP을 선언하지 않는다.

이유 (DESIGN §7 "STOP은 full 점수로만 인정"):

- per-round는 **방금 고친 dimension만** 본다. 그 fix가 **다른 dimension을 회귀**시켰을 수 있다
  (교차 오염 — 예: 공유 모듈 수정이 옆 도메인의 계약을 깸).
- per-round 점수는 "이 축은 좋아졌다"는 국소 신호일 뿐, **시스템 전체가 target에 도달했다**는 권위 신호가
  아니다. binding constraint(최약 고리)는 *전수 재채점*에서만 드러난다.

### STOP 판정 알고리즘

```
on each round-end:
  if mode != "full":
      # per-round 라운드는 STOP 후보가 될 수 없다. consecutive 카운터 건드리지 않음.
      return CONTINUE (or ESCALATE/BLOCKED per §1)

  # 여기 도달 = 이번 라운드가 full audit
  if active_total >= stop.target:
      consecutive_at_target += 1            # checkpoint.json 에 영속
  else:
      consecutive_at_target = 0             # 한 번이라도 미달이면 리셋 (연속이 깨짐)

  if consecutive_at_target >= stop.consecutive:
      verdict = STOP                        # done — 루프 종료, 브랜치 핸드오프
  else:
      verdict = CONTINUE                    # 아직 — 다음 deficit
```

- `stop.target` (기본 98) — 활성 종합 임계.
- `stop.consecutive` (기본 2) — target을 **연속** full-audit 횟수. 1회로는 STOP 금지(우연·일시 통과 배제).
  연속이 깨지면(중간에 한 번이라도 < target) 카운터를 0으로 리셋한다 — "연속"의 의미를 지킨다.
- `consecutive_at_target`는 **full audit에서만** 증감한다. per-round 라운드는 이 카운터를 건드리지 않는다.

### full audit는 언제 도나

`stop.full_audit_every` (기본 3)가 권위 재채점 주기다. 두 시점에 full을 돈다:

1. **주기 도달** — `round_index % full_audit_every == 0` (예: 3, 6, 9라운드째).
2. **STOP 후보 도달** — per-round working score가 target을 넘긴 직후 → **즉시 full audit 1회 강제**해서
   진짜 STOP인지 확정. (per-round가 "넘은 것 같다"고 하면 full로 검증 — per-round 단독 STOP 금지의 실천.)

> 즉 STOP 선언에는 항상 **연속 `consecutive`개의 full audit**가 깔려 있다. per-round 라운드들은 그 사이를
> 메우지만 STOP 판정에 표를 던지지 못한다.

### `consecutive`개 full을 어떻게 모으나 (해석)

`full_audit_every:3`, `consecutive:2`라면, STOP은 *최소 두 번의 full audit*에서 연속으로 target을 넘겨야
선언된다. 두 full audit는 보통 `full_audit_every` 라운드 간격으로 떨어져 있다(예: R6 full=98, R9 full=99 →
연속 2 → STOP). 그 사이 per-round 라운드(R7·R8)에서 target 아래로 떨어졌더라도, **full audit 간**의 연속만
보면 되므로 카운터는 full 시점에서만 비교한다 — per-round 노이즈가 STOP을 막지도, 앞당기지도 않는다.

---

## 3. ESCALATE — no-progress 2라운드

같은 deficit을 `consecutive`(기본 2) 라운드 연속 시도했는데도 **점수가 안 오르고 verify가 계속 FAIL**이면,
루프는 그 자리에서 헛도는 것이다. 무한 재시도 대신 **사람을 부른다**(ESCALATE).

### no_progress 카운터 규칙 (checkpoint.json)

```
after verify + re-score, for the deficit this round targeted:
  if verify == FAIL  OR  dimension_score_delta <= 0:   # 안 고쳐졌거나 점수 정체/후퇴
      if same_deficit_as_last_round:
          no_progress += 1
      else:
          no_progress = 1            # 새 deficit에서 막힘 — 1부터
  else:                              # 진전 있음 (verify PASS & 점수 상승)
      no_progress = 0                # 리셋

  if no_progress >= stop.consecutive:   # 기본 2
      verdict = ESCALATE
```

- **"동일 deficit"** 판정 — `checkpoint.json.next`(이번 라운드가 고른 deficit id)가 직전 라운드와 같은지로
  본다. 라운드 안에서 maker가 1회 재fix해도 실패하면 그 라운드 자체가 no-progress 1로 집계된다
  (references/cycle.md [3] verify: 같은 라운드 1회 재fix → 또 실패면 no_progress +1).
- ESCALATE는 **그 deficit을 surface**하고 루프를 정지시킨다. 사람이 (a) 막힌 원인을 풀거나, (b) 그 deficit을
  BLOCKED로 재분류하거나, (c) triage 순서를 바꿔 우회하도록 재트리거한다.
- 진전이 한 번이라도 있으면(verify PASS + 점수 상승) `no_progress`를 0으로 리셋한다 — 누적 오판 방지.

> ESCALATE ≠ 실패. "이 deficit은 자동 루프가 풀 범위를 넘었다"는 **정직한 정지**다. 거짓 진전(점수만 만지작)
> 보다 정지가 싸다.

---

## 4. BLOCKED — 분모 제외 + surface

`blocker`는 **루프가 코드로 풀 수 없는 외부 의존**이다 — 미확정 비즈니스 룰, 미수령 스펙/cert, 외부 API
cutover(범위 밖), 인간 결정 대기 등. 이런 dimension은 점수를 올릴 *방법이 없으므로*, 분모에 남겨두면
`active_total`이 영원히 `target`에 못 닿는다(STOP 불능 → 무한 루프).

### 재정규화 (DESIGN §5 step 5 / scoring.md §4.1a와 동일 공식)

BLOCKED dimension은 **분자에서도 분모에서도** 뺀다. 분모에서만 빼면 틀린다.

```
blocked_weight  = Σ( weight_b )                 # BLOCKED dimension들의 가중 지분 합
blocked_contrib = Σ( score_b × weight_b )       # 그 dimension들의 점수 기여 합

active_total = ( Σ_all( score_i × weight_i )  −  blocked_contrib )
               ─────────────────────────────────────────────────
                          ( 1  −  blocked_weight )
```

검산 예시 (단일 BLOCKED dimension, weight 0.027, score 50, raw Σ = 57.98):

```
blocked_contrib = 50 × 0.027 = 1.35
active_total    = (57.98 − 1.35) / (1 − 0.027) = 56.63 / 0.973 ≈ 58   ✓
# ⚠️ 흔한 오류: 57.98 / 0.973 = 59.6  ← 분자에서 blocked_contrib를 안 뺀 값. 틀림.
```

`active_total`(분자·분모 양쪽 보정)이 §2 STOP 판정의 입력이다. raw total이 아니다.

### BLOCKED 동작 순서

1. **표기** — 해당 dimension을 `scorecard.json.blocked[]` + `checkpoint.json`에 BLOCKED로 기록.
   `scorecard.md` §점수이력 비고에 "왜 blocked인지"(부재한 룰/스펙) 1줄.
2. **제외** — 위 공식으로 `active_total` 재계산(blocked pool 분자·분모 동시 제외).
3. **surface** — 사람에게 "이 dimension은 X(룰/cert) 부재로 BLOCKED, 외부 입력 들어오면 활성 복귀" 보고.
4. **CONTINUE** — BLOCKED dimension은 triage 풀에서 빠지고, 루프는 **다음 deficit으로 계속**(BLOCKED 하나가
   전체 루프를 세우지 않는다).

### 활성 복귀 (un-block)

blocker가 해소되면(룰 확정·cert 수령) `blocked[]`에서 제거 → 분자·분모 원복 → 다음 full audit에서 그
dimension을 정상 재채점한다. 복귀 라운드는 점수가 출렁일 수 있으므로 그 라운드는 STOP 후보로 보지 않고
다음 full에서 안정화 확인.

### anti-gaming — BLOCKED 남용 금지

BLOCKED는 **외부 의존 부재**일 때만이다. "고치기 귀찮다/어렵다"를 BLOCKED로 숨기면 점수가 거짓 상승한다
(분모가 줄어 active_total↑). 판정 기준:

- ✅ BLOCKED: 우리 코드 밖의 입력(미확정 룰, 미수령 명세/cert, W13 외부 cutover)이 없으면 *원리적으로* 못
  고치는가? → 예면 BLOCKED.
- ❌ BLOCKED 아님: 코드로 고칠 수 있는데 시간/난이도 때문에 미룬 것 → 이건 그냥 미완 deficit(점수 낮게 유지,
  triage 풀에 남김). 막혀서 못 풀면 그건 ESCALATE(§3)지 BLOCKED가 아니다.

> 구분: **BLOCKED = 외부가 막음**(분모 제외 정당) / **ESCALATE = 루프가 못 풂**(분모 유지, 사람 호출).

---

## 5. checkpoint.json — 종료 상태의 디스크 표현

종료 판정에 필요한 상태는 전부 `.loop/checkpoint.json`에 영속한다(DESIGN §1 "디스크=메모리"). 메인 세션이나
driver.js는 컨텍스트가 아니라 이 파일만 읽고 종료를 재평가한다 — 멀티라운드 컨텍스트 폭발 방지.

```jsonc
{
  "next": "<다음 triage deficit id>",      // CONTINUE 시 다음 라운드가 집을 deficit
  "no_progress": 0,                         // §3 — 동일 deficit 연속 미진전 카운트
  "consecutive_at_target": 0,               // §2 — full audit 연속 target 도달 횟수
  "last_verdict": "CONTINUE",               // STOP | ESCALATE | BLOCKED | CONTINUE
  "active_total": 58,                       // §4 재정규화 후 종합(STOP 입력값)
  "last_full_round": 0,                     // 마지막 full audit 라운드 인덱스(주기 계산용)
  "blocked": []                             // BLOCKED dimension id 목록(분모 제외 대상)
}
```

- 매 라운드 끝에 verdict 산출 후 **이 파일을 먼저 갱신**하고 보고/정지한다(영속이 보고보다 먼저 — 중단돼도
  재진입 가능).
- `consecutive_at_target`·`no_progress`는 위 알고리즘대로만 증감(임의 조작 금지 — 그 자체가 anti-gaming).
- driver.js의 루프 탈출 조건 = `last_verdict ∈ {STOP, ESCALATE}` 또는 라운드 예산 소진(`maxRounds`).
  BLOCKED는 탈출이 아니라 CONTINUE(해당 dimension만 제외).

---

## 6. loop.yaml 파라미터 (이 문서가 읽는 설정)

```yaml
stop:
  target: 98              # active_total STOP 임계 (§2)
  consecutive: 2          # ① target 연속 full 횟수(STOP) ② 동일 deficit 미진전 한도(ESCALATE) 공용
  full_audit_every: 3     # 권위 재채점 주기(§2) — 이 주기 + STOP 후보 도달 시 full
```

- 세 값 모두 `loop.yaml.stop`에서 읽는다. **하드코딩 금지** — Stanley는 98×2×3이지만 다른 프로젝트는
  85×3×5일 수 있다. 이 문서의 알고리즘은 값에 독립.
- `consecutive`가 STOP(연속 target)과 ESCALATE(연속 no-progress) 양쪽에 쓰이는 건 의도된 단일 다이얼이다
  ("얼마나 끈질기게 볼까"의 한 노브). 두 의미를 분리하고 싶은 프로젝트는 yaml을 확장하되 기본은 공용.

---

## 7. 안전·범위 가드 (DESIGN §8 준수)

종료 로직도 보편 안전 불변을 넘지 않는다:

- **auto_merge: false 불변** — STOP은 "done 선언 + 브랜치 핸드오프"까지다. 머지는 항상 사람. STOP이 머지를
  트리거하지 않는다.
- **local-only** — BLOCKED 판정의 "외부 cutover/cert 부재"는 그 자체로 local 범위 밖 = 자동으로 BLOCKED
  처리(루프가 prod/remote write로 풀려 하지 않는다). `safety.forbid` 위반을 요구하는 deficit은 BLOCKED 또는
  ESCALATE로 surface, 자동 실행 금지.
- **마이그레이션** — local 적용·검증까지만 STOP 입력에 반영. 원격(QA/prod) 적용이 남았으면 그 dimension은
  "원격 게이트 대기"로 BLOCKED 표기 후 surface(루프가 원격에 쓰지 않는다).

---

## 8. 한 줄 요약

> **STOP** = full 점수 `active_total ≥ target`가 `consecutive`회 연속 (per-round 금지) ·
> **ESCALATE** = 동일 deficit `consecutive`라운드 미진전 ·
> **BLOCKED** = 외부 의존 부재 → 분자·분모 제외 + surface + 다음 deficit으로 CONTINUE ·
> 전부 `loop.yaml.stop`로 파라미터화, 상태는 `checkpoint.json`에 영속, 머지는 끝까지 사람.
