# Lab 2 · Endpoints 디버깅

실무에서 가장 흔한 Service 장애는 "Service를 만들었는데 접속이 안 된다"입니다. 원인의 대부분은 **selector와 label 불일치** 입니다. Service는 라벨이 맞는 Pod를 못 찾으면 조용히 빈 Endpoints를 갖게 되고, 트래픽은 갈 곳이 없어집니다. 이번 실습에서는 라벨 오타로 장애를 **일부러 재현** 한 뒤, `describe` 와 `get endpoints` 로 원인을 추적해 고칩니다.

!!! abstract "이 실습에서 배우는 것"
    - selector와 label이 안 맞으면 Endpoints가 비고 트래픽이 끊긴다는 것을 체감한다
    - `kubectl describe svc` / `get endpoints` / `get pods --show-labels` 로 장애를 추적한다
    - "접속 안 됨 → 제일 먼저 Endpoints를 보라"는 디버깅 순서를 익힌다

## 사전 조건

- Rancher Desktop의 k3s 클러스터가 실행 중이다
- Lab 1의 `web` Deployment가 없어도 됩니다(이 실습에서 새로 만듭니다)

## 1. 정상 Deployment 준비

먼저 정상 동작하는 Pod들을 띄웁니다. 아래를 `web-deploy.yaml` 로 저장하세요(Lab 1과 동일).

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
kubectl get pods --show-labels
```

```text
NAME                   READY   STATUS    RESTARTS   AGE   LABELS
web-6b7d8f9c4d-2xk9p   1/1     Running   0          15s   app=web,pod-template-hash=6b7d8f9c4d
web-6b7d8f9c4d-8vt4m   1/1     Running   0          15s   app=web,pod-template-hash=6b7d8f9c4d
```

Pod들의 실제 라벨이 `app=web` 임을 확인해 두세요. 이것이 나중에 비교 기준이 됩니다.

## 2. 오타가 있는 Service 만들기(장애 재현)

이번엔 일부러 selector에 오타를 낸 Service를 만듭니다. `app=web` 이 아니라 `app=weeb` 로 적었습니다. 아래를 `web-svc-bad.yaml` 로 저장하세요.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: weeb        # 오타! 실제 Pod 라벨은 app=web
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f web-svc-bad.yaml
kubectl get svc web
```

```text
NAME   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.43.88.10    <none>        80/TCP    5s
```

Service 자체는 아무 오류 없이 잘 만들어집니다. CLUSTER-IP도 정상적으로 할당됐습니다. **여기서 "만들어졌으니 되겠지"라고 넘어가면 장애를 놓칩니다.**

## 3. 접속이 안 되는 것 확인

테스트 Pod로 접속을 시도해 봅니다.

```bash
kubectl run tmp --image=busybox -it --rm --restart=Never -- sh
```

셸이 뜨면(`/ #`):

```bash
wget -qO- --timeout=5 http://web
```

```text
wget: download timed out
```

CLUSTER-IP는 있는데 응답이 없습니다. 셸에서 나옵니다.

```bash
exit
```

## 4. Endpoints로 원인 추적

접속이 안 될 때 **가장 먼저 볼 곳은 Endpoints** 입니다.

```bash
kubectl get endpoints web
```

```text
NAME   ENDPOINTS   AGE
web    <none>      1m
```

Endpoints가 `<none>` — 트래픽을 보낼 Pod가 하나도 없습니다. `describe` 로 더 자세히 봅니다.

```bash
kubectl describe svc web
```

```text
Name:              web
Namespace:         default
Selector:          app=weeb
Type:              ClusterIP
IP:                10.43.88.10
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         <none>
```

두 가지 단서가 보입니다. **`Selector: app=weeb`** 와 **`Endpoints: <none>`**. Service는 `app=weeb` 인 Pod를 찾고 있는데, 그런 Pod가 없어서 대상이 비어 있는 것입니다.

## 5. 실제 Pod 라벨과 대조

Service가 찾는 라벨과 Pod의 실제 라벨을 나란히 비교합니다.

```bash
kubectl get pods --show-labels
```

```text
NAME                   READY   STATUS    RESTARTS   AGE   LABELS
web-6b7d8f9c4d-2xk9p   1/1     Running   0          3m    app=web,pod-template-hash=6b7d8f9c4d
web-6b7d8f9c4d-8vt4m   1/1     Running   0          3m    app=web,pod-template-hash=6b7d8f9c4d
```

Pod 라벨은 `app=web`, Service selector는 `app=weeb`. **한 글자 차이로 매칭이 안 되고 있었습니다.** selector로 직접 Pod를 골라 보면 더 확실합니다.

```bash
kubectl get pods -l app=weeb
```

```text
No resources found in default namespace.
```

```bash
kubectl get pods -l app=web
```

```text
NAME                   READY   STATUS    RESTARTS   AGE
web-6b7d8f9c4d-2xk9p   1/1     Running   0          3m
web-6b7d8f9c4d-8vt4m   1/1     Running   0          3m
```

## 6. selector 수정

selector를 올바른 `app=web` 으로 고칩니다. 아래를 `web-svc.yaml` 로 저장하세요.

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
kubectl get endpoints web
```

```text
NAME   ENDPOINTS                      AGE
web    10.42.0.21:80,10.42.0.22:80    4m
```

Endpoints가 채워졌습니다. 이제 테스트 Pod에서 다시 접속하면 정상 응답이 옵니다.

```bash
kubectl run tmp --image=busybox -it --rm --restart=Never -- sh -c "wget -qO- http://web | head -5"
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
</head>
```

## 검증

- 수정 전 `kubectl get endpoints web` 이 `<none>` 이었다
- 수정 후 `kubectl get endpoints web` 에 Pod IP 두 개가 채워졌다
- 테스트 Pod에서 nginx 응답을 받는다

!!! tip "이 실습의 교훈"
    **라벨 불일치 = 트래픽 안 감.** Service는 selector로 Pod를 찾으므로, 접속이 안 될 때 디버깅 순서는 항상 이렇습니다.

    1. `kubectl get endpoints <svc>` — 비어 있는가?
    2. 비어 있으면 `kubectl describe svc <svc>` — selector가 무엇인가?
    3. `kubectl get pods --show-labels` — Pod의 실제 라벨과 대조

## 직접 해 보기

!!! question "도전 과제"
    1. 이번엔 Pod 쪽 라벨을 바꿔 장애를 내 보세요. `kubectl label pod <파드이름> app=broken --overwrite` 후 `get endpoints web` 이 어떻게 변하는지 보세요(Deployment가 곧 새 Pod를 다시 만들어 냅니다).
    2. `targetPort` 를 `8080` 처럼 nginx가 안 듣는 포트로 바꾸면, Endpoints는 채워지는데 접속만 안 됩니다. selector 문제와 어떻게 다른지 관찰해 보세요.

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **selector는 맞는데 Endpoints가 여전히 빔** — Pod가 아직 `Running`/`Ready` 상태가 아닐 수 있습니다. `kubectl get pods` 로 상태를 확인하세요. `Ready` 가 아닌 Pod는 Endpoints에서 제외됩니다.
    - **Endpoints는 채워졌는데 접속 안 됨** — 이건 selector가 아니라 `targetPort`(컨테이너가 실제 듣는 포트) 불일치일 가능성이 큽니다.
    - **`apply` 했는데 selector가 안 바뀜** — 이전 `web-svc-bad.yaml` 을 다시 apply하고 있지 않은지 파일 이름을 확인하세요.

## 정리

```bash
kubectl delete svc web
kubectl delete -f web-deploy.yaml
```

---

다음: [Lab 3 · NodePort와 port-forward](03-nodeport-portforward.md)
