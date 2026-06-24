# `loop` — Safety Guard (local-only · code-only)

> **이 문서는 `loop` 의 안전 계약(SSoT)이다.** DESIGN §8(결정 #4 local-only / 결정 #5 code-only)과 정수 §0.5(stay the engineer)를 절차로 구체화한다.
> init/score/run/driver 의 모든 모드가 fix·probe 를 실행하기 **직전**에 이 가드를 통과시켜야 한다.
> 위반은 "경고"가 아니라 **abort + surface**(즉시 정지하고 사람에게 보고)다. 자동 우회 없음.

핵심 한 줄: **루프는 자기 `<project>` 워킹트리의 local 범위 안에서만 코드를 만지고, 무엇이든 머지·배포·원격쓰기·자격증명·외부 cutover 직전에 멈춰 사람에게 넘긴다.**

---

## 0. 두 결정 (DESIGN §8 — 절대 불변)

| 결정 | 내용 | 이 문서가 강제하는 것 |
|------|------|----------------------|
| **#4 local-only** | probe·fix 는 **local 범위만**. prod/remote write·외부 cutover·자격증명·force push·원격 마이그레이션 금지. | `safety.forbid` 매칭 → abort. DB probe read-only. 마이그레이션 local 적용만. |
| **#5 code-only** | 코드 프로젝트 전용. `probe.kind` 는 코드 신호(db/http/test/shell/grep)만. | 글쓰기·리서치 루프·비-코드 probe kind 거부. |

그리고 정수 §0.5 에서 파생되는 **auto_merge 불변**:

> **`safety.auto_merge: false` 는 설정값이 아니라 불변식이다.** `loop.yaml` 에서 `true` 로 바꿔도 가드가 무시한다(아래 §4). 루프는 **언제나 브랜치 핸드오프**로 끝난다 — 머지는 사람이 한다.

---

## 1. 가드 적용 시점 (언제 검사하나)

```
score 모드  : probe 실행 전 — 각 dimension.probe.kind/cmd 를 §2·§3 으로 검증
run 1 모드  : fix(maker) 위임 전 + verify(probe) 전 + "정지/핸드오프" 시 §4
driver.js   : 매 라운드 fix·verify·rescore·audit 호출 전 + 종료 return 전 §4
init 모드   : loop.yaml 생성 시 — probe.kind allowlist·forbid 기본값·auto_merge:false 검산
```

원칙: **가드는 실행 직전의 마지막 관문**이다. maker/probe sub-agent 의 프롬프트에 forbid 목록을 그대로 주입하되(자기검열), 가드는 그것을 **신뢰하지 않고** 커맨드 문자열을 다시 매칭한다(이중 방어). sub-agent 가 forbid 를 어기면 그 결과를 PASS 로 세지 않고 abort 한다.

---

## 2. `probe.kind` allowlist (결정 #5 — 코드 신호만)

`loop.yaml` 의 모든 `dimensions[].probe.kind` 는 아래 5종 중 하나여야 한다. 그 외 값은 init 에서 거부, score 에서 STALE 처리.

| kind | 의미 | 안전 제약 |
|------|------|-----------|
| `db` | DB 질의로 실데이터 신호 (SUM·COUNT·제약 검증) | **read-only only** — §3.1. write/DDL 금지. |
| `http` | 기동된 앱에 요청해 상태코드·실데이터·권한(403) 관찰 | **local 호스트만** — §3.2. 외부 API outbound 금지. |
| `test` | 테스트 스위트 실행 결과 | local. 플래그/데몬 주의(프로젝트 env.test). |
| `shell` | 빌드·린트·스크립트 종료코드 | **forbid 매칭 후** 실행 — §3.3. 파괴/원격 커맨드 금지. |
| `grep` | 코드 신호 정적 검색 (보조 신호) | 부작용 없음. ⚠ anti-gaming: grep "0건"을 단독 PASS 로 세지 말 것(DESIGN §0.3). |

> **비-코드 kind 거부**: 사람손 QA, 외부 설문, "문서 읽고 판단" 같은 비결정적·비코드 신호는 probe 가 아니다. `kind` 가 위 5종이 아니면 init 인터뷰에서 코드 신호로 재정의하도록 되돌린다.

---

## 3. local-only 가드 (결정 #4) — kind 별 세칙

### 3.1 `db` — read-only 불변

DB probe 는 **읽기만** 한다. 마이그레이션은 별개(§3.4).

- **허용**: `SELECT`, `SHOW`, `EXPLAIN`, `DESCRIBE`, 그리고 read-only 트랜잭션.
- **금지(매칭 즉시 abort)**: `INSERT` / `UPDATE` / `DELETE` / `REPLACE` / `MERGE` / `TRUNCATE` / `DROP` / `ALTER` / `CREATE` / `GRANT` / `LOAD DATA` 및 모든 DDL·DML write.
- **원격 금지**: 접속 대상 host 가 local/dev 가 아니면(예: prod RDS·remote write 엔드포인트) abort. `env.db` 에 명시된 local probe 호스트만 허용.
- **계정**: 가능하면 read-only 권한 계정 사용을 init 인터뷰에서 권장(쓰기 권한 없는 계정이면 사고 시 물리적으로 차단됨).

```yaml
# 좋음 — read-only 실데이터 신호 (보존법칙·orphan 검출 등)
probe: { kind: db, cmd: "<local mysql client> -e 'SELECT SUM(qty) FROM stock'", expect: "SUM 불변(가차감≠확정차감)" }

# 금지 — write/DDL (abort)
probe: { kind: db, cmd: "... -e 'UPDATE stock SET qty=0'" }    # ✗ DML write
probe: { kind: db, cmd: "... -e 'TRUNCATE audit_log'" }        # ✗ 파괴
```

### 3.2 `http` — local 호스트만

- **허용**: `env.start` 로 띄운 local 앱(예: `http://localhost:<port>`, `127.0.0.1`)에 대한 요청. 권한 probe 는 비-권한 토큰(`env.auth`)으로 403/스코프를 관찰.
- **금지**: 외부 도메인 outbound(prod 도메인·서드파티 API·결제/배송/알림 outbound). 외부 cutover(mock→live 전환)는 범위 밖 — §3.5.
- **부작용 주의**: probe 가 mutation 엔드포인트를 호출해 **local DB 상태를 바꾸는 것**은 허용되지만(local 범위), 멱등하지 않아 다음 probe 를 오염시키면 신호가 거짓이 된다. 가능하면 read 경로로 신호를 잡고, mutation probe 는 정리(cleanup)까지 포함하거나 일회성으로 표시.

```yaml
# 좋음 — local 200 + 실데이터 / 비-권한 403
probe: { kind: http, cmd: "curl -s localhost:5300/... -H 'Authorization: Bearer $TECH1'", expect: "200 + 실데이터 비어있지 않음" }
probe: { kind: http, cmd: "curl -s -o /dev/null -w '%{http_code}' localhost:5300/<other-center> -H '$TECH1'", expect: "403(cross-center IDOR 차단)" }

# 금지 — 외부/prod outbound (abort)
probe: { kind: http, cmd: "curl https://<prod-domain>/..." }   # ✗ 원격
probe: { kind: http, cmd: "curl https://<popbill/holdfast/...>/send" }  # ✗ 외부 cutover
```

### 3.3 `shell` / fix 커맨드 — forbid 매칭 후 실행

`shell` probe 와 maker 의 fix(git·빌드·스크립트)는 **forbid 패턴 매칭을 통과한 뒤에만** 실행한다.

### 3.4 마이그레이션 — local 적용만

- **허용**: 새 스키마 마이그레이션을 **local/dev DB 에만** 적용·검증(예: local 재기동 시 flyway 자동 적용, 또는 local 수동 적용 후 read-only probe 로 검증).
- **금지(abort)**: 원격/QA/prod DB 로의 마이그레이션 적용. 원격 적용은 **인간 게이트** — 루프는 마이그레이션 파일을 만들고 local 검증까지만, 원격 적용은 사람이 SOP 로 한다.
- **함정(프로젝트 메모리 근거)**: maker 가 local DB 에 접속하지 않으면 실데이터 결함(예: UNIQUE 추가 시 기존 중복 1062)을 blind 로 통과시킨다 → UNIQUE/제약 마이그레이션은 dedup DML 선행 + local 재기동 검증을 fix 안에 포함해야 verify 가 잡는다.

### 3.5 외부 cutover — 범위 밖

mock 어댑터 → live 전환(외부 결제/배송/알림/벤더 API 의 운영 cert/key 사용)은 **루프 범위 밖**이다. 별도 인간 주도 단계(예: 릴리즈 cutover). probe 도 fix 도 외부 live 엔드포인트를 건드리지 않는다.

---

## 4. `safety.forbid` 차단 목록 — 매칭 → abort + surface

`loop.yaml` 의 `safety.forbid` 는 **커맨드 문자열에 대한 차단 패턴 목록**이다. fix·probe·driver 가 실행하려는 모든 커맨드를 이 목록과 매칭하고, 하나라도 걸리면:

```
1. 실행하지 않는다 (abort)
2. 점수에 반영하지 않는다 (그 라운드의 fix/probe 결과를 PASS 로 세지 않음)
3. surface: 어떤 커맨드가 어떤 forbid 패턴에 걸렸는지 사람에게 보고하고 정지
```

### 4.1 매칭 규칙

- 패턴은 커맨드 문자열의 **부분 일치**(substring) 또는 **glob**(예: `flyway*qa`)로 해석한다. 대소문자 무시 권장.
- sub-agent 의 자기검열을 **신뢰하지 않는다** — sub-agent 가 "안 했다"고 주장해도 가드가 실제 실행 커맨드를 다시 매칭한다(이중 방어, §1).
- 의심스러우면 차단(fail-safe): 패턴 모호성은 abort 쪽으로 해석.

### 4.2 기본 forbid (init 이 모든 프로젝트에 심는 baseline)

언어/스택 무관하게 항상 들어가야 하는 보편 차단 패턴. init 인터뷰에서 프로젝트별 항목을 **추가**할 수 있으나 아래는 **제거 불가**.

```yaml
safety:
  scope: local-only
  auto_merge: false                 # 불변 (true 로 바꿔도 가드가 무시 — §4.4)
  forbid:
    # ── 머지·원격 push (stay the engineer) ──
    - "push --force"                #  강제 push
    - "push -f"
    - "push --force-with-lease"     #  보수적으로 force 계열 전체 차단(원격쓰기는 인간)
    - "git push"                    #  ★루프는 push 안 함 — 브랜치는 local. 원격 push 는 인간 게이트
    - "merge"                       #  auto-merge 금지(브랜치 핸드오프만)
    - "gh pr merge"
    - "reset --hard origin"         #  원격 기준 파괴적 리셋
    # ── 파괴적 파일/디스크 ──
    - "rm -rf"
    - "rm -fr"
    - ":(){ :|:& };:"               #  fork bomb 류
    - "mkfs"
    - "> /dev/sd"                   #  raw device write
    - "dd if="                      #  디스크 덤프/덮어쓰기
    # ── 원격/prod DB write·마이그레이션 (결정 #4) ──
    - "flyway*migrate*qa"           #  원격 QA flyway 적용 금지(local 만)
    - "flyway*migrate*prod"
    - "flyway*migrate*remote"
    - "DROP "                       #  DDL (대문자, SQL)
    - "TRUNCATE "
    - "DELETE FROM"
    - "UPDATE "                     #  ⚠ DB probe 컨텍스트에서만 — §3.1 read-only 가드와 함께
    - "INSERT INTO"
    # ── 자격증명·secret ──
    - "prod.yml"                    #  운영 설정/키 파일 접근·수정 금지
    - ".env.prod"
    - "AWS_SECRET"                  #  자격증명 echo/주입 금지
    - "PRIVATE_KEY"
    - "credentials"
    # ── 외부 cutover (결정 #4 — §3.5) ──
    - "<external-live-endpoint>"    #  프로젝트별: 결제/배송/알림 vendor live URL
```

> `forbid` 항목 중 일부(예: `UPDATE `, `INSERT INTO`)는 **DB probe 의 read-only 가드(§3.1)와 중복**으로 잡힌다 — 의도된 이중 방어다. shell 커맨드 안에 SQL 이 인라인될 수도 있으므로 문자열 매칭으로도 한 번 더 거른다.

### 4.3 프로젝트별 forbid (init 인터뷰에서 추가)

스캔으로 감지된 위험 표면을 추가한다. 예시(범주):

- **원격 origin push 금지**가 정책인 fork/모노레포: 업스트림 origin URL 패턴.
- **외부 API live 엔드포인트**: 벤더 도메인/호스트.
- **prod 배포 스크립트**: 배포 트리거 커맨드명.
- **인프라 변경**: `terraform apply`, `kubectl ... delete`, `docker ... down -v`(볼륨 삭제) 등.

### 4.4 `auto_merge` 불변 강제

```
loop.yaml 의 safety.auto_merge 값이 무엇이든:
  - 가드는 auto_merge 를 항상 false 로 취급한다.
  - true 로 설정되어 있으면 → init/score 시 경고 surface + false 로 정정 제안.
  - driver.js 의 return 은 produced 브랜치 목록 + "사람이 리뷰·머지" 핸드오프 메시지로 끝난다.
  - merge/gh pr merge 커맨드는 §4.2 forbid 로도 막힌다(이중).
```

---

## 5. 정지·핸드오프 (auto-merge 금지의 실제 형태)

루프가 STOP/ESCALATE/BLOCKED/budget 어느 사유로 끝나든, **종료 행위는 동일**하다:

```
1. fix 결과는 feature/loop 브랜치에만 존재한다(머지 안 됨).
2. driver.return = { stop_reason, rounds, last_active, branch, produced[] }
3. surface 메시지: "produced 브랜치를 사람이 리뷰·머지하세요.
   원격 마이그레이션 필요 시 프로젝트 SOP 로 수동 적용. (auto-merge 안 함)"
```

forbid 위반으로 인한 abort 도 같은 핸드오프 형태로 surface 한다 — 단, 그 라운드는 점수 미반영, stop_reason 에 위반 커맨드·매칭 패턴을 적어 사람이 원인을 즉시 안다.

---

## 6. anti-gaming 과의 교차 (왜 안전이 점수 정직성과 묶이나)

local-only·read-only 가드는 **점수 정직성(§0.3)을 지키는 부수효과**가 있다:

- DB probe 가 read-only 이므로 probe 자신이 상태를 바꿔 "좋아 보이게" 만들 수 없다.
- 외부 outbound 금지이므로 "실제로는 mock 인데 live 인 척" 신호가 섞이지 않는다.
- auto-merge 금지이므로 검증 안 된 fix 가 main 으로 새지 않는다 — STOP 은 **full 점수로만** 인정(§0.4)되고, full 점수는 사람이 머지한 코드가 아니라 **브랜치 위 실probe** 로 잰다.

가드에 걸린 결과를 PASS 로 세지 않는다는 §4 규칙이 곧 anti-gaming 규칙이다: **차단된 fix 로 점수를 올리는 것은 점수 사기다.**

---

## 7. 빠른 체크리스트 (가드 통과 전 자문)

실행 직전, 가드(또는 가드를 호출하는 모드)는 다음을 자문한다:

- [ ] 이 커맨드가 `safety.forbid` 패턴에 걸리나? → 걸리면 abort+surface.
- [ ] DB 면 read-only(SELECT/SHOW/EXPLAIN)인가? write/DDL 이면 abort.
- [ ] HTTP/네트워크면 대상이 local(`env` 명시 호스트)인가? 외부면 abort.
- [ ] 마이그레이션이면 local DB 에만 적용하나? 원격이면 인간 게이트로 surface.
- [ ] probe.kind 가 db/http/test/shell/grep 중 하나인가? 아니면 코드 신호로 재정의.
- [ ] 종료 시 머지하려 하나? → auto_merge:false — 브랜치 핸드오프로만 끝낸다.
- [ ] 외부 cutover(mock→live)를 건드리나? → 범위 밖, 정지.

하나라도 "위반/모호"면 **fail-safe = abort + surface**. 루프는 멈추고 사람이 결정한다. 이것이 "stay the engineer" 의 운영적 형태다.
