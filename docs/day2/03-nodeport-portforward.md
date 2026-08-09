# Lab 3 · NodePort와 port-forward

지금까지 만든 `ClusterIP` Service는 클러스터 **내부에서만** 접근할 수 있었습니다. 이제 밖(내 노트북 브라우저)에서 접근하는 첫 번째 방법인 **NodePort** 를 배웁니다. 그리고 개발·디버깅 때 유용한 임시 접근 수단인 **`kubectl port-forward`** 도 함께 다룹니다. 둘 다 외부 접근을 열어 주지만 동작 방식이 완전히 다릅니다.

!!! abstract "이 실습에서 배우는 것"
    - `NodePort` 타입으로 노드의 포트를 열어 외부에서 접근한다
    - Service의 `port` / `targetPort` / `nodePort` 세 포트의 역할을 구분한다
    - `kubectl port-forward` 로 특정 리소스에 임시 터널을 뚫는다
    - NodePort(정식 노출)와 port-forward(임시 터널)의 차이를 이해한다

## 사전 조건

- Rancher Desktop의 k3s 클러스터가 실행 중이다
- Lab 1의 `web` Deployment가 필요합니다. 없으면 아래로 다시 만드세요.

아래를 `web-deploy.yaml` 로 저장하고 적용합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f web-deploy.yaml
```

## 1. NodePort Service 만들기

아래를 `web-nodeport.yaml` 로 저장하세요.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80          # Service의 포트(클러스터 내부용)
      targetPort: 80    # Pod(컨테이너)가 실제 듣는 포트
      nodePort: 30080   # 노드에 열리는 외부 포트(30000~32767)
```

```bash
kubectl apply -f web-nodeport.yaml
kubectl get svc web
```

```text
NAME   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
web    NodePort   10.43.120.55   <none>        80:30080/TCP   8s
```

`PORT(S)` 가 `80:30080/TCP` 로 바뀌었습니다. `80` 은 여전히 내부용 Service 포트, `30080` 은 새로 열린 외부 포트입니다.

## 2. 포트 3종 구분하기

NodePort Service에는 포트가 세 개 등장합니다. 헷갈리기 쉬우니 표로 정리합니다.

| 포트 | 위치 | 역할 | 이 실습의 값 |
|---|---|---|---|
| `nodePort` | 노드(호스트) | 외부에서 들어오는 입구 포트. 30000~32767만 허용 | `30080` |
| `port` | Service | 클러스터 내부에서 Service를 부를 때 쓰는 포트 | `80` |
| `targetPort` | Pod(컨테이너) | 트래픽이 최종 도착하는 컨테이너의 포트 | `80` |

트래픽이 흐르는 순서는 이렇습니다.

```text
브라우저 → 노드:30080(nodePort) → Service:80(port) → Pod:80(targetPort)
```

!!! info "왜 30000번대만 되나요?"
    `nodePort` 범위는 기본 30000~32767로 제한됩니다. 22·80·443 같은 잘 알려진 포트와 충돌을 피하기 위한 안전장치입니다. 생략하면 이 범위에서 자동으로 하나 배정됩니다.

## 3. 외부에서 접근하기

Rancher Desktop은 노드 포트를 호스트의 `localhost` 로 연결해 줍니다. 브라우저에서 `http://localhost:30080` 을 열거나, 터미널에서 확인합니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:30080
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost:30080
    ```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

클러스터 밖(내 노트북)에서 처음으로 Pod에 도달했습니다. Lab 1의 ClusterIP와 달리 테스트 Pod 없이 바로 접속됩니다.

## 4. port-forward로 임시 접근

이번엔 전혀 다른 방식인 `port-forward` 입니다. NodePort를 열지 않아도, `kubectl` 이 내 컴퓨터의 포트와 클러스터 리소스 사이에 **임시 터널** 을 만들어 줍니다.

```bash
kubectl port-forward svc/web 8080:80
```

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

이 명령은 **터미널을 점유한 채 계속 실행** 됩니다. 새 터미널을 하나 더 열어 접속해 봅니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8080
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost:8080
    ```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

확인이 끝나면 port-forward를 실행 중인 터미널에서 `Ctrl + C` 로 중단합니다. 중단하는 순간 터널도 사라져 `localhost:8080` 접속이 끊깁니다.

## 5. 두 방식의 차이

| 항목 | NodePort | port-forward |
|---|---|---|
| 성격 | 클러스터에 **상시** 열리는 정식 노출 | `kubectl` 이 살아 있는 동안만 유지되는 **임시** 터널 |
| 누가 접근 가능 | 노드에 네트워크로 닿는 누구나 | 명령을 실행한 내 컴퓨터에서만 |
| Service 경유 | 함. (Service→Endpoints→Pod 로드밸런싱) | `svc/web` 을 지정해도 실제로는 Pod 하나로 직접 터널을 뚫음 |
| 주 용도 | 개발·테스트 환경의 외부 노출 | 디버깅, 잠깐 확인, 대시보드 접속 |

!!! info "port-forward는 Service 로드밸런싱을 안 거친다"
    `port-forward svc/web` 이라고 Service를 지정해도, kubectl은 그 Service가 고른 **Pod 하나** 를 골라 직접 연결합니다. 즉 여러 Pod로 분산되지 않습니다. "이 Pod가 응답을 주긴 하나?"를 빠르게 확인할 때 쓰는 도구이지, 실제 트래픽 분산을 흉내 내는 도구가 아닙니다.

## 검증

- `kubectl get svc web` 의 `PORT(S)` 가 `80:30080/TCP` 로 보인다
- `http://localhost:30080` 에서 nginx 페이지가 열린다
- `port-forward` 실행 중에는 `http://localhost:8080` 이 되고, `Ctrl+C` 로 끊으면 접속도 끊긴다

## 직접 해 보기

!!! question "도전 과제"
    1. `web-nodeport.yaml` 에서 `nodePort: 30080` 줄을 통째로 지우고 다시 apply한 뒤 `kubectl get svc web` 을 보세요. 어떤 포트가 자동 배정됐나요?
    2. `nodePort: 80` 처럼 범위를 벗어난 값으로 바꿔 apply해 보세요. 어떤 에러가 나는지 읽어 보세요.
    3. `kubectl port-forward deployment/web 8080:80` 처럼 Deployment를 대상으로도 터널을 뚫을 수 있습니다. `svc/web` 대상과 무엇이 다른지 생각해 보세요.

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **`provided port is not in the valid range 30000-32767`** — `nodePort` 값이 범위를 벗어났습니다. 30000~32767 사이로 바꾸세요.
    - **`localhost:30080` 접속 안 됨** — Rancher Desktop이 실행 중인지, Service가 `NodePort` 타입인지(`get svc` 로 확인) 점검하세요.
    - **`port-forward` 가 `bind: address already in use`** — 로컬 8080 포트를 다른 프로그램이 쓰고 있습니다. `8081:80` 처럼 다른 로컬 포트로 바꾸세요.
    - **`port-forward` 터미널을 닫으니 접속이 끊김** — 정상입니다. port-forward는 명령이 살아 있는 동안만 유지되는 임시 터널입니다.

## 정리

```bash
kubectl delete -f web-nodeport.yaml
kubectl delete -f web-deploy.yaml
```

---

다음: [Lab 4 · LoadBalancer](04-loadbalancer.md)
