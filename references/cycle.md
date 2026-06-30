# `loop run` — 루프 사이클 절차 (the heart of run mode)

> SSoT: 본 파일은 [`../DESIGN.md`](../DESIGN.md) §6(run 분기)·§7(STOP)·§0(5 invariants)·§8(safety) 의 절차적 전개다.
> 보편(universal) 절차만 기술한다 — 프로젝트 고유값(어떤 deficit 부터·어떤 probe·어떤 토큰)은 전부 `<project>/.loop/loop.yaml` + `.loop/scorecard.md` 에서 읽는다.
> Stanley 의 `loop-round` 스킬을 일반화한 것: §6.2 같은 프로젝트특정 Rank 순서는 `loop.yaml: triage_order` 로, scorecard `§9` 는 `.loop/scorecard.md §history` 로, std_dev DB/tech1 토큰은 `loop.yaml: env`/`dimensions[].probe` 로 빠졌다.

루프를 한 라운드(또는 무인 N라운드) 돈다: **score → triage → fix → verify → rescore → STOP판정 → 영속 → 보고**.
교리는 *"Build the loop, stay the engineer"* (Addy Osmani). 그래서 한 라운드도 **maker≠verifier**(자기가 짠 코드를 자기가 검증 금지)·**auto-merge 금지**·**디스크=메모리**(컨텍스트 폭발 방지)를 불변으로 지킨다.

---

## 0. 사이클 한눈에

```
[1 triage]  .loop/scorecard.md §history 마지막 행 + .loop/checkpoint.json
            + loop.yaml: triage_order 만 선택적 Read → 다음 미완 deficit 선택
     ↓  ★ (N=1 only) 인간 게이트: "이번 라운드 = <deficit>, 맞습니까?"
[2 fix]     feat 브랜치 → maker(worktree sub-agent)가 해당 dimension 체크리스트대로 구현
     ↓
[3 verify]  maker≠verifier: 리뷰 fan-out + loop.yaml: dimensions[].probe 실환경 재probe
              verify FAIL → 같은 라운드 1회 재fix → 또 실패면 no_progress++ → 정지
     ↓
[4 rescore] `loop score` (per-round, 터치 dimension 1개만) → active_total 갱신
     ↓
[5 stop]    STOP 판정 (active_total ≥ target ×consecutive / no_progress≥2 / blocker)
     ↓
[6 persist] .loop/scorecard.md §history append(full이면) + scorecard.json + checkpoint.json
     ↓
[7 report]  점수 델타·변경파일·verify 결과·다음 deficit 보고 → ★(N=1)정지 / (N≥2)다음 round
```

각 단계가 어떤 디스크 파일을 읽고/쓰는지가 핵심이다 — **메인 세션은 얇은 드라이버**, heavy work 는 전부 sub-agent 가 자기 컨텍스트에서 수행하고 **compact 결과만** 반환한다(컨텍스트 방화벽).

---

## 1. 컨텍스트 규율 (모든 모드 불변 — 왜 sub-agent 위임인가)

scorecard 산문이 수만 토큰일 수 있고, 라운드를 한 세션에서 연달아 돌리면 컨텍스트가 폭발한다. Addy Osmani: *"메모리는 컨텍스트가 아니라 디스크에."* 그래서:

- **선택적 읽기**: triage 는 `.loop/scorecard.md` **§history 마지막 행 + `.loop/checkpoint.json` + 대상 dimension 의 rubric 블록만** Read(`offset`/`Grep`). 문서 통독 금지.
- **heavy work 전면 sub-agent 위임**: maker(구현)·실환경 probe·리뷰·re-score 는 전부 sub-agent 가 자기 컨텍스트에서 수행하고 **compact 결과만** 반환. 메인은 maker 의 파일내용/추론을 안 본다 = 컨텍스트 방화벽이자 maker≠verifier 의 구조적 보장.
- **상태는 디스크에**: 라운드 끝에 `.loop/scorecard.md §history` append + `scorecard.json` + `checkpoint.json`(다음 deficit·no_progress·last verdict·active_total·consecutive_at_target) 영속. 다음 라운드는 **fresh 세션이어도 디스크만 읽고 재진입** — 컨텍스트 의존 0.
- **멀티라운드는 인라인 금지**: N=1 은 이 attended 절차로 1라운드 후 정지. N≥2 무인 연속은 생성된 `.loop/driver.js`(`Workflow` 드라이버 — 스크립트가 compact 상태만 들고 라운드를 sub-agent 로 반복)로만. 인라인 "계속"은 컨텍스트 폭발.

---

## 2. N=1 (attended) vs N≥2 (무인) 분기

`loop run [N]` 의 단일 다이얼 N 이 게이트 강도를 결정한다(DESIGN §2·§6).

| | **N=1 (attended)** | **N≥2 (무인)** |
|--|--------------------|----------------|
| 주체 | 메인 세션이 본 `cycle.md` 절차를 1회 직접 오케스트레이션 | 생성된 `.loop/driver.js` 를 `Workflow` 로 실행 (driver 가 라운드 반복) |
| 인간 게이트 | **[1]triage 후 ★게이트** ("Rank N 맞나?") + 라운드 끝 정지 | 게이트 없음 — STOP/ESCALATE/BLOCKED/budget 정지조건에서만 멈춤 |
| 정지 | **1라운드 후 무조건 정지** (인간 재트리거 대기) | for r in 1..N, 정지조건 break 까지 |
| full re-audit | 선택(STOP 후보면 1회) | `loop.yaml: stop.full_audit_every` 주기 자동 |
| 용도 | 신중 — 루프 기판 검증·고위험 deficit·처음 도입 | 맡김 — 검증된 기판 위 연속 진전 |

> 첫 바퀴는 항상 보여주고(검증), 나머지는 맡긴다 — "stay the engineer". 그래서 권장 진입은 **항상 `loop run 1` 먼저**(기판/점수함수/verifier 분리가 실제로 도는지 1바퀴로 확인) → 이후 `loop run N`.
> 두 경로 모두 **본 cycle.md 의 [1]~[7] 동일 절차**를 돈다. N=1 은 메인이, N≥2 는 driver.js 가 각 라운드에서 같은 시퀀스를 실행할 뿐이다(driver = 본 절차의 코드화). 그래서 driver 의 phase 라벨(Triage/Fix/Verify/Rescore/Audit)과 본 단계가 1:1 대응한다.

---

## 3. 단계별 상세

### [1] Triage — 다음 deficit 선택

`.loop/scorecard.md §history` 마지막 행 + `.loop/checkpoint.json` + `loop.yaml: triage_order` 를 읽어 **다음 미완(☐) deficit** 을 고른다.

- `triage_order` 가 **명시 배열**이면 그 실행순서대로(raw weight 순위와 다를 수 있음 — 예: 단일 모듈·검증 명확·선행의존 없음인 deficit 을 파일럿으로 앞당김). `loop-round` 의 "Rank 2 가 첫 파일럿" 같은 프로젝트 판단이 여기 yaml 로 박힌다.
- `triage_order: auto` 이면 **weakest-link-first**(blend 종합을 지배하는 최약 dimension 부터, scoring.md 의 binding-constraint 원칙). epic 성 deficit 은 sub-deficit(예 `1a`/`1b`/`1c`)으로만 진입 — **한 라운드 = 한 sub-deficit**.
- 이미 PASS 도달했거나 BLOCKED 표기된 deficit 은 건너뛴다.

**checkpoint.json 에서 읽는 결정값** (driver TRIAGE_SCHEMA 와 동형):

```json
{
  "next": "<deficit-id>",          // 실행순서상 다음 미완
  "domain": "<dimension-id>",      // 그 deficit 의 dimension
  "active_total": 0,               // 현재 활성 재정규화 종합
  "no_progress": 0,                // 동일 deficit 연속 미해결 카운트
  "consecutive_at_target": 0,      // target 연속 full-audit 충족 횟수
  "blocked_only_remaining": false, // 남은 미완이 BLOCKED 뿐인가
  "last_verdict": "PASS"
}
```

**★ 인간 게이트 (N=1 only)**: 고른 deficit 을 사용자에게 확인받고 시작한다("이번 라운드 = `<deficit>`, 맞습니까?"). 사용자가 다른 deficit 을 원하면 따른다. **N≥2 무인은 게이트 없음** — checkpoint.next 를 그대로 진행.

**선(先)-게이트 STOP 체크** (triage 직후, fix 전에):
```
active_total ≥ target 이고 consecutive_at_target ≥ (consecutive-1)  → STOP-at-target  (이번 full 로 확정)
blocked_only_remaining == true                                     → BLOCKED-only-remaining
no_progress ≥ 2                                                     → ESCALATE-no-progress@<deficit>
```
(driver §1 Triage 의 STOP 게이트와 동일. 자세한 종료 규칙은 [`stop.md`](./stop.md).)

### [2] Fix — maker = worktree sub-agent (메인은 직접 안 짬)

maker 는 **worktree sub-agent** 가 수행한다(메인 세션이 직접 코드를 짜지 않는다 — 그래야 [3]의 verify 가 *자기가 안 짠 코드*를 보는 진짜 maker≠verifier 가 되고, 메인 컨텍스트도 보호된다). 검증된 패턴: teammate 가 구현만, 메인이 검증/통합 commit.

- **브랜치 선택 규칙** (run 진입 시 **오케스트레이터(메인)가 1회 결정** → `args.branch` 로 driver 에 고정 전달. driver.js 는 Workflow 런타임상 git 접근 불가 → 메인이 결정해 넘긴다):
  - **현재 브랜치가 이미 루프 브랜치(`loop/*`)면 → 그 브랜치에 계속 적층**(세션 이어가기).
  - **아니면(main 등 비-루프 브랜치) → `loop/<name>_YYYYMMDD` 를 현재 브랜치에서 새로 생성**해 시작. ⚠️ 기존 동명 루프 브랜치(`loop/<name>`)는 이미 인간이 머지해 stale(base 가 main 보다 뒤처짐 — 그 위에 쌓으면 옛 코드로 빌드/probe)일 수 있으므로 **재사용 금지** — 날짜 suffix 로 세션을 격리한다.
  - `loop.yaml` 에 명시 branch 컨벤션이 있으면 그게 우선(예 `feat/<deficit>`, 서브태스크 병렬 시 `-{k}`). 어느 경우든 **main 직접 커밋 금지** · `isolation: worktree`.
- **sub-agent 에 전달**: 대상 deficit + `.loop/scorecard.md` 의 해당 dimension 체크리스트(FULL 정의/잔여 갭) + rubric evidence anchor(file:line). loop.yaml 의 dimension.evidence 가 grounding.
- **sub-agent 반환 (compact만 — driver FIX_SCHEMA)**: `{branch, files[], summary(≤5줄), self_test_pass}`. 파일 전체내용·추론은 반환 금지 — 메인이 들고 있지 않아야 한다.
- **자체 빌드/테스트**: maker 가 `loop.yaml: env.test` 로 자체 검증(데몬/플래그 주의 — env.test 에 명시된 그대로. 임의 플래그 추가 금지).

**NEVER (safety 가드 — 루프가 절대 넘지 않는 선, [`safety.md`](./safety.md) + `loop.yaml: safety.forbid`):**
- **probe·fix 는 local 범위만**: prod/remote write·외부 cutover·자격증명·force push·`push` 금지. `safety.forbid` 목록을 maker 가 시도하면 abort + surface.
- **DB 마이그레이션은 local 적용만**. 원격(QA/prod) 적용은 **인간 수동 게이트** — 루프는 마이그 파일 작성 + local 검증까지만, 원격은 보고 후 정지.
- **auto_merge: false 불변** — maker 는 커밋만, 머지 금지.
- 프로젝트 도메인 NEVER(`loop.yaml: safety.forbid` 의 프로젝트별 항목 — 예 금지 식별자/금지 테이블 재도입)는 그대로 준수.

### [3] Verify — verifier ≠ maker ⚠️ 정적 리뷰만으로 끝내지 말 것

두 층을 모두 수행하되, **maker(worktree sub-agent)와 다른 주체가** 한다. 실환경 재probe 는 **별도 sub-agent(또는 N=1 이면 메인)**가 maker 브랜치를 체크아웃해 돌리고 compact verdict 만 반환 — **maker 자신의 자기검증 금지**. 정적 리뷰만으로 PASS 선언 금지 — 정적 grep "0건"·테스트 개수는 invariant #3(anti-gaming)이 막는 묘지다(정적 0건이어도 실DB 누설·native SQL·UNIQUE 위반은 실probe 로만 잡힌다).

1. **리뷰 fan-out** — 프로젝트 리뷰 메커니즘(예 `/harness-review`, 또는 코드리뷰 sub-agent)으로 maker 와 다른 에이전트가 정적 검토. verdict 수령.
2. **실환경 재probe (★필수, 이게 진짜 결함을 잡는다)** — `loop.yaml: dimensions[].probe` 의 해당 dimension probe 를 실행하고 `expect` 와 대조:
   - `probe.kind` 별 신호: `db`(read-only SUM/COUNT 불변·orphan 0), `http`(200+실데이터 / 비-권한 403/404 — IDOR·scope), `test`(env.test 데몬/플래그 준수), `shell`/`grep`(벤치·계약).
   - **비-권한 토큰**으로 권한/scope probe (`loop.yaml: env.auth` — 최고권한 계정은 전부 통과라 IDOR/scope 미검출).
   - **실DB**(`loop.yaml: env.db`, read-only) — mock/in-memory 는 native SQL·UNIQUE·암묵 INNER JOIN 미검출.
   - **사전조건**: `loop.yaml: env.start` 가 살아있나? 죽었으면 live probe 신호 = STALE(거짓상승 방지, scoring.md §1).

**verify 분기:**
```
PASS  (리뷰 OK AND 실probe expect 충족) → [4] rescore
FAIL  → 같은 라운드에서 1회 재fix([2] 재진입)
        또 실패 → no_progress++ , last_verdict="ITERATE" 를 checkpoint.json 에 write
               → 보고 후 정지 (2라운드 연속이면 다음 triage 가 ESCALATE 로 잡음)
```
verdict 반환 (compact — driver VERDICT_SCHEMA): `{pass, verdict: PASS|ITERATE|ESCALATE, evidence(상태코드/DB SUM/403 등 실신호)}`. **리뷰 verdict 만으로 PASS 금지 — evidence 에 실probe 신호가 없으면 PASS 무효.**

### [4] Rescore — `loop score` per-round

`loop score` 를 **per-round 모드**(터치한 dimension 1개만)로 호출([`scoring.md`](./scoring.md)). 절차:
- 그 dimension 의 probe 신호 → band(FULL/PARTIAL/CONFLICT/MISSING) → 도메인점수.
- BLOCKED 분자·분모 제외 → `active_total` 재정규화(scoring.md §normalize, DESIGN §5.5).
- `.loop/scorecard.json`(런타임 점수) + `.loop/checkpoint.json`(`next`·`no_progress=0`·`last_verdict="PASS"`·`active_total`·`consecutive_at_target`) write.

반환 (compact — driver SCORE_SCHEMA): `{domain, domain_score, active_total, appended}`.

> per-round 는 **빠른 국소 갱신**일 뿐 — STOP 확정에는 못 쓴다([5] 참조). 국소 회귀(다른 dimension 교차오염)는 per-round 가 못 본다.

### [5] STOP 판정 ([`stop.md`](./stop.md) SSoT)

`loop.yaml: stop`(`target`·`consecutive`·`full_audit_every`) 기준:
```
active_total ≥ target 이 직전 full 과 연속 consecutive 회   → STOP (done 선언)
동일 deficit 2라운드 미해결                                → ESCALATE (인간)
blocker(룰/스펙/cert 부재)                                → BLOCKED 표기 + 분모 제외 + surface, 다음 deficit
그 외                                                     → 계속, triage 다음
```
**STOP 은 full 점수로만 인정** — per-round 부분점수로 STOP 선언 금지(국소 회귀 미검출). 그래서:
- **N=1**: STOP 후보(active_total ≥ target)에 도달하면 `loop score` **full 모드** 1회로 확정 → §history append.
- **N≥2 (driver)**: `round % full_audit_every == 0` 마다 full re-audit(전 dimension sub-agent fan-out, 교차오염·회귀 검출) → consecutive_at_target 갱신 → `active_total ≥ target && consecutive_at_target ≥ consecutive` 면 STOP 확정.

### [6] Persist — 디스크가 메모리 (먼저 쓰고 그다음 보고)

**보고 전에 반드시 영속**(컨텍스트 의존 0 으로 다음 라운드 재진입 가능하게):
- `.loop/scorecard.md §history` append — **full 모드일 때만** 공식 행 추가(per-round 는 scorecard.json/checkpoint 만). 각 history 행 = `R번호 | 날짜 | deficit | dimension 점수 before→after | active_total before→after | verdict | 변경 브랜치`.
- `.loop/scorecard.json` — 런타임 dimension 점수표.
- `.loop/checkpoint.json` — `{next, domain, active_total, no_progress, consecutive_at_target, blocked_only_remaining, last_verdict}`. **다음 라운드(fresh 세션 포함)의 유일한 입력.**

> 이 순서(persist → report)가 불변인 이유: driver 가 중단/재개돼도, 메인 세션이 컨텍스트를 잃어도 checkpoint.json 에서 정확히 이어진다. "디스크=메모리".

### [7] Report & 정지

영속 후 결과를 보고한다. **N=1 은 보고 후 멈춘다**(자동으로 다음 라운드 진입 금지 — 인간 재트리거). **N≥2 는 driver 가 정지조건 아니면 다음 round 로**.

```
## Loop Round Rx 결과
- deficit: <id — 작업명> (dimension <id>)
- 변경: <파일 목록 / 브랜치>  (PR 링크 — ★auto-merge 안 함)
- verify: review=<verdict> · 실probe=<핵심 신호: 상태코드/DB SUM/403 등>
- 점수: <dimension> <before>→<after>, 활성 종합 <before>→<after>
- STOP: <미도달, 계속 / STOP / ESCALATE / BLOCKED>
- 다음 라운드 후보: <deficit-id>
- ★ 머지 게이트: 인간 리뷰 후 머지하세요. (원격 마이그/cutover 필요시 별도 수동 게이트 안내)
```

N≥2 driver 최종 반환:
```
{ stopReason, rounds_done, last_active_total, branch, produced[], human_gate }
// human_gate: "auto-merge 안 함 — produced 브랜치를 인간이 리뷰·머지. STOP/ESCALATE/BLOCKED/budget 에서 정지함."
```

---

## 4. 게이트 (매 라운드 불변 — 절대 우회 금지)

- **auto-merge 금지** — 브랜치/PR 만 열고 인간이 리뷰·머지(`loop.yaml: safety.auto_merge: false` 불변). 저위험군(lint/i18n/console-only)도 현재는 전부 게이트.
- **maker ≠ verifier** — fix 를 짠 sub-agent 가 verify 를 못 한다. 실환경 재probe 는 코드 안 짠 별도 주체(별도 sub-agent 또는 N=1 메인).
- **단일 Claude Code 드라이버 + Claude sub-agent OK** — maker/verify 를 Claude teammate(worktree)로 위임 권장. 무인 fan-out 도구로 PowerShell storm/크래시 유발하는 외부 async 위임은 금지(adversarial review 필요시 foreground only — DESIGN §6 결정 #5 "순수 sub-agent").
- **앱 재기동은 clean** — 서버측 코드 수정 후 증분/hot 재기동 금지, clean 재기동(kill + fresh start + readiness 폴링). 안 그러면 stale 프로세스가 옛 코드로 응답해 probe 가 거짓 PASS.
- **1라운드 후 정지 (N=1)** — 인간이 다음 라운드를 재트리거. 무인 연속은 `.loop/driver.js`(Workflow)로만 — 인라인 "계속"은 컨텍스트 폭발.
- **local-only** — 모든 probe/fix 는 local 범위(DESIGN §8). 원격 write·외부 cutover·자격증명 접근 0.

> 루프는 점수를 올리는 기계지만, 검증과 머지 판단은 사람 몫이다. 본 사이클은 한 라운드를 *준비*해 사람 앞에 가져다 놓는다 — 시작 버튼이 아니라 엔지니어로 남으라.

---

## 5. 참조 (cross-ref)

| 무엇 | 어디 |
|------|------|
| 모드 분기(init/score/run/status)·invariants | [`../DESIGN.md`](../DESIGN.md) §0·§2·§6 |
| band·blend·정규화·anti-gaming(점수 어떻게 매기나) | [`./scoring.md`](./scoring.md) |
| STOP/ESCALATE/BLOCKED 종료 로직 상세 | [`./stop.md`](./stop.md) |
| local-only 안전 가드·forbid 처리 | [`./safety.md`](./safety.md) |
| init(스캔+인터뷰→.loop/ 생성) | [`./init-flow.md`](./init-flow.md) |
| 설정 스키마(dimensions·env·probe·triage_order·safety) | [`../templates/loop.yaml.tmpl`](../templates/loop.yaml.tmpl) |
| 무인 N≥2 워크플로 뼈대 | [`../templates/driver.js.tmpl`](../templates/driver.js.tmpl) |
