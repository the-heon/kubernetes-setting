# Kubernetes Setting

## Prerequisites

1. Create image pull secret (`my-secret`) used by deployments.
2. Create application secrets from the example file.

```bash
kubectl apply -f app-secrets.example.yaml
```

## Core Services

```bash
kubectl apply -f api-server.yaml
kubectl apply -f api-server-service.yaml
kubectl apply -f api-gateway.yaml
kubectl apply -f api-gateway-service.yaml
kubectl apply -f realtime-server-config.yaml
kubectl apply -f realtime-server.yaml
kubectl apply -f realtime-server-service.yaml
kubectl apply -f react-nginx.yaml
kubectl apply -f react-nginx-service.yaml
kubectl apply -f react-nginx-ingress.yaml
```

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
