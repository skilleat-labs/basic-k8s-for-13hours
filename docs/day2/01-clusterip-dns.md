# Lab 1 · ClusterIP와 DNS

1일차에서 Pod의 IP가 재생성될 때마다 바뀌는 것을 확인했습니다. 이 휘발성 문제를 해결하는 것이 **Service** 입니다. Service는 여러 Pod 앞에 놓이는 **변하지 않는 가상 주소** 이자, 그 주소로 온 트래픽을 조건에 맞는 Pod들에게 나눠 주는 로드밸런서입니다. 이번 실습에서는 가장 기본이 되는 `ClusterIP` 타입 Service를 만들고, 클러스터 내부 DNS 이름으로 접근해 봅니다.

!!! abstract "이 실습에서 배우는 것"
    - `ClusterIP` Service를 만들어 여러 Pod에 안정적인 하나의 주소를 부여한다
    - Service의 `selector` 와 Pod의 `label` 이 어떻게 연결되는지 이해한다
    - `Endpoints` 가 실제로 트래픽이 갈 Pod IP 목록임을 확인한다
    - 클러스터 내부 DNS(CoreDNS)와 FQDN(`<svc>.<namespace>.svc.cluster.local`)으로 접근한다

## 사전 조건

- Rancher Desktop의 k3s 클러스터가 실행 중이고 `kubectl get nodes` 가 `Ready` 로 나온다
- 실습용 작업 폴더(`~/k8s-labs` 등)에서 명령을 실행한다

```bash
kubectl get nodes
```

## 1. Deployment 만들기

먼저 접속 대상이 될 웹 서버 Pod 두 개를 Deployment로 띄웁니다. 아래를 `web-deploy.yaml` 로 저장하세요.

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

적용하고 Pod가 뜨는지 확인합니다.

```bash
kubectl apply -f web-deploy.yaml
kubectl get pods -l app=web -o wide
```

```text
NAME                   READY   STATUS    RESTARTS   AGE   IP           NODE
web-6b7d8f9c4d-2xk9p   1/1     Running   0          20s   10.42.0.21   lima-rancher-desktop
web-6b7d8f9c4d-8vt4m   1/1     Running   0          20s   10.42.0.22   lima-rancher-desktop
```

두 Pod 모두 `app=web` 라벨을 달고 서로 다른 IP(`10.42.x.x`)를 받았습니다. 이 IP들은 언제든 바뀔 수 있는 값이라는 점을 기억하세요.

## 2. ClusterIP Service 만들기

이제 두 Pod 앞에 놓일 Service를 만듭니다. 아래를 `web-svc.yaml` 로 저장하세요.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f web-svc.yaml
kubectl get svc web
```

```text
NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.43.120.55    <none>        80/TCP    10s
```

`CLUSTER-IP`(`10.43.x.x`)가 바로 이 Service의 변하지 않는 주소입니다. `EXTERNAL-IP` 가 `<none>` 인 것에 주목하세요 — `ClusterIP` 타입은 **클러스터 내부에서만** 접근할 수 있습니다.

!!! info "selector가 Pod를 고르는 방식"
    Service의 `selector: app=web` 는 "라벨이 `app=web` 인 Pod에게 트래픽을 보내라"는 뜻입니다. Pod를 IP로 직접 지정하지 않기 때문에, Pod가 죽고 새 IP로 다시 떠도 라벨만 같으면 Service가 알아서 새 Pod를 대상에 포함시킵니다.

## 3. Endpoints 확인하기

Service가 실제로 어떤 Pod IP들로 트래픽을 보내는지는 `Endpoints` 에 담깁니다.

```bash
kubectl get endpoints web
```

```text
NAME   ENDPOINTS                       AGE
web    10.42.0.21:80,10.42.0.22:80     30s
```

앞서 본 두 Pod의 IP:포트가 그대로 들어 있습니다. 즉 **selector와 라벨이 매칭된 결과가 Endpoints** 이고, Service는 이 목록으로 요청을 분산합니다.

## 4. 클러스터 내부에서 접근하기

`ClusterIP` 는 외부에서 못 들어가므로, 클러스터 안에 임시 테스트 Pod를 띄워서 접근해 봅니다. 아래 명령은 `busybox` 컨테이너로 들어가 셸을 엽니다(종료 시 자동 삭제).

```bash
kubectl run tmp --image=busybox -it --rm --restart=Never -- sh
```

셸 프롬프트(`/ #`)가 뜨면 Service 이름으로 웹 서버에 요청해 봅니다.

```bash
wget -qO- http://web
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

`http://web` 만으로 접속됐다는 점이 핵심입니다. 같은 네임스페이스 안에서는 **Service 이름이 곧 호스트 이름** 이 됩니다.

## 5. DNS(FQDN)로 접근하기

같은 테스트 Pod 셸에서 DNS 조회를 해 봅니다.

```bash
nslookup web.default.svc.cluster.local
```

```text
Server:    10.43.0.10
Address:   10.43.0.10:53

Name:      web.default.svc.cluster.local
Address:   10.43.120.55
```

`web.default.svc.cluster.local` 이라는 **FQDN(전체 도메인 이름)** 이 Service의 CLUSTER-IP(`10.43.120.55`)로 해석됩니다. 규칙은 다음과 같습니다.

```text
<서비스이름>.<네임스페이스>.svc.cluster.local
   web    .   default   .svc.cluster.local
```

같은 네임스페이스에서는 `web` 만 써도 되지만, 다른 네임스페이스의 Service를 부를 때는 `web.default` 처럼 네임스페이스를 붙여야 합니다.

!!! info "이 이름을 해석해 주는 것은 누구?"
    쿠버네티스에는 **CoreDNS** 라는 클러스터 내부 DNS 서버가 기본으로 떠 있습니다(`10.43.0.10`). 모든 Pod는 자동으로 이 DNS를 바라보도록 설정되어, Service 이름을 IP로 바꿔 줍니다.

확인이 끝났으면 셸에서 나옵니다(`tmp` Pod는 자동 삭제됨).

```bash
exit
```

## 검증

아래가 모두 만족되면 성공입니다.

```bash
kubectl get svc web
kubectl get endpoints web
```

- `get svc web` 의 `CLUSTER-IP` 에 `10.43.x.x` 주소가 찍힌다
- `get endpoints web` 에 Pod 두 개의 `IP:80` 이 나열된다
- 테스트 Pod에서 `wget -qO- http://web` 로 nginx 응답이 온다

## 직접 해 보기

!!! question "도전 과제"
    1. `kubectl scale deployment web --replicas=4` 로 Pod를 4개로 늘린 뒤, `kubectl get endpoints web` 을 다시 보세요. Endpoints 목록이 4개로 늘어났나요?
    2. 다시 `--replicas=2` 로 줄이면 Endpoints도 2개로 줄어드는지 확인하세요. Service는 Pod 개수 변화를 어떻게 따라잡나요?
    3. 테스트 Pod에서 `wget -qO- http://web` 을 여러 번 실행하고, `kubectl logs -l app=web` 로 두 Pod 모두에 요청이 들어왔는지 확인해 보세요.

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **`wget: bad address 'web'`** — 테스트 Pod가 Service와 다른 네임스페이스에 있거나 Service 이름이 틀렸습니다. 같은 `default` 네임스페이스인지 확인하고, 다른 네임스페이스면 `web.<네임스페이스>` 로 부르세요.
    - **`get endpoints web` 이 비어 있음(`<none>`)** — Service의 `selector` 와 Pod의 `label` 이 안 맞는 경우입니다. 다음 Lab 2에서 이 상황을 일부러 만들어 원인을 추적합니다.
    - **`nslookup` 이 응답 없음** — CoreDNS Pod 상태를 `kubectl get pods -n kube-system` 로 확인하세요.

## 정리

다음 실습에서 이어서 쓰므로 지우지 않아도 되지만, 정리하려면 아래처럼 삭제합니다.

```bash
kubectl delete -f web-svc.yaml
kubectl delete -f web-deploy.yaml
```

---

다음: [Lab 2 · Endpoints 디버깅](02-endpoints-debug.md)
