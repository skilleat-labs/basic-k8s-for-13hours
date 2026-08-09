# Lab 4 · YAML 매니페스트

지금까지는 `kubectl run` 으로 명령을 직접 내렸습니다. 이제 원하는 상태를 **YAML 파일(매니페스트)** 로 적어 두고 `apply` 하는 **선언형** 방식을 배웁니다. 실무의 표준 방식입니다.

!!! abstract "이 실습에서 배우는 것"
    - 매니페스트의 4요소: `apiVersion` / `kind` / `metadata` / `spec`
    - `kubectl apply -f` 로 선언형 배포하기
    - 같은 파일을 다시 apply 하면 `unchanged` 가 되는 멱등성
    - `-o yaml` 로 실제 상태(status) 관찰, `--dry-run` 으로 초안 생성

## 사전 조건

- [Lab 3](03-pod-ip-ephemeral.md) 을 마치고 `default` 네임스페이스에 `web` Pod 가 없는 상태.
- 확인: `kubectl get pods` → `No resources found` 여야 합니다.

## 1. 매니페스트 작성하기

아래 내용을 **`pod.yaml`** 이라는 파일로 저장하세요. (메모장, VS Code 등 아무 에디터나 됩니다.)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - containerPort: 80
```

매니페스트는 항상 이 **4요소**로 시작합니다.

| 요소 | 의미 | 예시 값 |
|---|---|---|
| `apiVersion` | 이 리소스가 속한 API 그룹·버전 | `v1` |
| `kind` | 리소스 종류 | `Pod` |
| `metadata` | 이름·라벨 등 식별 정보 | `name: web` |
| `spec` | 원하는 상태(무엇을, 어떻게) | 컨테이너·이미지·포트 |

!!! tip "필드가 헷갈리면 explain"
    "`spec` 밑에 뭘 쓸 수 있더라?" 싶으면 [Lab 1](01-cluster-access.md) 에서 배운 `kubectl explain pod.spec` 을 다시 써 보세요. 매니페스트를 쓸 때 늘 곁에 두는 도구입니다.

## 2. 적용하기 (apply)

작성한 파일을 클러스터에 적용합니다.

```bash
kubectl apply -f pod.yaml
```

```text
pod/web created
```

```bash
kubectl get pod web
```

```text
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          8s
```

## 3. 다시 적용하면? (멱등성)

방금 그 명령을 **한 번 더** 실행합니다. 파일을 바꾸지 않았습니다.

```bash
kubectl apply -f pod.yaml
```

```text
pod/web unchanged
```

`created` 가 아니라 `unchanged` 입니다.

!!! info "선언형의 핵심 · 멱등성"
    `apply` 는 "이 파일과 같은 상태로 맞춰라"라는 뜻입니다. 이미 그 상태면 아무 일도 하지 않습니다(멱등성). 그래서 같은 매니페스트를 몇 번을 apply 해도 안전합니다. 파일 일부를 고쳐 다시 apply 하면 **바뀐 부분만** `configured` 로 반영됩니다.

## 4. 실제 상태 관찰하기 (-o yaml)

내가 쓴 건 `spec`(원하는 상태) 뿐이지만, 클러스터는 여기에 실제 운영 정보(`status`)를 채워 넣습니다. 전체를 YAML 로 뽑아 봅니다.

```bash
kubectl get pod web -o yaml
```

```text
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: default
  labels:
    app: web
  uid: 6a1f...
spec:
  containers:
    - name: web
      image: nginx
      ...
status:
  phase: Running
  podIP: 10.42.0.33
  startTime: "2026-08-10T06:25:02Z"
  containerStatuses:
    - name: web
      ready: true
      restartCount: 0
...
```

- 위쪽 `spec` 은 내가 선언한 것.
- 아래쪽 `status` 는 쿠버네티스가 채운 실제 현황(`podIP`, `phase`, `ready` 등)입니다.

!!! note "spec 은 소망, status 는 현실"
    쿠버네티스는 항상 `status`(현실)를 `spec`(소망)에 맞추려고 끊임없이 조정합니다. 이 "원하는 상태로 수렴"이 쿠버네티스의 작동 원리 그 자체입니다.

## 5. 초안 자동 생성 (--dry-run)

YAML 을 처음부터 손으로 다 쓰긴 번거롭습니다. `run` 에 `--dry-run=client -o yaml` 을 붙이면 **적용하지 않고** 매니페스트 초안만 출력해 줍니다.

```bash
kubectl run web --image=nginx --dry-run=client -o yaml
```

```text
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: web
  name: web
spec:
  containers:
  - image: nginx
    name: web
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

이 출력을 파일로 저장해 원하는 대로 다듬어 쓰면 됩니다. 파일로 바로 저장할 수도 있습니다.

=== "Windows (PowerShell)"
    ```powershell
    kubectl run web --image=nginx --dry-run=client -o yaml > draft.yaml
    ```

=== "macOS / Linux"
    ```bash
    kubectl run web --image=nginx --dry-run=client -o yaml > draft.yaml
    ```

!!! tip "실무 팁"
    복잡한 리소스도 `--dry-run=client -o yaml` 로 뼈대를 뽑은 뒤 필요한 필드만 손보는 게 빠릅니다. `--dry-run=client` 는 클러스터에 아무 영향도 주지 않으니 마음껏 실험하세요.

## 명령형 vs 선언형

| | 명령형 (imperative) | 선언형 (declarative) |
|---|---|---|
| 방식 | `kubectl run/create/delete` 로 매번 지시 | 원하는 상태를 YAML 로 적고 `apply` |
| 비유 | "지금 이걸 해" | "결과가 이렇게 돼 있어야 해" |
| 이력·재현 | 어려움(명령을 기억해야 함) | 파일이 곧 문서·버전관리 대상 |
| 실무 | 빠른 실험·일회성 | 표준 배포 방식 |

!!! info "왜 선언형이 표준인가"
    매니페스트 파일은 Git 에 넣어 버전관리하고, 리뷰하고, 그대로 다시 배포할 수 있습니다. "이 클러스터가 어떤 상태여야 하는가"가 파일 하나에 다 적혀 있으니 재현성과 협업에 압도적으로 유리합니다.

## 검증

```bash
kubectl get pod web -o wide
kubectl apply -f pod.yaml
```

- `web` Pod 가 `Running` 이다.
- 다시 apply 했을 때 `unchanged` 가 나온다.

## 직접 해 보기

1. `pod.yaml` 의 `metadata.labels` 에 `tier: frontend` 를 추가하고 다시 `apply` 해 보세요. 출력이 `configured` 로 바뀌고, `kubectl get pod web --show-labels` 로 라벨이 반영됐는지 확인합니다.
2. `kubectl run box --image=busybox --dry-run=client -o yaml` 로 busybox Pod 의 초안을 뽑아 보고, nginx 초안과 무엇이 같고 다른지 비교해 보세요.

!!! failure "자주 만나는 오류"
    **증상**: `error: error parsing pod.yaml: ... mapping values are not allowed`
    **원인**: YAML 들여쓰기 오류(탭 사용, 칸 수 불일치). YAML 은 **공백 2칸** 들여쓰기를 쓰며 탭을 허용하지 않습니다.
    **해결**: 에디터에서 탭을 공백으로 바꾸고 들여쓰기를 맞춘 뒤 다시 apply.

    ---

    **증상**: `the path "pod.yaml" does not exist`
    **원인**: 명령을 실행한 폴더에 `pod.yaml` 이 없음.
    **해결**: 파일을 저장한 폴더에서 명령을 실행하거나, 전체 경로를 지정(`kubectl apply -f ./경로/pod.yaml`).

    ---

    **증상**: `error validating data: ValidationError(Pod.spec): unknown field ...`
    **원인**: 필드 이름 오타(예: `container` 대신 `containers`)나 위치 오류.
    **해결**: `kubectl explain pod.spec` 으로 정확한 필드명을 확인 후 수정.

## 정리

이번에 만든 Pod 는 파일로 관리했으니, 파일 기준으로 깔끔히 삭제할 수 있습니다.

```bash
kubectl delete -f pod.yaml
```

```text
pod "web" deleted
```

---

다음: [Lab 5 · Deployment와 자가복구](05-deployment-selfheal.md)
