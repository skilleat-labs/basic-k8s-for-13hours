# Lab 5 · Ingress 경로 라우팅

앞선 실습에서 서비스마다 NodePort나 LoadBalancer를 하나씩 붙였습니다. 서비스가 10개면 포트나 LB도 10개가 필요하죠. **Ingress** 는 이 문제를 해결합니다. **문(진입점) 하나** 를 두고, 들어온 요청의 **URL 경로(또는 호스트)** 를 보고 알맞은 Service로 나눠 보내는 L7 라우터입니다. 이번 실습에서는 프론트엔드와 백엔드 두 앱을 만들고, `/` 는 프론트로, `/api` 는 백엔드로 보내는 경로 라우팅을 구성합니다.

!!! abstract "이 실습에서 배우는 것"
    - `Ingress` 로 하나의 진입점에서 URL 경로별로 트래픽을 분기한다
    - `ingressClassName: traefik` 으로 k3s 내장 Ingress 컨트롤러를 사용한다
    - `pathType: Prefix` 와 경로 매칭 규칙을 이해한다
    - NodePort/LoadBalancer의 한계를 Ingress가 어떻게 보완하는지 안다

## 사전 조건

- Rancher Desktop의 k3s 클러스터가 실행 중이다
- **이전 Lab의 LoadBalancer(`web-lb`)를 삭제했는지 확인** 하세요. Traefik이 호스트의 80 포트를 써야 하므로 겹치면 안 됩니다.

```bash
kubectl get svc
```

`LoadBalancer` 타입이 80 포트를 잡고 있지 않은지 확인합니다.

!!! info "Ingress 컨트롤러란 — k3s는 Traefik 내장"
    Ingress 리소스(규칙)는 **그 규칙을 실제로 실행할 컨트롤러** 가 있어야 동작합니다. k3s에는 **Traefik** 이라는 Ingress 컨트롤러가 기본 설치되어 있고, 호스트의 **80 포트(localhost:80)** 로 들어온 요청을 받습니다. 그래서 우리는 컨트롤러를 따로 설치할 필요 없이 `http://localhost/` 로 바로 테스트할 수 있습니다. 설치 여부는 아래로 확인됩니다.

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik
```

```text
NAME                       READY   STATUS    RESTARTS   AGE
traefik-5d45fc8cc9-xk7pq   1/1     Running   0          3d
```

## 1. 프론트엔드 배포

아래를 `frontend.yaml` 로 저장하세요. Deployment와 Service를 한 파일에 담습니다(`---` 로 구분).

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
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f frontend.yaml
```

## 2. 백엔드 배포

백엔드는 간단한 응답을 돌려주는 `hashicorp/http-echo` 를 씁니다. 이 이미지는 컨테이너 포트 `5678` 에서 지정한 텍스트를 응답합니다. 아래를 `backend.yaml` 로 저장하세요.

```yaml
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
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 5678
      targetPort: 5678
```

```bash
kubectl apply -f backend.yaml
kubectl get pods
```

```text
NAME                        READY   STATUS    RESTARTS   AGE
backend-7c9f8d5b6c-4wq2n    1/1     Running   0          20s
frontend-6b7d8f9c4d-hp8kt   1/1     Running   0          40s
```

두 앱 모두 `ClusterIP` Service만 가지고 있어, 아직 외부에서는 접근할 수 없습니다. 이 둘을 Ingress로 묶어 하나의 문으로 노출할 차례입니다.

## 3. Ingress 규칙 만들기

`/` 요청은 `frontend:80`, `/api` 요청은 `backend:5678` 로 보내는 규칙을 정의합니다. 아래를 `ingress.yaml` 로 저장하세요.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: traefik
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 5678
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
```

```text
NAME          CLASS     HOSTS   ADDRESS     PORTS   AGE
app-ingress   traefik   *       127.0.0.1   80      10s
```

`CLASS` 가 `traefik`, `ADDRESS` 에 `127.0.0.1` 이 채워지면 Traefik이 이 규칙을 받아들였다는 뜻입니다.

!!! info "규칙 순서와 pathType: Prefix"
    `pathType: Prefix` 는 "경로가 이 접두사로 시작하면 매칭"입니다. `/` 는 모든 경로에 매칭되므로, 더 구체적인 `/api` 규칙을 함께 두면 Traefik은 **가장 길게 일치하는 규칙** 을 우선합니다. 그래서 `/api/...` 요청은 `/` 가 아니라 `/api` 로 갑니다.

## 4. 경로별로 접근 확인

Traefik이 `localhost:80` 을 듣고 있으므로 `http://localhost/` 로 테스트합니다.

먼저 `/` — 프론트엔드(nginx)로 가야 합니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost/
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost/
    ```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

다음 `/api` — 백엔드(http-echo)로 가야 합니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost/api
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost/api
    ```

```text
hello from backend
```

**같은 주소(`localhost`), 같은 80 포트인데 경로에 따라 다른 앱** 이 응답했습니다. 이것이 Ingress의 경로 라우팅입니다.

## 5. Ingress가 해결한 문제

앞선 방식들과 비교하면 Ingress의 가치가 분명해집니다.

| 방식 | 앱 2개를 노출하려면 | 접근 주소 |
|---|---|---|
| NodePort | 포트 2개(예: 30080, 30081) 필요 | `localhost:30080`, `localhost:30081` |
| LoadBalancer | LB 2개 필요(클라우드에선 2배 과금) | 서로 다른 외부 IP 2개 |
| **Ingress** | **진입점 1개** 로 통합 | `localhost/`, `localhost/api` |

!!! info "Ingress는 문 하나, Service는 여러 개"
    Ingress는 외부 진입점을 하나로 통합하고, 경로/호스트 규칙으로 뒤쪽의 여러 ClusterIP Service에 트래픽을 분배합니다. 실무에서 웹 앱 여러 개를 도메인·경로로 나눠 노출하는 표준 방식입니다.

## 검증

- `kubectl get ingress` 에서 `CLASS=traefik`, `ADDRESS` 에 IP가 찍힌다
- `http://localhost/` → nginx 환영 페이지
- `http://localhost/api` → `hello from backend`

## 직접 해 보기

!!! question "도전 과제"
    1. `curl http://localhost/api/anything` 처럼 `/api` 하위 경로로 요청해 보세요. 여전히 백엔드로 가나요? (`Prefix` 매칭 확인)
    2. Ingress 규칙에서 `/api` 블록을 지우고 다시 apply한 뒤 `curl http://localhost/api` 를 하면 어디로 가는지 보세요.
    3. `backend.yaml` 의 `-text=` 값을 다른 문자열로 바꾸고 `kubectl rollout restart deployment/backend` 후 응답이 바뀌는지 확인하세요.

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **`http://localhost/` 가 404** — Ingress 규칙의 `service.name`/`port.number` 가 실제 Service와 일치하는지 확인하세요. `kubectl get svc` 로 대조합니다.
    - **`/api` 인데 프론트로 감** — 규칙에 `/api` 블록이 빠졌거나 `pathType` 이 틀렸습니다. `Prefix` 인지 확인하세요.
    - **접속 자체가 안 됨(연결 거부)** — 80 포트를 이전 Lab의 LoadBalancer가 잡고 있을 수 있습니다. `kubectl get svc` 로 `LoadBalancer` 타입을 찾아 지우세요.
    - **`backend` Pod가 `CrashLoopBackOff`** — `-listen=:5678` args 오타가 흔합니다. `kubectl logs -l app=backend` 로 확인하세요.

## 정리

```bash
kubectl delete -f ingress.yaml
kubectl delete -f backend.yaml
kubectl delete -f frontend.yaml
```

---

다음: [Lab 6 · Gateway API와 HTTPRoute](06-gateway-api.md)
