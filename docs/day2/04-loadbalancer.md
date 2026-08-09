# Lab 4 · LoadBalancer

NodePort는 외부 접근을 열어 주지만, 접속할 때 `30080` 같은 어색한 포트 번호를 붙여야 했습니다. 실무에서 사용자에게 "포트 30080으로 오세요"라고 안내할 수는 없겠죠. **LoadBalancer** 타입 Service는 외부에서 접근할 **전용 외부 IP** 를 할당받아, 표준 포트로 서비스를 노출합니다. 이번 실습에서는 LoadBalancer를 만들고, k3s가 로컬에서 어떻게 외부 IP를 흉내 내는지 확인합니다.

!!! abstract "이 실습에서 배우는 것"
    - `LoadBalancer` 타입 Service를 만들고 `EXTERNAL-IP` 할당을 확인한다
    - `ClusterIP ⊂ NodePort ⊂ LoadBalancer` 계층 관계를 이해한다
    - 로컬(k3s)과 클라우드에서 LoadBalancer가 어떻게 다르게 동작하는지 안다

## 사전 조건

- Rancher Desktop의 k3s 클러스터가 실행 중이다
- 아래 `web` Deployment가 필요합니다.

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

## 1. LoadBalancer Service 만들기

아래를 `web-lb.yaml` 로 저장하세요.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f web-lb.yaml
```

## 2. EXTERNAL-IP 할당 확인

Service 목록을 봅니다. LoadBalancer는 외부 IP를 배정받기까지 잠깐 `<pending>` 상태를 거칠 수 있습니다.

```bash
kubectl get svc web-lb
```

```text
NAME     TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
web-lb   LoadBalancer   10.43.55.201   127.0.0.1     80:31562/TCP   15s
```

앞선 NodePort와 달리 **`EXTERNAL-IP` 에 실제 값(`127.0.0.1`)** 이 찍혔습니다. Rancher Desktop의 k3s는 로컬 환경이라 외부 IP로 `127.0.0.1`(localhost)을 할당합니다.

!!! info "이 외부 IP는 누가 주는가 — klipper-lb"
    클라우드가 아닌 로컬에는 진짜 로드밸런서 장비가 없습니다. 대신 k3s에는 **ServiceLB(별칭 klipper-lb)** 라는 내장 로드밸런서 컨트롤러가 있어, LoadBalancer Service가 생기면 노드의 호스트 포트를 이용해 `EXTERNAL-IP` 를 흉내 내 줍니다. 그래서 로컬에서도 LoadBalancer 실습이 가능합니다.

## 3. 외부에서 접근하기

`EXTERNAL-IP` 가 `127.0.0.1` 이므로 `localhost` 로, 그것도 **표준 80 포트** 로 바로 접근됩니다(포트 번호를 붙일 필요 없음).

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost
    ```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

NodePort 때 붙였던 `:30080` 없이 접속됩니다. 사용자 입장에서 훨씬 자연스러운 주소입니다.

## 4. 서비스 타입 계층 이해

세 가지 Service 타입은 서로 별개가 아니라 **포함 관계** 입니다. 상위 타입은 하위 타입의 기능을 그대로 포함합니다.

```text
LoadBalancer  ──▶ 외부 IP로 노출  (안에 NodePort를 포함)
    └ NodePort  ──▶ 노드 포트로 노출 (안에 ClusterIP를 포함)
          └ ClusterIP ──▶ 클러스터 내부 전용 주소
```

| 타입 | 얻는 것 | 접근 범위 |
|---|---|---|
| ClusterIP | 내부용 가상 IP | 클러스터 내부만 |
| NodePort | 위 + 노드의 30000~32767 포트 | 노드에 닿는 외부 |
| LoadBalancer | 위 + 전용 EXTERNAL-IP | 외부에서 표준 포트로 |

이 포함 관계는 방금 만든 Service의 `PORT(S)` 에서도 드러납니다. LoadBalancer인데 `80:31562/TCP` 처럼 **NodePort(31562)도 함께 열려** 있죠. LoadBalancer가 내부적으로 NodePort를 품고 있기 때문입니다.

## 5. 로컬과 클라우드의 차이

!!! info "클라우드에서는 진짜 LB가 생긴다"
    AWS·GCP·Azure 같은 클라우드에서 동일한 `type: LoadBalancer` Service를 만들면, 클라우드 공급자의 컨트롤러가 **실제 로드밸런서 장비(AWS ELB, GCP Forwarding Rule 등)를 프로비저닝** 하고 공인 IP나 도메인을 `EXTERNAL-IP` 에 채워 줍니다. 인터넷 어디서나 접속할 수 있는 진짜 외부 주소입니다.

    로컬(k3s ServiceLB)에서는 이를 `127.0.0.1` 로 흉내 낼 뿐이지만, **매니페스트는 완전히 동일** 합니다. 즉 로컬에서 검증한 YAML을 그대로 클라우드에 올리면 됩니다 — 이것이 쿠버네티스의 이식성입니다.

!!! warning "LoadBalancer의 비용 함정"
    클라우드에서 Service마다 `type: LoadBalancer` 를 쓰면 LB가 개수만큼 생기고 각각 과금됩니다. 서비스가 많아지면 비쌉니다. 이 문제를 해결하는 것이 다음 실습의 **Ingress** 입니다.

## 검증

- `kubectl get svc web-lb` 의 `EXTERNAL-IP` 에 `127.0.0.1`(또는 localhost)이 찍힌다
- `http://localhost` 에서 포트 번호 없이 nginx 페이지가 열린다
- `PORT(S)` 에 NodePort(3xxxx)도 함께 보여, LoadBalancer가 NodePort를 포함함을 알 수 있다

## 직접 해 보기

!!! question "도전 과제"
    1. `kubectl get svc web-lb -o wide` 로 좀 더 자세한 정보를 보세요.
    2. Deployment를 `kubectl scale deployment web --replicas=3` 으로 늘린 뒤 `curl http://localhost` 를 여러 번 실행하고, `kubectl logs -l app=web` 로 여러 Pod에 요청이 분산됐는지 확인하세요.
    3. `EXTERNAL-IP` 가 오래 `<pending>` 이면 `kubectl describe svc web-lb` 의 이벤트를 읽어 보세요.

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **`EXTERNAL-IP` 가 계속 `<pending>`** — 로컬에서 klipper-lb가 쓰려는 호스트 포트(80 등)를 다른 프로그램이 이미 점유했을 수 있습니다. `port` 를 `8081` 등으로 바꿔 보거나 충돌 프로그램을 끄세요.
    - **`http://localhost` 접속 안 됨** — 다른 LoadBalancer/Ingress가 이미 80 포트를 쓰고 있는지 확인하세요. 특히 다음 Lab의 Ingress(Traefik)와 80 포트가 겹칠 수 있으니, 이 실습이 끝나면 `web-lb` 를 꼭 정리하세요.
    - **관리형이 아닌 순수 온프레미스 클러스터** — 별도 LB 컨트롤러(MetalLB 등)가 없으면 `EXTERNAL-IP` 가 영영 `<pending>` 입니다. 로컬 k3s는 ServiceLB가 내장이라 그냥 됩니다.

## 정리

다음 Lab의 Ingress와 80 포트가 겹치지 않도록 이 실습 자원은 반드시 정리하세요.

```bash
kubectl delete -f web-lb.yaml
kubectl delete -f web-deploy.yaml
```

---

다음: [Lab 5 · Ingress 경로 라우팅](05-ingress-routing.md)
