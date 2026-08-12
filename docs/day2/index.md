# 2일차 · 서비스와 네트워킹

Pod는 언제든 죽고 다시 생기며 IP가 바뀝니다. 오늘은 **Service** 로 안정적인 주소를 부여하고, **NodePort·LoadBalancer·Ingress·Gateway API** 로 외부에 노출하며, **ConfigMap·Secret** 으로 설정과 자격증명을 코드에서 분리합니다. 마지막에는 프론트·백엔드·DB로 이루어진 **3-tier 앱** 을 통째로 배포합니다.

## 오늘의 실습

| # | 실습 | 무엇을 배우나 |
|---|---|---|
| 1 | [ClusterIP와 DNS](01-clusterip-dns.md) | selector-label 매칭, 클러스터 내부 DNS |
| 2 | [Endpoints 디버깅](02-endpoints-debug.md) | 라벨 오타 장애 재현 → Endpoints로 원인 추적 |
| 3 | [NodePort와 port-forward](03-nodeport-portforward.md) | 외부 노출 기초, 포트 3종, 임시 접근 |
| 4 | [LoadBalancer](04-loadbalancer.md) | 서비스 타입 계층, 외부 IP 할당 |
| 5 | [Ingress 실습](05-ingress-routing.md) | NGINX Ingress Controller로 경로·호스트 라우팅 |
| 6 | [Gateway API 실습](06-gateway-api.md) | Envoy Gateway·역할 분리·가중치 카나리 |
| 7 | [3-tier 앱 배포](07-three-tier-app.md) | 프론트·백엔드·DB 통합 배포 |
| 8 | [ConfigMap](08-configmap.md) | 환경변수·볼륨으로 설정 주입 |
| 9 | [Secret](09-secret.md) | DB 자격증명 분리, `rollout restart` |

!!! note "사전 준비"
    1일차를 마쳤거나, 최소한 Pod/Deployment 개념과 `kubectl apply -f` 사용에 익숙해야 합니다.

시작: **[Lab 1 · ClusterIP와 DNS](01-clusterip-dns.md)**
