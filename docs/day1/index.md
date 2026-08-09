# 1일차 · 입문과 워크로드

도커의 한계를 직접 체감하는 것에서 출발해, 로컬 클러스터에 접속하고, 쿠버네티스의 가장 기본 단위인 **Pod** 와 이를 관리하는 **Deployment** 를 직접 만들고 다뤄 봅니다. 오늘이 끝나면 "왜 컨테이너를 직접 안 돌리고 쿠버네티스에 맡기는가"를 손으로 체감하게 됩니다.

## 오늘의 실습

| # | 실습 | 무엇을 배우나 |
|---|---|---|
| 1 | [도커의 한계와 쿠버네티스](01-docker-and-k8s.md) | 단일 호스트의 한계 체감, 아키텍처·선언적 관리 미리보기 |
| 2 | [클러스터 접속과 조회](02-cluster-access.md) | `kubectl get/describe/explain`, kubeconfig·컨텍스트 |
| 3 | [첫 Pod 실행](03-first-pod.md) | `kubectl run`, 로그·exec, 도커 CLI와의 대응 |
| 4 | [Pod IP는 휘발성](04-pod-ip-ephemeral.md) | Pod 재생성 시 IP 변경 재현 → Service의 필요성 |
| 5 | [YAML 매니페스트](05-yaml-manifest.md) | 선언형 배포, 매니페스트 4요소, `--dry-run` |
| 6 | [Deployment와 자가복구](06-deployment-selfheal.md) | Deployment→ReplicaSet→Pod, 자가복구 |
| 7 | [스케일링](07-scaling.md) | `kubectl scale`, replicas 조정 |
| 8 | [롤링 업데이트와 롤백](08-rolling-update.md) | `set image`, `rollout status/history/undo` |

!!! note "사전 준비"
    [실습 환경 준비](../setup/prerequisites.md)를 마치고 `kubectl get nodes` 가 `Ready` 로 나오는 상태여야 합니다.

시작: **[Lab 1 · 도커의 한계와 쿠버네티스](01-docker-and-k8s.md)**
