# Lab 6 · 토큰 마운트 차단

쿠버네티스는 기본적으로 모든 Pod 안에 **ServiceAccount 토큰**을 자동으로 넣어 줍니다. 이 토큰은 API 서버에 말을 걸 수 있는 자격 증명입니다. 문제는 대부분의 앱이 이 토큰을 전혀 쓰지 않는데도 컨테이너 안에 그대로 놓여 있다는 점입니다. 공격자가 컨테이너에 침입하면 이 토큰을 주워 클러스터를 정찰하는 발판으로 삼습니다. 이번 실습에서는 토큰이 자동으로 마운트되는 것을 확인하고(공격 재현), `automountServiceAccountToken: false` 로 그 통로를 막습니다(방어 적용).

!!! abstract "이 실습에서 배우는 것"
    - 모든 Pod 에 SA 토큰이 자동 마운트되는 위치와 위험
    - 탈취된 토큰으로 공격자가 무엇을 할 수 있는지(정찰)
    - `automountServiceAccountToken: false` 로 토큰 마운트를 차단하는 법
    - RBAC(Lab 5)와 결합해 방어를 강화하는 원리

## 사전 조건

- Lab 5(RBAC)을 먼저 이해하면 이 실습의 방어 원리가 더 잘 이해됩니다.
- 클러스터가 정상인지 확인합니다.

```bash
kubectl get nodes
```

## 1. 공격 재현 · 자동 마운트된 토큰 찾기

!!! danger "공격 재현 · 컨테이너 안에 자격 증명이 놓여 있다"
    아무 설정 없는 평범한 Pod 를 배포하고, 그 안에 토큰이 저절로 들어와 있는지 확인합니다.

`default-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-pod
spec:
  containers:
    - name: app
      image: curlimages/curl
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f default-pod.yaml
kubectl wait --for=condition=Ready pod/default-pod --timeout=60s
```

```text
pod/default-pod created
pod/default-pod condition met
```

토큰이 마운트되는 표준 경로를 들여다봅니다.

```bash
kubectl exec default-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

```text
ca.crt
namespace
token
```

!!! danger "요청하지 않은 자격 증명이 이미 들어와 있습니다"
    우리는 토큰을 넣어 달라고 한 적이 없는데도 `token` 파일이 컨테이너 안에 존재합니다. `ca.crt`(API 서버 인증서)와 `namespace` 까지 세트로 들어 있어, 이 컨테이너에 침입한 공격자는 별도 준비 없이 곧바로 API 서버와 통신할 재료를 손에 쥡니다.

토큰 내용의 앞부분만 확인해 봅니다(실제 값은 JWT 문자열입니다).

```bash
kubectl exec default-pod -- head -c 40 /var/run/secrets/kubernetes.io/serviceaccount/token
```

```text
eyJhbGciOiJSUzI1NiIsImtpZCI6Ii0tLS0t...
```

## 2. 공격 재현 · 토큰으로 정찰하기

공격자가 이 토큰으로 무엇을 할 수 있는지 재현합니다. 컨테이너 안에서 그 토큰을 헤더에 실어 API 서버에 직접 질의해 봅니다.

```bash
kubectl exec default-pod -- sh -c 'TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token); curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H "Authorization: Bearer $TOKEN" https://kubernetes.default.svc/api'
```

```text
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddress...": "..."
}
```

!!! danger "토큰이 실제로 통합니다"
    컨테이너 안에서 아무런 외부 도구 없이 API 서버로부터 응답을 받았습니다. 이 `default` ServiceAccount 자체의 기본 권한은 낮지만, 공격자는 이 발판으로 클러스터를 정찰합니다. 밖에서보다 훨씬 편하게 API 를 두드릴 수 있는 상태입니다.

`default` ServiceAccount 가 어디까지 할 수 있는지 정찰하는 관점에서 확인해 봅니다(Lab 5 의 `auth can-i` 활용).

```bash
kubectl auth can-i --list --as=system:serviceaccount:default:default
```

```text
Resources                                       Non-Resource URLs   Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                  []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                  []               [create]
...
```

!!! warning "기본 권한이 낮아도 방심 금물"
    다행히 `default` SA 의 권한은 최소한입니다. 하지만 누군가 이 SA 에 RBAC 로 권한을 더해 줬거나(흔한 실수), 앱이 더 강한 SA 를 쓰고 있다면 이야기가 달라집니다. **애초에 쓰지 않을 토큰이라면 컨테이너에 두지 않는 것**이 가장 확실합니다.

## 3. 방어 적용 · 토큰 자동 마운트 끄기

!!! success "방어 적용 · 필요 없는 토큰은 넣지 않는다"
    이 앱은 쿠버네티스 API 를 호출할 일이 없다고 가정하고, 토큰 자동 마운트를 끕니다. Pod 스펙에 `automountServiceAccountToken: false` 한 줄이면 됩니다.

`no-token-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
spec:
  automountServiceAccountToken: false
  containers:
    - name: app
      image: curlimages/curl
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f no-token-pod.yaml
kubectl wait --for=condition=Ready pod/no-token-pod --timeout=60s
```

```text
pod/no-token-pod created
pod/no-token-pod condition met
```

이제 같은 경로를 확인하면 토큰이 없습니다.

```bash
kubectl exec no-token-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

```text
ls: /var/run/secrets/kubernetes.io/serviceaccount/: No such file or directory
command terminated with exit code 1
```

!!! success "토큰 통로가 사라졌습니다"
    디렉터리 자체가 존재하지 않습니다. 이 컨테이너에 공격자가 침입해도 주워 갈 자격 증명이 없습니다. API 를 쓰지 않는 앱이라면 잃을 것 없이 공격 표면만 줄이는, 비용 대비 효과가 매우 큰 방어입니다.

## 4. ServiceAccount 단위로 끄기

Pod 마다 설정하는 대신, **ServiceAccount** 에 걸어 두면 그 SA 를 쓰는 모든 Pod 에 일괄 적용됩니다.

`sa-no-token.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: restricted-sa
  namespace: default
automountServiceAccountToken: false
```

```bash
kubectl apply -f sa-no-token.yaml
```

```text
serviceaccount/restricted-sa created
```

!!! tip "Pod 설정이 SA 설정보다 우선합니다"
    ServiceAccount 에서 `false` 로 꺼 두더라도, 특정 Pod 에서 `automountServiceAccountToken: true` 로 지정하면 그 Pod 만 토큰을 받습니다. 기본은 SA 에서 끄고(안전 우선), 정말 API 가 필요한 Pod 에서만 개별적으로 켜는 방식이 깔끔합니다.

## RBAC 와 함께 쓰기 (Lab 5 연계)

토큰 차단은 강력하지만, API 를 실제로 써야 하는 앱에는 쓸 수 없습니다. 그런 앱에는 두 방어를 함께 적용합니다.

1. **토큰이 필요 없는 앱** → `automountServiceAccountToken: false` 로 아예 토큰을 빼앗습니다(이번 실습).
2. **토큰이 필요한 앱** → 전용 ServiceAccount 를 만들고, Lab 5 의 RBAC 로 **딱 필요한 권한만** 부여합니다. 토큰이 탈취돼도 할 수 있는 일이 최소한이 됩니다.

!!! note "심층 방어(defense in depth)"
    "토큰을 안 준다"와 "줘도 권한이 없다"는 서로 다른 층의 방어입니다. 두 층을 겹쳐 두면 한쪽이 뚫려도 다른 쪽이 버팁니다. 특히 Secret 을 읽을 수 있는 권한은 꼭 필요한 SA 에만 주고, 나머지는 RBAC 로 확실히 막아야 합니다.

## 검증

- `default-pod` 에는 `/var/run/secrets/kubernetes.io/serviceaccount/token` 이 존재했다.
- `no-token-pod` 에서는 해당 디렉터리가 없어 `No such file or directory` 가 났다.
- `restricted-sa` 에 `automountServiceAccountToken: false` 가 설정됐다.

## 직접 해 보기

1. `restricted-sa` 를 사용하는 Pod 를 배포해(`spec.serviceAccountName: restricted-sa`) 토큰이 정말 마운트되지 않는지 확인해 보세요. 그런 다음 그 Pod 에서만 `automountServiceAccountToken: true` 로 덮어써서, Pod 설정이 SA 설정을 이기는 것을 확인해 보세요.
2. `no-token-pod` 안에서 2단계의 `curl` 명령을 그대로 실행해 보세요. 토큰 파일이 없어 인증이 실패하는 것을 확인할 수 있습니다.

!!! failure "자주 만나는 오류"
    **증상**: `automountServiceAccountToken: false` 를 넣었는데도 토큰이 여전히 있음
    **원인**: 필드 위치 오류. Pod 에서는 `spec.automountServiceAccountToken`(컨테이너가 아니라 Pod 레벨)이어야 합니다.
    **해결**: 들여쓰기를 확인해 `spec:` 바로 아래, `containers:` 와 같은 레벨에 두었는지 봅니다.

    ---

    **증상**: API 를 쓰는 앱을 배포했더니 토큰이 없어 동작하지 않음
    **원인**: 토큰이 꼭 필요한 앱에까지 차단을 적용함.
    **해결**: 그 앱만 토큰을 켜고, RBAC 로 최소 권한을 부여합니다(위 "RBAC 와 함께 쓰기" 참고).

## 정리

```bash
kubectl delete -f default-pod.yaml
kubectl delete -f no-token-pod.yaml
kubectl delete -f sa-no-token.yaml
```

```text
pod "default-pod" deleted
pod "no-token-pod" deleted
serviceaccount "restricted-sa" deleted
```

---

다음: [Lab 7 · NetworkPolicy](07-networkpolicy.md)
