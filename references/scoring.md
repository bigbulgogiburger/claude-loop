# `loop score` — 측정 엔진 (universal)

> DESIGN §5 의 구현 레퍼런스. `loop` 스킬이 **score 모드**로 분기하면 이 절차를 그대로 인코딩해 실행한다.
> 이 문서는 *점수 함수의 실행기*다. 채점 대상(축·가중치·probe·증거 anchor)의 **데이터 SSoT 는
> `<project>/.loop/loop.yaml`(기계) + `.loop/scorecard.md`(rubric 산문)** 이고, 이 문서는 **절차만**
> 인코딩한다 — 중복 정의 금지(jira-ingest 가 `INDEX-SCHEMA.md` 로 정책을 빼낸 것과 동형).
>
> Stanley CS 의 `completeness-score` 스킬(7도메인 고정 probe map)을 보편화한 것 — Stanley 의 7도메인은
> 이제 그 프로젝트 `.loop/loop.yaml` 의 `dimensions[]` 한 인스턴스일 뿐이다. 여기엔 도메인 이름이 없다.

---

## 0. 왜 이렇게 채점하나 (5 invariants 중 anti-gaming 의 자리)

DESIGN §0 invariant #3: **"숫자를 못 속인다"**. 점수가 의미 있으려면
**"코드가 존재한다"가 아니라 "관찰된 동작이 목표를 충족한다"** 로만 매겨야 한다.

정적분석(grep / 타입체커 / 테스트 개수)은 **구조 결함만** 잡고, 런타임 500·캐시 HIT 시에만
터지는 버그·IDOR·권한 누설·보존법칙 위반(원장 불변식)·계약 미스를 **못 본다**. 그래서 이 엔진의
모든 점수 상승은 **실환경 신호**(`probe.kind` 의 db/http/test/shell/grep 결과 — DB SUM 불변 /
200+실데이터 / 비-권한 403 / 스택트레이스 부재 / 벤치)에 결박한다. 이게 score-gaming
(실패테스트 삭제·미구현 "done" 체크·정적 0건을 PASS 로 셈)을 막는 유일한 방법이다.

> **probe 는 local 범위만** (DESIGN §8, safety.scope=local-only). prod/remote write·외부 cutover·
> 자격증명·force push 금지. DB probe 는 read-only. `safety.forbid` 가 차단 목록이며 위반 시 abort+surface.

---

## 1. 입력: 채점 범위 결정 (per-round vs full)

먼저 무엇을 재채점할지 정한다. 호출 의도에서 추론한다:

| 모드 | 시점 | 범위 | 비용 | 트리거 |
|------|------|------|------|--------|
| **per-round** (기본) | fix 라운드 직후 | **터치한 dimension 1개만** | 저 | 도메인/축 지정·"방금 고친 거 재채점" |
| **full** | `stop.full_audit_every` 라운드마다 + STOP 후보 도달 시 | **전 dimensions** | 고 | "full"·"전체"·"re-audit"·STOP 판정 직전 |

규칙:
- 호출자가 특정 축을 지정 → **per-round**(그 축 + 직접 의존 축만).
- "full"/"전체"/"re-audit" 또는 `active_total` 이 `stop.target` 근접 → **full**.
- **STOP(`active_total ≥ target` ×`consecutive`) 판정은 full 점수로만 인정** — per-round 부분점수로
  STOP 선언 금지(국소 회귀·교차오염 미검출. DESIGN §7 "STOP 은 full 점수로만").

---

## 2. 절차 개요 (DESIGN §5 의 6단계)

```
1 사전조건: env.start 살았나? 죽었으면 live-probe 신호 = STALE (거짓 상승 방지)
2 신호수집: dimension 별 probe 실행
            - full:      dimension 당 sub-agent fan-out(병렬), compact 결과만 회수
            - per-round: 터치 dimension 1개만(sub-agent 1 또는 경량 inline)
3 band:     관찰된 동작 → FULL / PARTIAL / CONFLICT / MISSING 점수대
4 blend:    dimension 점수 = 가중 blend (단순평균 ❌) — binding constraint(최약 고리) 지배
5 normalize: BLOCKED 항목 분자·분모 제외 → active_total (§4.1a 공식)
6 출력:     scorecard.json + checkpoint.json + (full이면)scorecard.md §이력 append + STOP 판정
```

### 컨텍스트 규율 (오케스트레이터를 얇게 — 디스크=메모리)

이 스킬은 메인 세션 컨텍스트에서 돈다. 통독·전수 inline 채점은 컨텍스트를 폭발시킨다. 규율:

- **선택적 읽기**: `scorecard.md`(클 수 있음)를 통독하지 말 것. per-round 면 해당 dimension 의
  rubric 절 + §점수이력 마지막 행만, full 이면 dimension 헤더 점수들 + weights 만 Read(offset/Grep).
  증거 anchor 는 필요할 때 그 `file:line` 만.
- **probe 는 sub-agent 에 위임, compact 만 회수**: dimension probe(curl/DB/grep 묶음)는 sub-agent 가
  자기 컨텍스트에서 실행하고 **{dimension, 점수, 등급, 핵심증거 3줄}만 반환**. full 모드 =
  dimension N개 sub-agent fan-out(병렬), 메인은 N개 compact 결과만 합산. probe 출력 raw 를 메인이
  들고 있지 말 것.
- **상태는 디스크에**: 결과를 컨텍스트가 아니라 `.loop/scorecard.json` + `scorecard.md` §이력 +
  `.loop/checkpoint.json` 으로 영속(6단계). 다음 라운드/세션은 디스크만 읽고 재진입.

---

## 3. 1단계 — 사전조건 확인 (실환경 살아있나)

실환경 probe 가 점수의 큰 비중(런타임/보안 신호)을 좌우하므로, 죽어 있으면 해당 신호를 **PASS 로 세지
말고 STALE 로 표기**한다(거짓 상승 방지. DESIGN §5-1 "live 죽었으면 STALE").

`.loop/loop.yaml` 의 `env` 블록을 그대로 실행:

| env 키 | 확인 내용 | 흔한 함정 |
|--------|-----------|-----------|
| `env.start` | 앱 기동 + **readiness 폴링** (포트 LISTEN 아니라 "ready" 신호 확인) | stale 프로세스가 포트만 squat → 옛 코드 응답. PID + ready 로그로 확인 |
| `env.db` | read-only DB probe 도달 (local only) | 원격 DB 절대 write 금지. 접속 옵션(ssl mode 등)은 loop.yaml 에 |
| `env.test` | 테스트 커맨드 1회 smoke | 데몬/플래그 주의(예: gradle `--no-daemon` 이 mock self-attach 깨짐 등 프로젝트별 함정은 loop.yaml 주석에) |
| `env.auth` | **비-권한 토큰** 획득 (IDOR/scope probe 용) | 슈퍼유저-only 검증은 전부 200 이라 권한 누설 미검출 → 반드시 저권한 계정 |

**라이브가 죽어 있고 띄울 수 없으면**: 결정론적-정적 신호(test/grep/db-readonly)만 채점하고
live-probe(http/auth) 파트는 **STALE(직전 점수 유지 + 플래그)** 로 두고, 종합에 "live-probe N건 STALE"
을 명시한다. STALE 동안엔 **STOP 선언 금지**(거짓 98 위험).

---

## 4. 2단계 — dimension 별 신호 수집 (probe 실행)

각 dimension 을 `loop.yaml` 의 `dimensions[].probe` 로 측정한다. `probe.kind` 별 의미:

| `probe.kind` | 무엇을 묻나 | "PASS 모습"(`probe.expect`) 예 |
|--------------|-------------|--------------------------------|
| `db` | 원장/제약/보존 불변식 (read-only) | 출고 전후 `SUM(qty)` 불변 · UNIQUE 제약 DDL 에 존재 · orphan FK 0건 |
| `http` | 엔드포인트 실동작 (200+실데이터·계약·에러매핑) | 정상 입력 200+비어있지 않은 body · 잘못된 입력 4xx(200=gap) |
| `test` | 회귀 기준선 (count·특정 케이스 PASS) | 기준 test 수 유지+추가 케이스 GREEN (개수만으론 부족 — anti-gaming §7) |
| `shell` | 빌드/벤치/스크립트 신호 | build GREEN · 벤치 임계 이하 · 마이그 local 적용 성공 |
| `grep` | dead-path / 미배선 탐지 (구조만) | 호출처 grep **0건=dead**(소비자 없음) — **반드시 실환경 신호로 교차확인**(정적 거짓음성) |

> **grep 은 보조 신호일 뿐.** "0건"은 dead path 의심을 띄울 뿐 그 자체로 PASS/FAIL 이 아니다 —
> 반드시 db/http 실신호로 확정. 반대로 "코드 존재"(grep 매칭)도 배선/동작 증명이 아니다.

### full 모드 = sub-agent fan-out

full 모드는 dimension 별 sub-agent fan-out(컨텍스트 규율 §2). 각 agent 가 자기 dimension 의
**결정론 신호(직접 실행: db/test/shell/grep)** + **판단 신호(에이전트가 행동충족·목표충돌 해석)** 를
모두 보고 **compact 결과만 반환** — `{dimension, 점수, 등급, 핵심증거 3줄, blocked?, stale?}`.
메인은 compact 결과 N개만 합산한다. per-round 는 터치 dimension 1개만(sub-agent 1 또는 경량 inline).

`evidence[]` 의 `file:line` anchor 는 `loop.yaml`/`scorecard.md` 에서 읽어 **그 지점을 재probe** —
이전 점수를 신뢰하지 말고 매 라운드 신호를 새로 관찰한다.

---

## 5. 3단계 — rubric 밴드 적용 (band)

각 dimension(또는 그 scenario)을 관찰된 동작에 따라 등급→점수대로 매긴다:

| 등급 | 점수대 | 정의 |
|------|--------|------|
| **FULL** | 80~90 | 실 전계층(예: BE/FE/DB·API/UI·service/store) LIVE + 게이트/불변식 강제 |
| **PARTIAL** | 40~78 | 일부 계층만 결선 or 단일 계층 의존(우회 가능) |
| **CONFLICT** | 25~55 | 코드는 LIVE 이나 **목표 명세와 정면충돌**(예: spec-out-of-scope 인데 live·잘못된 의미로 배선) |
| **MISSING** | 12~30 | 진입점·producer 부재(dead path 포함 — 소비자/조회는 있는데 write 진입점 없어 늘 빈 결과) |

### 5대 감점 원칙 (이게 anti-gaming — 반드시 적용)

1. **자산 존재 ≠ 운영 배선** — 코드/메서드/테이블이 있어도 호출처가 없으면(dead path) 감점.
   "grep 으로 자산 찾음"은 동작 증명이 아니다.
2. **빌드 GREEN ≠ 정합** — 테스트가 통과해도 목표 축(명세)을 못 채우면 감점. 테스트는 *작성한 것*만
   검증하지 *빠뜨린 불변식*은 검증 못 한다.
3. **단일 계층 의존** — FE/UI 한 군데에만 가드가 있고 BE/API/다른 진입점으로 우회 가능하면 CRITICAL.
   진실의 원천은 가장 낮은(우회 불가) 계층에 있어야 한다.
4. **제약 부재** — 앱레벨 순회 검사만 있고 DB UNIQUE/FK/CHECK 제약이 없으면 race 로 무력화 → 감점.
5. **보존법칙 위반** — 원장(ledger) SSoT 없이 단방향 수량 증가/감소(예: onHand 만 올림, 차감
   source 없음)는 최중대. 들어온 만큼 나간 만큼이 **불변식**으로 강제돼야 한다.

### 가중 blend (단순 평균 금지 — binding constraint 지배)

**dimension 보정점수 = scenario 중요도·심각도 가중 blend.** binding constraint(CRITICAL·최저
deficit 인 최약 고리)가 지배하고, FULL outlier 는 자기 시나리오 점유분만큼만 반영한다 — 실세계
정합은 *가장 약한 고리*가 결정하기 때문. 단순 평균을 쓰면 FULL 한두 개가 CRITICAL 을 가려 거짓
상승한다.

> 예(추상): 한 dimension 에 scenario A=55(CONFLICT)·B=88(FULL)·C=48(CRITICAL 불변식 위반).
> 단순평균=64 지만 C 가 binding 이면 C/A 가중↑·B↓ → 보정점수 ≈ 58. 대조: dominant FULL outlier 가
> 없는 dimension 은 ≈단순평균에 수렴.
>
> 가중은 **판단**이므로 라운드간 드리프트를 막기 위해 **verifier 가 근거를 1줄 기록**한다
> (full re-audit 의 다에이전트가 판단 분산을 평균). per-round 에서 임의로 가중을 바꾸지 말 것 —
> 같은 binding 을 유지하고 그 deficit 의 변화만 반영.

---

## 6. 4단계 — 활성 재정규화 (BLOCKED 제외) ⚠️ 필수

DESIGN §7: blocker(룰/스펙/cert 부재로 **지금 고칠 수 없는** 항목)는 `BLOCKED` 표기 후
**분자·분모 양쪽에서 제외**한다. 안 빼면 `target`(예: 98)이 상한에 막혀 **도달 불가 → 루프가
영원히 안 끝난다**.

```
blocked_weight  = Σ (BLOCKED 항목들의 가중 지분 weight_b)
blocked_contrib = Σ (BLOCKED score_b × weight_b)

active_total = ( Σ_dim score_i × weight_i  −  blocked_contrib ) / ( 1 − blocked_weight )
```

⚠️ **흔한 오류**: `Σ / (1 − blocked_weight)` 만 하면 틀림 — BLOCKED 를 분모뿐 아니라
**분자에서도** 빼야 한다(분자에 BLOCKED score 가 남으면 과대평가).

```
검산 예: 전체 Σ(score×weight)=57.98, BLOCKED 1건(score 50, weight 0.027)
   blocked_contrib = 50 × 0.027 = 1.35
   active_total = (57.98 − 1.35) / (1 − 0.027) = 56.63 / 0.973 ≈ 58  ✓
   (틀린 식 57.98 / 0.973 = 59.6 ✗)
```

`active_total` 이 **STOP 판정 기준**. BLOCKED-pool 은 룰/cert 확정 시 활성 복귀(분자·분모 원복 +
재채점). BLOCKED 는 `safety`/blocker 사유와 함께 surface — 억지로 점수 올리지 않는다(accept-by-design).

---

## 7. 5단계 — anti-gaming 자기점검 (출력 직전 필수)

출력 전 아래를 반드시 자문한다(DESIGN §5 "출력 전 필수"). 하나라도 의심되면 **낮게 매기고
STALE/UNCERTAIN 표기** — 거짓 `target` 점이 진짜 결함보다 비싸다.

- [ ] 테스트 **개수 증가만으로** 점수 올리지 않았나? (커버리지·실probe 동시 확인 — `test` 신호만으론 부족)
- [ ] 정적 grep **"0건"을 PASS 로** 셌나? → db/http 실환경 신호로 재확인했나? (정적 거짓음성)
- [ ] **BLOCKED 를 억지로** 점수 올렸나? → accept-by-design 으로 surface 했나?
- [ ] **live 죽었는데** live-probe(http/auth)를 PASS 로 셌나? → STALE 로 표기했나?
- [ ] 가중 blend 가 **binding constraint 를 가리지** 않았나? (FULL outlier 가 CRITICAL 을 덮어 거짓 상승?)

> 이 엔진의 점수는 **주장이 아니라 측정**이다. 의심스러우면 낮게 + STALE.

---

## 8. 6단계 — 출력 (디스크 영속)

### (A) `scorecard.json` (매 모드 — 다음 라운드/세션 재진입용)

경로 `<project>/.loop/scorecard.json`. 메인·`driver.js` 는 이 JSON 만 읽고 다음 라운드를 시작
(문서 통독 불필요 — 컨텍스트 절약).

```json
{
  "round": "Rx",
  "date": "YYYY-MM-DD",
  "mode": "full",
  "active_total": 58,
  "raw_total": 57,
  "target": 98,
  "dimensions": {
    "<dim-id-1>": 37,
    "<dim-id-2>": 58,
    "<dim-id-3>": 62
  },
  "blocked": ["<dim-id-or-scenario>"],
  "stale": []
}
```

### (A2) `checkpoint.json` (매 모드 — STOP/triage 상태)

경로 `<project>/.loop/checkpoint.json`.

```json
{
  "next": "<triage 다음 dimension-or-rank>",
  "no_progress": 0,
  "last_verdict": "ITERATE",
  "active_total": 58,
  "consecutive_at_target": 0
}
```

`consecutive_at_target` = `active_total ≥ target` 인 **연속 full 라운드** 수. `stop.consecutive`
도달 시 STOP.

### (B) `scorecard.md` §점수이력 append (**full 모드만**, append-only — 기존 행 수정 금지)

```
| Rx | YYYY-MM-DD | <active_total> | <dim1> | <dim2> | <dim3> | … | <비고: 무엇 fix·CRITICAL 잔여·STALE·BLOCKED> |
```

> 헤더(dimension 열 순서)는 `loop.yaml` 의 `dimensions[]` 순서를 따른다. per-round 모드면 (B)
> 대신 "working score" 갱신만 보고하고 §이력 append 는 다음 full 에서.

### (C) 검증 메타 (보고에 포함)

- FALSE_POSITIVE 강등 근거(이전 라운드에서 결함으로 봤으나 실신호로 무효 확인된 것).
- score-gaming 방어 근거(§7 체크 중 걸린 것·왜 낮췄나).
- 다음 STOP 후보 / 가중 blend 판단 1줄.

### (D) STOP 판정 (DESIGN §7 — full 점수로만)

```
active_total ≥ target  이 직전 full 과 연속 consecutive 회   → STOP (done 선언)
동일 deficit 가 2라운드 미해결                                → ESCALATE (인간)
blocker (룰/스펙/cert 부재)                                  → BLOCKED 표기 + 분모 제외 + surface
그 외                                                       → 계속, triage 다음 (checkpoint.next)
```

per-round 모드에서는 (D) 의 STOP/ESCALATE 를 **선언하지 않는다** — working score 만 갱신하고
다음 full 에서 권위 판정. STALE 신호가 남아 있는 동안에도 STOP 금지.

---

## 9. 요약 — 이 엔진이 보장하는 것

- 점수는 **실환경 신호**(db/http/test/shell/grep, local only)에 결박된다 — 정적 자산 존재로
  올리지 않는다(invariant #3).
- dimension 점수는 **최약 고리(binding constraint)** 가 지배한다 — FULL outlier 가 CRITICAL 을 못 가린다.
- BLOCKED 는 **분자·분모 양쪽 제외**로 정규화 — `target` 이 도달 가능해진다.
- STOP 은 **full 점수 ×연속**으로만 — per-round 부분점수로 done 선언 안 한다.
- 의심 시 **낮게 + STALE** — 거짓 target 이 진짜 결함보다 비싸다.
- 모든 상태는 **디스크(JSON + md §이력)** 에 — 오케스트레이터/driver 는 얇게, 컨텍스트 안 폭발.
