# Lab 6 · Deployment와 자가복구

Pod 를 직접 만들면 죽어도 아무도 되살리지 않습니다. **Deployment** 는 "항상 N개의 Pod 가 떠 있어야 한다"고 선언해 두면, 하나가 사라져도 알아서 다시 만들어 줍니다. 이 **자가복구(self-healing)** 를 직접 재현해 봅니다.

!!! abstract "이 실습에서 배우는 것"
    - Deployment 매니페스트 작성과 `replicas`, `selector`, `template` 의 의미
    - Deployment → ReplicaSet → Pod 의 3계층 소유 관계
    - `ownerReferences` 로 누가 누구를 만들었는지 확인
    - Pod 를 지워도 자동 재생성되는 자가복구 관찰

## 사전 조건

- [Lab 5](05-yaml-manifest.md) 를 마치고 `default` 에 `web` Pod 가 없는 상태(`kubectl get pods` → `No resources found`).

## 1. Deployment 매니페스트 작성

아래 내용을 **`deployment.yaml`** 로 저장하세요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx
          ports:
            - containerPort: 80
```

세 개의 핵심 필드를 이해하고 넘어갑시다.

| 필드 | 의미 |
|---|---|
| `replicas: 3` | Pod 를 항상 **3개** 유지하라 |
| `selector.matchLabels` | 이 Deployment 가 관리할 Pod 를 고르는 라벨 조건 |
| `template` | 만들어 낼 Pod 의 설계도(그 자체가 하나의 Pod 정의) |

!!! warning "selector 와 template 라벨은 일치해야 합니다"
    `spec.selector.matchLabels` (`app: web`) 와 `spec.template.metadata.labels` (`app: web`) 가 서로 맞아야 합니다. 다르면 Deployment 가 자기가 만든 Pod 를 관리 대상으로 인식하지 못해 apply 시 오류가 납니다.

## 2. 적용하고 계층 확인하기

```bash
kubectl apply -f deployment.yaml
```

```text
deployment.apps/web created
```

Deployment, ReplicaSet, Pod 를 한 번에 조회합니다.

```bash
kubectl get deploy,rs,pods
```

```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web   3/3     3            3           20s

NAME                            DESIRED   CURRENT   READY   AGE
replicaset.apps/web-6f9c8d7b5   3         3         3       20s

NAME                        READY   STATUS    RESTARTS   AGE
pod/web-6f9c8d7b5-2xkqr     1/1     Running   0          20s
pod/web-6f9c8d7b5-7hd4p     1/1     Running   0          20s
pod/web-6f9c8d7b5-p8n9m     1/1     Running   0          20s
```

한 번의 apply 로 3계층이 만들어졌습니다.

- **Deployment `web`** — 내가 선언한 리소스. 업데이트·롤백 전략을 담당.
- **ReplicaSet `web-6f9c8d7b5`** — Deployment 가 만든 하위 리소스. "Pod 개수 유지"를 담당.
- **Pod `web-...-xxxxx`** — ReplicaSet 이 만든 실제 워크로드. 이름 뒤에 랜덤 접미사가 붙습니다.

```text
Deployment (web)
└── ReplicaSet (web-6f9c8d7b5)
    ├── Pod (web-6f9c8d7b5-2xkqr)
    ├── Pod (web-6f9c8d7b5-7hd4p)
    └── Pod (web-6f9c8d7b5-p8n9m)
```

!!! info "왜 ReplicaSet 이 중간에 끼나요?"
    개수 유지는 ReplicaSet 이, 버전 전환(롤링 업데이트·롤백)은 Deployment 가 담당하도록 역할을 나눈 구조입니다. 새 버전을 배포하면 Deployment 가 **새 ReplicaSet** 을 하나 더 만들어 트래픽을 옮깁니다. 이 장면은 [Lab 8](08-rolling-update.md) 에서 봅니다.

## 3. 소유 관계 확인 (ownerReferences)

"이 Pod 를 누가 만들었나?"는 Pod 의 `ownerReferences` 에 적혀 있습니다. Pod 이름 하나를 골라 확인합니다. (아래 이름은 여러분 환경 값으로 바꾸세요.)

```bash
kubectl get pod web-6f9c8d7b5-2xkqr -o yaml
```

```text
metadata:
  name: web-6f9c8d7b5-2xkqr
  ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: web-6f9c8d7b5
      controller: true
      uid: ...
```

Pod 의 주인은 Deployment 가 아니라 **ReplicaSet** 입니다. 마찬가지로 ReplicaSet 의 주인은 Deployment 입니다.

!!! tip "이름만 빠르게 뽑기"
    Pod 이름 복사가 귀찮다면 `kubectl get pods -o name` 으로 이름 목록만 얻거나, `kubectl get pods -l app=web` 처럼 라벨로 필터링하세요.

## 4. 자가복구 재현하기

이제 핵심입니다. Pod 하나를 강제로 삭제하고 무슨 일이 일어나는지 지켜봅니다. 먼저 감시 모드를 켭니다.

```bash
kubectl get pods -w
```

이 창은 켜 둔 채로 **새 터미널**에서 Pod 하나를 삭제합니다. (이름은 여러분 값으로)

```bash
kubectl delete pod web-6f9c8d7b5-2xkqr
```

`-w` 창을 보면, 지운 Pod 가 사라지자마자 **새 Pod 가 곧바로 생성**됩니다.

```text
NAME                    READY   STATUS        RESTARTS   AGE
web-6f9c8d7b5-2xkqr     1/1     Terminating   0          3m
web-6f9c8d7b5-9wq2t     0/1     Pending       0          0s
web-6f9c8d7b5-9wq2t     0/1     ContainerCreating   0    0s
web-6f9c8d7b5-9wq2t     1/1     Running       0          3s
```

개수를 다시 세어 보면 여전히 3개입니다.

```bash
kubectl get pods
```

```text
NAME                    READY   STATUS    RESTARTS   AGE
web-6f9c8d7b5-7hd4p     1/1     Running   0          4m
web-6f9c8d7b5-p8n9m     1/1     Running   0          4m
web-6f9c8d7b5-9wq2t     1/1     Running   0          30s
```

!!! note "이게 바로 자가복구입니다"
    ReplicaSet 은 "실제 Pod 수 = 원하는 수(3)"를 끊임없이 감시합니다. 하나가 사라져 2개가 되는 순간, 즉시 새 Pod 를 만들어 3개로 되돌립니다. [Lab 4](04-pod-ip-ephemeral.md) 에서 명령형 Pod 는 지우면 끝이었던 것과 정반대죠. **개별 Pod 는 소모품, 원하는 상태는 유지된다** — 이것이 쿠버네티스를 쓰는 가장 큰 이유입니다.

++ctrl+c++ 로 `-w` 감시를 종료합니다.

## 검증

```bash
kubectl get deploy web
kubectl get pods -l app=web
```

- Deployment `web` 이 `READY 3/3`.
- `app=web` 라벨을 가진 Pod 가 3개 `Running`(그중 하나는 방금 재생성돼 AGE 가 짧다).

## 직접 해 보기

1. Pod 를 **두 개** 동시에 삭제해 보세요(`kubectl delete pod <A> <B>`). 그래도 금세 3개로 복구되는지 확인합니다.
2. Deployment 는 그대로 두고 하위 **ReplicaSet 을 직접 삭제**해 보세요(`kubectl delete rs <이름>`). Deployment 가 ReplicaSet 마저 다시 만들어 내는지 관찰합니다. (계층 위로 갈수록 더 끈질기게 복구됩니다.)

!!! failure "자주 만나는 오류"
    **증상**: apply 시 `selector does not match template labels`
    **원인**: `selector.matchLabels` 와 `template.metadata.labels` 가 다름.
    **해결**: 두 라벨을 `app: web` 으로 동일하게 맞춘다.

    ---

    **증상**: Pod 를 지웠는데 다시 안 생김
    **원인**: 사실은 Deployment 가 아니라 명령형 Pod 였거나, Deployment 가 삭제된 상태.
    **해결**: `kubectl get deploy` 로 Deployment 존재를 확인. 없으면 `kubectl apply -f deployment.yaml` 다시 실행.

    ---

    **증상**: `get pods` 에 Pod 가 6개처럼 너무 많이 보임
    **원인**: 이전 실습의 Pod 나 다른 Deployment 가 함께 남아 있음.
    **해결**: `kubectl get pods -l app=web` 로 이 Deployment 소속만 필터링해서 확인.

## 정리

다음 [Lab 7](07-scaling.md), [Lab 8](08-rolling-update.md) 에서 이 Deployment 를 계속 사용하므로 **지금은 삭제하지 마세요.** (전부 끝내고 정리하고 싶다면 `kubectl delete -f deployment.yaml`.)

---

다음: [Lab 7 · 스케일링](07-scaling.md)
