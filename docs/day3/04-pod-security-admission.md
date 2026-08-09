# Lab 4 · Pod Security Admission

Lab 3 에서 `securityContext` 로 Pod 하나하나를 안전하게 만드는 법을 배웠습니다. 하지만 개발자가 실수로 하드닝을 빠뜨리면 어떻게 될까요? 위험한 Pod 가 그대로 배포됩니다. **Pod Security Admission(PSA)** 은 네임스페이스 단위로 "이 네임스페이스에는 안전 기준을 만족하는 Pod 만 들어올 수 있다"는 **문지기(정책)** 를 세웁니다. 이번 실습에서는 `restricted` 정책을 건 네임스페이스에 위험한 Pod 를 넣으려다 **거부당하는** 것을 재현하고, 규격을 갖춘 Pod 는 통과하는 것을 확인합니다.

!!! abstract "이 실습에서 배우는 것"
    - 네임스페이스 라벨로 Pod Security 정책을 적용하는 법
    - 세 가지 **레벨**: `privileged`, `baseline`, `restricted`
    - 세 가지 **모드**: `enforce`(차단), `audit`(기록), `warn`(경고)
    - 폐지된 PodSecurityPolicy(PSP)를 PSA 가 대체했다는 배경

## 사전 조건

- Lab 3 에서 `securityContext` 필드에 익숙해져 있어야 합니다.
- k8s 1.25 이상이면 PSA 가 내장되어 있습니다(k3s 포함). 버전을 확인합니다.

```bash
kubectl version
```

```text
Client Version: v1.29.3+k3s1
Server Version: v1.29.3+k3s1
```

!!! info "PSP 는 폐지되었습니다"
    예전에는 PodSecurityPolicy(PSP)라는 리소스로 비슷한 통제를 했지만, 복잡하고 다루기 어려워 **쿠버네티스 1.25 에서 완전히 제거**되었습니다. 그 자리를 대신하는 것이 지금 배우는 **Pod Security Admission** 이며, 별도 설치 없이 네임스페이스 라벨만으로 동작합니다.

## 1. 정책을 건 네임스페이스 만들기

`restricted` 레벨을 `enforce`(차단)와 `warn`(경고) 두 모드로 적용한 네임스페이스를 만듭니다. 네임스페이스와 라벨을 한 파일에 담습니다.

`secure-ns.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

```bash
kubectl apply -f secure-ns.yaml
```

```text
namespace/secure-ns created
```

라벨이 잘 붙었는지 확인합니다.

```bash
kubectl get namespace secure-ns --show-labels
```

```text
NAME        STATUS   AGE   LABELS
secure-ns   Active   10s   kubernetes.io/metadata.name=secure-ns,pod-security.kubernetes.io/enforce=restricted,pod-security.kubernetes.io/warn=restricted
```

!!! note "라벨 형식이 정책의 전부입니다"
    PSA 는 `pod-security.kubernetes.io/<모드>: <레벨>` 형식의 네임스페이스 라벨만 보고 동작합니다. 별도의 정책 리소스를 만들 필요가 없습니다. 여기서는 `enforce=restricted`(위반 시 차단)와 `warn=restricted`(위반 시 경고 메시지)를 함께 걸었습니다.

## 2. 공격 재현 · 위험한 Pod 배포 시도

!!! danger "공격 재현 · 규격 미달 Pod 를 밀어 넣는다"
    하드닝을 전혀 하지 않은(그래서 root 로 도는) Pod 를 이 네임스페이스에 배포하려 시도합니다. `restricted` 기준에 한참 못 미치는 Pod 입니다.

`bad-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: secure-ns
spec:
  containers:
    - name: app
      image: nginx
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f bad-pod.yaml
```

```text
Error from server (Forbidden): error when creating "bad-pod.yaml": pods "bad-pod" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "app" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "app" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "app" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "app" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

!!! success "위험한 Pod 가 아예 거부됐습니다"
    `Error from server (Forbidden): ... violates PodSecurity "restricted"` — Pod 가 **생성조차 되지 못하고** 거부됐습니다. 오류 메시지가 무엇이 부족한지 친절하게 알려줍니다: `allowPrivilegeEscalation`, `capabilities.drop`, `runAsNonRoot`, `seccompProfile` 를 설정하라고요. Lab 3 에서는 개발자가 하드닝을 "해야" 안전했다면, 여기서는 하드닝하지 않으면 "배포가 막힙니다". 정책이 강제되는 것입니다.

정말 아무것도 안 만들어졌는지 확인합니다.

```bash
kubectl get pods -n secure-ns
```

```text
No resources found in secure-ns namespace.
```

## 3. 방어 규격을 갖춘 Pod 배포

!!! success "방어 적용 · 규격을 지키면 통과한다"
    이번엔 `restricted` 기준을 모두 만족하는 Pod 를 배포합니다. Lab 3 의 하드닝에 `seccompProfile` 을 더한 형태입니다.

`good-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  namespace: secure-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      image: nginx
      command: ["sleep", "3600"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
```

```bash
kubectl apply -f good-pod.yaml
```

```text
pod/good-pod created
```

```bash
kubectl get pods -n secure-ns
```

```text
NAME       READY   STATUS    RESTARTS   AGE
good-pod   1/1     Running   0          15s
```

!!! success "규격을 지킨 Pod 는 정상 생성됩니다"
    같은 네임스페이스인데 이 Pod 는 통과했습니다. PSA 는 "무조건 막는" 것이 아니라 "안전 기준을 만족하면 허용"합니다. 개발팀에게 명확한 규격을 요구하는 셈입니다.

## 레벨과 모드 이해하기

PSA 는 **레벨(무엇을 요구하나)** 과 **모드(위반하면 어떻게 하나)** 의 조합으로 동작합니다.

### 3가지 레벨

| 레벨 | 의미 | 용도 |
|---|---|---|
| `privileged` | 제한 없음(모두 허용) | 시스템·인프라용 네임스페이스 |
| `baseline` | 알려진 위험만 차단(privileged 컨테이너, hostPath 등) | 일반 애플리케이션의 최소 기준 |
| `restricted` | 강력히 제한(비-root, capabilities drop, seccomp 등 강제) | 보안이 중요한 워크로드 권장 |

### 3가지 모드

| 모드 | 위반 시 동작 |
|---|---|
| `enforce` | **차단** — Pod 생성을 거부 |
| `audit` | **기록** — 감사 로그에만 남기고 생성은 허용 |
| `warn` | **경고** — 사용자에게 경고 메시지를 보여주되 생성은 허용 |

!!! tip "세 모드를 함께 쓰면 안전하게 도입할 수 있습니다"
    운영 중인 네임스페이스에 갑자기 `enforce=restricted` 를 걸면 기존 Pod 배포가 무더기로 막힐 수 있습니다. 그래서 실무에서는 먼저 `warn=restricted` + `audit=restricted` 만 걸어 **무엇이 걸리는지 관찰**하고, 앱을 고친 뒤 마지막에 `enforce=restricted` 로 조입니다. 세 라벨을 동시에 지정할 수 있습니다.

## 검증

- `secure-ns` 에 `pod-security.kubernetes.io/enforce=restricted` 라벨이 있다.
- `bad-pod` 배포는 `violates PodSecurity "restricted"` 로 거부됐다.
- `good-pod` 는 같은 네임스페이스에서 `Running` 이다.

## 직접 해 보기

1. `secure-ns` 의 라벨을 `enforce=baseline` 으로 바꿔 보세요(`kubectl label namespace secure-ns pod-security.kubernetes.io/enforce=baseline --overwrite`). 그런 다음 `bad-pod` 를 다시 배포해 보세요. `baseline` 은 root 실행 자체는 막지 않으므로 이번엔 통과할 것입니다. 레벨에 따라 허용 범위가 어떻게 달라지는지 체감해 보세요.
2. `enforce` 라벨은 두고 `warn` 을 `baseline` 으로 낮춘 뒤, 규격 미달 Pod 를 배포하면 어떤 메시지가 나오는지 관찰해 보세요.

!!! failure "자주 만나는 오류"
    **증상**: 라벨을 걸었는데 위험한 Pod 가 그대로 생성됨
    **원인**: 오타로 라벨 키가 틀렸거나(`pod-security.kubernetes.io/enforce` 가 정확), 모드가 `warn`/`audit` 뿐이라 차단이 안 됨.
    **해결**: `kubectl get ns secure-ns --show-labels` 로 `enforce` 모드 라벨이 정확히 있는지 확인합니다.

    ---

    **증상**: 기존에 잘 돌던 앱이 네임스페이스 라벨 추가 후 배포 실패
    **원인**: 앱이 `restricted` 기준을 만족하지 못함.
    **해결**: 먼저 `warn`·`audit` 으로 관찰하며 `securityContext` 를 보완한 뒤 `enforce` 를 적용합니다.

## 정리

```bash
kubectl delete namespace secure-ns
```

```text
namespace "secure-ns" deleted
```

네임스페이스를 지우면 그 안의 `good-pod` 등 모든 리소스가 함께 삭제됩니다.

---

다음: [Lab 5 · RBAC 최소 권한](05-rbac.md)
