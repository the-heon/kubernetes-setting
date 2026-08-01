# Kubernetes Setting

## Prerequisites

1. Create image pull secret (`my-secret`) used by deployments.
2. Create application secrets from the example file.
3. Enable Metrics Server for HPA.
4. Use a CNI that enforces NetworkPolicy.

```bash
kubectl apply -f app-secrets.example.yaml
```

## Core Services

```bash
kubectl apply -f api-server.yaml
kubectl apply -f api-server-service.yaml
kubectl apply -f auth-server.yaml
kubectl apply -f auth-server-service.yaml
kubectl apply -f auth-server-hpa.yaml
kubectl apply -f auth-server-pdb.yaml
kubectl apply -f auth-server-networkpolicy.yaml
kubectl apply -f matchmaking-server.yaml
kubectl apply -f matchmaking-server-service.yaml
kubectl apply -f matchmaking-server-hpa.yaml
kubectl apply -f matchmaking-server-pdb.yaml
kubectl apply -f matchmaking-server-networkpolicy.yaml
kubectl apply -f session-orchestrator.yaml
kubectl apply -f session-orchestrator-service.yaml
kubectl apply -f session-orchestrator-hpa.yaml
kubectl apply -f session-orchestrator-pdb.yaml
kubectl apply -f session-orchestrator-networkpolicy.yaml
kubectl apply -f economy-server.yaml
kubectl apply -f economy-server-service.yaml
kubectl apply -f economy-server-hpa.yaml
kubectl apply -f economy-server-pdb.yaml
kubectl apply -f economy-server-networkpolicy.yaml
kubectl apply -f liveops-config-server.yaml
kubectl apply -f liveops-config-server-service.yaml
kubectl apply -f liveops-config-server-hpa.yaml
kubectl apply -f liveops-config-server-pdb.yaml
kubectl apply -f liveops-config-server-networkpolicy.yaml
kubectl apply -f ranking-stats-server.yaml
kubectl apply -f ranking-stats-server-service.yaml
kubectl apply -f ranking-stats-server-hpa.yaml
kubectl apply -f ranking-stats-server-pdb.yaml
kubectl apply -f ranking-stats-server-networkpolicy.yaml
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

## Monitoring Stack

Prometheus scrapes:

1. `api-server-service:8080/actuator/prometheus`
2. `auth-server-service:8080/metrics`
3. `matchmaking-server-service:8080/metrics`
4. `session-orchestrator-service:8080/metrics`
5. `economy-server-service:8080/metrics`
6. `liveops-config-server-service:8080/metrics`
7. `ranking-stats-server-service:8080/metrics`
8. `api-gateway-service:80/readyz`
9. `realtime-server-service:8081/metrics`

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
신규로 `AUTH_HEALTH_*`, `MATCHMAKING_HEALTH_*`, `SESSION_ORCHESTRATOR_HEALTH_*`를 사용해
운영 요약에 auth/matchmaking/orchestrator 상태를 함께 집계합니다.
추가로 `ECONOMY_HEALTH_*`, `LIVEOPS_HEALTH_*`, `RANKING_HEALTH_*`를 사용해
economy/liveops/ranking 상태도 함께 집계합니다.

## Resilience Controls

1. `api-gateway-hpa.yaml`: gateway CPU 기반 자동 스케일링
2. `auth-server-hpa.yaml`: auth CPU 기반 자동 스케일링
3. `matchmaking-server-hpa.yaml`: matchmaking CPU/메모리 기반 자동 스케일링
4. `session-orchestrator-hpa.yaml`: orchestrator CPU/메모리 기반 자동 스케일링
5. `realtime-server-hpa.yaml`: realtime CPU/메모리 기반 자동 스케일링
6. `auth-server-pdb.yaml`: 노드 드레인 시 auth 최소 1개 Pod 유지
7. `matchmaking-server-pdb.yaml`: 노드 드레인 시 matchmaking 최소 1개 Pod 유지
8. `session-orchestrator-pdb.yaml`: 노드 드레인 시 orchestrator 최소 1개 Pod 유지
9. `realtime-server-pdb.yaml`: 노드 드레인 시 realtime 최소 1개 Pod 유지
10. `auth-server-networkpolicy.yaml`: auth ingress/egress 허용 범위 제한
11. `matchmaking-server-networkpolicy.yaml`: matchmaking ingress/egress 허용 범위 제한
12. `session-orchestrator-networkpolicy.yaml`: orchestrator ingress/egress 허용 범위 제한
13. `realtime-server-networkpolicy.yaml`: realtime ingress/egress 허용 범위 제한
14. `economy-server-hpa.yaml`: economy CPU/메모리 기반 자동 스케일링
15. `liveops-config-server-hpa.yaml`: liveops CPU 기반 자동 스케일링
16. `ranking-stats-server-hpa.yaml`: ranking CPU/메모리 기반 자동 스케일링
17. `economy-server-pdb.yaml`: 노드 드레인 시 economy 최소 1개 Pod 유지
18. `liveops-config-server-pdb.yaml`: 노드 드레인 시 liveops 최소 1개 Pod 유지
19. `ranking-stats-server-pdb.yaml`: 노드 드레인 시 ranking 최소 1개 Pod 유지
20. `economy-server-networkpolicy.yaml`: economy ingress/egress 허용 범위 제한
21. `liveops-config-server-networkpolicy.yaml`: liveops ingress/egress 허용 범위 제한
22. `ranking-stats-server-networkpolicy.yaml`: ranking ingress/egress 허용 범위 제한
