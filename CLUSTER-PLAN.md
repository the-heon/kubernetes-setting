# 클러스터 운영 계획

## 지금 상태

- **k8s 를 실제로 도입할지는 아직 정하지 않았다.**
- 초창기에 테스트로 **노드 3대** 구성을 띄워 이 매니페스트들로 운영되는 것까지 확인했다.
- 초기 개발·운영은 **서버 1대**로 한다. 노드 3대 구성은 추후에 고려한다.

## 매니페스트를 읽을 때 주의

- `replicas: 2`, HPA, PDB 처럼 여러 대를 전제로 한 설정은 3대 테스트 때 쓰던 것이다.
  지금 그 구성으로 운영 중이라는 뜻이 아니다.
- 아래는 3대 테스트 **이후**에 추가·변경되어, 실제 클러스터에서는 아직 확인하지 않았다 (2026-10-09).
  - notification-server 매니페스트 (`notification-server*.yaml`), api-gateway 의 `/notify/` 설정, nginx 의 `/notify/` 경로
  - Postgres 이미지 `postgres:18` 고정과 마운트 위치 변경 (`/var/lib/postgresql`)
- Postgres(`emptyDir`)와 Redis(볼륨 없음)는 파드가 다시 뜨면 데이터가 사라진다.

## 다시 k8s 를 쓰게 되면 볼 것

- Postgres 영구 저장(PVC·StorageClass)과 백업
- 서버 1대라면: `replicas`·HPA·PDB 를 1대에 맞게 줄일지 (PDB 는 옮겨 갈 노드가 없어 노드 점검(drain)이 멈출 수 있음)
- api-gateway 에 사용자 실제 IP 전달: 지금은 nginx 를 거치면서 모든 요청이 nginx 파드 IP 로 보여,
  IP별 요청 제한이 사이트 전체에 하나로 걸린다 (`TRUSTED_PROXY_CIDRS`, `X-Forwarded-For`).
