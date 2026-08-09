# Lab 7 · 롤링 업데이트와 롤백

서비스를 멈추지 않고 새 버전으로 교체하는 **롤링 업데이트**, 그리고 문제가 생겼을 때 이전 버전으로 되돌리는 **롤백**을 직접 해 봅니다. 일부러 잘못된 이미지로 배포를 실패시켜 보고, 한 줄로 복구까지 합니다.

!!! abstract "이 실습에서 배우는 것"
    - `kubectl set image` 로 이미지 버전 교체(롤링 업데이트)
    - `rollout status / history` 로 진행·이력 확인
    - 업데이트 시 **새 ReplicaSet** 이 생기는 이유
    - 배포 실패 재현 후 `rollout undo` 로 롤백
    - RollingUpdate 와 Recreate 전략의 차이

## 사전 조건

- [Lab 6](06-scaling.md) 까지 마쳐 Deployment `web` 이 실행 중(replicas 4개 예상).
- 확인: `kubectl get deploy web`

## 1. 현재 이미지 확인

지금 어떤 이미지로 떠 있는지 봅니다.

```bash
kubectl get deploy web -o jsonpath='{.spec.template.spec.containers[0].image}'
```

```text
nginx
```

`nginx`(= 최신 latest)입니다. 이제 특정 버전으로 교체해 보겠습니다.

## 2. 롤링 업데이트 실행

이미지를 `nginx:1.27` 로 바꿉니다.

```bash
kubectl set image deployment/web web=nginx:1.27
```

```text
deployment.apps/web image updated
```

!!! note "`web=nginx:1.27` 의 의미"
    `컨테이너이름=새이미지` 형식입니다. 우리 매니페스트의 컨테이너 이름이 `web` 이라 `web=nginx:1.27` 이 됩니다. 컨테이너 이름은 `kubectl describe deploy web` 의 `Containers` 에서 확인할 수 있습니다.

업데이트 진행 상황을 실시간으로 봅니다.

```bash
kubectl rollout status deployment/web
```

```text
Waiting for deployment "web" rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for deployment "web" rollout to finish: 3 out of 4 new replicas have been updated...
deployment "web" successfully rolled out
```

!!! info "무중단 교체의 원리"
    쿠버네티스는 옛 Pod 를 한꺼번에 죽이지 않습니다. 새 Pod 를 몇 개 띄워 준비되면, 옛 Pod 를 그만큼 줄이는 식으로 **조금씩 교체**합니다. 그래서 교체 중에도 요청을 처리할 Pod 가 항상 남아 있습니다.

## 3. 새 ReplicaSet 확인

ReplicaSet 목록을 보면, 업데이트로 **새 ReplicaSet 이 하나 더 생긴** 것을 볼 수 있습니다.

```bash
kubectl get rs
```

```text
NAME              DESIRED   CURRENT   READY   AGE
web-6f9c8d7b5     0         0         0       20m
web-7c4d9f8a6     4         4         4       2m
```

- 옛 ReplicaSet(`web-6f9c8d7b5`)은 Pod 수가 `0` 으로 비워졌습니다.
- 새 ReplicaSet(`web-7c4d9f8a6`)이 4개를 담당합니다.

!!! tip "옛 ReplicaSet 을 왜 안 지우나요?"
    비워진 옛 ReplicaSet 은 **롤백을 위해 남겨 둡니다.** 문제가 생기면 이 ReplicaSet 을 다시 4개로 채워 즉시 예전 버전으로 되돌릴 수 있습니다. 다음 단계에서 바로 써먹습니다.

## 4. 배포 이력 보기

지금까지의 배포 이력을 확인합니다.

```bash
kubectl rollout history deployment/web
```

```text
deployment.apps/web
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

- REVISION 1: 처음의 `nginx`
- REVISION 2: 방금의 `nginx:1.27`

## 5. 일부러 실패시키기

이번엔 **존재하지 않는 이미지 태그**로 업데이트해 배포 실패를 재현합니다.

```bash
kubectl set image deployment/web web=nginx:no-such-tag
```

```text
deployment.apps/web image updated
```

```bash
kubectl rollout status deployment/web
```

```text
Waiting for deployment "web" rollout to finish: 1 out of 4 new replicas have been updated...
```

한참 기다려도 끝나지 않습니다. Pod 상태를 보면 원인이 드러납니다. ++ctrl+c++ 로 status 를 멈추고 확인합니다.

```bash
kubectl get pods -l app=web
```

```text
NAME              READY   STATUS             RESTARTS   AGE
web-7c4d9f8a6-aa11  1/1   Running            0          5m
web-7c4d9f8a6-bb22  1/1   Running            0          5m
web-7c4d9f8a6-cc33  1/1   Running            0          5m
web-5b8e6c7d9-dd44  0/1   ImagePullBackOff   0          40s
```

새 버전 Pod 하나가 `ImagePullBackOff`(이미지를 못 받음) 상태로 멈춰 있습니다.

!!! info "실패해도 서비스는 살아 있다"
    잘 보면 옛 버전(`nginx:1.27`) Pod 3개는 여전히 `Running` 입니다. 새 Pod 가 준비되기 전까지 옛 Pod 를 죽이지 않기 때문에, **잘못된 배포를 해도 서비스 전체가 죽지는 않습니다.** 이것이 롤링 업데이트의 안전장치입니다.

원인을 `describe` 로 한 번 더 확인해 봅니다.

```bash
kubectl describe pod -l app=web | grep -A3 Events
```

```text
Events:
  Type     Reason     Age   From     Message
  ----     ------     ----  ----     -------
  Warning  Failed     30s   kubelet  Failed to pull image "nginx:no-such-tag": not found
```

## 6. 롤백하기

한 줄로 직전 버전으로 되돌립니다.

```bash
kubectl rollout undo deployment/web
```

```text
deployment.apps/web rolled back
```

```bash
kubectl rollout status deployment/web
```

```text
deployment "web" successfully rolled out
```

이미지가 정상 버전으로 돌아왔는지 확인합니다.

```bash
kubectl get deploy web -o jsonpath='{.spec.template.spec.containers[0].image}'
```

```text
nginx:1.27
```

`ImagePullBackOff` 이던 Pod 는 사라지고, 다시 4개가 모두 `Running` 입니다.

```bash
kubectl get pods -l app=web
```

```text
NAME                READY   STATUS    RESTARTS   AGE
web-7c4d9f8a6-aa11  1/1     Running   0          8m
web-7c4d9f8a6-bb22  1/1     Running   0          8m
web-7c4d9f8a6-cc33  1/1     Running   0          8m
web-7c4d9f8a6-ee55  1/1     Running   0          30s
```

!!! tip "특정 리비전으로 롤백"
    바로 직전이 아니라 특정 버전으로 되돌리려면 `kubectl rollout undo deployment/web --to-revision=1` 처럼 리비전 번호를 지정합니다. 번호는 `rollout history` 에서 확인합니다.

## 배포 전략: RollingUpdate vs Recreate

Deployment 의 `spec.strategy` 로 교체 방식을 정합니다. 지정하지 않으면 기본은 `RollingUpdate` 입니다.

| 전략 | 동작 | 중단 여부 |
|---|---|---|
| **RollingUpdate**(기본) | 새 Pod 를 조금씩 늘리며 옛 Pod 를 줄임 | 무중단 |
| **Recreate** | 옛 Pod 를 전부 죽인 뒤 새 Pod 를 만듦 | 잠깐 중단 발생 |

RollingUpdate 의 속도는 두 값으로 조절합니다.

- `maxSurge`: 원하는 개수보다 **몇 개까지 더** 띄울 수 있는가(교체 속도↑).
- `maxUnavailable`: 교체 중 **몇 개까지 부족해도** 되는가.

예시(YAML 에 추가할 경우):

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

!!! note "언제 Recreate 를 쓰나"
    옛 버전과 새 버전이 동시에 떠 있으면 안 되는 경우(예: DB 스키마 호환성 문제)에는 `Recreate` 를 씁니다. 잠깐의 다운타임을 감수하고 버전이 섞이는 것을 막는 선택입니다. 웹 서버 같은 무상태 앱은 기본값 RollingUpdate 가 적합합니다.

## 검증

```bash
kubectl rollout history deployment/web
kubectl get deploy web
```

- 이미지가 `nginx:1.27` 로 정상 복구됐고 `READY 4/4`.
- `rollout history` 에 여러 리비전이 기록돼 있다.

## 직접 해 보기

1. `kubectl set image deployment/web web=nginx:1.26` 으로 한 번 더 정상 업데이트한 뒤, `kubectl rollout history deployment/web` 로 리비전이 늘어나는 걸 확인하세요.
2. `kubectl rollout undo deployment/web --to-revision=1` 로 아주 처음 버전(`nginx`)까지 되돌려 보고, 이미지가 바뀌는지 확인하세요.

!!! failure "자주 만나는 오류"
    **증상**: `set image` 후 `error: unable to find container named "..."`
    **원인**: 지정한 컨테이너 이름이 매니페스트와 다름.
    **해결**: `kubectl describe deploy web` 의 `Containers` 에서 실제 이름(`web`)을 확인 후 `web=...` 형식으로 재실행.

    ---

    **증상**: `rollout status` 가 끝나지 않고 계속 대기
    **원인**: 새 이미지 Pull 실패(`ImagePullBackOff`) 등으로 새 Pod 가 Ready 되지 못함.
    **해결**: `kubectl get pods -l app=web` 로 상태 확인 → `kubectl rollout undo deployment/web` 로 롤백.

    ---

    **증상**: `rollout undo` 했는데 되돌아갈 이력이 없음
    **원인**: 아직 업데이트를 한 번도 하지 않았거나 리비전이 하나뿐.
    **해결**: `kubectl rollout history deployment/web` 로 리비전이 2개 이상 있는지 먼저 확인.

## 정리

1일차 실습이 모두 끝났습니다. 만든 리소스를 정리합니다.

```bash
kubectl delete deployment web
```

```text
deployment.apps "web" deleted
```

```bash
kubectl get all
```

```text
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   12d
```

`web` 관련 리소스가 모두 사라지고, 기본 `kubernetes` Service 만 남으면 깨끗이 정리된 것입니다.

!!! note "수고하셨습니다"
    오늘 Pod 를 만들고, IP 가 휘발성임을 겪고, Deployment 로 자가복구·스케일링·무중단 배포까지 손으로 해 봤습니다. 그런데 [Lab 3](03-pod-ip-ephemeral.md) 에서 남은 숙제가 있죠 — "IP 가 계속 바뀌는데 어떻게 안정적으로 접속하지?" **2일차에서 이어집니다.** Service 로 그 문제를 해결하고, 클러스터 안팎의 네트워킹을 다룹니다.
