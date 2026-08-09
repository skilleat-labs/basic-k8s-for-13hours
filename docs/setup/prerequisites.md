# 실습 환경 준비

모든 실습은 **Rancher Desktop** 이 노트북에 띄워 주는 로컬 쿠버네티스 클러스터(k3s)에서 진행합니다. 클라우드 계정이나 결제가 필요 없습니다.

!!! info "왜 Rancher Desktop인가"
    macOS·Windows·Linux 어디서든 동일하게 **로컬 단일 노드 쿠버네티스 클러스터**를 한 번에 띄워 주는 무료 오픈소스 도구입니다. 내부적으로 CNCF 인증 경량 배포판 **k3s** 를 사용합니다. 컨트롤플레인과 워커 역할을 노드 하나가 함께 수행합니다.

## 1. Rancher Desktop 설치

=== "Windows"

    1. [rancherdesktop.io](https://rancherdesktop.io) 에서 Windows용 설치 파일(`.msi`)을 내려받습니다.
    2. 설치 프로그램을 실행합니다. WSL2가 필요하며, 없으면 설치 과정에서 안내됩니다.
        - WSL2가 없다면 PowerShell(관리자)에서 아래를 먼저 실행하고 재부팅하세요.
        ```powershell
        wsl --install
        ```
    3. 설치 후 Rancher Desktop을 실행합니다.

=== "macOS / Linux"

    1. [rancherdesktop.io](https://rancherdesktop.io) 에서 설치 파일을 내려받습니다. (macOS: `.dmg`)
        - Apple Silicon(M1~) / Intel 칩에 맞는 빌드를 선택하세요.
        - Homebrew를 쓴다면:
        ```bash
        brew install --cask rancher
        ```
    2. macOS는 내부적으로 경량 VM(Lima)을 사용합니다. 별도 설정은 필요 없습니다.
    3. 설치 후 Rancher Desktop을 실행합니다.

## 2. 최초 실행 설정

처음 실행하면 아래 두 가지를 묻습니다. 이 과정 실습 기준에 맞춰 선택하세요.

| 항목 | 선택 | 이유 |
|---|---|---|
| **Kubernetes 버전** | 최신 안정 버전(기본값) | 실습 명령은 버전 무관하게 동작 |
| **Container Engine** | **containerd** | 실무 클러스터 대부분이 containerd를 사용 |

설정을 마치면 VM이 부팅되고 k3s 클러스터가 자동으로 뜹니다. 앱 창의 상태 표시가 **초록색(정상)** 이 될 때까지 기다리세요.

!!! note "설치 중 자동으로 처리되는 것"
    - `kubectl` 바이너리가 함께 설치됩니다.
    - `~/.kube/config`(kubeconfig)에 `rancher-desktop` 컨텍스트가 추가되고, 현재 컨텍스트가 그것으로 전환됩니다.

## 3. 설치 검증

터미널을 새로 열고 아래 세 명령으로 클러스터가 살아 있는지 확인합니다. `kubectl` 명령 자체는 **모든 OS에서 동일**합니다. (Windows는 PowerShell, macOS/Linux는 터미널에서 실행)

```bash
kubectl version
kubectl get nodes
kubectl cluster-info
```

정상이라면 이런 출력이 나옵니다(버전·이름은 다를 수 있음).

```text
$ kubectl get nodes
NAME                   STATUS   ROLES                  AGE   VERSION
lima-rancher-desktop   Ready    control-plane,master   3d    v1.29.3+k3s1
```

??? question "출력이 이렇게 나오는 이유는?"
    - **ROLES에 control-plane,master가 함께** — k3s 단일 노드가 컨트롤플레인 역할을 겸하기 때문
    - **VERSION 뒤 +k3s1** — 표준 쿠버네티스가 아니라 k3s로 빌드된 버전이라는 표시
    - **cluster-info 주소가 127.0.0.1** — 내 컴퓨터 안 VM에 떠 있는 로컬 클러스터이기 때문

## 4. 편의 설정(선택이지만 강력 추천)

실습 내내 `kubectl` 을 수십 번 칩니다. 별칭과 자동완성을 걸어 두면 훨씬 빠릅니다.

=== "Windows (PowerShell)"

    PowerShell 프로필에 별칭과 자동완성을 추가합니다.

    ```powershell
    # 프로필 파일이 없으면 생성
    if (!(Test-Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }

    # 프로필 편집기 열기
    notepad $PROFILE
    ```

    편집기가 열리면 아래 내용을 붙여넣고 저장합니다.

    ```powershell
    # kubectl 별칭
    Set-Alias -Name k -Value kubectl

    # 자동완성
    kubectl completion powershell | Out-String | Invoke-Expression
    ```

    PowerShell을 새로 열면 `k get nodes` 처럼 쓸 수 있습니다.

    !!! warning "스크립트 실행 정책"
        프로필 로딩이 막히면 관리자 PowerShell에서 실행 정책을 완화하세요.
        ```powershell
        Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
        ```

=== "macOS / Linux (zsh · bash)"

    사용 중인 셸의 설정 파일(`~/.zshrc` 또는 `~/.bashrc`)에 추가합니다.

    ```bash
    # zsh 사용자
    echo 'alias k=kubectl' >> ~/.zshrc
    echo 'source <(kubectl completion zsh)' >> ~/.zshrc
    source ~/.zshrc
    ```

    ```bash
    # bash 사용자
    echo 'alias k=kubectl' >> ~/.bashrc
    echo 'source <(kubectl completion bash)' >> ~/.bashrc
    source ~/.bashrc
    ```

    이제 `k get nodes` 처럼 쓸 수 있습니다.

!!! note "이 가이드의 명령 표기"
    가독성을 위해 문서 본문에서는 `kubectl` 을 그대로 씁니다. 별칭을 걸었다면 `kubectl` 대신 `k` 를 쳐도 됩니다.

## 5. 실습용 작업 폴더 만들기

실습에서 만드는 YAML 파일을 한 곳에 모아 둡니다.

=== "Windows (PowerShell)"

    ```powershell
    mkdir $HOME\k8s-labs
    cd $HOME\k8s-labs
    ```

=== "macOS / Linux"

    ```bash
    mkdir -p ~/k8s-labs
    cd ~/k8s-labs
    ```

## 자주 만나는 문제

| 증상 | 원인 · 해결 |
|---|---|
| `kubectl: command not found` | 터미널을 새로 열거나, PATH에 Rancher Desktop의 `bin` 경로가 있는지 확인 |
| `get nodes` 가 응답 없음(hang) | VM이 아직 부팅 중 — 앱에서 상태(초록 점)를 확인하고 잠시 후 재시도 |
| 다른 로컬 클러스터와 충돌 | `kubectl config current-context` 가 `rancher-desktop` 인지 확인 (아래) |

```bash
kubectl config current-context
kubectl config use-context rancher-desktop
```

---

준비가 끝났으면 **[1일차 Lab 1 · 도커의 한계와 쿠버네티스](../day1/01-docker-and-k8s.md)** 로 이동하세요.
