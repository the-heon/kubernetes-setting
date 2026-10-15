# 클러스터 운영 계획

## 지금 상태

- **k8s 를 실제로 도입할지는 아직 정하지 않았다.**
- 초창기에 테스트로 **노드 3대** 구성을 띄워 이 매니페스트들로 운영되는 것까지 확인했다.
- 초기 개발·운영은 **서버 1대**로 한다. 노드 3대 구성은 추후에 고려한다.

## 매니페스트를 읽을 때 주의

- `replicas: 2`, HPA, PDB 처럼 여러 대를 전제로 한 설정은 3대 테스트 때 쓰던 것이다.
  지금 그 구성으로 운영 중이라는 뜻이 아니다.
- 아래는 3대 테스트 **이후**에 추가·변경되어, 실제 클러스터에서는 아직 확인하지 않았다 (2026-10-09, 2026-10-15).
  - notification-server 매니페스트 (`notification-server*.yaml`), api-gateway 의 `/notify/` 설정, nginx 의 `/notify/` 경로
  - Postgres 이미지 `postgres:18` 고정과 마운트 위치 변경 (`/var/lib/postgresql`)
  - realtime-server → game-server 이름 변경 (`game-server*.yaml`, 이미지 태그 `game-server`, Secret 키 `game-server-api-key`)
  - mobile-api-server 매니페스트 (`mobile-api-server*.yaml`), api-gateway 의 `/mobile/` 설정, nginx 의 `/mobile/` 경로
  - 사용자 실제 IP 전달 (아래 "사용자 실제 IP" 참고)
  - Postgres 영구 저장(PVC `postgres-data`, 기본 StorageClass 필요)과 매일 백업(`postgres-backup.yaml`) — README "Postgres 영구 저장과 백업"
  - 종료 전 5초 대기(`preStop: sleep`, k8s 1.30 이상): api-gateway·api-server·mobile-api-server·notification-server·react-nginx
  - api-gateway 를 root 가 아닌 distroless 이미지로 (`runAsNonRoot`, 읽기 전용 루트)
- Redis(볼륨 없음)는 파드가 다시 뜨면 데이터가 사라진다 (토큰·서명 캐시라 다시 로그인하면 된다).

## 사용자 실제 IP (2026-10-15 설정)

요청은 사용자 → ingress-nginx → react-nginx → api-gateway 순서로 온다. 연결마다 상대가 바뀌므로 그대로 두면 게이트웨이에는
모든 요청이 react-nginx 파드 IP 하나로 보이고, IP별 속도 제한이 사이트 전체에 하나로 걸린다. 그래서:

- react-nginx (`config/nginx.conf`): 사설 대역에서 온 연결의 `X-Forwarded-For` 를 오른쪽부터 보며 사설 주소를 건너뛰고 처음 만나는 주소를
  사용자로 본다 (`real_ip_recursive`). 게이트웨이에는 그 값 **하나만** `X-Forwarded-For` 로 넘긴다 (사용자가 보낸 값은 버림).
- api-gateway (`api-gateway.yaml`): `TRUSTED_PROXY_CIDRS` 를 사설 대역 전체로 둔다. 게이트웨이 Service 는 ClusterIP 라 클러스터 안에서만 들어온다.
- nginx 동작은 컨테이너로 확인했다 (꾸민 맨 왼쪽 값을 버리고 실제 IP 하나만 넘김). 실제 클러스터에서는 확인하지 않았다.

실제 클러스터에서 맞춰야 하는 것:

1. **ingress-nginx 컨트롤러 Service 의 `externalTrafficPolicy: Local`.** `Cluster`(기본)면 노드를 거치며 사용자 IP 가 노드 IP 로 바뀌어,
   여전히 사용자들이 노드 IP 몇 개로 묶인다. 게이트웨이 로그의 IP 가 노드 IP 로만 보이면 이것부터 본다.
2. **react-nginx 의 NodePort(30000)를 인터넷에 열지 않는다.** Ingress 를 거치지 않고 NodePort 로 바로 들어오면 노드를 거쳐 사설 IP 로 보여
   믿는 연결이 되므로, 사용자가 `X-Forwarded-For` 에 아무 IP 나 넣어 속도 제한을 피할 수 있다. Ingress 를 쓰면 방화벽으로 막거나
   `react-nginx-service` 를 ClusterIP 로 바꾼다.

서버 1대(k8s 없이)로 운영하면 그 서버의 nginx 에 같은 설정(`set_real_ip_from`·`real_ip_recursive`·`X-Forwarded-For $remote_addr`)을 쓰고,
게이트웨이 포트는 밖에 열지 않는다.

## 다시 k8s 를 쓰게 되면 볼 것

- 기본 StorageClass 가 있는지 (Postgres PVC). 백업을 클러스터 밖으로 옮기는 방법 (지금은 같은 저장소에 7일치)
- 서버 1대라면: `replicas`·HPA·PDB 를 1대에 맞게 줄일지 (PDB 는 옮겨 갈 노드가 없어 노드 점검(drain)이 멈출 수 있음)
- 위 "사용자 실제 IP" 의 조건 두 가지
