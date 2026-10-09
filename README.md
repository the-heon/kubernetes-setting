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

## Data Services

```bash
kubectl apply -f postgres.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f redis.yaml
kubectl apply -f redis-service.yaml
```

Data layer:

1. Redis: `api-server` (토큰/서명 캐시)
2. PostgreSQL 18 (서버 1대, 데이터베이스 2개)
	- `game_platform`: `api-server` (prod 프로파일)
	- `notification_server`: `notification-server` (전용 계정 `notification_server`, `game_platform` 테이블은 못 읽음)

PostgreSQL 이미지는 메이저 버전(`postgres:18`)으로 고정합니다. 18 부터는 데이터 위치가 `/var/lib/postgresql` 이라
볼륨도 그 위치에 붙입니다(예전 위치 `/var/lib/postgresql/data` 에 붙이면 시작을 거부함).
데이터는 `emptyDir` 이라 Postgres 파드가 다시 뜨면 사라집니다.

## Core Services

```bash
kubectl apply -f api-server.yaml
kubectl apply -f api-server-service.yaml
kubectl apply -f api-gateway.yaml
kubectl apply -f api-gateway-service.yaml
kubectl apply -f api-gateway-hpa.yaml
kubectl apply -f realtime-server-config.yaml
kubectl apply -f realtime-server.yaml
kubectl apply -f realtime-server-service.yaml
kubectl apply -f realtime-server-hpa.yaml
kubectl apply -f realtime-server-pdb.yaml
kubectl apply -f realtime-server-networkpolicy.yaml
kubectl apply -f react-nginx.yaml
kubectl apply -f react-nginx-service.yaml
kubectl apply -f react-nginx-ingress.yaml
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

## Monitoring Stack

Prometheus scrapes:

1. `api-server-service:8080/actuator/prometheus`
2. `api-gateway-service:80/readyz`
3. `realtime-server-service:8081/metrics`

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
3. realtime-server:
	- liveness: `/health`
	- readiness: `/health`

realtime-server는 `X-Api-Key`를 사용해 다음 API를 주기 호출합니다.

1. `POST /api/game-servers/heartbeat`
2. `GET /api/game-servers/traffic-policy`

api-gateway는 운영 요약(`/api/ops/summary`)에 realtime 상태를 합치기 위해
`REALTIME_SERVER_HOST`, `REALTIME_SERVER_HEALTH_PORT`를 사용해 `/health`를 조회합니다.

## Resilience Controls

1. `api-gateway-hpa.yaml`: gateway CPU 기반 자동 스케일링
2. `realtime-server-hpa.yaml`: realtime CPU/메모리 기반 자동 스케일링
3. `realtime-server-pdb.yaml`: 노드 드레인 시 realtime 최소 1개 Pod 유지
4. `realtime-server-networkpolicy.yaml`: realtime ingress/egress 허용 범위 제한
5. `notification-server-networkpolicy.yaml`: notification-server 는 api-gateway 에서 오는 요청만 받음
