# Lab 1 · 도커의 한계와 쿠버네티스

도커로 컨테이너 하나를 띄우는 건 쉽습니다. 그런데 **그 컨테이너가 죽으면 누가 다시 살려줄까요?** 트래픽이 몰려 10대로 늘려야 하면요? 새 버전을 무중단으로 배포하려면요? 이번 실습에서는 **도커(단일 호스트)의 한계를 직접 체감** 하고, 같은 일을 쿠버네티스가 어떻게 **자동으로** 처리하는지 눈으로 비교합니다. 오늘 과정 전체를 관통하는 "왜 쿠버네티스인가"에 대한 답을 손으로 얻는 시간입니다.

!!! abstract "이 실습에서 배우는 것"
    - 컨테이너를 직접 실행하고, **죽었을 때 자동 복구가 안 되는** 단일 호스트의 한계를 체감한다
    - 쿠버네티스 아키텍처(컨트롤플레인 · 워커노드)를 명령으로 실물 확인한다
    - 같은 앱을 쿠버네티스에 올려 **Pod를 지워도 자동으로 되살아나는** 선언적 관리를 미리 본다
    - 명령형(도커) vs 선언형(쿠버네티스)의 차이를 이해한다

## 사전 조건

- [실습 환경 준비](../setup/prerequisites.md)를 마치고 Rancher Desktop이 실행 중이다
- `kubectl get nodes` 가 `Ready` 로 나온다

!!! note "컨테이너 엔진 확인"
    이 실습의 앞부분은 컨테이너를 **직접** 실행합니다. Rancher Desktop 설치 시 고른 엔진에 따라 명령이 다릅니다.

    - **containerd** 를 골랐다면 → `nerdctl` 명령 (이 과정 기본)
    - **dockerd(moby)** 를 골랐다면 → `docker` 명령

    아래 단계마다 두 엔진 탭을 모두 제공합니다. 자신이 고른 엔진 탭을 선택하세요.

## 1. 컨테이너 직접 실행하기

먼저 nginx 웹 서버 컨테이너 하나를 띄웁니다.

=== "containerd (nerdctl)"

    ```bash
    nerdctl run -d --name web -p 8080:80 nginx
    nerdctl ps
    ```

=== "dockerd (docker)"

    ```bash
    docker run -d --name web -p 8080:80 nginx
    docker ps
    ```

```text
CONTAINER ID   IMAGE     ...   STATUS         PORTS                  NAMES
a1b2c3d4e5f6   nginx     ...   Up 5 seconds   0.0.0.0:8080->80/tcp   web
```

웹 서버에 접속해 봅니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8080
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost:8080
    ```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

잘 뜹니다. 여기까지는 도커의 세계에서 아무 문제가 없습니다.

## 2. 컨테이너가 죽으면? — 단일 호스트의 한계

실제 운영에서는 프로세스가 죽거나, 컨테이너가 강제 종료되는 일이 생깁니다. 그 상황을 흉내 내 컨테이너를 강제로 없애 봅니다.

=== "containerd (nerdctl)"

    ```bash
    nerdctl rm -f web
    nerdctl ps
    ```

=== "dockerd (docker)"

    ```bash
    docker rm -f web
    docker ps
    ```

```text
CONTAINER ID   IMAGE     COMMAND   ...   STATUS   PORTS   NAMES
(비어 있음)
```

다시 접속해 봅니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost:8080
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost:8080
    ```

```text
curl: (7) Failed to connect to localhost port 8080: Connection refused
```

!!! warning "핵심 — 아무도 대신 살려주지 않는다"
    컨테이너가 사라졌지만 **자동으로 다시 뜨지 않습니다.** 사람이 직접 `run` 명령을 다시 쳐야 서비스가 복구됩니다. 새벽 3시에 컨테이너가 죽으면? 누군가 깨어나 손으로 살려야 합니다. 서버가 여러 대라면 어느 서버에 올릴지도 사람이 정해야 하죠. 이것이 **단일 호스트에서 컨테이너를 수동으로 운영할 때의 한계** 입니다.

정리하면, 도커만으로는 이런 것들을 **사람이 직접** 해야 합니다.

| 상황 | 도커(단일 호스트)에서는 |
|---|---|
| 컨테이너가 죽음 | 자동 복구 없음 → 수동 재시작 |
| 트래픽 급증 | 수동으로 여러 대 실행 + 부하 분산 직접 구성 |
| 새 버전 배포 | 정지 → 교체 구간에 다운타임 발생 |
| 여러 서버로 확장 | 어느 서버에 올릴지 사람이 결정 |

## 3. 쿠버네티스 아키텍처를 눈으로 보기

쿠버네티스는 이 문제들을 **자동으로** 풀어 주는 오케스트레이션 시스템입니다. 먼저 우리 클러스터의 구조를 명령으로 확인합니다.

```bash
kubectl get nodes
```

```text
NAME                   STATUS   ROLES                  AGE   VERSION
lima-rancher-desktop   Ready    control-plane,master   3d    v1.29.3+k3s1
```

노드가 하나인데 `ROLES` 에 `control-plane,master` 가 함께 붙어 있습니다. 학습용 k3s라 **한 노드가 컨트롤플레인과 워커 역할을 겸하기** 때문입니다. 실무에서는 이 역할이 여러 노드로 나뉩니다.

컨트롤플레인·애드온 구성요소는 `kube-system` 네임스페이스에서 볼 수 있습니다.

```bash
kubectl get pods -n kube-system
```

```text
NAME                                      READY   STATUS    RESTARTS   AGE
coredns-6799fbcd5-8k2pl                   1/1     Running   0          3d
local-path-provisioner-857df8ff65-z4tqs   1/1     Running   0          3d
metrics-server-648b5df564-c2n9v           1/1     Running   0          3d
traefik-5d45fc8cc9-xk7pq                  1/1     Running   0          3d
```

| 구분 | 하는 일 | 대표 구성요소 |
|---|---|---|
| **컨트롤플레인** | 클러스터의 두뇌. 무엇을 어디에 배치할지 결정·관리 | kube-apiserver · etcd · scheduler · controller-manager |
| **워커노드** | 실제로 컨테이너(Pod)를 실행 | kubelet · kube-proxy · 컨테이너 런타임 |

!!! info "apiserver·etcd·scheduler가 안 보이는 이유"
    k3s는 이 컨트롤플레인 구성요소들을 **하나의 프로세스로 통합** 해 실행하기 때문에, 일반 클러스터처럼 개별 Pod로 보이지 않을 수 있습니다. 구조 자체는 동일합니다 — 두뇌(컨트롤플레인)와 일꾼(워커)이 존재합니다.

## 4. 같은 앱을 쿠버네티스에 — 자동 복구 미리 보기

이제 아까 죽으면 끝이었던 nginx를, 쿠버네티스에 맡겨 봅니다.

```bash
kubectl create deployment web --image=nginx
kubectl get pods
```

```text
NAME                   READY   STATUS    RESTARTS   AGE
web-5d4f9c8b7c-abcde   1/1     Running   0          15s
```

Pod가 하나 떴습니다. 이제 **아까 도커에서 했던 것처럼** 이 Pod를 강제로 지워 봅니다.

```bash
kubectl delete pod -l app=web
kubectl get pods
```

```text
NAME                   READY   STATUS              RESTARTS   AGE
web-5d4f9c8b7c-fghij   0/1     ContainerCreating   0          2s
```

!!! success "핵심 — 쿠버네티스는 스스로 되살린다"
    Pod를 지웠는데 **새 Pod가 즉시 자동으로 생성** 됩니다(이름의 뒷부분이 바뀐 것에 주목). 도커에서는 사람이 다시 살려야 했지만, 쿠버네티스는 "web이라는 앱이 1개 떠 있어야 한다"는 **원하는 상태(desired state)** 를 기억하고, 현재 상태가 어긋나면 **스스로 맞춥니다.** 이것을 자가복구(self-healing) 또는 조정 루프(reconciliation loop)라고 합니다.

## 5. 명령형 vs 선언형

방금 본 차이를 개념으로 정리합니다.

| | 도커(명령형) | 쿠버네티스(선언형) |
|---|---|---|
| 방식 | "이 명령을 실행해라" | "이런 상태이길 원한다" |
| 컨테이너가 죽으면 | 그대로 멈춤 | 자동으로 다시 생성 |
| 관리 범위 | 한 대(단일 호스트) | 여러 노드(클러스터) |
| 확장 | 수동 | 선언한 개수로 자동 유지 |

!!! note "오늘의 여정"
    오늘 남은 실습에서 이 "선언적 관리"를 하나씩 손으로 익힙니다. Pod를 직접 다루고(→ Lab 3), YAML로 원하는 상태를 선언하고(→ Lab 5), Deployment로 자가복구·확장·무중단 배포까지(→ Lab 6~8) 이어집니다.

## 검증

- 도커 컨테이너를 지웠을 때 `curl` 이 실패했다(자동 복구 없음)
- `kubectl get nodes` 로 단일 노드가 `control-plane,master` 임을 확인했다
- 쿠버네티스 Pod를 지웠을 때 새 Pod가 자동 생성됐다

## 직접 해 보기

!!! question "도전 과제"
    1. 쿠버네티스 Pod를 **한 번 더** 지워 보세요. 몇 번을 지워도 계속 되살아나나요?
    2. `kubectl get deployment web` 을 실행해 `READY` 열(예: `1/1`)이 무엇을 의미하는지 생각해 보세요.
    3. 도커로 컨테이너 2개(`web1`, `web2`)를 서로 다른 포트로 띄워 보세요. 부하 분산은 누가 해 주나요? (쿠버네티스의 Service는 2일차에서 배웁니다.)

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **`nerdctl: command not found`** — dockerd(moby) 엔진을 골랐을 가능성이 큽니다. `docker` 탭을 사용하세요. 반대의 경우도 마찬가지입니다.
    - **`docker: command not found`** — containerd 엔진이면 `nerdctl` 을 쓰세요.
    - **포트 8080 이미 사용 중** — 이전 컨테이너가 남아 있을 수 있습니다. `nerdctl rm -f web`(또는 `docker rm -f web`)로 정리 후 다시 실행하세요.
    - **`curl` 이 HTML 대신 이상한 출력** — Windows PowerShell의 `curl` 은 `Invoke-WebRequest` 별칭입니다. 반드시 `curl.exe` 를 쓰세요.

## 정리

컨테이너와 Deployment를 정리합니다.

=== "containerd (nerdctl)"

    ```bash
    nerdctl rm -f web 2>/dev/null; kubectl delete deployment web
    ```

=== "dockerd (docker)"

    ```bash
    docker rm -f web 2>/dev/null; kubectl delete deployment web
    ```

!!! tip "Windows(PowerShell) 사용자"
    위 정리 명령의 `2>/dev/null` 은 macOS/Linux 문법입니다. PowerShell에서는 두 명령을 그냥 나눠서 실행하세요.
    ```powershell
    nerdctl rm -f web   # 또는 docker rm -f web
    kubectl delete deployment web
    ```

---

다음: [Lab 2 · 클러스터 접속과 조회](02-cluster-access.md)
