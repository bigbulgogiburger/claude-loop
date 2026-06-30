# `loop init` — 코드베이스 스캔 + 풀 인터뷰 → `.loop/` 생성

> DESIGN §4 의 절차적 SSoT. `init` 모드는 loop 프레임워크의 **"기획" 엔진**이다 —
> read-only 스캔이 `dimensions`·`probe`·`env` 초안을 *제안*하고, 풀 인터뷰가 그것을 *확정*한다.
> 스캔만 믿으면 dimension 오판, 인터뷰만이면 매번 처음부터 — 둘을 합쳐 자동의 속도 + 사람의 정확도를 얻는다.
>
> 패턴 차용: `jira-ingest` SKILL 의 first-run onboarding (Guard → 추론 → 사용자 확인 → 생성 → baseline chain).
> 거기서 "ISSUE_PREFIX 추론 → schema 작성 → wiki-lint baseline" 이었던 흐름을
> 여기서는 "스택 추론 → loop.yaml 작성 → loop score baseline(R0)" 으로 동형 대응한다.

---

## ⛔ Guard — 언제 init 으로 들어오나 (SKILL.md 가 호출)

`loop` SKILL.md 진입 즉시 분기 (DESIGN §2 Guard):

| 점검 | 결과 |
|------|------|
| `<project>/.loop/loop.yaml` **부재** | → **init 모드** (이 문서) |
| `.loop/loop.yaml` 존재 | → score / run / status 의도 분류 (init 아님) |
| 사용자가 명시적으로 "루프 셋업/시작/init" | → init 모드 (재-init 이면 § 0 참조) |

### § 0. 재-init (이미 `.loop/` 가 있는데 init 요청)

기존 자산을 **덮어쓰지 않는다**. 다음을 사용자에게 제시 후 1회 확인:

```
ℹ️ .loop/ 가 이미 존재합니다 (loop.yaml: <N> dimensions, 마지막 score: R<k>=<active_total>).
   무엇을 하시겠어요?
     1) 재-스캔만 (스택 변동 감지 → 제안만, 기존 yaml 보존)
     2) dimension 추가/수정 (인터뷰 §B 부분 실행 → 해당 항목만 patch)
     3) 전체 재생성 (.loop/loop.yaml.bak 백업 후 처음부터 — 점수이력은 scorecard.md 에 보존)
     4) 취소
```

`scorecard.md` 의 §점수이력은 **절대 파기하지 않는다** (R0~Rn append-only 자산). 전체 재생성 시에도 §이력 블록은 carry-over.

### § 0.5. init 파라미터 (목표점수 등 사전 입력)

init 호출 인자(ARGUMENTS)에 `key=value` 형식이 있으면 해당 `stop.*` 값을 **사전 채택**하고 [B-5] 의 그 질문을 생략한다(나머지 인터뷰 단계는 그대로 진행). 지원 파라미터:

| param | 매핑 | 예 | 미지정 시 |
|-------|------|-----|----------|
| `target=<N>` | `stop.target` (목표 종합점수, 1~100) | `target=95` | [B-5] 에서 질문 — 기본 **90** |
| `consecutive=<N>` | `stop.consecutive` (연속 full-audit 회수) | `consecutive=3` | 기본 **2** |
| `full_audit_every=<N>` | `stop.full_audit_every` (권위 재채점 주기) | `full_audit_every=5` | 기본 **3** |

- **파싱**: ARGUMENTS 에서 `key=value`(공백·콤마 구분 허용, 예 `target=90 consecutive=3`) 추출 → 범위 검증(`target` 1~100, 나머지 ≥1). 범위 밖/형식오류면 무시하고 질문으로 폴백.
- **표기**: 사전 채택값은 [C] 생성 카드에 `param 적용: target=90` 으로 명시하고, [B-5] 는 해당 항목을 `✓ param 으로 설정됨(target=90) — 확인만` 으로 압축(미지정 항목은 정상 질문).
- **범위**: param 은 **목표·종료 조건만** 받는다 — `dimensions`/`weights`/`probe`/`env` 는 코드베이스 고유라 param 화하지 않고 스캔+인터뷰로 확정(invariant #5: 사람이 루프 안에).

---

## [A] 스캔 — 코드베이스 자동 탐지 (read-only, 부작용 0)

목표: 사람이 처음부터 채우지 않도록 `dimensions`·`env`·`probe` **초안**을 제안한다.
**모든 스캔은 read-only** — 빌드/테스트/DB write 실행 금지(스캔 단계에서는 신호 *수집처*만 식별, 실행은 [D] baseline 에서). `safety.scope=local-only` 선언 전이므로 원격 접근도 금지.

### A-1. 언어 / 프레임워크 / 빌드 도구

manifest 파일 존재로 스택을 판정 (우선순위: 루트 → 서브디렉토리):

| 탐지 파일 | 스택 | 추론하는 `env` 초안 |
|-----------|------|---------------------|
| `package.json` | Node / JS·TS (scripts 파싱) | `test`= `scripts.test`, `start`= `scripts.dev\|start` |
| `build.gradle(.kts)` / `gradlew` | JVM (Gradle) | `test`= `./gradlew test`, `start`= `bootRun`(Spring) |
| `pom.xml` | JVM (Maven) | `test`= `mvn test`, `start`= `spring-boot:run` |
| `go.mod` | Go | `test`= `go test ./...`, `start`= `go run .` |
| `requirements.txt`/`pyproject.toml`/`setup.py` | Python | `test`= `pytest`, `start`= uvicorn/manage.py |
| `Cargo.toml` | Rust | `test`= `cargo test`, `start`= `cargo run` |
| `Gemfile` | Ruby | `test`= `rspec\|rake test` |
| `composer.json` | PHP | `test`= `phpunit` |

> **Grep 우선** (Bash `cat`/`find` 금지): `Glob "**/{package.json,build.gradle,go.mod,pom.xml,Cargo.toml,requirements.txt,pyproject.toml}"` 로 매니페스트 위치 수집 → 각 파일 `Read`.
> Monorepo (매니페스트 다수) → 서브프로젝트별 스택을 표로 모아 인터뷰에서 "어느 경계를 dimension 으로 쓸지" 질문.

### A-2. 테스트 / 기동 / DB / 인증 신호 채집

`env.{start,test,db,auth}` 초안의 근거를 코드/설정/스크립트에서 채집:

- **test 커맨드**: 매니페스트 scripts + CI 파일(`.github/workflows/*.yml`, `.gitlab-ci.yml`, `Makefile`)에서 실제 테스트 호출 라인 추출. CI 가 SSoT — 로컬 README 추정보다 신뢰.
  - ⚠️ **함정 grep**: 데몬/플래그 주의 라인이 있는지(`--no-daemon`, `-T`, `--runInBand`, `maxWorkers`). 프로젝트별 테스트 함정(예: Gradle daemon 필수)을 `env.test` 주석에 기록.
- **기동(start) + readiness**: dev 서버 명령 + "떴다" 신호 패턴. 포트만으로 판정 금지 — 로그 토큰(`Started`/`Listening on`/`ready in`) + PID 가 readiness SSoT(stale 프로세스 포트 squat 방지). `env.start` 는 "기동 + readiness 폴링"을 한 줄로.
- **DB 설정**: `application*.yml`/`.env`/`docker-compose*.yml`/`*.properties` 에서 dev DB host·port·schema·driver 추출. **read-only probe 명령**만 초안(`SELECT … LIMIT`/`SHOW`). connection 문자열·자격증명은 yaml 에 **평문 저장 금지** — env var 참조 또는 "사용자가 로컬에 보유" 로 표기.
- **인증(auth)**: 로그인 엔드포인트 + 토큰 발급 방식 탐지(`/login`, JWT, 세션). IDOR/scope probe 를 위해 **비-권한(non-admin) 계정**으로 토큰 얻는 법이 핵심 — admin-only 로 probe 하면 전부 200 이라 누설 미검출. 테스트 계정이 코드/문서에 있으면 후보로, 없으면 인터뷰에서 질문.

### A-3. 모듈 경계 → dimension 후보 자동 제안

dimension(=score function 의 축)을 코드 구조에서 유추:

1. **도메인 디렉토리**: `src/{domain,modules,features}/*`, 패키지 트리(`com.x.{order,payment,...}`), `apps/*`(monorepo) → 각 1차 후보.
2. **라우트/컨트롤러 그룹**: REST 컨트롤러·라우터 파일의 prefix 묶음(`/orders`, `/users`) → 기능 축.
3. **기존 품질 신호**: 커버리지 리포트·E2E 디렉토리·`SECURITY.md`·성능 벤치 → "테스트/보안/성능" 같은 **횡단(cross-cutting) dimension** 후보.
4. **기존 scorecard 가 있으면 import**: `docs/*scorecard*.md`/`dev-plan-*.md` 발견 시 그 도메인·가중치를 그대로 초안으로(예: Stanley 의 7도메인 → loop.yaml dimensions). 이게 "첫 인스턴스 마이그레이션" 경로(DESIGN §9).

> dimension 은 **3~8개** 권장. 너무 잘게 쪼개면(파일 단위) blend 가 무의미, 너무 뭉치면(전체 1개) binding-constraint 진단 불가. 스캔은 후보를 넉넉히 제안하고 인터뷰에서 병합/삭제.

### A-4. probe 후보 수집 (실동작 신호 ≠ 코드 존재)

각 dimension 후보마다 **"코드가 있다"가 아니라 "관찰된 동작이 충족한다"** 를 증명할 probe 초안(DESIGN 스키마 `probe.{kind,cmd,expect}`). 스캔이 제안하는 5종 kind:

| kind | 무엇을 신호로 | 초안 출처 | anti-gaming 가치 |
|------|--------------|-----------|------------------|
| `db` | 불변식·보존법칙 (예: 출고 전후 `SUM(qty)` 보존, UNIQUE 제약 존재) | DDL/마이그레이션 + 도메인 규칙 | 앱레벨 race 우회를 DB 진실로 검증 |
| `http` | 200 + **실데이터** / 비-권한 토큰 **403** / 계약 위반시 **4xx 기대** | 컨트롤러 + auth 신호 | 정적 grep 이 못 보는 런타임 500·IDOR |
| `test` | 특정 스위트 통과(개수 아님, **회귀 기준선** + 커버 지점) | CI 테스트 호출 | "개수만 늘림" gaming 차단(커버 동시 확인) |
| `shell` | 벤치/스크립트의 정량 임계(레이턴시·exit code) | Makefile/bench | 성능 회귀 |
| `grep` | dead-path 탐지 (호출처 0건 = producer 부재) — **보조로만** | 소스 트리 | "자산 존재 ≠ 운영 배선" |

> ⚠️ `grep` 단독으로 PASS 매기지 말 것. "grep 0건"은 거짓음성(이름만 다른 호출처 존재 가능) — 가능하면 db/http 실신호로 교차확인. probe.expect 는 **PASS 의 모습**을 한 줄로(예: `"출고 후 SUM 불변(증가=0)"`, `"tech1 토큰 403"`, `"200 & body[].id 존재"`).

### A-5. 스캔 요약 (인터뷰 입력)

스캔이 끝나면 사용자에게 **초안 카드**를 한 번에 제시(여기서 결정하지 않음 — [B] 로 넘김):

```
🔍 코드베이스 스캔 결과 (read-only, 0 부작용)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
스택        : <Spring Boot 3.x (Java 21, Gradle) + Vue 3 (Vue CLI)> [monorepo 2-subdir]
env 초안    :
  start = <JAVA_HOME=… ./gradlew bootRun --args='--spring.profiles.active=local' → "Started" 폴링>
  test  = <./gradlew test  ⚠ 데몬 ON 필수 (--no-daemon=Mockito 깨짐)>
  db    = <mysql … std_dev (read-only: SELECT/SHOW)>  ⚠ 자격증명은 yaml 평문 금지
  auth  = <POST /login (JWT). 비-admin 토큰 필요 — 계정 미발견, 인터뷰에서 질문>
dimension 후보 (스캔 제안, weight 미정):
  1. material   (src/.../material — 재고/보존)        probe: db SUM 보존
  2. reception  (src/.../reception — 접수)            probe: http native INSERT 영속
  3. product    (src/.../product — 제품마스터)        probe: db UNIQUE 제약
  4. security   (횡단 — IDOR/scope)                   probe: http 비-admin 403
  … (총 <K>개)
기존 scorecard 발견: <docs/dev-plan-tobe-flow-scorecard.md → 7도메인 import 가능 | 없음>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
이제 인터뷰로 이 초안을 확정합니다. (5단계)
```

---

## [B] 인터뷰 — 풀 확정 (스캔 제안 → 사람 보정)

스캔 초안을 사람이 확정/보정한다. **풀 인터뷰**(DESIGN §4 결정 #3) — 5 단계 순서대로, 각 단계는 스캔 제안을 보여주고 수정만 받는다(빈 화면에서 묻지 않음). 사용자가 "스캔대로 OK" 하면 그 단계 즉시 통과.

### B-1. 목표 (goal — 한 줄 성공 기준)

```
1️⃣ 목표 — 루프가 끝나면 "무엇이 참"이어야 하나요? (한 줄)
   예: "PM 6대 시나리오가 실환경에서 정합 — 누설 0, 보존법칙 위반 0, 활성 종합 ≥98×2연속"
   스캔 추정: <기존 scorecard goal | 미정 — 직접 입력>
```

`goal` 은 막연("더 좋게") 금지 — **검증 가능한 종단 상태**여야(invariant #1). 모호하면 1회 되물음.

### B-2. dimensions + weights (Σ = 1.0 검산 ★)

스캔 후보를 제시하고 **병합/삭제/추가 + weight 배정**:

```
2️⃣ dimensions + weights — 점수 축과 비중을 정합니다. Σ weights = 1.0
   (스캔 제안 <K>개. 비중은 "실패 시 가장 아픈 곳"에 높게)

   id          label            weight   probe(요약)
   material    재고/보존        0.22     db SUM 보존
   reception   접수             0.14     http native INSERT
   product     제품마스터       0.16     db UNIQUE
   security    IDOR/scope       0.10     http 비-admin 403
   …                            ─────
                                Σ=1.00   ← 검산
   조정할 축/비중을 알려주세요 (병합·삭제·추가·weight 변경). OK면 'OK'.
```

⚠️ **Σ=1.0 검산은 [B] 종료 게이트** — 합이 1.0(±0.005) 이 아니면 진행 불가. 어긋나면:
- 자동 제안: 비례 정규화(`w_i / Σw`) 결과를 보여주고 수용 여부 확인.
- 사람이 "각 축 점수 못 올림" 으로 막힐 BLOCKED 축이 있으면, 그 축은 **포함하되** `safety`/triage 에서 BLOCKED 로 다룸(분모 재정규화는 scoring.md §4.1a). weight 에서 빼지 않음 — 나중에 활성 복귀.

각 dimension 의 `scenarios`(세부 시나리오 id)·`evidence`(file:line) 는 선택 — 스캔이 채운 게 있으면 확인만, 없으면 비워두고 첫 score 때 채움.

### B-3. 각 축 probe = 실동작 신호 (확정)

dimension 별로 스캔 probe 초안을 **확정**:

```
3️⃣ probe — 각 축의 "점수를 올렸다"를 증명할 실환경 신호입니다.
   (코드 존재가 아니라 관찰된 동작 — anti-gaming 핵심)

   material  →  kind: db
                cmd : SELECT SUM(on_hand_qty) FROM material_stock   (출고 전후 2회)
                expect: 출고 후 SUM 불변 (증가분 0 — 보존법칙)
   security  →  kind: http
                cmd : curl -H "Bearer <tech1>" /api/orders/<남의것>
                expect: 403 (비-권한 토큰)
   …
   probe 가 비현실적이거나(환경 부재) 더 강한 신호가 있으면 알려주세요.
```

probe 가 환경상 불가능한 축(예: 외부 cert 필요)은 **BLOCKED** 로 표기 후 진행(분모 제외 대상). `expect` 가 비어 있으면 채우도록 요구 — expect 없는 probe 는 PASS/FAIL 판정 불가.

### B-4. env 확정 (기동/DB/test/auth)

```
4️⃣ env — 루프가 "세계를 살리는" 법입니다. (스캔 초안 확인)
   start: <…>                     ← 기동+readiness, 맞나요?
   test : <…>  ⚠ 함정: <데몬…>    ← 테스트 커맨드 + 주의
   db   : <… read-only>           ← local only, 자격증명은 env var/로컬 보유
   auth : <비-admin 토큰 획득법>   ← 미발견 시 여기서 입력 (예: tech1 / pw=string)
```

**핵심 게이트**: `auth` 의 비-권한 계정이 확정돼야 IDOR/scope probe 가 의미 있음 — 누락 시 명시적으로 질문. DB 자격증명을 yaml 에 평문으로 받지 않음(safety.md).

### B-5. stop.target + 범위 가드

```
5️⃣ 종료 조건 + 안전 범위   (★ §0.5 param 으로 받은 값은 질문 생략·확인만)
   stop.target      : <param target=N 있으면 그 값 / 없으면 질문 — 기본 90>  ← 이 점수 이상이 done
   stop.consecutive : <param 없으면 기본 2>      ← 연속 full-audit 회수 (국소회귀 방지)
   full_audit_every : <param 없으면 기본 3>      ← 권위 재채점 주기
   safety.scope     : local-only            (고정 — 변경 불가)
   safety.forbid    : [push, flyway*qa, rm -rf, <prod write 패턴…>]
   safety.auto_merge: false                 (고정 — 변경 불가)
   추가로 금지할 명령/경로가 있으면 알려주세요.
```

`safety.scope=local-only` · `auto_merge=false` 는 **불변(DESIGN 결정 #4)** — 사용자가 바꾸려 해도 거부하고 이유 설명. `forbid` 는 프로젝트별 prod-write 패턴을 추가(예: 원격 마이그레이션 명령, 운영 도메인 curl).

---

## [C] 생성 — `.loop/` 4종 작성

인터뷰 확정값으로 템플릿을 채워 디스크에 쓴다. **생성 전 1회 최종 확인**(jira-ingest first-run 의 "진행할까요?" 패턴):

```
📁 다음 자산을 .loop/ 에 생성합니다:
   loop.yaml       (설정 — dimensions <K>개, Σweight=1.0 ✓, stop.target=<98>)
   scorecard.md    (rubric 산문 + §점수이력 헤더)
   driver.js       (무인 워크플로 — 템플릿 + 프로젝트값)
   scorecard.json  (런타임 점수 — baseline 후 채워짐)
   checkpoint.json (런타임 상태 — baseline 후 채워짐)
진행할까요? (yes / 항목 수정 / 취소)
```

승인 시:

1. **`loop.yaml`** ← `templates/loop.yaml.tmpl` 채움. version·name·goal·dimensions(id/label/weight/scenarios/evidence/probe)·env·stop·triage_order·safety. **Σweight=1.0 재검산**(쓰기 직전 마지막 가드). 자격증명 평문 0건 확인.
2. **`scorecard.md`** ← `templates/scorecard.md.tmpl` 채움. 각 dimension 의 FULL/PARTIAL/CONFLICT/MISSING 산문 정의 + `## 점수이력` 빈 표(헤더만). 이게 사람이 읽는 rubric SSoT(yaml 은 기계용).
3. **`driver.js`** ← `templates/driver.js.tmpl` 채움. for r in 1..N 루프, triage→fix→verify→score→영속→STOP/ESCALATE/BLOCKED 체크. 프로젝트값(env 명령·stop 파라미터) 주입. `auto_merge` 호출 경로 **부재** 확인.
   - **STOP 상수**: `{{stop_target}}`/`{{stop_consecutive}}`/`{{full_audit_every}}` 를 `loop.yaml.stop` 값으로 치환(driver 는 런타임상 yaml 미파싱 → 생성시점 고정). ★값 변경 시 driver 손편집 금지 — `loop.yaml.stop` 을 고치고 **재-init** 으로 재생성(손편집 fork 는 SSoT drift·STOP off-by-one 의 근원).
   - **forbid 치환(2종)**: `safety.forbid` 를 ① 산문 `{{forbid_list}}`(maker 프롬프트 주입용) ② **JS 배열** `{{forbid_array}}`(`checkForbid()` 코드 가드용 — 예 `['git push','force','rm -rf','flyway*qa','merge']`) **둘 다**로 치환. 배열이 있어야 driver 가 maker/verifier 의 `risky_cmds` 를 재매칭해 abort 할 수 있다(safety.md §4.1).
4. **`scorecard.json` / `checkpoint.json`**: baseline 전이라 **빈 골격**만(`{"round":null,...}`) — [D] 에서 R0 로 채움.

> 멱등성: 재-init(§0)이 아닌 한 기존 파일 덮어쓰기 금지. 모든 쓰기 전 `Read` 로 존재/내용 확인(부분 patch 모드 대비).

---

## [D] Baseline — R0 측정 (생성 직후 자동 chain)

jira-ingest 가 bootstrap 후 wiki-lint baseline 을 chain 하듯, init 은 생성 후 **첫 score(full)** 를 제안:

```
✅ .loop/ 생성 완료. 지금 baseline 점수를 잴까요?
   → loop score (full) 1회 → §점수이력에 R0 기록 → 다음 STOP 후보 식별
   (live 환경이 떠 있어야 정확 — 죽어 있으면 STALE 표기로 진행)
   진행? (yes / 나중에)
```

승인 시:
1. **사전조건 확인**(scoring.md §1): `env.start` 살아있나(Started+PID), DB 도달, 비-admin 토큰 확보. 죽었으면 해당 probe = **STALE**(거짓상승 방지) 로 표기하고 결정론-정적 파트만 채점.
2. `loop score (full)` 실행 — dimension 별 sub-agent fan-out(병렬), compact 결과만 회수(컨텍스트 규율).
3. **R0 영속**: `scorecard.md` §점수이력에 R0 행 append + `scorecard.json`/`checkpoint.json` 초기화(`next_rank`=weakest-link, `no_progress`=0, `consecutive`=0).
4. **출력**(아래 § 종료 출력).

> baseline 을 "나중에" 하면 `checkpoint.json` 의 `round`=null 로 남고, 첫 `loop score` 호출이 R0 를 채운다.

---

## § 종료 출력 (init 완료 카드)

```
🔧 loop init 완료 — <project>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✓ loop.yaml       : <K> dimensions, Σweight=1.00, target=<98>×<2>연속
✓ scorecard.md    : rubric 산문 + §점수이력
✓ driver.js       : 무인 워크플로 (auto_merge=false ✓)
✓ baseline (R0)   : active_total=<58>  [material 37 / reception 70 / …]   <STALE: live N건 | clean>
✓ safety          : local-only, forbid=<push,flyway*qa,…>
⚠ BLOCKED         : <settlement-6.1 (Q3 룰 미확정) — 분모 제외 | 없음>

다음 단계:
  loop run 1     # attended 1라운드 (다음 deficit triage→fix→verify→rescore, 게이트)
  loop run <N>   # 무인 N라운드 (.loop/driver.js, STOP/ESCALATE/BLOCKED 에서 정지)
  loop status    # 현황·다음 deficit
```

---

## § 안전·범위 재확인 (init 단계 불변)

- **read-only 스캔**: [A] 는 부작용 0. 빌드/테스트/DB write 는 [D] baseline 에서만, 그것도 local·read-only(db) 또는 격리(test).
- **자격증명 비-영속**: DB·토큰 자격증명을 `loop.yaml` 에 평문 저장 금지 — env var 참조 또는 "로컬 보유" 표기.
- **코드 프로젝트 전용**(DESIGN 결정 #5): probe.kind 는 db/http/test/shell/grep 만. 매니페스트·테스트·라우트가 전무한 비-코드 리포면 init 거부하고 사유 설명.
- **사람이 루프 안에**: dimension·weight·probe·target 은 전부 사람이 확정(스캔은 제안만). auto_merge·scope 는 고정 불변.
