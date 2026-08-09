# Lab 7 · NetworkPolicy

기본 상태의 쿠버네티스에서는 **모든 Pod 가 서로 자유롭게 통신**할 수 있습니다. 프론트엔드가 데이터베이스에 직접 접속하는 것도, 침입당한 Pod 가 옆의 다른 Pod 로 옮겨 가는 것(내부 이동, lateral movement)도 아무 제약이 없습니다. **NetworkPolicy** 는 Pod 사이의 트래픽에 방화벽 규칙을 세워, "누가 누구에게 말을 걸 수 있는가"를 통제합니다. 이번 실습에서는 3계층(frontend·backend·db) 앱을 배포하고, **전면 차단(default deny)** 후 **의도한 경로만 선별 허용**하는 정석적인 접근을 실습합니다.

!!! abstract "이 실습에서 배우는 것"
    - 기본적으로 모든 Pod 통신이 열려 있다는 사실과 그 위험
    - default deny-all 로 전면 차단한 뒤 필요한 경로만 여는 패턴
    - Ingress(들어오는)와 Egress(나가는) 규칙의 구분
    - default deny 시 **DNS 도 끊긴다**는 함정과 DNS egress 허용
    - `podSelector` 와 `namespaceSelector`, AND vs OR 결합 규칙

!!! info "k3s 는 NetworkPolicy 를 실제로 강제합니다"
    NetworkPolicy 는 CNI(네트워크 플러그인)가 지원해야 실제로 동작합니다. **k3s 는 kube-router 기반의 NetworkPolicy 컨트롤러를 내장**하고 있어, 이 실습의 차단·허용이 실제로 적용됩니다. 만약 NetworkPolicy 를 지원하지 않는 CNI 를 쓰는 환경이라면, 정책을 만들어도 **조용히 무시**되어 통신이 계속 열려 있을 수 있으니 주의하세요.

## 사전 조건

- Service·DNS(2일차)와 라벨/셀렉터 개념에 익숙해야 합니다.
- 클러스터가 정상인지 확인합니다.

```bash
kubectl get nodes
```

## 1. 3계층 앱 배포하기

`netpol-demo` 네임스페이스에 frontend·backend·db 세 계층을 배포합니다. backend·db 는 확인하기 쉽도록 nginx 로, 각 계층에 Service 도 붙입니다. 라벨 `app=frontend|backend|db` 로 계층을 구분합니다.

`three-tier.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: netpol-demo
---
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: netpol-demo
  labels:
    app: frontend
spec:
  containers:
    - name: shell
      image: curlimages/curl
      command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: netpol-demo
  labels:
    app: backend
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: netpol-demo
spec:
  selector:
    app: backend
  ports:
    - port: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: db
  namespace: netpol-demo
  labels:
    app: db
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: db
  namespace: netpol-demo
spec:
  selector:
    app: db
  ports:
    - port: 80
```

```bash
kubectl apply -f three-tier.yaml
kubectl wait --for=condition=Ready pod --all -n netpol-demo --timeout=90s
```

```text
namespace/netpol-demo created
pod/frontend created
pod/backend created
service/backend created
pod/db created
service/db created
pod/frontend condition met
pod/backend condition met
pod/db condition met
```

!!! note "frontend 는 테스트 클라이언트 역할"
    frontend Pod 는 `curlimages/curl` 로 띄웠습니다. 여기서 `curl` 로 backend·db 에 접속해 보며 정책의 효과를 확인할 것입니다.

## 2. 정책이 없을 때 모두 통신됨을 확인

먼저 아무 정책도 없는 상태에서 frontend 가 backend 와 db 에 모두 접속되는지 봅니다.

frontend → backend:

```bash
kubectl exec -n netpol-demo frontend -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 backend
```

```text
200
```

frontend → db:

```bash
kubectl exec -n netpol-demo frontend -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 db
```

```text
200
```

!!! danger "프론트엔드가 DB 에 직접 접속됩니다"
    frontend 는 원래 backend 하고만 이야기하면 됩니다. 그런데 지금은 **db 에도 곧바로 접속**됩니다(`200`). 만약 frontend 가 탈취되면 공격자는 곧장 데이터베이스로 향할 수 있습니다. 기본값이 "전부 허용"이라 내부 이동을 막을 방벽이 하나도 없는 상태입니다.

## 3. 방어 적용 · 전면 차단(default deny-all)

!!! success "방어 적용 · 일단 전부 막는다"
    보안의 정석은 "필요한 것만 열기(allowlist)"입니다. 그러려면 먼저 모든 것을 막아야 합니다. 네임스페이스의 모든 Pod 에 대해 Ingress·Egress 를 전면 차단하는 정책을 겁니다.

`deny-all.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: netpol-demo
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

- `podSelector: {}`: 빈 셀렉터는 **네임스페이스의 모든 Pod** 를 대상으로 합니다.
- `policyTypes: [Ingress, Egress]`: 들어오는 트래픽과 나가는 트래픽을 모두 통제 대상으로 삼습니다.
- `ingress`·`egress` 규칙을 하나도 적지 않았으므로 **아무것도 허용하지 않음** = 전면 차단입니다.

```bash
kubectl apply -f deny-all.yaml
```

```text
networkpolicy.networking.k8s.io/default-deny-all created
```

이제 다시 frontend → backend 를 시도합니다.

```bash
kubectl exec -n netpol-demo frontend -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 backend
```

```text
command terminated with exit code 28
```

!!! success "모든 통신이 끊겼습니다"
    `curl` 이 5초 동안 응답을 받지 못하고 타임아웃(exit code 28)으로 실패합니다. frontend → db 도 마찬가지입니다. 전면 차단이 실제로 강제되고 있습니다(k3s 의 내장 컨트롤러 덕분). 이제 여기서부터 필요한 경로만 하나씩 열어 갑니다.

## 4. 함정 · DNS 도 끊겼다

이름(`backend`)이 아니라 진짜 통신이 막힌 것인지 확인하다 보면 흥미로운 현상을 만납니다. 이름 해석부터 실패합니다.

```bash
kubectl exec -n netpol-demo frontend -- nslookup backend
```

```text
;; connection timed out; no servers could be reached
command terminated with exit code 1
```

!!! warning "Egress 를 막으면 DNS 조회도 막힙니다"
    쿠버네티스에서 `backend` 같은 이름을 IP 로 바꾸려면 `kube-system` 의 CoreDNS 에 질의해야 합니다. 그런데 default deny 가 **나가는 트래픽(Egress)** 을 전부 막았으니, CoreDNS 로 가는 DNS 질의(포트 53)까지 차단됩니다. 그래서 이름 해석 단계에서부터 실패합니다. 선별 허용을 설계할 때 **DNS egress 를 반드시 함께 열어 줘야** 하는 이유입니다. 이 함정을 모르면 "규칙을 제대로 열었는데 왜 안 되지?"로 오래 헤매게 됩니다.

## 5. 방어 적용 · 의도한 경로만 선별 허용

!!! success "방어 적용 · 필요한 길만 낸다"
    이제 실제 앱 흐름(frontend → backend → db)과 DNS 만 허용합니다. 여러 정책을 하나의 파일에 담습니다.

`allow-rules.yaml`

```yaml
# (1) 모든 Pod 가 DNS(CoreDNS) 로 질의할 수 있게 egress 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: netpol-demo
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
# (2) backend 는 frontend 에서 오는 ingress 만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
---
# (3) frontend 는 backend 로 나가는 egress 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-allow-backend
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80
---
# (4) db 는 backend 에서 오는 ingress 만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-backend
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 80
---
# (5) backend 는 db 로 나가는 egress 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-db
  namespace: netpol-demo
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: db
      ports:
        - protocol: TCP
          port: 80
```

!!! note "Ingress 와 Egress 는 양쪽 다 열어야 통합니다"
    frontend 가 backend 를 호출하려면, **frontend 의 Egress(나감)** 와 **backend 의 Ingress(들어옴)** 가 둘 다 허용돼야 합니다. default deny 가 양방향을 막았기 때문입니다. 그래서 규칙 (2)+(3), (4)+(5)가 짝을 이룹니다. 하나만 열면 여전히 막힙니다.

```bash
kubectl apply -f allow-rules.yaml
```

```text
networkpolicy.networking.k8s.io/allow-dns created
networkpolicy.networking.k8s.io/backend-allow-frontend created
networkpolicy.networking.k8s.io/frontend-allow-backend created
networkpolicy.networking.k8s.io/db-allow-backend created
networkpolicy.networking.k8s.io/backend-allow-db created
```

## 6. 의도한 경로만 통하는지 검증

허용해야 하는 길 — frontend → backend:

```bash
kubectl exec -n netpol-demo frontend -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 backend
```

```text
200
```

차단해야 하는 길 — frontend → db(직접 접속은 막혀야 함):

```bash
kubectl exec -n netpol-demo frontend -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 db
```

```text
command terminated with exit code 28
```

허용해야 하는 길 — backend → db:

```bash
kubectl exec -n netpol-demo backend -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 db
```

```text
200
```

!!! success "의도한 경로만 통합니다"
    frontend → backend(`200`)와 backend → db(`200`)는 허용되고, frontend → db 직접 접속(`timeout`)은 차단됩니다. 정확히 설계한 트래픽 흐름만 남았습니다. 이제 frontend 가 탈취되어도 공격자는 backend 까지만 도달할 수 있고, 데이터베이스로 바로 넘어가지 못합니다. **내부 이동이 차단**된 것입니다. DNS 를 함께 열어 둔 덕분에 이름 해석도 정상 동작합니다.

## from 셀렉터와 AND vs OR

`from`(또는 `to`)을 쓸 때 셀렉터의 결합 방식을 정확히 이해해야 의도대로 열 수 있습니다.

**같은 `-` 항목 안에서 `podSelector` 와 `namespaceSelector` 를 함께 쓰면 AND** (둘 다 만족):

```yaml
from:
  - namespaceSelector:
      matchLabels:
        team: web
    podSelector:
      matchLabels:
        app: frontend
# → "team=web 네임스페이스" 그리고(AND) "app=frontend Pod" 에서 오는 트래픽만
```

**`-` 항목을 여러 개 나열하면 OR** (둘 중 하나라도 만족):

```yaml
from:
  - podSelector:
      matchLabels:
        app: frontend
  - podSelector:
      matchLabels:
        app: admin
# → "app=frontend" 또는(OR) "app=admin" 에서 오는 트래픽 허용
```

| 셀렉터 | 의미 |
|---|---|
| `podSelector` | (같은 네임스페이스 안에서) 라벨로 대상 Pod 선택 |
| `namespaceSelector` | 라벨로 대상 네임스페이스 선택 |
| 한 `from` 블록 안에 둘 다 | **AND** — 네임스페이스와 Pod 조건을 모두 만족 |
| 여러 `from` 블록 나열 | **OR** — 어느 하나라도 만족 |

!!! tip "이 차이가 보안 사고의 단골 원인"
    AND 로 좁히려던 것을 실수로 OR 로 써서 의도보다 넓게 열리는 일이 잦습니다. `-`(대시)의 위치로 결합 방식이 완전히 달라지니, 정책 작성 후에는 이 실습처럼 **실제로 접속 테스트**해 검증하는 습관이 중요합니다.

## 검증

- 정책 적용 전: frontend → backend, frontend → db 모두 `200`.
- default deny 후: 모든 접속 타임아웃, `nslookup` 도 실패.
- 선별 허용 후: frontend → backend `200`, backend → db `200`, frontend → db 는 타임아웃.

## 직접 해 보기

1. `allow-dns` 정책만 삭제(`kubectl delete networkpolicy allow-dns -n netpol-demo`)한 뒤 frontend → backend 를 다시 시도해 보세요. IP 는 통하는데 이름은 안 통하는(DNS 실패) 상황을 재현하고, DNS egress 의 중요성을 체감합니다. 확인 후 다시 적용하세요.
2. `db-allow-backend` 의 `from` 을 frontend 도 허용하도록 `-` 항목을 하나 더 추가(OR)한 뒤, frontend → db 가 이제 통하는지 확인해 보세요. 보안상 원래는 막아야 하는 경로이니, 실습 후 되돌리세요.

!!! failure "자주 만나는 오류"
    **증상**: 규칙을 다 열었는데도 접속이 안 됨(타임아웃)
    **원인**: 대개 DNS egress 를 빠뜨렸거나, Ingress/Egress 한쪽만 열었음.
    **해결**: `allow-dns` 가 적용됐는지, 그리고 호출하는 쪽 Egress 와 받는 쪽 Ingress 가 **둘 다** 열렸는지 확인합니다.

    ---

    **증상**: 정책을 걸어도 통신이 전혀 차단되지 않음
    **원인**: CNI 가 NetworkPolicy 를 지원하지 않는 환경(k3s 는 지원하므로 이 실습에서는 해당 없음).
    **해결**: k3s/Rancher Desktop 환경인지 확인합니다. 다른 클러스터라면 CNI 가 NetworkPolicy 를 지원하는지 점검합니다.

    ---

    **증상**: `namespaceSelector` 로 kube-system 을 지정했는데 DNS 가 안 열림
    **원인**: kube-system 네임스페이스에 매칭 라벨이 없음.
    **해결**: 이 실습처럼 모든 네임스페이스에 기본으로 있는 `kubernetes.io/metadata.name` 라벨을 쓰거나, CoreDNS Pod 라벨(`k8s-app: kube-dns`)이 맞는지 `kubectl get pods -n kube-system --show-labels` 로 확인합니다.

## 정리

```bash
kubectl delete namespace netpol-demo
```

```text
namespace "netpol-demo" deleted
```

네임스페이스를 지우면 그 안의 Pod·Service·NetworkPolicy 가 모두 함께 삭제됩니다.

---

## 3일 과정을 모두 마쳤습니다 :material-party-popper:

축하합니다! 도커의 한계에서 시작해 쿠버네티스의 **배포·자가복구·스케일링**(1일차), **Service·Ingress·설정 관리**(2일차), 그리고 **영속성과 보안**(3일차)까지 13시간의 여정을 완주했습니다. 이제 여러분은 앱을 안전하게 배포하고, 데이터를 지키고, 침입의 사슬을 여러 층에서 끊을 수 있습니다.

!!! tip "다음 학습 여정"
    더 나아가고 싶다면 **Helm**(패키지 관리로 복잡한 앱을 한 번에 배포), **Prometheus·Grafana 모니터링**, **GitOps(Argo CD)** 로 배포 자동화, 그리고 실제 클라우드의 관리형 쿠버네티스(EKS·GKE·AKS)를 다음 목표로 삼아 보세요. 여기서 다진 기본기가 든든한 출발점이 되어 줄 것입니다. 수고하셨습니다!
