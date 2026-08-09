# Lab 6 · 스케일링

트래픽이 늘면 Pod 개수를 늘려 부하를 나눕니다(수평 확장). 명령형(`kubectl scale`)과 선언형(YAML 수정 후 apply) 두 가지로 Pod 수를 조절하고, `kubectl top` 으로 사용량도 확인해 봅니다.

!!! abstract "이 실습에서 배우는 것"
    - `kubectl scale` 로 즉시 Pod 수 늘리기(명령형)
    - YAML 의 `replicas` 를 고쳐 apply 하기(선언형)
    - 수평 확장(scale out)의 개념
    - `kubectl top` 으로 CPU·메모리 사용량 보기

## 사전 조건

- [Lab 5](05-deployment-selfheal.md) 에서 만든 Deployment `web` 이 `READY 3/3` 로 실행 중.
- 확인: `kubectl get deploy web`

## 1. 명령형으로 스케일 아웃

`scale` 명령으로 replicas 를 5로 늘립니다.

```bash
kubectl scale deployment web --replicas=5
```

```text
deployment.apps/web scaled
```

새 창이나 같은 창에서 변화를 지켜봅니다.

```bash
kubectl get pods -w
```

```text
NAME                    READY   STATUS              RESTARTS   AGE
web-6f9c8d7b5-7hd4p     1/1     Running             0          8m
web-6f9c8d7b5-p8n9m     1/1     Running             0          8m
web-6f9c8d7b5-9wq2t     1/1     Running             0          5m
web-6f9c8d7b5-c4rtx     0/1     ContainerCreating   0          1s
web-6f9c8d7b5-k2m8v     0/1     ContainerCreating   0          1s
web-6f9c8d7b5-c4rtx     1/1     Running             0          4s
web-6f9c8d7b5-k2m8v     1/1     Running             0          4s
```

두 개가 추가돼 총 5개가 됩니다. ++ctrl+c++ 로 감시를 멈추고 확인합니다.

```bash
kubectl get deploy web
```

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    5/5     5            5           9m
```

!!! info "수평 확장(scale out)이란"
    한 Pod 를 더 크게 키우는 게 아니라(수직 확장), **같은 Pod 를 여러 개 복제**해 부하를 나눠 받는 방식입니다. 쿠버네티스는 수평 확장에 최적화돼 있고, 앞서 본 것처럼 늘린 개수도 자동으로 유지됩니다.

## 2. 스케일 인(축소)도 같은 방법

줄일 때도 똑같이 `scale` 을 씁니다. 2개로 줄여 봅니다.

```bash
kubectl scale deployment web --replicas=2
```

```text
deployment.apps/web scaled
```

```bash
kubectl get pods -l app=web
```

```text
NAME                    READY   STATUS    RESTARTS   AGE
web-6f9c8d7b5-7hd4p     1/1     Running   0          10m
web-6f9c8d7b5-p8n9m     1/1     Running   0          10m
```

초과분 3개가 종료되고 2개만 남습니다.

!!! warning "명령형 스케일은 파일과 어긋납니다"
    방금 `scale` 로 2개로 바꿨지만, `deployment.yaml` 파일에는 아직 `replicas: 3` 이라고 적혀 있습니다. 지금 이 파일을 그대로 다시 `apply` 하면 개수가 다시 3으로 **되돌아갑니다.** 파일이 곧 원하는 상태이기 때문입니다. 그래서 실무에서는 스케일도 파일로 관리하는 게 안전합니다 — 바로 다음 단계입니다.

## 3. 선언형으로 스케일링

`deployment.yaml` 을 열어 `replicas` 값을 `4` 로 수정합니다.

```yaml
spec:
  replicas: 4      # 3 → 4 로 변경
  selector:
    matchLabels:
      app: web
  ...
```

저장한 뒤 apply 합니다.

```bash
kubectl apply -f deployment.yaml
```

```text
deployment.apps/web configured
```

```bash
kubectl get deploy web
```

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    4/4     4            4           12m
```

`configured`(바뀐 부분만 반영)가 나오고 Pod 가 4개로 맞춰집니다.

!!! note "명령형 vs 선언형 스케일"
    - **명령형(`kubectl scale`)**: 지금 당장 빠르게 조절할 때. 단, 파일과 어긋날 수 있음.
    - **선언형(`replicas` 수정 후 apply)**: 변경 이력이 파일·Git 에 남아 재현·협업에 유리. 실무 기본.

## 4. 사용량 확인 (kubectl top)

이 클러스터에는 `metrics-server` 가 설치돼 있어, Pod 와 노드의 실시간 CPU·메모리 사용량을 볼 수 있습니다.

```bash
kubectl top pods
```

```text
NAME                    CPU(cores)   MEMORY(bytes)
web-6f9c8d7b5-7hd4p     0m           4Mi
web-6f9c8d7b5-p8n9m     0m           4Mi
web-6f9c8d7b5-c4rtx     0m           4Mi
web-6f9c8d7b5-k2m8v     0m           4Mi
```

노드 전체 사용량도 볼 수 있습니다.

```bash
kubectl top nodes
```

```text
NAME                   CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
lima-rancher-desktop   180m         2%     1450Mi          18%
```

- `CPU(cores)` 의 `m` 은 milli-core(1000m = 1코어). nginx 는 유휴 상태라 `0m` 에 가깝습니다.
- `MEMORY(bytes)` 는 실제 사용 메모리입니다.

!!! info "top 은 오토스케일링의 토대"
    실무에서는 `kubectl top` 이 보는 이 사용량 지표를 기준으로 **HPA(HorizontalPodAutoscaler)** 가 Pod 수를 자동으로 늘리고 줄입니다. 오늘은 수동 스케일까지만 다루지만, 자동 확장도 결국 "지표를 보고 replicas 를 조절"하는 같은 원리입니다.

!!! warning "metrics-server 가 없다면"
    다른 환경에서 `kubectl top` 이 `Metrics API not available` 이라고 나오면 metrics-server 가 없거나 준비 중인 것입니다. 이 실습 환경(Rancher Desktop/k3s)에는 기본 설치돼 있으니, 방금 켰다면 1~2분 뒤 다시 시도하세요.

## 검증

```bash
kubectl get deploy web
kubectl top pods
```

- Deployment `web` 이 `READY 4/4`.
- `kubectl top pods` 가 4개 Pod 의 사용량을 표로 보여 준다.

## 직접 해 보기

1. `kubectl scale deployment web --replicas=1` 로 1개까지 줄였다가, 파일은 `4` 인 채로 `kubectl apply -f deployment.yaml` 을 실행해 보세요. 개수가 다시 4로 돌아오는 걸 확인하고, "파일이 원하는 상태"라는 의미를 되새겨 봅니다.
2. `kubectl top pods --sort-by=memory` 로 메모리 사용량 기준 정렬을 해 보세요.

!!! failure "자주 만나는 오류"
    **증상**: `kubectl top pods` → `error: Metrics API not available`
    **원인**: metrics-server 가 아직 준비되지 않았거나 재시작 중.
    **해결**: `kubectl get pods -n kube-system | grep metrics-server` 로 `Running` 확인 후, 1~2분 뒤 재시도.

    ---

    **증상**: `scale` 후에도 개수가 안 늘어남
    **원인**: 대상 이름 오타 또는 Deployment 가 아님(예: Pod 를 지정).
    **해결**: `kubectl get deploy` 로 정확한 Deployment 이름 확인 후 재실행.

    ---

    **증상**: apply 했더니 스케일이 되돌아감
    **원인**: 이건 오류가 아니라 정상 동작입니다. 파일의 `replicas` 값이 우선합니다.
    **해결**: 유지하려는 개수를 파일에 반영한 뒤 apply.

## 정리

다음 [Lab 7](07-rolling-update.md) 에서 이 Deployment 를 계속 사용하므로 **지금은 삭제하지 마세요.**

---

다음: [Lab 7 · 롤링 업데이트와 롤백](07-rolling-update.md)
