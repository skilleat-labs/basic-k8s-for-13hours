# Lab 3 · Pod IP는 휘발성

Pod 를 지웠다가 다시 만들면 IP 가 바뀝니다. 직접 재현해 보고, 왜 Pod IP 를 절대 하드코딩하면 안 되는지 — 그래서 왜 **Service** 가 필요한지 — 를 이해합니다.

!!! abstract "이 실습에서 배우는 것"
    - Pod 를 삭제·재생성하면 IP 가 달라짐을 직접 확인
    - Pod 는 언제든 사라지고 다시 만들어지는 "일회용" 자원이라는 개념
    - Pod IP 하드코딩이 위험한 이유
    - 다음 강의 주제인 Service 의 필요성

## 사전 조건

- [Lab 2](02-first-pod.md) 에서 만든 `web` Pod 가 `Running` 상태.
- 없다면 먼저 실행합니다.

```bash
kubectl run web --image=nginx
```

## 1. 현재 IP 기록하기

지금 `web` Pod 의 IP 를 확인하고 어딘가에 적어 둡니다.

```bash
kubectl get pod web -o wide
```

```text
NAME   READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
web    1/1     Running   0          5m    10.42.0.24   lima-rancher-desktop   <none>           <none>
```

!!! note "메모"
    이 예시의 IP 는 `10.42.0.24` 입니다. 여러분의 값은 다를 수 있으니, 화면에 나온 실제 값을 적어 두세요.

## 2. Pod 삭제하기

Pod 를 지웁니다.

```bash
kubectl delete pod web
```

```text
pod "web" deleted
```

```bash
kubectl get pods
```

```text
No resources found in default namespace.
```

!!! warning "명령형 Pod 는 되살아나지 않습니다"
    `kubectl run` 으로 직접 만든 Pod 는 삭제하면 그대로 끝입니다. 아무도 다시 만들어 주지 않습니다. (이걸 자동으로 되살려 주는 게 [Lab 5](05-deployment-selfheal.md) 의 Deployment 입니다.)

## 3. 다시 만들기

똑같은 이름·이미지로 Pod 를 다시 띄웁니다.

```bash
kubectl run web --image=nginx
```

```text
pod/web created
```

## 4. IP 가 바뀌었다

새로 뜬 Pod 의 IP 를 확인합니다.

```bash
kubectl get pod web -o wide
```

```text
NAME   READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
web    1/1     Running   0          10s   10.42.0.31   lima-rancher-desktop   <none>           <none>
```

이름은 그대로 `web` 이지만 **IP 가 `10.42.0.24` → `10.42.0.31` 로 바뀌었습니다.**

!!! info "왜 바뀌나요?"
    쿠버네티스에서 Pod 는 "언제든 사라지고 새로 만들어지는" 소모품입니다. 삭제, 노드 재시작, 스케일링, 롤링 업데이트 — 이 모든 순간에 Pod 는 새로 생성되고, 그때마다 Pod 네트워크에서 새 IP 를 할당받습니다. **Pod 의 IP 는 그 Pod 의 수명 동안만 유효한 임시 값**입니다.

## 5. 그래서 하드코딩하면 안 된다

만약 프론트엔드 앱의 설정에 백엔드 Pod 의 IP 를 `10.42.0.24` 라고 박아 두었다고 생각해 봅시다.

- 백엔드 Pod 가 한 번만 재시작돼도 IP 가 바뀌어 연결이 끊깁니다.
- 스케일 아웃으로 백엔드가 3개가 되면, 그중 어느 IP 를 써야 할까요?
- 롤링 업데이트 중에는 옛 Pod 와 새 Pod 가 섞여 IP 가 계속 변합니다.

즉, **Pod IP 로는 안정적인 통신 대상을 지정할 수 없습니다.**

??? question "그럼 무엇으로 지정하나요?"
    바뀌지 않는 **이름(주소)** 이 필요합니다. 쿠버네티스는 이를 위해 **Service** 라는 리소스를 제공합니다. Service 는 고정된 가상 IP(ClusterIP)와 DNS 이름을 가지며, 뒤에 있는 Pod 들이 바뀌어도 알아서 살아있는 Pod 로 트래픽을 넘겨 줍니다. 앱은 Pod IP 가 아니라 Service 이름(예: `backend`)만 알면 됩니다.

!!! note "다음 강의 예고 · Service"
    2일차 첫 실습에서 바로 이 **Service** 를 만들어, "IP 가 바뀌어도 끊기지 않는 통신"을 직접 구현합니다. 오늘은 "왜 그게 필요한가"를 손으로 겪은 것으로 충분합니다.

## 검증

```bash
kubectl get pod web -o wide
```

- 3번에서 다시 만든 `web` Pod 가 `Running` 이고, 1번에 기록해 둔 IP 와 **다른** IP 를 갖고 있다.

## 직접 해 보기

1. 위 삭제 → 재생성을 **한 번 더** 반복하며 IP 를 기록해 보세요. 매번 값이 달라지는지 확인합니다.
2. `kubectl get pod web -o jsonpath='{.status.podIP}'` 를 실행해 IP 만 딱 뽑아내 보세요. (스크립트에서 값을 추출할 때 쓰는 방식입니다.)

!!! failure "자주 만나는 오류"
    **증상**: 삭제 직후 재생성 시 `pods "web" already exists`
    **원인**: 이전 Pod 가 아직 완전히 종료(Terminating)되지 않음.
    **해결**: `kubectl get pods` 로 목록이 비워질 때까지 잠깐 기다린 뒤 다시 `run`.

    ---

    **증상**: 재생성했는데 IP 가 이전과 같아 보임
    **원인**: 우연히 같은 IP 가 재할당됐거나, 예전 `get` 출력을 보고 있음.
    **해결**: 대체로 다르게 나오지만, 같더라도 정상입니다. IP 는 "보장되지 않는다"는 점이 핵심입니다. 몇 번 더 반복하면 달라지는 걸 보게 됩니다.

## 정리

다음 실습([Lab 4](04-yaml-manifest.md))에서 같은 `web` 을 YAML 로 다시 만들 것이므로, 지금 명령형으로 만든 Pod 는 삭제하고 넘어갑니다.

```bash
kubectl delete pod web
```

---

다음: [Lab 4 · YAML 매니페스트](04-yaml-manifest.md)
