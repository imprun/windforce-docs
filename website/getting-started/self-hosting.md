# 로컬 평가 (Docker Compose)

Docker Compose 스택은 windforce를 **로컬에서 몇 분 안에 평가**하기 위한 것이다 — 콘솔 저작 경험(Create app → Deploy → Run)을 직접 만져 보고 기능을 테스트하는 용도다. **운영(production) self-host가 아니다**: 운영 self-host는 Kubernetes를 기준으로 하며 [Kubernetes 배포](../operating/deployment.md)가 다룬다.

> **같은 앱 저작 경험, 다른 운영 모델.** compose로 띄운 콘솔(web UI)은 SaaS·Kubernetes self-host와 **동일한 콘솔**이다 — 차이는 누가 인스턴스를 운영하느냐, 그리고 그 아래 운영 모델(격리·스케일·배치)뿐이다. compose는 그 운영 모델을 대표하지 않는다(아래 표 참조).

## 무엇이 포함되고, 무엇이 아닌가

| 포함 — 로컬 평가 | 미포함 — Kubernetes self-host에서 |
|---|---|
| 콘솔(web UI) · API (`localhost:8080`) | production 격리(gVisor RuntimeClass + bubblewrap) |
| Postgres + 1회성 migration·super-admin seed | KEDA 큐깊이 오토스케일 |
| 단일 worker(`default` 태그) | 멀티노드 워커 그룹 Deployment·롤아웃 |
| 앱 create / deploy / run 스모크 | 노드 배치(taint·nodeSelector·toleration)·배치 스케줄러 |
| 콘솔로 워크스페이스·앱·잡 관리 | Flux/GitOps reconcile · read-only fleet observer |
| — | production object cache(S3)·백업(PITR)·NetworkPolicy·HA |

compose는 `SECRET_KEY` 플레이스홀더, 비보안 세션 쿠키, 셀프서비스 가입(`SIGNUP_MODE`를 지정하지 않아 기본값 open), 단일 worker로 떠 있어 **평가·로컬 개발 전용**이다. 운영 격리·스케일·배치는 [Kubernetes 배포](../operating/deployment.md)에서 켜진다.

## 사전 준비

- **Docker / Docker Compose** — 스택 전체(Postgres·server·worker)를 컨테이너로 띄운다.
- 호스트 5432 포트가 비어 있어야 한다 — 따로 떠 있는 Postgres가 있으면 먼저 내린다.

## 1. 스택 기동

레포 루트에서 한 줄로 전체 스택을 올린다. Postgres, DB 뷰어(Adminer), 1회성 마이그레이션·super-admin seed, `server`(API·콘솔·sync·reaper), `worker`가 함께 뜬다.

```bash
docker compose up -d --build
```

- **콘솔(웹 UI) · API** — `http://localhost:8080`. 콘솔 SPA는 `server` 바이너리에 임베드되어 같은 주소에서 열린다(별도 UI 서버가 없다 — SaaS·Kubernetes self-host와 같은 콘솔).
- **Adminer(DB 뷰어)** — `http://localhost:8081` (server `postgres`, user·password 모두 `windforce`).

## 2. 콘솔 로그인

스택이 뜨면 `seed` 단계가 **super-admin 계정을 자동 생성**한다(멱등). 그 자격증명으로 `http://localhost:8080` 콘솔에 로그인한다 — CLI 없이 바로 web UI로 시작한다.

- 이메일/비밀번호: compose의 `SUPERADMIN_EMAIL` / `SUPERADMIN_PASSWORD`(기본 `root@demo.test` / `correct-password`).

로그인 후 워크스페이스가 없으면 콘솔이 **워크스페이스 생성**으로 안내한다 — 이름과 URL에 쓰이는 워크스페이스 ID를 정하면 그 워크스페이스의 첫 admin이 된다. (compose는 `SIGNUP_MODE`를 지정하지 않아 기본값 `open`으로 동작 → 콘솔에서 새 계정 가입도 열려 있다. 프로덕션은 `invite`로 닫는다.)

워크스페이스를 CLI로 미리 만들고 싶으면(자동화·스크립트) **1회성 부트스트랩** 명령도 있다 — 콘솔 경로의 대안일 뿐 필수는 아니다:

```bash
# (선택) 워크스페이스 + 첫 admin을 CLI로 생성
docker compose exec server /app/windforce bootstrap-workspace demo "Demo" \
  --admin-email dev@demo.test --admin-password correct-password --admin-username dev
```

## 3. 콘솔로 — 빠른 시작 따라가기

워크스페이스에 들어왔으면 [빠른 시작](quickstart.md)으로 넘어가 앱을 만들고(Start coding + 템플릿), Deploy하고, 실행한다. 콘솔이 git 저장소 생성·커밋·배포를 대신 처리하므로, **로컬 평가에서도 SaaS·Kubernetes self-host와 똑같은 저작 경험**이다.

## (참고) API로 직접 실행 · 외부 git 연결

콘솔 없이 CLI/API로만 쓰는 흐름도 있다 — 외부 git 저장소를 source로 등록하고 `curl`로 실행한다.

```bash
# API 호출용 토큰 발급 (기본 scope "*")
docker compose exec server /app/windforce create-token demo dev@demo.test --admin

# 외부 git 저장소를 source로 등록하고 sync (이름 "repo", 브랜치 main; 첫 source id는 1)
docker compose exec server /app/windforce add-gitsource demo repo <git-url> main
docker compose exec server /app/windforce sync demo 1

# 실행 — 공개 주소 /api/w/{workspace}/jobs/run/{app}/{action}
curl -X POST localhost:8080/api/w/demo/jobs/run/greet/hello \
  -H "Authorization: Bearer <token>" -d '{"name":"world"}'
# => {"job_id":"…"}
```

액션 소스 레이아웃(`windforce.json` + entrypoint)과 실행·결과 API의 전체 계약은 [앱·액션 만들기](../guide/apps-and-actions.md)·[잡 실행·결과·로그](../guide/jobs.md)·[스크립트 개발자 계약](https://github.com/imprun/windforce/blob/main/docs/contracts/author-contract.md)에서 다룬다.

## 운영 self-host는 Kubernetes다

로컬 평가를 넘어 **실제 운영**하려면 Docker Compose가 아니라 **Kubernetes**가 기준이다 — [Kubernetes 배포](../operating/deployment.md)가 production self-host 경로다. compose 스택에는 다음이 **없다**: production 격리(gVisor RuntimeClass + bubblewrap), KEDA 오토스케일, 멀티노드 워커 그룹 Deployment·롤아웃, 노드 배치, Flux/GitOps reconcile, read-only fleet observer, production object cache·백업(PITR)·NetworkPolicy·HA. 이 운영 모델은 [Kubernetes 배포](../operating/deployment.md)와 [워커 그룹·스케일](../operating/worker-groups.md)에서 구성한다.

## 더 보기

- [빠른 시작](quickstart.md) — 콘솔에서 첫 액션 만들기·실행 (이 페이지 다음 단계)
- [핵심 개념](concepts.md) — Workspace · App · Action · Job
- [Kubernetes 배포](../operating/deployment.md) — production self-host 경로
- [워커 그룹·스케일](../operating/worker-groups.md) — 운영 모델(워커 그룹·KEDA·배치)
