# Kubernetes Setting

> 운영 전제(k8s 도입 여부 미정, 초기에는 서버 1대)는 [CLUSTER-PLAN.md](CLUSTER-PLAN.md) 를 먼저 봅니다.

## Prerequisites

1. Create image pull secret (`my-secret`) used by deployments.
2. Create application secrets from the example file.
3. Enable Metrics Server for HPA.
4. Use a CNI that enforces NetworkPolicy.

```bash
kubectl apply -f app-secrets.example.yaml
```

## api-server 보안 설정 (2026-10-17)

- **비밀값 32바이트 이상**: `gateway-secret-key`(= `hmac-secret`), `jwt-secret`, `jwt-refresh-secret` 이 짧으면 api-gateway·api-server 가 시작하지 않습니다.
- **첫 관리자**: 운영 DB 에는 개발용 관리자(`admin@naver.com`)를 만들지 않습니다. 첫 배포 때 `app-secrets` 의
  `initial-admin-email`·`initial-admin-password`(12자 이상) 를 넣으면, 관리자가 한 명도 없을 때만 그 계정을 관리자로 만듭니다.
  만든 뒤에는 두 키를 지워도 됩니다. 이 변경 전에 띄운 DB 에 `admin@naver.com` 이 기본 비밀번호 그대로 있으면 api-server 가 ERROR 로그로 알립니다.
- **배포하면 로그인이 한 번 풀립니다**: 토큰을 Redis 에 지문(SHA-256)으로 저장하도록 바뀌어, 예전에 받은 토큰은 통과하지 못합니다.
- **로그인 잠금 기본 켜짐**: 같은 IP 에서 5번 틀리면 그 IP 에서 15분, 모든 IP 를 합쳐 50번이면 계정 전체를 잠급니다 (보안정책 화면에서 조정).
  api-gateway 가 판단한 사용자 IP(`TRUSTED_PROXY_CIDRS`)로 세므로, react-nginx 의 realip 설정과 ingress 의 `externalTrafficPolicy: Local` 이 맞아야 합니다
  (안 맞으면 모든 사용자가 같은 IP 로 보여 한 사람의 실패가 모두를 잠급니다).

## 내부망 제한·DB·Redis 비밀번호 (2026-10-17)

- **NetworkPolicy**: 클러스터 안의 파드 하나가 뚫려도 api-server·Redis·Postgres 에 바로 붙지 못하게 합니다.
  - `api-server-networkpolicy.yaml`: api-gateway·game-server·prometheus 만
  - `redis-networkpolicy.yaml`: api-server 만
  - `postgres-networkpolicy.yaml`: api-server·notification-server(worker)·mobile-api-server, 그리고 `role: postgres-admin` 라벨의 DB 초기화 Job·백업 CronJob
  - **DB 나 Redis 를 쓰는 서버를 새로 더하면 이 정책에도 더해야** 붙을 수 있습니다. NetworkPolicy 를 지키는 CNI 여야 효과가 있습니다.
- **비밀번호를 Secret 으로**: `app-secrets` 의 `postgres-password`(Postgres 관리자 계정, 예전에는 매니페스트에 `postgres` 로 적혀 있었음),
  `redis-password`(예전에는 없음). api-server 는 이제 DB·Redis 주소와 비밀번호를 환경변수로 받습니다 (예전 매니페스트에는 없어서 prod 로 시작하지 못했음).
- **이미 만든 Postgres 의 비밀번호 바꾸기**: `POSTGRES_PASSWORD` 는 비어 있는 데이터 폴더를 처음 만들 때만 쓰입니다. 이미 있는 DB 는 먼저 바꾼 뒤 Secret 을 맞춥니다.
  ```bash
  kubectl exec deploy/postgres -- psql -U postgres -c "ALTER USER postgres PASSWORD '<새 비밀번호>'"
  # app-secrets 의 postgres-password 를 같은 값으로 바꾸고 api-server·postgres-backup 등을 다시 띄움
  ```
- 적용 순서: Secret → `redis.yaml`·`postgres.yaml` → `*-networkpolicy.yaml` → `api-server.yaml`.

## Images (2026-10-15)

각 저장소의 GitHub Actions(`.github/workflows/ci.yml`)가 main/master 에 올라온 코드로 이미지를 만들어 Docker Hub `daev681/daev681` 에 올립니다.

| 저장소 | 태그 |
|---|---|
| api-server | `api-server`, `api-server-<커밋 7자리>` |
| api-gateway | `api-gateway`, `api-gateway-<커밋>` |
| mobile-api-server | `mobile-api-server`, `mobile-api-server-<커밋>` |
| notification-server | `notification-server`, `notification-server-<커밋>` |
| web-front | `react-nginx`, `react-nginx-<커밋>` |
| game-server | CI 없음 (Windows 전용 IOCP 서버라 리눅스에서 빌드하지 않음) |

- 저장소마다 시크릿 `DOCKERHUB_USERNAME`·`DOCKERHUB_TOKEN` 이 있어야 올립니다 (없으면 그 단계만 건너뜀). Docker Hub → Account settings → Personal access tokens 에서 Read & Write 토큰을 만들어 넣습니다.
- 배포: 매니페스트는 서비스 이름 태그에 `imagePullPolicy: Always` 라 `kubectl rollout restart deploy/<이름>` 이면 새 이미지를 받습니다.
- **되돌리기**: `kubectl set image deploy/mobile-api-server mobile-api-server=daev681/daev681:mobile-api-server-<커밋>`
  (어느 커밋이 문제인지는 저장소의 `git log`). 다시 최신으로 돌아갈 때는 서비스 이름 태그로 `set image` 합니다.

## Data Services

```bash
kubectl get storageclass          # (default) 표시가 있는 StorageClass 가 있어야 합니다 (아래 참고)
kubectl apply -f postgres.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f postgres-backup.yaml
kubectl apply -f redis.yaml
kubectl apply -f redis-service.yaml
```

Data layer:

1. Redis: `api-server` (토큰/서명 캐시)
2. PostgreSQL 18 (서버 1대, 데이터베이스 3개)
	- `game_platform`: `api-server` (prod 프로파일)
	- `notification_server`: `notification-server` (전용 계정 `notification_server`, `game_platform` 테이블은 못 읽음)
	- `mobile_api_server`: `mobile-api-server` (전용 계정 `mobile_api_server`, 다른 데이터베이스는 못 읽음)

PostgreSQL 이미지는 메이저 버전(`postgres:18`)으로 고정합니다. 18 부터는 데이터 위치가 `/var/lib/postgresql` 이라
볼륨도 그 위치에 붙입니다(예전 위치 `/var/lib/postgresql/data` 에 붙이면 시작을 거부함).

### Postgres 영구 저장과 백업 (2026-10-15)

- 데이터는 PVC `postgres-data`(10Gi)에 둡니다. 파드가 다시 떠도 남습니다. 클러스터의 **기본 StorageClass** 를 쓰므로,
  `kubectl get storageclass` 에 `(default)` 가 없으면 PVC 가 Pending 이고 Postgres 가 뜨지 않습니다
  (k3s·Docker Desktop 은 기본으로 있고, kubeadm 으로 직접 만든 클러스터는 따로 깔아야 합니다).
- 매일 03:00(서울) `postgres-backup` CronJob 이 `pg_dumpall` 로 전체를 백업해 PVC `postgres-backup` 에 7일치 둡니다.
- 백업은 같은 클러스터 저장소에 있어 디스크가 통째로 망가지면 같이 잃습니다. 가끔 밖으로 복사합니다:

```bash
# 백업 PVC 를 붙인 임시 파드로 꺼내기
kubectl run backup-reader --rm -i --restart=Never --image=busybox \
  --overrides='{"spec":{"volumes":[{"name":"b","persistentVolumeClaim":{"claimName":"postgres-backup"}}],"containers":[{"name":"backup-reader","image":"busybox","command":["sleep","300"],"volumeMounts":[{"name":"b","mountPath":"/backup"}]}]}}' &
kubectl cp backup-reader:/backup ./postgres-backup
# 지금 바로 백업 한 번
kubectl create job --from=cronjob/postgres-backup postgres-backup-manual
```

되살리기 (빈 Postgres 에): `gunzip -c all-YYYYMMDD-HHMM.sql.gz | kubectl exec -i deploy/postgres -- psql -U postgres -d postgres`

10/9 에 한 번 PVC 를 넣었다가 emptyDir 로 되돌린 적이 있습니다 (그때는 k8s 도입이 정해지지 않았음). 기본 StorageClass 가 없는
테스트 클러스터라 Postgres 가 뜨지 않으면, 그 클러스터에서만 `postgres.yaml` 의 volumes 를 `emptyDir: {}` 로 바꿔 씁니다 (데이터는 사라짐).

## Core Services

```bash
kubectl apply -f api-server.yaml
kubectl apply -f api-server-service.yaml
kubectl apply -f api-gateway.yaml
kubectl apply -f api-gateway-service.yaml
kubectl apply -f api-gateway-hpa.yaml
kubectl apply -f game-server-config.yaml
kubectl apply -f game-server.yaml
kubectl apply -f game-server-service.yaml
kubectl apply -f game-server-hpa.yaml
kubectl apply -f game-server-pdb.yaml
kubectl apply -f game-server-networkpolicy.yaml
kubectl apply -f react-nginx.yaml
kubectl apply -f react-nginx-service.yaml
kubectl apply -f react-nginx-ingress.yaml
```

게임 서버는 2026-10-15 에 `realtime-server` 에서 `game-server` 로 이름을 바꿨습니다 (저장소 `the-heon/game-server`).
이미지는 `daev681/daev681:game-server` 태그로 올리고, 실행 파일 이름도 `game-server` 입니다.
예전 이름으로 띄운 클러스터에서는 예전 리소스를 지우고, Secret 의 키 이름도 옮깁니다.

```bash
kubectl delete deploy/realtime-server svc/realtime-server-service configmap/realtime-server-config \
  hpa/realtime-server-hpa pdb/realtime-server-pdb networkpolicy/realtime-server-networkpolicy --ignore-not-found
# app-secrets 의 realtime-api-key 값을 game-server-api-key 로 옮긴 뒤 위 game-server 매니페스트를 적용합니다.
```

## Notification Server

여러 앱이 같이 쓰는 푸시 알림 서버입니다. 앱·백엔드는 api-gateway 의 `/notify/...` 로 부르고,
게이트웨이가 앞의 `/notify` 를 떼고 `notification-server-service` 로 넘깁니다.
바깥 요청은 react-nginx 가 받으므로 `config/nginx.conf` 에서 `/notify/` 도 게이트웨이로 넘깁니다.
nginx 설정을 바꾼 뒤에는 `config/nginx-setting.sh` 로 ConfigMap 을 다시 만들고 react-nginx 를 재시작합니다.

1. `app-secrets` 에 `notification-admin-token`, `notification-credentials-key`, `notification-db-password` 를 실제 값으로 넣습니다.
   예시 파일의 `change-me` 그대로면 알림 서버가 시작하지 않습니다.
2. 데이터베이스·계정을 만들고 서버를 띄웁니다.

```bash
kubectl apply -f postgres.yaml -f postgres-service.yaml
kubectl apply -f notification-server-db-init.yaml
kubectl wait --for=condition=complete job/notification-server-db-init --timeout=180s
kubectl apply -f notification-server.yaml
kubectl apply -f notification-server-service.yaml
kubectl apply -f notification-server-networkpolicy.yaml
```

- `notification-server`(api, 2개)와 `notification-server-worker`(1개)는 같은 이미지(`daev681/daev681:notification-server`)를
  `-mode=api` / `-mode=worker` 로 띄운 것입니다. 테이블은 시작할 때 자동으로 만듭니다.
- 관리자 API(`/admin/...`)는 게이트웨이에서 막혀 있으므로 port-forward 로 부릅니다. 앱을 처음 연결할 때:

```bash
kubectl port-forward svc/notification-server-service 8090:80
ADMIN_TOKEN=...   # app-secrets 의 notification-admin-token
curl -X POST localhost:8090/admin/v1/apps -H "Authorization: Bearer $ADMIN_TOKEN"   -H 'Content-Type: application/json' -d '{"id":"pet-app","name":"반려동물 건강관리","default_locale":"ko"}'
curl -X POST localhost:8090/admin/v1/apps/pet-app/keys -H "Authorization: Bearer $ADMIN_TOKEN"   -H 'Content-Type: application/json' -d '{"kind":"client","label":"pet-app android/ios"}'   # ns_pub_ (앱에 넣음)
curl -X POST localhost:8090/admin/v1/apps/pet-app/keys -H "Authorization: Bearer $ADMIN_TOKEN"   -H 'Content-Type: application/json' -d '{"kind":"server","label":"pet-app backend"}'      # ns_sec_ (서버에만 둠)
curl -X PUT localhost:8090/admin/v1/apps/pet-app/credentials/fcm -H "Authorization: Bearer $ADMIN_TOKEN"   -H 'Content-Type: application/json' --data-binary @firebase-service-account.json
```

자세한 API 는 notification-server 저장소의 README 를 봅니다.

## Mobile API Server

모바일 앱 전용 API 서버입니다 (Kotlin · Spring Boot, 저장소 `the-heon/mobile-api-server`). 웹은 api-server 가 맡습니다.
여러 앱이 같이 쓰고, 앱은 앱 공개 키(`X-Api-Key: ms_pub_...`)로 어느 앱인지 밝힙니다.

- 8070: 앱 API. 앱은 `https://<도메인>/mobile/v1/...` 로 부릅니다: react-nginx(`config/nginx.conf` 의 `/mobile/`) → api-gateway
  (`MOBILE_API_SERVER_URL`, `/mobile/v1/` 만 넘김) → 이 서버의 `/v1/...`. nginx 설정을 바꿨으면 `config/nginx-setting.sh` 로 ConfigMap 을
  다시 만들고 react-nginx 를 재시작합니다.
- 사용자 실제 IP: react-nginx 가 앞단(ingress-nginx)이 넘긴 `X-Forwarded-For` 에서 실제 사용자 IP 를 찾아 그 값 하나만 게이트웨이로 넘기고,
  게이트웨이는 사설 대역(`TRUSTED_PROXY_CIDRS`)에서 온 그 값을 IP별 속도 제한에 씁니다. 실제 클러스터에서 확인할 조건 두 가지는
  [CLUSTER-PLAN.md](CLUSTER-PLAN.md) 에 있습니다.
- 8071: 관리 포트 (`/actuator/health/liveness`·`readiness`, `/actuator/prometheus`, `/actuator/openapi`). Service 에는 Prometheus 수집용으로만 열고, NetworkPolicy 가 prometheus 파드만 들여보냅니다.
- 1개로 띄웁니다 (서버 1대 운영). 상태를 서버에 두지 않으므로 `replicas` 만 올리면 늘어납니다. HPA·PDB 는 두지 않았습니다 (노드 1대에서 PDB 는 노드 점검을 막을 수 있음).

1. `app-secrets` 에 `mobile-api-admin-token`(32자 이상), `mobile-api-db-password` 를 실제 값으로 넣습니다.
   관리자 토큰이 `change-me` 그대로면(32자 미만) 서버가 시작하지 않습니다.
2. 이미지 `daev681/daev681:mobile-api-server` 를 올립니다 (mobile-api-server 저장소의 Dockerfile).
3. 데이터베이스·계정을 만들고 서버를 띄웁니다. 테이블은 서버가 시작할 때 Flyway 가 만듭니다.

```bash
kubectl apply -f postgres.yaml -f postgres-service.yaml
kubectl apply -f mobile-api-server-db-init.yaml
kubectl wait --for=condition=complete job/mobile-api-server-db-init --timeout=180s
kubectl apply -f mobile-api-server.yaml
kubectl apply -f mobile-api-server-service.yaml
kubectl apply -f mobile-api-server-networkpolicy.yaml
```

관리자 API(`/admin/...`)는 port-forward 로 부릅니다. 앱을 처음 연결할 때:

```bash
kubectl port-forward svc/mobile-api-server-service 8095:80
ADMIN_TOKEN=...   # app-secrets 의 mobile-api-admin-token
curl -X POST localhost:8095/admin/v1/apps -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' -d '{"id":"pet-app","name":"Pet Health Care"}'
curl -X POST localhost:8095/admin/v1/apps/pet-app/keys -H "Authorization: Bearer $ADMIN_TOKEN" -H 'Content-Type: application/json' -d '{"label":"pet-app android/ios"}'   # ms_pub_ (앱에 넣음)
```

자세한 API 는 mobile-api-server 저장소의 README 를 봅니다.

## Monitoring Stack

Prometheus scrapes:

1. `api-server-service:8080/actuator/prometheus`
2. `api-gateway-service:80/readyz`
3. `game-server-service:8081/metrics`
4. `mobile-api-server-service:8071/actuator/prometheus`

Deploy:

```bash
kubectl apply -f monitoring/prometheus-config.yaml
kubectl apply -f monitoring/prometheus.yaml
kubectl apply -f monitoring/prometheus-service.yaml
```

## Health Probes

1. api-gateway:
	- liveness: `/healthz`
	- readiness: `/readyz`
2. api-server:
	- liveness: `/actuator/health/liveness`
	- readiness: `/actuator/health/readiness`
3. game-server:
	- liveness: `/health`
	- readiness: `/health`
4. mobile-api-server (관리 포트 8071):
	- startup·liveness: `/actuator/health/liveness` (DB 를 보지 않음)
	- readiness: `/actuator/health/readiness` (DB 에 닿아야 UP)

game-server는 `X-Api-Key`를 사용해 다음 API를 주기 호출합니다.

1. `POST /api/game-servers/heartbeat`
2. `GET /api/game-servers/traffic-policy`

api-gateway는 운영 요약(`/api/ops/summary`)에 게임 서버 상태를 합치기 위해
`GAME_SERVER_HOST`, `GAME_SERVER_HEALTH_PORT`를 사용해 `/health`를 조회합니다.

## Resilience Controls

1. `api-gateway-hpa.yaml`: gateway CPU 기반 자동 스케일링
2. `game-server-hpa.yaml`: game-server CPU/메모리 기반 자동 스케일링
3. `game-server-pdb.yaml`: 노드 드레인 시 game-server 최소 1개 Pod 유지
4. `game-server-networkpolicy.yaml`: game-server ingress/egress 허용 범위 제한
5. `notification-server-networkpolicy.yaml`: notification-server 는 api-gateway 에서 오는 요청만 받음
6. `mobile-api-server-networkpolicy.yaml`: 앱 API 는 api-gateway 에서만, 관리 포트는 prometheus 에서만 받음
7. `api-server-networkpolicy.yaml`: api-gateway·game-server·prometheus 에서만
8. `redis-networkpolicy.yaml`: api-server 에서만
9. `postgres-networkpolicy.yaml`: DB 를 쓰는 서버와 `role: postgres-admin`(DB 초기화 Job·백업)에서만
