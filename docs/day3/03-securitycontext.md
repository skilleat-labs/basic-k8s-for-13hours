# Lab 3 · securityContext 하드닝

기본 설정으로 배포된 컨테이너는 대부분 **root(관리자)** 권한으로 실행됩니다. 공격자가 컨테이너 내부로 침입하면, root 권한은 침입을 컨테이너 탈출이나 노드 장악으로 키우는 발판이 됩니다. 이번 실습에서는 먼저 root 로 도는 컨테이너의 위험을 재현한 뒤, `securityContext` 로 권한을 하나씩 깎아 방어하는 과정을 실습합니다. 여기서부터는 **공격 재현(빨강)** 과 **방어 적용(파랑)** 을 나눠 진행합니다.

!!! abstract "이 실습에서 배우는 것"
    - 컨테이너가 기본적으로 root 로 실행된다는 사실과 그 위험
    - `securityContext` 의 핵심 필드: `runAsNonRoot`, `runAsUser`, `allowPrivilegeEscalation`, `readOnlyRootFilesystem`, `capabilities`
    - `runAsNonRoot` 만 켜고 사용자를 지정하지 않았을 때 생기는 함정
    - Pod-level 과 Container-level securityContext 의 차이

## 사전 조건

- 앞 실습 리소스가 정리된 상태여야 합니다.
- 클러스터가 정상인지 확인합니다.

```bash
kubectl get nodes
```

## 1. 공격 재현 · root 로 도는 컨테이너

!!! danger "공격 재현 · 컨테이너가 root 로 돌고 있다"
    아무 설정 없이 배포한 컨테이너가 어떤 권한으로 실행되는지 직접 확인합니다.

아무 하드닝도 하지 않은 평범한 nginx Pod 를 배포합니다.

`root-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
spec:
  containers:
    - name: app
      image: nginx
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f root-pod.yaml
kubectl wait --for=condition=Ready pod/root-pod --timeout=60s
```

```text
pod/root-pod created
pod/root-pod condition met
```

컨테이너 안에서 지금 어떤 사용자로 실행 중인지 확인합니다.

```bash
kubectl exec root-pod -- whoami
```

```text
root
```

```bash
kubectl exec root-pod -- id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

!!! danger "uid=0 은 관리자입니다"
    `uid=0` 은 리눅스의 최고 권한 계정, 즉 root 입니다. 컨테이너 안에서 root 라는 것은, 만약 커널 취약점이나 잘못된 볼륨 마운트가 있다면 그 권한이 **노드(호스트)로 번질 수 있다**는 뜻입니다. 공격자가 노려볼 수 있는 가장 흔한 발판입니다. 파일시스템도 쓰기 가능하므로, 침입 후 악성 도구를 내려받아 심어 둘 수도 있습니다.

root 컨테이너에서는 파일시스템에 자유롭게 쓸 수 있습니다. 공격자가 임의 파일을 심을 수 있다는 뜻입니다.

```bash
kubectl exec root-pod -- touch /malware.sh
kubectl exec root-pod -- ls -l /malware.sh
```

```text
-rw-r--r-- 1 root root 0 Aug 10 09:00 /malware.sh
```

파일 생성이 아무 저항 없이 성공했습니다. 이것이 하드닝하지 않은 컨테이너의 기본 상태입니다.

## 2. 방어 적용 · 비-root 사용자로 실행하기

!!! success "방어 적용 · root 를 버린다"
    이제 `securityContext` 로 컨테이너를 비-root 사용자로 실행하고 불필요한 권한을 모두 제거합니다.

`hardened-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
    - name: app
      image: nginx
      command: ["sleep", "3600"]
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
```

각 설정의 의미는 이렇습니다.

| 필드 | 위치 | 효과 |
|---|---|---|
| `runAsNonRoot: true` | Pod | 이미지가 root 로 실행되려 하면 **아예 기동을 거부** |
| `runAsUser: 1000` | Pod | UID 1000 사용자로 프로세스 실행 |
| `allowPrivilegeEscalation: false` | Container | `sudo`·setuid 등으로 권한을 높이는 것을 차단 |
| `readOnlyRootFilesystem: true` | Container | 루트 파일시스템을 읽기 전용으로 만들어 파일 심기 차단 |
| `capabilities.drop: [ALL]` | Container | 리눅스 커널 권한(capability)을 전부 제거 |

적용합니다.

```bash
kubectl apply -f hardened-pod.yaml
kubectl wait --for=condition=Ready pod/hardened-pod --timeout=60s
```

```text
pod/hardened-pod created
pod/hardened-pod condition met
```

이제 다시 사용자를 확인하면 root 가 아닙니다.

```bash
kubectl exec hardened-pod -- id
```

```text
uid=1000 gid=3000 groups=2000,3000
```

!!! success "더 이상 root 가 아닙니다"
    `uid=1000` 으로 실행됩니다. 공격자가 이 컨테이너에 침입해도 일반 사용자 권한밖에 얻지 못합니다.

## 3. 방어 적용 · 읽기 전용 파일시스템 검증

`readOnlyRootFilesystem: true` 가 실제로 파일 쓰기를 막는지 확인합니다. Lab 1단계에서 성공했던 것과 같은 `touch` 를 시도합니다.

```bash
kubectl exec hardened-pod -- touch /malware.sh
```

```text
touch: cannot touch '/malware.sh': Read-only file system
```

```text
command terminated with exit code 1
```

!!! success "파일 심기가 차단됐습니다"
    루트 파일시스템이 읽기 전용이라 공격자가 악성 파일을 심으려는 시도가 거부됩니다. 앱이 임시 파일을 써야 한다면, 필요한 경로에만 `emptyDir` 볼륨을 마운트해 쓰기를 허용하면 됩니다(전체를 열지 않고 최소한만).

## 4. 함정 · runAsNonRoot 만 켜면 생기는 문제

가장 흔히 겪는 실수를 일부러 재현해 봅니다. `runAsNonRoot: true` 만 켜고 **`runAsUser` 를 지정하지 않은** 채, root 로 실행되도록 만들어진 이미지를 쓰면 어떻게 될까요?

`trap-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: trap-pod
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: app
      image: nginx
      command: ["sleep", "3600"]
```

```bash
kubectl apply -f trap-pod.yaml
```

```text
pod/trap-pod created
```

잠시 뒤 상태를 봅니다.

```bash
kubectl get pod trap-pod
```

```text
NAME       READY   STATUS                       RESTARTS   AGE
trap-pod   0/1     CreateContainerConfigError   0          20s
```

원인을 봅니다.

```bash
kubectl describe pod trap-pod
```

```text
...
Events:
  Type     Reason     Age   From     Message
  ----     ------     ----  ----     -------
  Warning  Failed     10s   kubelet  Error: container has runAsNonRoot and image will run as root
...
```

!!! warning "runAsNonRoot 는 root 를 막을 뿐, 대신 실행할 사용자를 정해 주지 않습니다"
    `runAsNonRoot: true` 는 "root 면 실행하지 마라"는 **금지 규칙**입니다. 그런데 이 nginx 이미지는 별도 사용자 지정이 없으면 root 로 실행하려 하므로, 쿠버네티스가 규칙 위반으로 판단해 컨테이너 생성을 거부합니다(`CreateContainerConfigError`). 해결책은 **`runAsUser: 1000` 처럼 실행할 비-root 사용자를 명시**하는 것입니다(2단계의 `hardened-pod` 가 바로 그렇게 했습니다). 이 함정은 실무에서 매우 자주 만나므로 꼭 기억하세요.

확인이 끝났으니 이 함정 Pod 는 지웁니다.

```bash
kubectl delete pod trap-pod
```

## Pod-level vs Container-level securityContext

`securityContext` 는 두 곳에 지정할 수 있고, 적용 범위가 다릅니다.

| 위치 | 적용 범위 | 대표 필드 |
|---|---|---|
| **Pod** `spec.securityContext` | Pod 안 **모든 컨테이너**에 공통 적용 | `runAsUser`, `runAsNonRoot`, `fsGroup` |
| **Container** `containers[].securityContext` | **해당 컨테이너에만** 적용(Pod 설정을 덮어씀) | `allowPrivilegeEscalation`, `readOnlyRootFilesystem`, `capabilities` |

!!! tip "겹치면 Container 가 이깁니다"
    같은 필드를 Pod 와 Container 양쪽에 지정하면 **Container 쪽 설정이 우선**합니다. 공통 정책은 Pod 레벨에, 컨테이너별 예외는 Container 레벨에 두는 식으로 조합합니다.

## 검증

- `root-pod` 에서 `id` 는 `uid=0(root)` 였다.
- `hardened-pod` 에서 `id` 는 `uid=1000` 이다.
- `hardened-pod` 에서 `touch /malware.sh` 가 `Read-only file system` 으로 거부됐다.
- `runAsNonRoot` 만 켠 `trap-pod` 는 `CreateContainerConfigError` 로 뜨지 않았다.

## 직접 해 보기

1. `hardened-pod` 에서 `kubectl exec hardened-pod -- touch /tmp/test` 를 해 보세요. `/tmp` 역시 읽기 전용이라 실패합니다. 그런 다음 매니페스트에 `emptyDir` 볼륨을 `/tmp` 에 마운트하고 재배포해, 이제는 쓰기가 되는지 확인해 보세요.
2. `capabilities.drop: [ALL]` 대신 특정 권한만 남기는 실험을 해 보세요. 예를 들어 `add: ["NET_BIND_SERVICE"]` 를 추가하면 1024 미만 포트 바인딩이 가능해집니다. 최소 권한 원칙에 맞게 "필요한 것만 더하기"를 연습합니다.

!!! failure "자주 만나는 오류"
    **증상**: `hardened-pod` 가 `CrashLoopBackOff` (일반 nginx 이미지를 그대로 웹 서버로 띄웠을 때)
    **원인**: nginx 공식 이미지는 시작 시 root 로 여러 디렉터리에 써야 해서, 위 하드닝과 그대로는 충돌합니다. 이 실습은 `command: ["sleep", "3600"]` 으로 nginx 를 실제로 구동하지 않아 문제를 피했습니다.
    **해결**: 실무에서는 `nginxinc/nginx-unprivileged` 처럼 비-root 로 동작하도록 만들어진 이미지를 사용합니다.

    ---

    **증상**: `hardened-pod` 도 `CreateContainerConfigError`
    **원인**: `runAsUser` 를 빠뜨렸을 가능성.
    **해결**: 매니페스트에 `runAsUser: 1000` 이 있는지 확인합니다(4단계 함정과 같은 원인).

## 정리

```bash
kubectl delete -f root-pod.yaml
kubectl delete -f hardened-pod.yaml
```

```text
pod "root-pod" deleted
pod "hardened-pod" deleted
```

(`trap-pod` 는 4단계에서 이미 삭제했습니다.)

---

다음: [Lab 4 · Pod Security Admission](04-pod-security-admission.md)
