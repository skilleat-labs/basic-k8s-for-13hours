# Lab 3 · 첫 Pod 실행

명령 한 줄로 첫 **Pod** 를 띄우고, 로그를 보고, 컨테이너 안으로 들어가 봅니다. 도커를 써 봤다면 명령이 얼마나 비슷한지도 확인합니다.

!!! abstract "이 실습에서 배우는 것"
    - `kubectl run` 으로 Pod 를 실행하고 IP·노드를 확인하기
    - `describe` 의 Events 로 Pod 가 뜨는 과정 읽기
    - `logs` 와 `exec` 로 컨테이너 관찰·접속하기
    - `port-forward` 로 로컬에서 접속 테스트하기
    - 도커 CLI 와 kubectl 명령의 대응 관계

## 사전 조건

- [Lab 2](02-cluster-access.md) 을 마치고 `kubectl get nodes` 가 `Ready` 상태.

## 1. Pod 실행하기

`nginx` 이미지로 `web` 이라는 이름의 Pod 를 하나 띄웁니다.

```bash
kubectl run web --image=nginx
```

```text
pod/web created
```

곧바로 상태를 확인합니다. 이미지를 내려받는 동안 잠깐 `ContainerCreating` 이었다가 `Running` 이 됩니다.

```bash
kubectl get pods
```

```text
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          15s
```

!!! tip "상태가 계속 바뀌는 걸 보고 싶다면"
    `kubectl get pods -w` 를 쓰면 변화가 있을 때마다 새 줄이 추가됩니다(watch). 멈추려면 ++ctrl+c++ 를 누르세요.

## 2. IP 와 노드 확인하기

`-o wide` 로 Pod 의 IP 와 어느 노드에 배치됐는지 봅니다.

```bash
kubectl get pods -o wide
```

```text
NAME   READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
web    1/1     Running   0          40s   10.42.0.24   lima-rancher-desktop   <none>           <none>
```

- `IP` (예: `10.42.0.24`)는 클러스터 내부 Pod 네트워크 대역의 주소입니다.
- `NODE` 는 이 Pod 가 실제로 실행 중인 노드입니다. 단일 노드라 항상 `lima-rancher-desktop` 입니다.

!!! warning "이 IP, 외워 두세요"
    다음 실습([Lab 4](04-pod-ip-ephemeral.md))에서 이 IP 가 어떻게 되는지 확인합니다. 지금 값을 메모해 두면 좋습니다.

## 3. Events 로 뜨는 과정 읽기

Pod 가 스케줄되고 이미지를 받고 시작되기까지의 흐름은 `describe` 맨 아래 `Events` 에 남습니다.

```bash
kubectl describe pod web
```

```text
Name:         web
Namespace:    default
Node:         lima-rancher-desktop/192.168.5.15
Status:       Running
IP:           10.42.0.24
Containers:
  web:
    Image:          nginx
    State:          Running
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  50s   default-scheduler  Successfully assigned default/web to lima-rancher-desktop
  Normal  Pulling    49s   kubelet            Pulling image "nginx"
  Normal  Pulled     45s   kubelet            Successfully pulled image "nginx"
  Normal  Created    45s   kubelet            Created container web
  Normal  Started    45s   kubelet            Started container web
```

!!! info "Events 는 트러블슈팅의 1순위"
    Pod 가 안 뜰 때(`ImagePullBackOff`, `CrashLoopBackOff` 등) 원인은 거의 항상 이 `Events` 에 적혀 있습니다. "Pod 가 이상하다" 싶으면 반사적으로 `describe` 를 치세요.

## 4. 로그 보기

컨테이너가 표준출력에 남긴 로그를 봅니다. nginx 는 시작 시 설정 관련 메시지를 출력합니다.

```bash
kubectl logs web
```

```text
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/08/10 06:20:11 [notice] 1#1: using the "epoll" event method
2026/08/10 06:20:11 [notice] 1#1: nginx/1.27.0
2026/08/10 06:20:11 [notice] 1#1: start worker processes
```

!!! tip "실시간 로그"
    `kubectl logs -f web` 로 로그를 실시간으로 따라볼 수 있습니다(도커의 `docker logs -f` 와 같습니다). ++ctrl+c++ 로 종료합니다.

## 5. 컨테이너 안으로 들어가기

`exec` 로 실행 중인 컨테이너 안에서 셸을 엽니다.

```bash
kubectl exec -it web -- sh
```

셸이 열리면(`#` 프롬프트) 안에서 몇 가지를 확인해 봅니다.

```bash
hostname -i
ls
exit
```

```text
# hostname -i
10.42.0.24
# ls
bin  boot  dev  docker-entrypoint.d  etc  home  ...
# exit
```

- `hostname -i` 로 나온 IP 가 3번에서 본 Pod IP 와 같습니다. **Pod 의 IP = 그 안 컨테이너의 IP** 입니다.
- `exit` 로 다시 내 터미널로 돌아옵니다.

!!! note "`--` 의 의미"
    `kubectl exec -it web -- sh` 에서 `--` 뒤부터는 "컨테이너 안에서 실행할 명령"입니다. `-it` 는 대화형 터미널(interactive + tty)을 뜻하며, 도커의 `docker exec -it` 와 같습니다.

## 6. 접속 테스트: port-forward

Pod IP(`10.42.x.x`)는 클러스터 내부 주소라 브라우저에서 바로 못 엽니다. 임시로 로컬 포트를 Pod 포트에 연결해 테스트합니다.

```bash
kubectl port-forward pod/web 8080:80
```

```text
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

이 창은 연결이 유지되는 동안 켜 둔 채로, **새 터미널**을 하나 더 열어 접속을 확인합니다.

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

브라우저에서 `http://localhost:8080` 을 열어도 nginx 기본 페이지가 보입니다. 확인했으면 port-forward 를 실행한 터미널에서 ++ctrl+c++ 로 종료합니다.

!!! warning "Windows 의 `curl` 주의"
    PowerShell 에서 `curl` 은 `Invoke-WebRequest` 의 별칭이라 동작이 다릅니다. 위와 같은 결과를 보려면 실제 curl 인 **`curl.exe`** 를 쓰세요. 아니면 그냥 브라우저로 열어도 됩니다.

## 도커 CLI ↔ kubectl 대응표

도커를 써 봤다면 명령이 자연스럽게 겹칩니다.

| 하고 싶은 일 | 도커 | kubectl |
|---|---|---|
| 컨테이너/Pod 실행 | `docker run` | `kubectl run` |
| 실행 목록 보기 | `docker ps` | `kubectl get pods` |
| 로그 보기 | `docker logs <c>` | `kubectl logs <pod>` |
| 안으로 들어가기 | `docker exec -it <c> sh` | `kubectl exec -it <pod> -- sh` |
| 삭제 | `docker rm <c>` | `kubectl delete pod <pod>` |

!!! info "닮았지만 다릅니다"
    도커는 "이 컨테이너를 실행"하는 명령형 도구입니다. 쿠버네티스는 "이런 상태를 원한다"고 선언하면 그 상태를 **유지**해 주는 시스템입니다. 그 차이는 [Lab 6](06-deployment-selfheal.md) 의 자가복구에서 확실히 체감하게 됩니다.

## 검증

```bash
kubectl get pod web -o wide
kubectl logs web
```

- `web` Pod 가 `Running`, `READY 1/1` 이다.
- 로그에 nginx 시작 메시지가 보인다.

## 직접 해 보기

1. `kubectl exec -it web -- sh` 로 들어가서 `cat /usr/share/nginx/html/index.html` 로 nginx 기본 페이지의 원본 HTML 을 직접 확인해 보세요.
2. `kubectl run box --image=busybox -it --rm -- sh` 로 일회용 busybox Pod 를 띄운 뒤, 그 안에서 `wget -qO- 10.42.0.24`(Lab 3 에서 본 web 의 IP)를 실행해 보세요. Pod 끼리는 IP 로 직접 통신됨을 확인할 수 있습니다. (`--rm` 이라 `exit` 하면 Pod 가 자동 삭제됩니다.)

!!! failure "자주 만나는 오류"
    **증상**: `Error from server (AlreadyExists): pods "web" already exists`
    **원인**: 같은 이름의 Pod 가 이미 있음.
    **해결**: `kubectl delete pod web` 로 지운 뒤 다시 실행하거나, 다른 이름을 사용.

    ---

    **증상**: port-forward 후 `curl` 이 `Connection refused`
    **원인**: port-forward 터미널을 닫았거나, `curl` 을 같은 터미널에서 실행해 port-forward 가 중단됨.
    **해결**: port-forward 는 켜 둔 채, **새 터미널**에서 `curl`(Windows 는 `curl.exe`)을 실행.

    ---

    **증상**: `unable to upgrade connection: pod does not have a host assigned` 또는 exec 실패
    **원인**: Pod 가 아직 `Running` 이 아님.
    **해결**: `kubectl get pod web` 으로 `Running` 이 될 때까지 기다린 뒤 재시도.

## 정리

다음 실습에서 이어 쓰므로 지금은 지우지 않아도 됩니다. 만약 처음부터 다시 하고 싶다면 아래로 삭제합니다.

```bash
kubectl delete pod web
```

---

다음: [Lab 4 · Pod IP는 휘발성](04-pod-ip-ephemeral.md)
