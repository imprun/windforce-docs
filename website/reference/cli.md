# windforce CLI

`windforce`는 플랫폼 전체를 담은 **단일 바이너리**다. 첫 번째 인자가 서브커맨드이고, 같은 바이너리로 플랫폼을 띄우는 일(`server`·`worker`·`standalone`)과 운영자 부트스트랩 작업(워크스페이스·계정·토큰·git source 생성, sync)을 모두 한다. 이 페이지는 운영자가 쓰는 서브커맨드의 인자·플래그·용도를 정리한다.

운영 모드(`server`/`worker`/`standalone`)와 마이그레이션은 컨테이너 entrypoint나 Deployment가 호출하고, 나머지 부트스트랩 커맨드는 운영자가 손으로 또는 배포 hook에서 실행한다. docker compose에서는 `server` 컨테이너에 CLI가 내장돼 있어 `docker compose exec server /app/windforce <command>`로 호출한다.

## 공통 동작

- **인자 위치**: `windforce <command> <positional...> [--flags]`. 위치 인자가 부족하면 usage를 출력하고 종료 코드 `2`로 빠진다.
- **설정은 환경변수로 온다.** 모든 커맨드가 시작 시 환경을 읽으므로 최소한 `DATABASE_URL`은 있어야 한다(로컬 dev 기본값(sslmode=disable 포함): `postgres://windforce:windforce@localhost:5432/windforce?sslmode=disable`). 작업 디렉터리에 `.env`가 있으면 자동 로드되며, 실제 환경변수가 우선한다.
- **`SECRET_KEY`**: 시크릿 봉인(envelope 암호화)과 잡 토큰 서명에 쓰는 키. 비워 두면 안전하지 않은 dev 기본값이 쓰이고 시작 시 경고가 찍힌다 — 프로덕션에서는 모든 server/worker가 **같은** 랜덤 값을 공유해야 한다.
- **플래그 표기**: 값을 받는 플래그는 `--flag value`와 `--flag=value` 둘 다 받는다.

## 커맨드 한눈에

아래는 **운영자가 손으로 자주 쓰는 부트스트랩 커맨드**다(전체 서브커맨드 목록이 아니다 — 배포·키 운영 커맨드는 표 아래 "배포·시크릿 관리 커맨드" 참고).

| 커맨드 | 용도 |
|---|---|
| `migrate` | DB 스키마 마이그레이션 적용 |
| `server` | control-plane API + reaper·retention·schedule 루프 실행 |
| `worker` | PG 큐를 consume하는 워커 실행 |
| `standalone` | 한 프로세스에서 server + worker 동시 실행 |
| `seed-superadmin` | 첫 설치 super-admin 시드(기본은 배포 hook이 자동 실행) |
| `bootstrap-workspace <id> <name>` | 워크스페이스를 첫 admin과 함께 생성 |
| `create-account <email>` | 전역 password 계정 생성 |
| `add-workspace-member <ws> <email>` | 기존 계정을 워크스페이스 멤버로 추가 |
| `create-token <ws> <email>` | API 토큰 발급(원문 1회 출력) |
| `add-gitsource <ws> <name> <url>` | sync가 쓸 git source 등록 |
| `sync <ws> <git_source_id>` | git source 1회 ingest(clone→manifest→materialize→카탈로그) |

**배포·시크릿 관리 커맨드**(`rekey-secrets`·`rotate-kek`·`render-workers`·`observe` 등)는 배포·키 운영 흐름에서만 쓰며, 여기서는 다루지 않는다 — [Kubernetes 배포](../operating/deployment.md)·[시크릿 관리 (SOPS + age)](../operating/secrets.md) 페이지를 참고한다.

## 플랫폼 실행

### `migrate`

DB 스키마 마이그레이션을 적용한다. server/worker가 뜨기 전에 한 번 실행하는 것이 원칙이며, 여러 pod가 동시에 마이그레이션하지 않도록 단일 실행 Job(Helm pre-install/pre-upgrade hook)으로 돌린다.

```bash
windforce migrate
# => migrations applied
```

### `server`

control-plane을 띄운다 — HTTP API, 콘솔 SPA(프로덕션 이미지에 임베드), git sync, 그리고 백그라운드 루프(reaper·retention·aging·schedule 발화). 기본 리슨 주소는 `:8080`이며 `HTTP_ADDR`로 바꾼다. `SIGINT`/`SIGTERM`을 받으면 graceful shutdown한다.

```bash
windforce server
```

### `worker`

PG 큐를 consume하는 워커를 띄운다. 자기 태그에 맞는 잡을 claim해 격리 실행하고 결과·로그를 기록한다. `SIGTERM`은 drain 신호로, 새 claim을 멈추고 진행 중인 잡을 마저 끝낸다.

```bash
windforce worker
```

### `standalone`

server와 worker를 **한 프로세스**에 함께 띄운다. 로컬 개발이나 단일 노드 데모용이다 — 별도 워커 프로세스 없이 API와 잡 실행이 한 바이너리에서 돈다.

```bash
windforce standalone
```

## 워크스페이스·계정·토큰 부트스트랩

### `bootstrap-workspace`

워크스페이스를 만들고 **그 안의 첫 admin 계정을 함께** 생성한다. 워크스페이스별 봉인 키(wrapped DEK)도 이때 발급된다. 새 테넌트를 처음 세팅할 때의 시작점이다.

```text
windforce bootstrap-workspace <id> <name> \
  --admin-email <email> --admin-password <password> [--admin-username <username>]
```

| 인자/플래그 | 필수 | 설명 |
|---|---|---|
| `<id>` | 예 | 워크스페이스 키(실행 주소 `/api/w/{workspace}/…`의 `{workspace}`) |
| `<name>` | 예 | 표시용 이름 |
| `--admin-email` | 예 | 첫 admin 계정 이메일(정규화됨) |
| `--admin-password` | 예 | 첫 admin 비밀번호(해시 후 저장) |
| `--admin-username` | 아니오 | admin 표시 이름. 생략하면 이메일에서 기본값을 만든다 |

```bash
windforce bootstrap-workspace demo "Demo" \
  --admin-email dev@demo.test --admin-password correct-password --admin-username dev
# => workspace "demo" created with admin "dev@demo.test"
```

### `create-account`

워크스페이스와 무관한 **전역 password 계정**(콘솔 로그인 신원)을 만든다. 이미 만든 계정을 나중에 `add-workspace-member`로 워크스페이스에 붙인다.

```text
windforce create-account <email> --password <password> [--super-admin]
```

| 인자/플래그 | 필수 | 설명 |
|---|---|---|
| `<email>` | 예 | 계정 이메일 |
| `--password` | 예 | 비밀번호(해시 후 저장) |
| `--super-admin` | 아니오 | instance-scoped 운영자(super-admin) 권한 부여 |

```bash
windforce create-account ops@demo.test --password correct-password --super-admin
# => account "ops@demo.test" created
```

### `add-workspace-member`

이미 존재하는 계정을 워크스페이스 멤버로 추가한다. 계정이 없으면 실패하므로 먼저 `create-account`로 만들어 둔다.

```text
windforce add-workspace-member <workspace> <email> [--username <username>] [--admin]
```

| 인자/플래그 | 필수 | 설명 |
|---|---|---|
| `<workspace>` | 예 | 대상 워크스페이스 키 |
| `<email>` | 예 | 추가할 기존 계정 이메일 |
| `--username` | 아니오 | 워크스페이스 내 표시 이름. 생략하면 이메일에서 기본값 |
| `--admin` | 아니오 | admin 역할로 추가(생략 시 member) |

```bash
windforce add-workspace-member demo ops@demo.test --admin
# => account "ops@demo.test" added to workspace "demo" as admin
```

### `create-token`

API 토큰을 발급한다. **원문 토큰은 발급 시 단 한 번 stdout으로 출력**되며 이후 다시 볼 수 없다 — 바로 보관한다. `Authorization: Bearer <token>` 헤더로 API를 호출할 때 쓴다.

```text
windforce create-token <workspace> <email> [--admin] [--super-admin] [--scope <scope>...]
```

| 인자/플래그 | 필수 | 설명 |
|---|---|---|
| `<workspace>` | 예 | 토큰이 속할 워크스페이스 키 |
| `<email>` | 예 | 토큰을 소유할 계정 이메일 |
| `--admin` | 아니오 | 워크스페이스 admin 권한 토큰 |
| `--super-admin` | 아니오 | instance-scoped 운영자 권한 토큰 |
| `--scope <scope>` | 아니오 | 토큰 스코프. **여러 번 반복**해 여러 스코프를 줄 수 있다. 생략하면 기본 스코프 `*`(전체) |

```bash
# 기본 스코프 "*"
windforce create-token demo dev@demo.test --admin
# => wf_… (원문 토큰, 1회만 출력)

# 스코프를 명시적으로 좁힌 토큰
windforce create-token demo ci@demo.test --scope jobs:run --scope jobs:read
```

## git source 등록과 sync

### `add-gitsource`

sync가 사용할 **git source**(repo·branch·subpath)를 워크스페이스에 등록한다. `id`는 등록 순서대로 발급되는 전역 정수다(워크스페이스별이 아니라 테이블 전역 IDENTITY 시퀀스). 갓 초기화한 인스턴스에서는 첫 git source가 `1`을 받는다. 출력된 `id=<n>` 값을 다음 단계 `sync`의 두 번째 인자로 쓴다.

```text
windforce add-gitsource <ws> <name> <url> [branch] [subpath]
```

| 인자 | 필수 | 기본값 | 설명 |
|---|---|---|---|
| `<ws>` | 예 | — | 워크스페이스 키 |
| `<name>` | 예 | — | git source 표시 이름 |
| `<url>` | 예 | — | clone할 repo URL |
| `branch` | 아니오 | `main` | sync할 브랜치 |
| `subpath` | 아니오 | (비움) | repo 안에서 app 루트로 쓸 하위 경로 |

```bash
windforce add-gitsource demo repo https://example.com/acme/actions.git main
# => git_source id=1
```

### `sync`

등록된 git source를 **1회 ingest**한다. repo를 현재 커밋으로 clone하고, `windforce.json`(manifest)과 companion 스키마 파일을 읽어 소스를 오브젝트 캐시에 materialize한 뒤 app·action 카탈로그에 upsert한다. sync는 **manifest와 스키마 파일만 읽고 코드(TS/Python/Go)는 파싱하지 않는다.** repo를 새로 커밋·푸시한 뒤 다시 실행하면 카탈로그가 새 커밋으로 갱신된다.

```text
windforce sync <workspace> <git_source_id>
```

| 인자 | 필수 | 설명 |
|---|---|---|
| `<workspace>` | 예 | 워크스페이스 키 |
| `<git_source_id>` | 예 | `add-gitsource`가 출력한 숫자 id |

```bash
windforce sync demo 1
# => synced commit=<sha> app=<app_key> actions=[<action_key> …]
```

## 부트스트랩 흐름

새 워크스페이스를 처음부터 세팅해 첫 잡을 돌리기까지의 커맨드 순서다.

```mermaid
flowchart TD
  migrate["migrate<br/>(스키마 적용)"]
  ws["bootstrap-workspace demo Demo<br/>(워크스페이스 + 첫 admin)"]
  token["create-token demo dev@demo.test --admin<br/>(API 토큰, 1회 출력)"]
  gs["add-gitsource demo repo <url> main<br/>(git source id=1)"]
  sync["sync demo 1<br/>(clone → manifest → materialize → 카탈로그)"]
  run["POST /api/w/demo/jobs/run/{app}/{action}"]

  migrate --> ws
  ws --> token
  ws --> gs
  gs --> sync
  token --> run
  sync --> run
```

`docker compose`로 띄운 스택에서는 각 줄 앞에 `docker compose exec server /app/windforce`를 붙여 실행한다.

## 더 보기

- [빠른 시작](../getting-started/quickstart.md) — 이 커맨드들을 써서 첫 잡을 돌리는 가장 짧은 경로.
- [Kubernetes 배포](../operating/deployment.md) — `migrate`·`server`·`worker`를 K8s 워크로드로 운영하는 방법.
- [시크릿 관리 (SOPS + age)](../operating/secrets.md) — `SECRET_KEY`를 비롯한 자격증명을 git에 선언적으로 관리.
- [멀티테넌시·운영자 평면](../operating/multitenancy.md) — 워크스페이스·super-admin 권한 모델.
- 엔지니어링 원문: [아키텍처 정본](https://github.com/imprun/windforce/blob/main/docs/foundation/architecture.md) · [스크립트 개발자 계약](https://github.com/imprun/windforce/blob/main/docs/contracts/author-contract.md)
