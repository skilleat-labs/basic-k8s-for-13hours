# Lab 1 · 데이터 소실 재현

지금까지 배포한 앱들은 상태가 없었습니다(stateless). 하지만 데이터베이스처럼 **데이터를 저장해야 하는 앱**은 이야기가 다릅니다. 이번 실습에서는 볼륨 없이 MySQL 을 배포하고 데이터를 넣은 뒤, Pod 를 한 번 삭제해 봅니다. 그러면 데이터가 **흔적도 없이 사라지는** 것을 직접 확인하게 됩니다. 이 뼈아픈 경험이 다음 실습(PV/PVC)의 필요성을 몸으로 이해시켜 줍니다.

!!! abstract "이 실습에서 배우는 것"
    - 컨테이너 파일시스템은 **임시(ephemeral)** 라는 사실
    - `kubectl exec` 로 실행 중인 컨테이너 안에 들어가 명령 실행하기
    - Deployment 가 Pod 를 다시 만들 때 이전 데이터를 이어받지 못하는 이유
    - `emptyDir` 볼륨조차 Pod 수명과 함께 사라진다는 점

## 사전 조건

- 1·2일차를 마쳤거나, `kubectl apply -f` 와 Deployment 개념에 익숙해야 합니다.
- 클러스터가 정상인지 먼저 확인합니다.

```bash
kubectl get nodes
```

## 1. 볼륨 없는 MySQL 배포하기

아래 매니페스트를 파일로 저장합니다. 스토리지를 전혀 붙이지 않은, 순수하게 컨테이너 파일시스템에만 데이터를 쓰는 MySQL 입니다.

`mysql-novolume.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-novolume
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-novolume
  template:
    metadata:
      labels:
        app: mysql-novolume
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "rootpw123"
          ports:
            - containerPort: 3306
```

적용합니다.

```bash
kubectl apply -f mysql-novolume.yaml
```

```text
deployment.apps/mysql-novolume created
```

Pod 가 `Running` 이 될 때까지 기다립니다. MySQL 은 처음 기동할 때 초기화 과정이 있어 30초~1분 정도 걸릴 수 있습니다.

```bash
kubectl get pods -l app=mysql-novolume -w
```

```text
NAME                              READY   STATUS    RESTARTS   AGE
mysql-novolume-6c9f7b8d4c-abcde   1/1     Running   0          45s
```

!!! tip "`-w` 는 watch"
    `-w` 를 붙이면 상태 변화를 실시간으로 지켜볼 수 있습니다. `1/1 Running` 이 되면 `Ctrl+C` 로 빠져나옵니다.

## 2. 데이터 넣기

Pod 이름은 매번 다르므로 변수에 담아두면 편합니다.

=== "Windows (PowerShell)"
    ```powershell
    $POD = kubectl get pod -l app=mysql-novolume -o jsonpath='{.items[0].metadata.name}'
    echo $POD
    ```

=== "macOS / Linux"
    ```bash
    POD=$(kubectl get pod -l app=mysql-novolume -o jsonpath='{.items[0].metadata.name}')
    echo $POD
    ```

이제 컨테이너 안의 `mysql` 클라이언트로 접속해 데이터베이스와 테이블을 만들고 데이터를 한 줄 넣습니다.

=== "Windows (PowerShell)"
    ```powershell
    kubectl exec -it $POD -- mysql -uroot -prootpw123 -e "CREATE DATABASE shop; USE shop; CREATE TABLE users(id INT, name VARCHAR(20)); INSERT INTO users VALUES (1, 'alice');"
    ```

=== "macOS / Linux"
    ```bash
    kubectl exec -it "$POD" -- mysql -uroot -prootpw123 -e "CREATE DATABASE shop; USE shop; CREATE TABLE users(id INT, name VARCHAR(20)); INSERT INTO users VALUES (1, 'alice');"
    ```

```text
mysql: [Warning] Using a password on the command line interface can be insecure.
```

!!! note "비밀번호 경고는 정상"
    명령줄에 비밀번호를 직접 쓰면 위 경고가 뜹니다. 실습에서는 무시해도 됩니다. 실무에서는 3일차 Lab 6에서 배울 Secret 으로 관리합니다.

데이터가 잘 들어갔는지 조회합니다.

=== "Windows (PowerShell)"
    ```powershell
    kubectl exec -it $POD -- mysql -uroot -prootpw123 -e "SELECT * FROM shop.users;"
    ```

=== "macOS / Linux"
    ```bash
    kubectl exec -it "$POD" -- mysql -uroot -prootpw123 -e "SELECT * FROM shop.users;"
    ```

```text
+------+-------+
| id   | name  |
+------+-------+
|    1 | alice |
+------+-------+
```

`alice` 가 잘 저장되어 있습니다. 여기까지는 아무 문제 없어 보입니다.

## 3. Pod 삭제하기

이제 결정적인 순간입니다. Pod 를 삭제해 봅니다. Deployment 가 관리하고 있으므로, 삭제하면 쿠버네티스가 **곧바로 새 Pod 를 하나 만들어** 줍니다(자가복구).

```bash
kubectl delete pod "$POD"
```

```text
pod "mysql-novolume-6c9f7b8d4c-abcde" deleted
```

새 Pod 가 뜰 때까지 기다립니다.

```bash
kubectl get pods -l app=mysql-novolume
```

```text
NAME                              READY   STATUS    RESTARTS   AGE
mysql-novolume-6c9f7b8d4c-fghij   1/1     Running   0          40s
```

!!! info "이름 뒷부분이 바뀐 것에 주목"
    `...-abcde` 였던 Pod 가 `...-fghij` 로 바뀌었습니다. 이름이 다르다는 건 **완전히 다른 새 컨테이너**라는 뜻입니다. 이전 컨테이너의 파일시스템은 함께 사라졌습니다.

## 4. 데이터 소실 확인하기

새 Pod 이름을 다시 담고, 아까 넣은 데이터를 조회해 봅니다.

=== "Windows (PowerShell)"
    ```powershell
    $POD = kubectl get pod -l app=mysql-novolume -o jsonpath='{.items[0].metadata.name}'
    kubectl exec -it $POD -- mysql -uroot -prootpw123 -e "SELECT * FROM shop.users;"
    ```

=== "macOS / Linux"
    ```bash
    POD=$(kubectl get pod -l app=mysql-novolume -o jsonpath='{.items[0].metadata.name}')
    kubectl exec -it "$POD" -- mysql -uroot -prootpw123 -e "SELECT * FROM shop.users;"
    ```

```text
ERROR 1049 (42000) at line 1: Unknown database 'shop'
```

데이터베이스 `shop` 자체가 **사라졌습니다.** `alice` 는 물론이고 데이터베이스와 테이블 구조까지 전부 없어졌습니다. 새 컨테이너는 `mysql:8.0` 이미지 그대로, 아무것도 없는 깨끗한 상태로 시작했기 때문입니다.

!!! danger "이것이 상태 없는 컨테이너의 본질"
    컨테이너의 파일시스템은 컨테이너가 살아 있는 동안만 존재하는 **임시 저장소**입니다. 컨테이너가 죽으면(재시작·삭제·노드 이동) 그 안에 쓴 모든 데이터는 함께 사라집니다. 데이터베이스, 사용자 업로드 파일, 로그 등 **살아남아야 하는 데이터**는 반드시 외부 볼륨에 저장해야 합니다.

## 검증

아래를 확인하면 이 실습을 제대로 마친 것입니다.

- 데이터를 넣은 직후에는 `SELECT` 가 `alice` 를 보여줬다.
- Pod 삭제 후 새 Pod 에서는 `Unknown database 'shop'` 오류가 났다.
- Pod 이름의 뒷부분(해시)이 삭제 전후로 달라졌다.

## 직접 해 보기

1. `emptyDir` 볼륨을 붙여도 결과가 같은지 확인해 보세요. 매니페스트의 `containers` 아래에 `volumeMounts` 로 `/var/lib/mysql` 을 마운트하고, `volumes:` 에 `emptyDir: {}` 를 추가한 뒤 같은 실험을 반복합니다. `emptyDir` 역시 **Pod 가 사라지면 함께 삭제**되므로 결과는 동일합니다. (왜 그런지 생각해 보세요.)
2. `kubectl exec -it $POD -- bash` 로 컨테이너 안에 직접 들어가, `ls /var/lib/mysql` 로 MySQL 데이터 파일이 어디에 저장되는지 살펴보세요. 다음 실습에서 바로 이 경로에 볼륨을 붙이게 됩니다.

!!! failure "자주 만나는 오류"
    **증상**: `exec` 시 `MySQL server has gone away` 또는 접속 거부
    **원인**: MySQL 이 아직 초기화 중.
    **해결**: `kubectl logs $POD` 로 `ready for connections` 메시지가 나올 때까지 기다린 뒤 다시 시도.

    ---

    **증상**: `error: unable to upgrade connection: container not found`
    **원인**: 변수에 담긴 Pod 이름이 이미 삭제된 옛 Pod.
    **해결**: `POD=...` 명령을 다시 실행해 현재 살아 있는 Pod 이름을 새로 담습니다.

## 정리

다음 실습에서 이어서 쓰지 않으므로 삭제합니다.

```bash
kubectl delete -f mysql-novolume.yaml
```

```text
deployment.apps "mysql-novolume" deleted
```

---

다음: [Lab 2 · PV·PVC·StorageClass](02-pv-pvc.md)
