# `loop` — Universal Loop Engineering Skill (DESIGN)

> Addy Osmani "Build the loop, stay the engineer" 를 **아무 코드 프로젝트에나** 까는 재사용 프레임워크.
> Stanley CS 의 3개 전용 스킬(completeness-score / loop-round / completeness-loop.js)을 일반화한 것 — Stanley 는 이 프레임워크의 **첫 인스턴스**가 된다.
>
> 결정 반영(2026-06-24): ①SSoT=yaml(설정)+md(rubric) 분리 ②이름=`loop` ③init=코드베이스 스캔+풀 인터뷰 ④probe=local 범위만 ⑤코드 프로젝트만.

---

## 0. 정수 (5 invariants — 모든 모드가 지킨다)

1. **"잘됨"을 숫자로** — 성공을 계산 가능한 0~100 score 로. 막연한 "make it better" 금지.
2. **숫자를 올린다** — score→triage→fix→verify→rescore 사이클.
3. **숫자를 못 속인다 (anti-gaming)** — 모든 점수 상승은 **실환경 신호**(DB SUM/200+실데이터/비-권한 403/벤치)에 결박. 테스트 개수·정적 grep "0건"을 PASS 로 세지 않는다.
4. **결정론적 종료** — score ≥ target ×N연속 → STOP / no-progress → ESCALATE / blocker → SURFACE.
5. **사람이 루프 안에 (stay the engineer)** — 브랜치만, **auto-merge 금지**, 게이트.

---

## 1. 아키텍처 — 보편 엔진 + 프로젝트 설정

```
~/.claude/skills/loop/                 # 보편 (모든 프로젝트 공유)
├── SKILL.md                           # 단일 진입점 (init/score/run/status 모드 분기)
├── references/
│   ├── cycle.md                       # 루프 사이클 절차
│   ├── scoring.md                     # band·가중blend·정규화·anti-gaming
│   ├── stop.md                        # 종료 로직
│   ├── init-flow.md                   # 스캔+인터뷰→생성 절차
│   └── safety.md                      # local-only 가드 (결정 #4)
└── templates/
    ├── loop.yaml.tmpl                 # 설정 스키마
    ├── scorecard.md.tmpl              # 사람용 rubric SSoT
    └── driver.js.tmpl                 # 무인 워크플로 뼈대

<project>/.loop/                       # 프로젝트별 (init 이 생성)
├── loop.yaml                          # 설정: dimensions·env·stop·triage·safety
├── scorecard.md                       # rubric 산문 + §점수이력 (사람 편집 가능)
├── scorecard.json                     # 런타임 점수 (디스크=메모리)
├── checkpoint.json                    # 런타임 상태 (next·no_progress·verdict)
└── driver.js                          # 생성된 무인 워크플로 (템플릿+프로젝트값)
```

**보편 = 엔진·절차. 프로젝트별 = rubric·probe·env.** (jira-ingest 이 `INDEX-SCHEMA.md` 로 정책을 빼낸 것과 동형.)

---

## 2. `loop` 스킬 — 단일 진입점, 의도 추론 (플래그 explosion 없음)

| 모드 | 트리거(자연어) | 동작 | 정지 |
|------|---------------|------|------|
| **init** | `.loop/` 없거나 "루프 셋업/시작" | 코드베이스 스캔 → 풀 인터뷰 → `.loop/` 생성 | 생성 후 baseline score 제안 |
| **score** | "재채점/완성도 몇 점/re-score" | rubric+probe → 점수 (per-round/full) | 측정 후 |
| **run [N]** | "루프 N라운드 돌려" | **N=1**: attended 1바퀴+게이트 / **N≥2**: `driver.js` 무인 | STOP/ESCALATE/BLOCKED/max |
| **status** | "루프 현황/다음 뭐" | scorecard.json+checkpoint 요약 | — |

⛔ **Guard (스킬 시작 즉시)**: `.loop/loop.yaml` 없으면 → **init 모드**. 있으면 → 의도 분류(score/run/status).

> **N 단일 다이얼**: "루프 10" = 10라운드(직관 일치). N=1 게이트(신중), N≥2 무인. attended↔무인 전환 자동.

---

## 3. `loop.yaml` 스키마 (결정 #1 — 기계 설정)

```yaml
version: 1
name: <project>
goal: <한 줄 성공 기준 — "끝나면 뭐가 참인가">

dimensions:                       # score function 의 축들
  - id: <slug>
    label: <사람 라벨>
    weight: 0.22                  # Σ weights = 1.0 (init 이 검산)
    scenarios: [<sub-id>, ...]    # 선택: 세부 시나리오
    evidence:                     # rubric grounding (file:line)
      - "<path:line — 무엇을 증명>"
    probe:                        # ★실환경 신호 (anti-gaming, local only)
      kind: db | http | test | shell | grep
      cmd: "<커맨드>"
      expect: "<PASS 모습 — 예: SUM 불변·200+데이터·403>"

env:                              # 루프의 세계를 살리는 법
  start: "<앱 기동 + readiness 폴링>"
  db:    "<read-only DB probe>"   # local only
  test:  "<테스트 커맨드>"          # 데몬/플래그 주의
  auth:  "<비-권한 토큰 획득법>"     # IDOR/scope probe 용

stop:
  target: 98
  consecutive: 2                  # target 연속 full-audit 횟수
  full_audit_every: 3             # 권위 재채점 주기

triage_order: [<id>, ...]         # 또는 "auto" (weakest-link-first)

safety:                           # 결정 #4 — local only
  scope: local-only
  forbid: ["push", "flyway*qa", "rm -rf", "<prod write>"]
  auto_merge: false
```

`scorecard.md` = 위 rubric 의 **산문 SSoT**(각 dimension 의 FULL/PARTIAL/MISSING 정의 + §점수이력 append). 사람이 읽고 편집. driver/score 는 yaml 을 파싱, 사람은 md 를 읽음.

---

## 4. `init` — 스캔 + 풀 인터뷰 (결정 #3, 여기가 "기획" 엔진)

```
[A 스캔]  코드베이스 자동 탐지 (read-only):
          - 언어/프레임워크 (package.json·build.gradle·go.mod·requirements…)
          - 테스트 커맨드·기동 방법·DB 설정·인증 방식
          - 기존 도메인/모듈 경계 → dimension 후보 자동 제안
          - CI/스크립트에서 probe 후보 수집
   ↓
[B 인터뷰] 스캔 결과를 보여주고 사람이 확정/보정 (풀):
          1. 목표 (성공 기준 한 줄)
          2. dimensions + weights (스캔 제안 → 사람 조정, Σ=1.0)
          3. 각 축 probe = "코드 존재 아닌 실동작" 신호 (스캔 후보 → 확정)
          4. env (기동/DB/test/auth — 스캔값 확인)
          5. stop.target / 범위 가드
   ↓
[C 생성]  .loop/loop.yaml + scorecard.md(rubric 산문) + driver.js(템플릿 채움)
   ↓
[D baseline] "지금 baseline 잴까요?" → loop score (full) 1회 → §점수이력 R0
```

> 스캔이 **초안을 제안**, 인터뷰가 **확정** — 자동의 마법 + 사람의 정확도 둘 다. (스캔만 믿으면 dimension 오판 risk, 인터뷰만이면 매번 처음부터.)

---

## 5. `score` — 측정 엔진 (generic, references/scoring.md)

```
1 사전조건: env.start 살아있나? 죽었으면 live-probe 신호 = STALE(거짓상승 방지)
2 신호수집: dimension 별 probe 실행
            - full: dimension 당 sub-agent fan-out(병렬), compact 결과만 회수
            - per-round: 터치 dimension 1개만
3 band:     관찰된 동작 → FULL(80~90)/PARTIAL(40~78)/CONFLICT(25~55)/MISSING(12~30)
4 blend:    도메인점수 = 가중 blend (단순평균 ❌) — binding constraint(최약 고리) 지배
5 normalize: BLOCKED 항목 분자·분모 제외 → active_total (§4.1a)
6 출력:     scorecard.json + checkpoint.json + (full이면)scorecard.md §이력 append + STOP 판정
```

**anti-gaming 자기점검(출력 전 필수)**: 테스트 개수로만 올렸나 / 정적 grep 0건을 PASS로 셌나 / BLOCKED 억지상승 / live 죽었는데 PASS로 셌나 → 의심되면 낮게+STALE.

---

## 6. `run [N]` — attended / 무인 분기

**N=1 (attended, references/cycle.md):**
```
triage(다음 deficit) →★게이트 "Rank N 맞나?" → fix(maker=worktree sub-agent)
  → verify(maker≠verifier 실probe) → score(per-round) → STOP판정 → 보고 후 정지
```

**N≥2 (무인):** 생성된 `.loop/driver.js` 를 `Workflow` 로 실행. driver = 템플릿:
```
for r in 1..N:
  triage(checkpoint.next) → fix(agent worktree) → verify(다른 agent 실probe)
  → score(per-round) → 디스크 영속 → STOP/ESCALATE/BLOCKED 체크 → 정지조건이면 break
  → r%full_audit_every==0 이면 full re-audit
return {rounds, last_active, branch, stop_reason}  # auto-merge 안 함
```

> 첫 바퀴는 항상 보여주고(검증), 나머지는 맡긴다 — "stay the engineer".

---

## 7. STOP 로직 (references/stop.md)

```
active_total ≥ target 이 직전 full 과 연속 consecutive 회 → STOP (done 선언)
동일 deficit 2라운드 미해결                              → ESCALATE (인간)
blocker (룰/스펙/cert 부재)                              → BLOCKED 표기 + 분모 제외 + surface
그 외                                                    → 계속, triage 다음
```
**STOP 은 full 점수로만 인정** (per-round 부분점수로 STOP 선언 금지 — 국소회귀 미검출).

---

## 8. 안전 (결정 #4 — local only) + 범위 (결정 #5 — 코드만)

- **probe·fix 는 local 범위만**: prod/remote write·외부 cutover·자격증명·force push 금지. `safety.forbid` 가 차단 목록, driver/maker 가 시도 시 abort+surface.
- DB probe 는 **read-only**. 마이그레이션은 local 적용만(원격은 인간 게이트).
- **auto_merge: false** 불변 — 항상 브랜치 핸드오프.
- **코드 프로젝트 전용**: probe.kind 는 코드 신호(db/http/test/shell/grep). 글쓰기·리서치 루프는 범위 밖(추후 별도).

---

## 9. 마이그레이션 — Stanley = 첫 인스턴스

| 지금 (Stanley 전용) | → 보편 `loop` |
|--------------------|--------------|
| `completeness-score` 스킬 | `loop score` 모드 (7도메인 → loop.yaml dimensions) |
| `loop-round` 스킬 | `loop run 1` |
| `completeness-loop.js` | 생성된 `.loop/driver.js` |
| `dev-plan-tobe-flow-scorecard.md` | Stanley `.loop/scorecard.md` + `loop.yaml` |
| `.claude/runtime/completeness/*.json` | `.loop/*.json` |

→ 일반화 후 Stanley 에 `loop init` 한 번 돌려 `.loop/` 생성(기존 scorecard 를 import)하면 전용 스킬 3개를 보편 1개로 대체.

---

## 10. 빌드 계획 (다음 단계)

1. **보편 스킬 골격** — `SKILL.md`(모드 분기) + `references/`(cycle·scoring·stop·init-flow·safety) + `templates/`(loop.yaml·scorecard.md·driver.js)
2. **레퍼런스 인스턴스** — Stanley 로 `loop init` dry-run → 기존 scorecard import 정합 확인
3. **검증** — 2번째 프로젝트(작은 OSS)에 `loop init`→`score`→`run 1` 으로 보편성 입증

> 빌드는 컴포넌트가 비교적 독립적(SKILL/references/templates) → Workflow 로 병렬 초안 + 메인이 contract(본 DESIGN) 기준 정합 조립.
