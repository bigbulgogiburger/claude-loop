---
name: loop
description: >-
  Universal loop-engineering skill — 임의의 코드 프로젝트에 까는 재사용 완성도 루프 프레임워크
  (Addy Osmani "Build the loop, stay the engineer"). "잘됨"을 계산 가능한 0~100 score 로 정의하고
  score→triage→fix→verify→rescore 사이클을 돌려 숫자를 올리되, 모든 상승은 실환경 신호(DB SUM·200+실데이터·
  비-권한 403·벤치)에 결박한다(anti-gaming). 단일 진입점에서 4모드를 의도 추론으로 분기:
  init(코드베이스 스캔+인터뷰→.loop/ 생성) · score(재채점) · run N(N=1 attended 1바퀴+게이트, N≥2 무인 driver.js) · status(현황).
  **반드시 사용**: '루프 셋업', '루프 시작', 'loop init', '재채점', 'completeness 몇 점', 're-score',
  '루프 N라운드 돌려', 'completeness-loop N바퀴', 'loop engineering N라운드', '다음 deficit', 'STOP 판정',
  '루프 현황', '다음 뭐 해야 돼' — 완성도 루프를 셋업·측정·전진·조회하려는 모든 맥락.
  코드 프로젝트 전용(probe/fix=local 범위만, auto-merge 금지, maker≠verifier). SSoT contract=DESIGN.md.
  단순 단일 버그수정·일반 구현 요청엔 쓰지 말 것(그건 일반 작업 흐름).
---

# `loop` — Universal Loop Engineering (단일 진입점)

> *"Build the loop, stay the engineer."* — Addy Osmani
> 임의의 코드 프로젝트를 *계산 가능한 완성도 점수*로 측정→triage→fix→verify→re-score 하는 루프를
> 한 스킬에 담는다. Stanley CS 전용 3개 스킬(completeness-score / loop-round / completeness-loop.js)을
> 일반화한 보편 엔진이며, 프로젝트별 rubric·probe·env 는 `<project>/.loop/` 에 외화한다.
>
> **이 스킬은 *디스패처*다.** 절차 본문은 `references/*.md` 에, 데이터(rubric/weights/evidence/env)는
> `<project>/.loop/loop.yaml` + `scorecard.md` 에 산다 — 중복 정의 금지. 각 모드는 해당 reference 만
> 펼쳐 읽고 위임한다(통독 금지 — 컨텍스트 규율 참조).

---

## 0. 정수 — 5 invariants (모든 모드가 지킨다)

루프가 의미를 가지려면 이 5개가 깨지면 안 된다. 어떤 모드든 출력 전에 이 5개로 자기점검한다.

1. **"잘됨"을 숫자로** — 성공을 계산 가능한 0~100 score 로. 막연한 "make it better"·"production-ready 인 것 같다" 금지.
2. **숫자를 올린다** — score → triage(최약 고리) → fix → verify → rescore 사이클로만 전진.
3. **숫자를 못 속인다 (anti-gaming)** — 모든 점수 상승은 **실환경 신호**(DB SUM 불변·200+실데이터·비-권한 403·벤치)에 결박. 테스트 *개수* 증가·정적 grep "0건"·"코드가 존재함"을 PASS 로 세지 않는다. (자세히 → `references/scoring.md`)
4. **결정론적 종료** — `active_total ≥ target` 이 직전 full 과 연속 `consecutive` 회 → STOP / no-progress → ESCALATE / blocker → BLOCKED+surface. (자세히 → `references/stop.md`)
5. **사람이 루프 안에 (stay the engineer)** — 브랜치만, **auto-merge 금지**, 게이트. fix·probe 는 **local 범위만**(prod/remote/자격증명/force push 금지). (자세히 → `references/safety.md`)

---

## ⛔ Guard — 스킬 시작 즉시 (가장 먼저 실행)

```
1) <cwd>/.loop/loop.yaml 존재 확인 (Read 또는 Glob)
2) 없으면        → 무조건 init 모드 (다른 의도여도 먼저 셋업해야 측정 가능)
   있으면        → 사용자 발화로 의도 분류 → score / run / status
```

`.loop/` 가 없는데 "재채점/루프 돌려"를 요청받았다면, 측정할 rubric 자체가 없는 것이다 →
"아직 루프가 셋업되지 않았습니다. `loop init` 을 먼저 돌려 `.loop/` 를 만들까요?" 로 안내하고 init 으로 진입.

---

## 1. 모드 분기표 (의도 추론 — 플래그 explosion 없음)

| 모드 | 트리거(자연어) | 무엇을 하나 | 위임 reference | 정지 |
|------|---------------|------------|----------------|------|
| **init** | `.loop/` 없음 · "루프 셋업/시작/init" | 코드베이스 스캔 → 풀 인터뷰 → `.loop/`(loop.yaml·scorecard.md·driver.js) 생성 | `references/init-flow.md` | 생성 후 baseline score 제안 |
| **score** | "재채점/완성도 몇 점/re-score/full re-audit" | rubric+probe 로 측정 → 점수(per-round 또는 full) | `references/scoring.md` | 측정 후 |
| **run [N]** | "루프 N라운드 돌려/N바퀴/loop engineering N" | **N=1**: attended 1바퀴+게이트 / **N≥2**: `.loop/driver.js` 무인 | `references/cycle.md`(+`stop.md`) | STOP/ESCALATE/BLOCKED/max |
| **status** | "루프 현황/다음 뭐/지금 몇 점" | `scorecard.json`+`checkpoint.json` 요약(측정 안 함) | — (디스크만 읽음) | 즉시 |

> **N 단일 다이얼**: "루프 10" = 10 라운드(직관 일치). N 만으로 attended↔무인 자동 전환 —
> **N=1 게이트**(첫 바퀴는 항상 보여주고 검증), **N≥2 무인 driver**(나머지는 맡긴다). 별도 플래그 없음.
> N 미지정 "루프 돌려"는 **N=1 로 해석**(보수적 — 한 바퀴 보여주고 사람이 이어가게).

각 모드는 **그 모드의 reference 만** 펼쳐 읽는다. 4개를 한꺼번에 읽지 않는다(컨텍스트 규율).

---

## 2. 컨텍스트 규율 — "메모리는 컨텍스트가 아니라 디스크에"

이 스킬은 메인 세션에서 돈다. 통독·전수 inline 채점·멀티라운드 인라인은 컨텍스트를 폭발시킨다.
어느 모드든 다음 3규율을 지킨다:

- **선택적 읽기**: `scorecard.md`(산문 rubric, 클 수 있음)·DESIGN·reference 를 통독하지 말 것. 필요한 절(per-round 면 해당 dimension 의 §정의 + `scorecard.md` 마지막 점수이력 행만 / full 이면 §헤더 점수 + weights 만)만 offset/Grep 으로 읽는다. evidence anchor 는 필요할 때 그 `file:line` 만.
- **heavy work 는 sub-agent 위임, compact 만 회수**: dimension probe(curl/DB/grep 묶음)·maker(구현)·verifier(실 probe)·harness-review 는 sub-agent 가 자기 컨텍스트에서 실행하고 **{대상, 점수/verdict, 핵심증거 3~5줄}만** 반환. full 모드 = dimension 별 sub-agent fan-out(병렬), 메인은 compact 결과만 합산. probe 출력 raw·파일 전체내용을 메인이 들고 있지 말 것 = **컨텍스트 방화벽** + 진짜 maker≠verifier.
- **상태는 디스크에**: 결과를 컨텍스트가 아니라 `<project>/.loop/scorecard.json` + `scorecard.md §점수이력` + `checkpoint.json` 으로 영속. 다음 라운드/세션(fresh 여도)은 디스크만 읽고 재진입 — 컨텍스트 의존 0.
- **멀티라운드는 인라인 금지**: `run N≥2` 는 인라인 "계속"이 아니라 **`.loop/driver.js` 를 Workflow 로 실행**. 스크립트가 compact 상태(`scorecard.json`+`checkpoint.json`)만 들고 라운드를 sub-agent 로 반복하므로 메인 컨텍스트가 폭발하지 않는다.

---

## 3. `.loop/` 레이아웃 (init 이 생성, 다른 모드가 읽음)

```
<project>/.loop/
├── loop.yaml        # 기계 설정: dimensions·weights·evidence·probe·env·stop·triage·safety (score/run 이 파싱)
├── scorecard.md     # 산문 SSoT: 각 dimension FULL/PARTIAL/CONFLICT/MISSING 정의 + §점수이력(append-only, 사람이 읽고 편집)
├── scorecard.json   # 런타임 점수 (디스크=메모리: round·active_total·domains·blocked·stale)
├── checkpoint.json  # 런타임 상태 (next·no_progress·last_verdict·consecutive_target)
└── driver.js        # 생성된 무인 멀티라운드 워크플로 (run N≥2 가 Workflow 로 실행)
```

**보편(이 스킬·references·templates) = 엔진·절차. 프로젝트별(`.loop/`) = rubric·probe·env.**
(jira-ingest 이 정책을 `INDEX-SCHEMA.md` 로 외화한 것과 동형.) `loop.yaml` 스키마·`scorecard.md`
구조는 `~/.claude/skills/loop/templates/*.tmpl` + DESIGN §3 참조 — init 이 채워 쓴다.

---

## 4. 모드별 진입 (요지 + 위임)

> 각 항목은 *언제 무엇을 위임하나*만 적는다. 절차 본문은 reference 가 SSoT.

### init — 스캔 + 풀 인터뷰 → `.loop/` 생성
`.loop/` 가 없거나 사용자가 셋업을 원할 때. **여기가 "기획" 엔진** — 스캔이 초안을 제안하고 인터뷰가 확정한다.

```
A 스캔(read-only)  언어/프레임워크(package.json·build.gradle·go.mod·requirements…)·테스트/기동/DB/인증·
                   도메인/모듈 경계(→ dimension 후보)·CI 스크립트(→ probe 후보) 자동 탐지
B 인터뷰(풀)        스캔 결과를 보여주고 사람이 확정/보정: ①목표 한 줄 ②dimensions+weights(Σ=1.0 검산)
                   ③각 축 probe="코드 존재 아닌 실동작" 신호 ④env(기동/DB/test/auth) ⑤stop.target·범위 가드
C 생성             .loop/loop.yaml + scorecard.md(rubric 산문) + driver.js(templates/*.tmpl 채움)
D baseline         "지금 baseline 잴까요?" → score(full) 1회 → scorecard.md §점수이력 R0
```

**위임**: 전체 절차·스캔 휴리스틱·인터뷰 질문 세트·Σweights=1.0 검산·템플릿 채우기는
`references/init-flow.md` 가 SSoT. 스캔만 믿으면 dimension 오판, 인터뷰만이면 매번 처음부터 —
**둘 다** 한다.

### score — 측정 엔진 (재채점)
"재채점/몇 점/re-score". 먼저 범위를 정한다:

| 모드 | 시점 | 범위 | 비용 |
|------|------|------|------|
| **per-round**(기본) | fix 라운드 직후 | 터치한 deficit + 그 dimension 1개만 | 저 |
| **full** | `full_audit_every` 라운드마다 + STOP 후보 도달 시 | 전 dimension | 고 |

"full/전체/re-audit"이면 full(dimension fan-out), 도메인 지정이면 per-round.
**STOP 판정은 full 점수로만 인정**(per-round 부분점수로 STOP 선언 금지 — 국소회귀·교차오염 미검출).

핵심 6단계(사전조건 STALE 가드 → 신호수집 → band → 가중 blend → BLOCKED 재정규화 → 출력)와
band 표(FULL 80~90 / PARTIAL 40~78 / CONFLICT 25~55 / MISSING 12~30)·5대 감점 원칙·
active 재정규화 공식·anti-gaming 자기점검은 **`references/scoring.md` 가 SSoT**.
출력은 `scorecard.json`+`checkpoint.json`(매 모드) + (full 이면)`scorecard.md §점수이력` append +
STOP 판정. → 위임.

### run [N] — attended / 무인 분기
**N=1 (attended)**: 한 바퀴를 사람 앞에서 돈다.
```
triage(다음 deficit) →★게이트 "Rank N 맞나?" → fix(maker=worktree sub-agent)
  → verify(verifier≠maker: harness-review + 실 probe) → score(per-round) → STOP 판정 → 보고 후 정지
```
**N≥2 (무인)**: 인라인 반복 금지 → `.loop/driver.js` 를 `Workflow` 로 실행. driver 가 라운드를
sub-agent 로 반복하며 `scorecard.json`/`checkpoint.json` 만 들고 돈다. STOP/ESCALATE/BLOCKED/max
에서 멈추고 `{rounds, last_active, branch, stop_reason}` 반환 — **auto-merge 안 함**.

사이클 6단계(triage→fix→verify→rescore→stop→persist&report)·maker 위임 규약·verifier 규약·
human 게이트·재진입은 **`references/cycle.md`**, STOP 로직은 **`references/stop.md`** 가 SSoT. → 위임.
첫 바퀴는 항상 보여주고(검증), 나머지는 맡긴다.

### status — 현황 (측정 안 함)
`scorecard.json` + `checkpoint.json` 만 Read 해서 요약: 현재 active_total·dimension별 점수·
다음 Rank(triage)·no_progress·last_verdict·BLOCKED/STALE 목록. probe·fix 안 함, 디스크만 읽음.

---

## 5. 불변식 — 어떤 모드도 넘지 않는 선

루프는 점수를 올리는 기계지만, 검증·머지 판단은 사람 몫이다. 다음은 reference 가 무엇을 시키든
**항상** 강제된다(`references/safety.md` SSoT, `loop.yaml.safety.forbid` 가 차단 목록):

- **local-only** — probe·fix·마이그레이션은 local 범위만. prod/remote write·외부 API cutover·자격증명/AES키·force push 금지. DB probe 는 **read-only**. 원격 마이그/배포는 인간 게이트.
- **auto_merge: false** — 항상 브랜치/PR 핸드오프. 루프가 직접 머지하지 않는다.
- **maker ≠ verifier** — fix 한 주체(worktree sub-agent)와 검증한 주체가 달라야 한다. 자기검증·harness-review(정적)만으로 PASS 선언 금지 — 실환경 재 probe(비-권한 토큰·실DB·캐시 더블콜)가 진짜 결함을 잡는다.
- **anti-gaming** — 테스트 개수·정적 grep 0건·BLOCKED 억지상승·죽은 live 를 PASS 로 세지 않는다. 의심되면 낮게 + STALE/UNCERTAIN.
- **코드 프로젝트 전용** — probe.kind 는 코드 신호(db/http/test/shell/grep). 글쓰기·리서치 루프는 범위 밖.
- **결정론적 종료** — STOP 은 full 점수로만, ESCALATE 는 동일 deficit 2라운드 미해결, BLOCKER 는 표기+분모 제외+surface.

> 이 스킬은 한 라운드(또는 셋업/측정/현황)를 *준비*해서 당신 앞에 가져다 놓는다 —
> 시작 버튼이 아니라 엔지니어로 남으라.
