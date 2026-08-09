# Lab 6 · Gateway API와 HTTPRoute

Ingress는 훌륭하지만 두 가지 아쉬움이 있습니다. ① TLS·경로 재작성 같은 고급 기능이 표준이 아니라 **벤더별 어노테이션** 으로 갈리고, ② 인프라 담당과 앱 담당의 **역할이 한 리소스에 뒤섞여** 있습니다. **Gateway API** 는 이를 표준 필드로 정리하고 역할을 분리한 차세대 규격입니다. 이번 실습에서는 Gateway API의 세 리소스(GatewayClass · Gateway · HTTPRoute)로 앞 실습과 똑같은 경로 라우팅(`/`→프론트, `/api`→백엔드)을 구성합니다.

!!! abstract "이 실습에서 배우는 것"
    - Gateway API의 **역할 분리** 구조(GatewayClass · Gateway · HTTPRoute)를 이해한다
    - Gateway API **CRD를 설치** 하고 세 리소스를 직접 만든다
    - `HTTPRoute` 로 경로 기반 라우팅을 정의한다
    - Ingress와 Gateway API의 차이를 비교한다

## Gateway API의 역할 분리

Ingress는 규칙 하나에 모든 걸 담지만, Gateway API는 관심사를 세 리소스로 나눕니다.

| 리소스 | 무엇을 정의하나 | 보통 누가 관리 |
|---|---|---|
| **GatewayClass** | 어떤 구현체(컨트롤러)를 쓸지 등록 | 클러스터 관리자 |
| **Gateway** | 리스너 — 어떤 포트/프로토콜로 받을지 | 인프라팀 |
| **HTTPRoute** | 경로·호스트 라우팅 규칙 (어느 Service로) | 앱 개발팀 |

!!! info "Ingress와 같은 원리"
    Gateway API도 Ingress처럼 **규격(리소스)** 일 뿐이고, 실제 트래픽 처리는 별도의 **컨트롤러** 가 합니다. Ingress에서 Traefik이 그 역할을 했듯, Gateway API에도 컨트롤러가 필요합니다.

## 사전 조건

- Rancher Desktop의 k3s 클러스터가 실행 중이다
- 이전 Lab의 Ingress(`app-ingress`)를 삭제했다 (`kubectl delete ingress app-ingress --ignore-not-found`)

## 1. Gateway API CRD 설치

Gateway API는 쿠버네티스 기본 리소스가 아니라 **CRD(사용자 정의 리소스)** 로 추가합니다. 표준(standard) 채널을 설치합니다.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

```text
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
...
```

이제 클러스터가 `gateway`, `httproute` 같은 리소스를 이해합니다.

```bash
kubectl get gatewayclass
```

```text
No resources found
```

아직 GatewayClass가 없습니다. **CRD는 규격만 추가했을 뿐, 이를 처리할 컨트롤러가 없기 때문** 입니다. 다음 단계에서 컨트롤러를 설치합니다.

## 2. Gateway 컨트롤러 설치

!!! warning "Rancher Desktop 기본 Traefik은 Gateway API가 꺼져 있다"
    k3s에 내장된 Traefik은 Ingress는 처리하지만 Gateway API 프로바이더가 기본 비활성입니다. 그래서 여기서는 대표 구현체인 **NGINX Gateway Fabric(NGF)** 을 설치해 확인합니다.

!!! note "버전·URL은 공식 릴리스 기준으로"
    아래 명령의 버전(`v1.4.0`)은 예시입니다. 설치가 안 되면 [NGF 릴리스 페이지](https://github.com/nginx/nginx-gateway-fabric/releases)에서 **최신 버전 번호로 바꿔** 실행하세요. 이 실습의 **핵심(3~5단계의 리소스 생성·검사)은 컨트롤러 없이도 동작** 하며, 컨트롤러는 6단계의 실제 트래픽 확인에만 필요합니다.

NGF의 CRD와 컨트롤러를 설치합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.4.0/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.4.0/deploy/default/deploy.yaml
```

컨트롤러 Pod가 뜨는지 확인합니다.

```bash
kubectl get pods -n nginx-gateway
```

```text
NAME                             READY   STATUS    RESTARTS   AGE
nginx-gateway-6b8f9c7d84-2xk9q   2/2     Running   0          30s
```

이제 GatewayClass가 등록됩니다.

```bash
kubectl get gatewayclass
```

```text
NAME    CONTROLLER                                   ACCEPTED   AGE
nginx   gateway.nginx.org/nginx-gateway-controller   True       20s
```

`ACCEPTED=True` 면 컨트롤러가 이 GatewayClass를 받아들였다는 뜻입니다.

## 3. 앱 배포 (프론트엔드 · 백엔드)

Ingress 실습과 동일한 두 앱을 배포합니다. 아래를 `apps.yaml` 로 저장하세요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
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
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: http-echo
          image: hashicorp/http-echo
          args:
            - "-text=hello from backend"
            - "-listen=:5678"
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 5678
      targetPort: 5678
```

```bash
kubectl apply -f apps.yaml
```

## 4. Gateway 만들기 (인프라팀 역할)

80 포트로 HTTP를 받는 리스너를 정의합니다. 아래를 `gateway.yaml` 로 저장하세요.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same
```

```bash
kubectl apply -f gateway.yaml
kubectl get gateway
```

```text
NAME          CLASS   ADDRESS       PROGRAMMED   AGE
web-gateway   nginx   10.43.x.x     True         15s
```

`PROGRAMMED=True` 면 컨트롤러가 이 Gateway에 맞춰 실제 프록시를 구성했다는 뜻입니다.

## 5. HTTPRoute 만들기 (앱팀 역할)

이제 경로 라우팅 규칙입니다. `/` 는 frontend, `/api` 는 backend로 보냅니다. 아래를 `httproute.yaml` 로 저장하세요.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
spec:
  parentRefs:
    - name: web-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      backendRefs:
        - name: backend
          port: 5678
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend
          port: 80
```

```bash
kubectl apply -f httproute.yaml
kubectl describe httproute app-route
```

`parentRefs` 로 이 규칙이 어느 Gateway에 붙는지 지정하고, `backendRefs` 로 어느 Service로 보낼지 정합니다. Ingress와 달리 **리스너(Gateway)와 라우팅 규칙(HTTPRoute)이 분리** 되어 있는 점에 주목하세요.

## 6. 경로별 접근 확인

컨트롤러 서비스로 접근합니다. 서비스 이름은 환경에 따라 다를 수 있으니 먼저 확인합니다.

```bash
kubectl get svc -n nginx-gateway
```

가장 확실한 접근 방법은 `port-forward` 입니다(서비스 타입과 무관하게 동작). 컨트롤러 서비스를 로컬 8080 포트로 연결합니다.

```bash
kubectl port-forward -n nginx-gateway svc/nginx-gateway 8080:80
```

!!! note
    `port-forward` 는 터미널을 점유합니다. **새 터미널** 을 열어 아래 접속 테스트를 실행하세요.

`/` — 프론트엔드(nginx):

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8080/
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost:8080/
    ```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

`/api` — 백엔드(http-echo):

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8080/api
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost:8080/api
    ```

```text
hello from backend
```

Ingress 때와 똑같은 경로 라우팅을, 이번엔 **역할이 분리된 Gateway API** 로 구현했습니다.

## Ingress vs Gateway API

| | Ingress | Gateway API |
|---|---|---|
| 역할 분리 | 규칙 하나에 혼재 | GatewayClass / Gateway / HTTPRoute로 분리 |
| 고급 기능 | 벤더 어노테이션 의존 | 표준 필드(헤더·가중치·트래픽 분할 등) |
| 프로토콜 | 주로 HTTP/HTTPS | HTTP·TCP·gRPC 등 확장(TCPRoute·GRPCRoute) |
| 이식성 | 컨트롤러 바꾸면 어노테이션 재작성 | 표준이라 구현체 교체가 쉬움 |

!!! info "언제 무엇을 쓰나"
    지금도 많은 클러스터가 Ingress를 쓰지만, 쿠버네티스는 Gateway API를 차세대 표준으로 밀고 있습니다. 새 프로젝트라면 Gateway API를, 기존 자산은 Ingress를 유지하며 점진적으로 옮기는 방식이 일반적입니다.

## 검증

- `kubectl get gatewayclass` → `nginx`, `ACCEPTED=True`
- `kubectl get gateway` → `web-gateway`, `PROGRAMMED=True`
- `http://localhost:8080/` → nginx 환영 페이지, `http://localhost:8080/api` → `hello from backend`

## 직접 해 보기

!!! question "도전 과제"
    1. `kubectl get httproute app-route -o yaml` 로 `status` 를 확인해 규칙이 Gateway에 정상 연결(Accepted)됐는지 보세요.
    2. HTTPRoute에 규칙을 하나 더 추가해 `/health` 경로를 backend로 보내 보세요.
    3. Gateway API의 `TCPRoute`, `GRPCRoute` 가 무엇인지 찾아보고, Ingress로는 왜 이걸 표준으로 다루기 어려운지 생각해 보세요.

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **CRD 설치 URL이 404** — Gateway API 버전이 바뀌었습니다. [릴리스 페이지](https://github.com/kubernetes-sigs/gateway-api/releases)에서 최신 `standard-install.yaml` URL로 교체하세요.
    - **`kubectl get gatewayclass` 가 계속 비어 있음** — 컨트롤러(NGF)가 설치·실행 중인지 `kubectl get pods -n nginx-gateway` 로 확인하세요. NGF 버전이 안 맞으면 릴리스 페이지의 최신 명령을 쓰세요.
    - **`Gateway` 의 `PROGRAMMED` 가 `False`** — 컨트롤러가 아직 준비 중이거나 리스너 설정 오류입니다. `kubectl describe gateway web-gateway` 의 이벤트를 확인하세요.
    - **`port-forward` 연결 거부** — 서비스 이름이 다를 수 있습니다. `kubectl get svc -n nginx-gateway` 로 실제 이름을 확인해 명령을 맞추세요.

## 정리

```bash
kubectl delete -f httproute.yaml
kubectl delete -f gateway.yaml
kubectl delete -f apps.yaml
```

!!! tip "컨트롤러·CRD 정리(선택)"
    NGF 컨트롤러와 Gateway API CRD까지 완전히 지우려면(다음 실습에 불필요하므로 지워도 됩니다):
    ```bash
    kubectl delete -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.4.0/deploy/default/deploy.yaml
    ```

---

다음: [Lab 7 · 3-tier 앱 배포](07-three-tier-app.md)
