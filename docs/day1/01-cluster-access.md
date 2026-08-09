# Lab 1 · 클러스터 접속과 조회

`kubectl` 로 로컬 쿠버네티스 클러스터에 접속해, 노드와 시스템 구성요소를 들여다봅니다. 앞으로 모든 실습에서 반복하게 될 **조회 3종 세트(get → describe → explain)** 습관을 여기서 몸에 익힙니다.

!!! abstract "이 실습에서 배우는 것"
    - `kubectl get / describe / explain` 로 리소스를 조회하는 흐름
    - k3s 단일 노드 클러스터의 구성과 kube-system 네임스페이스
    - kubeconfig 4요소(clusters / users / contexts / current-context)와 컨텍스트 전환
    - 네임스페이스와 API 리소스의 개념

## 사전 조건

- Rancher Desktop 이 실행 중이고, 쿠버네티스가 켜져 있어야 합니다.
- 터미널에서 `kubectl` 이 실행되고, 아래 명령이 `Ready` 를 보여줘야 합니다.

```bash
kubectl get nodes
```

!!! note "실습 환경"
    이 과정은 Rancher Desktop 내장 **k3s 단일 노드** 클러스터에서 진행합니다. 노드가 하나뿐이라 control-plane 과 worker 역할을 한 노드가 겸합니다. 컨테이너 런타임은 containerd 입니다.

## 1. 노드 조회하기

가장 먼저 클러스터에 노드가 몇 개 있고 상태가 어떤지 봅니다.

```bash
kubectl get nodes
```

```text
NAME                   STATUS   ROLES                  AGE   VERSION
lima-rancher-desktop   Ready    control-plane,master   12d   v1.29.3+k3s1
```

- `STATUS` 가 `Ready` 면 정상입니다.
- `ROLES` 에 `control-plane,master` 가 함께 붙은 것은 단일 노드라 두 역할을 겸하기 때문입니다.
- `VERSION` 의 `+k3s1` 접미사가 이 클러스터가 k3s 배포판임을 알려 줍니다.

더 자세한 정보(내부 IP, OS, 런타임)를 보려면 `-o wide` 를 붙입니다.

```bash
kubectl get nodes -o wide
```

```text
NAME                   STATUS   ROLES                  AGE   VERSION        INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                      KERNEL-VERSION      CONTAINER-RUNTIME
lima-rancher-desktop   Ready    control-plane,master   12d   v1.29.3+k3s1   192.168.5.15   <none>        Alpine Linux v3.19            6.6.12-0-virt       containerd://1.7.13-k3s1
```

!!! tip "`-o wide` 는 습관처럼"
    대부분의 `get` 명령에 `-o wide` 를 붙이면 IP·노드 등 유용한 열이 추가로 나옵니다. Pod 를 볼 때 특히 유용합니다.

## 2. 노드 상세 보기 (describe)

`get` 이 목록이라면, `describe` 는 한 리소스의 상세 정보입니다. 노드에 할당된 리소스, 조건(Conditions), 실행 중인 Pod 등을 보여 줍니다.

```bash
kubectl describe node lima-rancher-desktop
```

```text
Name:               lima-rancher-desktop
Roles:              control-plane,master
Labels:             beta.kubernetes.io/arch=arm64
                    kubernetes.io/hostname=lima-rancher-desktop
                    node-role.kubernetes.io/control-plane=true
...
Conditions:
  Type             Status  Reason                       Message
  ----             ------  ------                       -------
  MemoryPressure   False   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   KubeletHasNoDiskPressure     kubelet has no disk pressure
  Ready            True    KubeletReady                 kubelet is posting ready status
Capacity:
  cpu:                8
  memory:             8039424Ki
  pods:               110
...
Non-terminated Pods:          (7 in total)
  Namespace   Name                                      CPU Requests   Memory Requests
  ---------   ----                                      ------------   ---------------
  kube-system coredns-6799fbcd5-abcde                   100m           70Mi
  kube-system traefik-7d5f6474df-fghij                  0 (0%)         0 (0%)
...
```

!!! info "출력이 길어요"
    `describe` 는 화면을 꽉 채웁니다. 지금은 `Conditions` 의 `Ready True` 와 맨 아래 `Non-terminated Pods` 목록만 눈여겨보세요. 문제 해결(트러블슈팅)의 대부분은 `describe` 로 시작합니다.

## 3. 시스템 구성요소 관찰하기

클러스터를 돌아가게 하는 시스템 Pod 들은 `kube-system` 네임스페이스에 있습니다. 모든 네임스페이스의 Pod 를 한 번에 보려면 `-A`(= `--all-namespaces`) 를 씁니다.

```bash
kubectl get pods -A
```

```text
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   coredns-6799fbcd5-abcde                   1/1     Running     0          12d
kube-system   local-path-provisioner-6f5d79df6-klmno    1/1     Running     0          12d
kube-system   metrics-server-54fd9b65b-pqrst            1/1     Running     0          12d
kube-system   svclb-traefik-xxxxxxxx-uvwxy              2/2     Running     0          12d
kube-system   traefik-7d5f6474df-fghij                  1/1     Running     0          12d
kube-system   helm-install-traefik-abcde                0/1     Completed   0          12d
```

각 구성요소의 역할을 간단히 정리하면 이렇습니다.

| 구성요소 | 역할 |
|---|---|
| `coredns` | 클러스터 내부 DNS (Service 이름 → IP 해석) |
| `traefik` | k3s 기본 Ingress 컨트롤러 |
| `svclb-traefik` | LoadBalancer 타입 Service 를 위한 k3s 내장 로드밸런서 |
| `local-path-provisioner` | 로컬 디스크 기반 동적 스토리지 프로비저너 |
| `metrics-server` | CPU·메모리 사용량 수집 (`kubectl top` 의 데이터 원천) |

!!! warning "apiserver / etcd / scheduler 는 왜 안 보이나요?"
    일반적인 쿠버네티스에서는 `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager` 가 `kube-system` 에 Pod 로 떠 있습니다. 하지만 **k3s 는 이들을 하나의 프로세스(`k3s server`)로 합쳐서 노드에서 직접 실행**하기 때문에 Pod 목록에는 나타나지 않습니다. 기능이 없는 게 아니라, 패키징 방식이 다를 뿐입니다.

## 4. 네임스페이스와 API 리소스

네임스페이스는 클러스터 안의 리소스를 논리적으로 나누는 폴더 같은 공간입니다.

```bash
kubectl get namespaces
```

```text
NAME              STATUS   AGE
default           Active   12d
kube-node-lease   Active   12d
kube-public       Active   12d
kube-system       Active   12d
```

우리가 앞으로 실습에서 만드는 리소스는 별도 지정이 없으면 모두 `default` 네임스페이스에 생성됩니다.

쿠버네티스가 다룰 수 있는 리소스 종류(그리고 축약형)는 `api-resources` 로 확인합니다.

```bash
kubectl api-resources
```

```text
NAME              SHORTNAMES   APIVERSION   NAMESPACED   KIND
pods              po           v1           true         Pod
services          svc          v1           true         Service
deployments       deploy       apps/v1      true         Deployment
replicasets       rs           apps/v1      true         ReplicaSet
namespaces        ns           v1           false        Namespace
nodes             no           v1           false        Node
...
```

!!! tip "축약형(SHORTNAMES)을 쓰면 빨라집니다"
    `kubectl get pods` 는 `kubectl get po`, `deployments` 는 `deploy`, `services` 는 `svc` 로 줄여 쓸 수 있습니다. `NAMESPACED` 열이 `false` 인 리소스(노드, 네임스페이스)는 특정 네임스페이스에 속하지 않는 클러스터 전역 리소스입니다.

## 5. 필드 탐색하기 (explain)

YAML 을 작성할 때 "이 리소스에 어떤 필드를 쓸 수 있지?" 가 궁금하면 웹 검색 대신 `explain` 을 씁니다. 클러스터가 직접 알려 주는 공식 설명입니다.

```bash
kubectl explain pod.spec
```

```text
KIND:     Pod
VERSION:  v1

FIELD:    spec <PodSpec>

DESCRIPTION:
    Specification of the desired behavior of the Pod.

FIELDS:
  containers    <[]Container> -required-
    List of containers belonging to the pod.

  nodeName      <string>
    NodeName indicates in which node this pod is scheduled.

  restartPolicy <string>
    Restart policy for all containers within the pod.
...
```

더 깊이 들어갈 수도 있습니다. 예를 들어 컨테이너에 어떤 필드가 있는지 보려면 경로를 이어 붙입니다.

```bash
kubectl explain pod.spec.containers
```

!!! note "조회 3종 세트를 기억하세요"
    앞으로 리소스를 다룰 때 이 순서가 기본기입니다.

    1. **get** — 뭐가 있는지 목록으로 본다
    2. **describe** — 하나를 골라 상세와 이벤트를 본다
    3. **explain** — YAML 을 쓰기 전에 필드를 확인한다

## 6. kubeconfig 와 컨텍스트

`kubectl` 이 "어느 클러스터에, 누구로, 어떤 네임스페이스로" 접속할지는 **kubeconfig** 파일에 담겨 있습니다.

=== "Windows (PowerShell)"
    ```powershell
    type $env:USERPROFILE\.kube\config
    ```

=== "macOS / Linux"
    ```bash
    cat ~/.kube/config
    ```

파일을 직접 열지 않고 요약해서 보려면 `config view` 가 편합니다.

```bash
kubectl config view
```

```text
apiVersion: v1
clusters:
- cluster:
    server: https://127.0.0.1:6443
  name: rancher-desktop
contexts:
- context:
    cluster: rancher-desktop
    user: rancher-desktop
  name: rancher-desktop
current-context: rancher-desktop
kind: Config
users:
- name: rancher-desktop
  user: ...
```

kubeconfig 는 **4가지 요소**로 이루어집니다.

| 요소 | 의미 |
|---|---|
| `clusters` | 접속할 클러스터(주소·인증서). "어디로" |
| `users` | 인증 정보(자격 증명). "누구로" |
| `contexts` | clusters + users(+namespace) 조합. "이 조합으로" |
| `current-context` | 지금 활성화된 컨텍스트 이름 |

현재 어떤 컨텍스트를 쓰고 있는지, 어떤 컨텍스트들이 있는지 확인합니다.

```bash
kubectl config current-context
```

```text
rancher-desktop
```

```bash
kubectl config get-contexts
```

```text
CURRENT   NAME              CLUSTER           AUTHINFO          NAMESPACE
*         rancher-desktop   rancher-desktop   rancher-desktop
```

컨텍스트가 여러 개일 때는 `use-context` 로 전환합니다. 지금은 하나뿐이지만, 우리가 쓸 컨텍스트를 명시적으로 지정하는 습관을 들여 둡니다.

```bash
kubectl config use-context rancher-desktop
```

```text
Switched to context "rancher-desktop".
```

!!! info "`*` 표시"
    `get-contexts` 의 `CURRENT` 열에 있는 `*` 가 현재 활성 컨텍스트입니다. 여러 클러스터를 오갈 때 지금 어디에 명령을 보내고 있는지 항상 이걸로 확인하세요.

## 검증

아래가 모두 통과하면 이 실습을 제대로 마친 것입니다.

```bash
kubectl get nodes
kubectl config current-context
kubectl get pods -n kube-system
```

- 노드 `lima-rancher-desktop` 이 `Ready` 로 보인다.
- 현재 컨텍스트가 `rancher-desktop` 이다.
- `kube-system` 에 `coredns`, `traefik`, `metrics-server` 등이 `Running` 이다.

## 직접 해 보기

1. `kubectl get pods -A -o wide` 를 실행해, 시스템 Pod 들이 모두 `lima-rancher-desktop` 노드에 떠 있음을 확인해 보세요. (단일 노드니 당연히 그렇겠죠?)
2. `kubectl explain node.status` 를 실행해, 노드 상태에 어떤 필드가 있는지 살펴보세요. 그중 `conditions` 필드의 설명을 읽어 보세요.

!!! failure "자주 만나는 오류"
    **증상**: `The connection to the server 127.0.0.1:6443 was refused`
    **원인**: Rancher Desktop 의 쿠버네티스가 아직 시작 중이거나 꺼져 있음.
    **해결**: Rancher Desktop 을 실행하고, 좌측 하단이 초록색(정상)이 될 때까지 1~2분 기다린 뒤 다시 시도.

    ---

    **증상**: `error: context "xxx" does not exist`
    **원인**: `use-context` 에 지정한 이름이 kubeconfig 에 없음.
    **해결**: `kubectl config get-contexts` 로 정확한 이름(`rancher-desktop`)을 확인 후 다시 입력.

    ---

    **증상**: `command not found: kubectl`
    **원인**: Rancher Desktop 설치 시 PATH 에 `kubectl` 이 등록되지 않음.
    **해결**: 터미널(또는 Windows 는 PowerShell)을 새로 열거나, Rancher Desktop 을 재시작.

## 정리

이번 실습은 조회만 했으므로 삭제할 리소스가 없습니다. 다음 실습부터 만든 리소스는 각 실습 끝에서 정리합니다.

---

다음: [Lab 2 · 첫 Pod 실행](02-first-pod.md)
