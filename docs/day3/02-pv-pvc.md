# Lab 2 · PV·PVC·StorageClass

앞 실습에서 Pod 를 삭제하자 데이터가 사라졌습니다. 이번에는 **영속 볼륨(PersistentVolume)** 을 MySQL 에 붙여, Pod 가 죽고 새로 태어나도 데이터가 그대로 남는 것을 확인합니다. 핵심 도구는 개발자가 "이만큼의 저장소를 주세요"라고 요청하는 **PVC(PersistentVolumeClaim)** 와, 그 요청을 받아 실제 볼륨을 자동으로 만들어 주는 **StorageClass** 입니다.

!!! abstract "이 실습에서 배우는 것"
    - PV·PVC·StorageClass 세 가지가 각각 무슨 역할을 하는지
    - PVC 로 저장소를 **요청**하면 StorageClass 가 PV 를 **동적 프로비저닝**하는 흐름
    - MySQL 데이터 디렉터리(`/var/lib/mysql`)에 볼륨을 마운트하는 법
    - Pod 를 삭제해도 데이터가 유지되는 것을 직접 확인
    - k3s 의 기본 StorageClass `local-path`

## 사전 조건

- Lab 1 을 마쳤고, `mysql-novolume` 리소스는 정리된 상태여야 합니다.
- 클러스터에 기본 StorageClass 가 있는지 확인합니다.

```bash
kubectl get storageclass
```

```text
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  12d
```

!!! note "`local-path` 가 기본 StorageClass"
    이름 옆의 `(default)` 표시가 핵심입니다. PVC 에서 `storageClassName` 을 생략하면 이 기본 StorageClass 가 자동으로 쓰입니다. k3s 의 `local-path` 프로비저너는 노드의 로컬 디스크에 디렉터리를 만들어 볼륨으로 제공합니다. `VOLUMEBINDINGMODE` 가 `WaitForFirstConsumer` 라서, PVC 를 만들어도 **실제로 쓰는 Pod 가 뜨기 전까지는 PV 가 생성되지 않습니다.**

## 1. PVC 만들기

저장소를 요청하는 PVC 매니페스트를 작성합니다. `storageClassName` 을 생략했으므로 기본 `local-path` 가 사용됩니다.

`mysql-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

- `accessModes: ReadWriteOnce` (RWO): 하나의 노드에서 읽기·쓰기로 마운트합니다. 단일 노드 클러스터와 대부분의 데이터베이스에 적합합니다.
- `resources.requests.storage: 1Gi`: 1기가바이트를 요청합니다.

적용합니다.

```bash
kubectl apply -f mysql-pvc.yaml
```

```text
persistentvolumeclaim/mysql-pvc created
```

상태를 봅니다.

```bash
kubectl get pvc
```

```text
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-pvc   Pending                                      local-path     8s
```

!!! info "왜 `Pending` 인가요?"
    아직 이 PVC 를 사용하는 Pod 가 없기 때문입니다. `local-path` 는 `WaitForFirstConsumer` 모드라, **실제로 볼륨을 쓸 Pod 가 스케줄되는 순간** PV 를 만들어 바인딩합니다. 다음 단계에서 MySQL 을 붙이면 `Bound` 로 바뀝니다.

## 2. PVC 를 붙인 MySQL 배포하기

이제 MySQL Deployment 에 볼륨을 마운트합니다. Lab 1 과의 차이는 `volumeMounts` 와 `volumes` 두 블록이 추가된 것뿐입니다.

`mysql-pv.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-pv
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-pv
  template:
    metadata:
      labels:
        app: mysql-pv
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "rootpw123"
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-data
          persistentVolumeClaim:
            claimName: mysql-pvc
```

- `volumeMounts.mountPath: /var/lib/mysql`: MySQL 이 실제 데이터를 저장하는 디렉터리에 볼륨을 연결합니다.
- `volumes.persistentVolumeClaim.claimName: mysql-pvc`: 앞에서 만든 PVC 를 이 볼륨의 실체로 지정합니다.

적용합니다.

```bash
kubectl apply -f mysql-pv.yaml
```

```text
deployment.apps/mysql-pv created
```

Pod 가 뜬 뒤 PVC 상태를 다시 봅니다.

```bash
kubectl get pods -l app=mysql-pv
kubectl get pv,pvc
```

```text
NAME                        READY   STATUS    RESTARTS   AGE
mysql-pv-7d4c8f9b6d-abcde   1/1     Running   0          50s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   AGE
persistentvolume/pvc-1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d   1Gi        RWO            Delete           Bound    default/mysql-pvc   local-path     20s

NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/mysql-pvc   Bound    pvc-1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d   1Gi        RWO            local-path     3m
```

!!! success "PV 가 자동으로 생겼습니다"
    우리가 PV 를 직접 만들지 않았는데도 `persistentvolume/pvc-...` 가 생겼습니다. Pod 가 뜨자 StorageClass 가 PV 를 **동적으로 만들어(dynamic provisioning)** PVC 에 바인딩(`Bound`)한 것입니다. 이것이 StorageClass 를 쓰는 이유입니다.

## 3. 데이터 넣기

Lab 1 과 똑같이 데이터를 한 줄 넣습니다.

=== "Windows (PowerShell)"
    ```powershell
    $POD = kubectl get pod -l app=mysql-pv -o jsonpath='{.items[0].metadata.name}'
    kubectl exec -it $POD -- mysql -uroot -prootpw123 -e "CREATE DATABASE shop; USE shop; CREATE TABLE users(id INT, name VARCHAR(20)); INSERT INTO users VALUES (1, 'alice');"
    ```

=== "macOS / Linux"
    ```bash
    POD=$(kubectl get pod -l app=mysql-pv -o jsonpath='{.items[0].metadata.name}')
    kubectl exec -it "$POD" -- mysql -uroot -prootpw123 -e "CREATE DATABASE shop; USE shop; CREATE TABLE users(id INT, name VARCHAR(20)); INSERT INTO users VALUES (1, 'alice');"
    ```

잘 들어갔는지 확인합니다.

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

## 4. Pod 삭제 후 데이터 유지 확인하기

이제 Lab 1 에서 데이터를 앗아갔던 바로 그 명령, `delete pod` 를 다시 실행합니다.

```bash
kubectl delete pod "$POD"
```

```text
pod "mysql-pv-7d4c8f9b6d-abcde" deleted
```

새 Pod 가 뜰 때까지 기다린 뒤, 이름을 다시 담고 데이터를 조회합니다.

=== "Windows (PowerShell)"
    ```powershell
    $POD = kubectl get pod -l app=mysql-pv -o jsonpath='{.items[0].metadata.name}'
    kubectl exec -it $POD -- mysql -uroot -prootpw123 -e "SELECT * FROM shop.users;"
    ```

=== "macOS / Linux"
    ```bash
    POD=$(kubectl get pod -l app=mysql-pv -o jsonpath='{.items[0].metadata.name}')
    kubectl exec -it "$POD" -- mysql -uroot -prootpw123 -e "SELECT * FROM shop.users;"
    ```

```text
+------+-------+
| id   | name  |
+------+-------+
|    1 | alice |
+------+-------+
```

!!! success "데이터가 살아남았습니다"
    Pod 가 완전히 새로 태어났는데도 `alice` 가 그대로 있습니다. 데이터가 컨테이너가 아니라 **PVC 가 가리키는 영속 볼륨**에 저장되었기 때문입니다. 새 Pod 는 같은 PVC 를 다시 마운트해 이전 데이터를 이어받았습니다. 이것이 Lab 1 과의 결정적 차이입니다.

## PV · PVC · StorageClass 역할 구분

세 가지가 헷갈리기 쉬우니 표로 정리합니다.

| 리소스 | 누가 만드나 | 역할 | 비유 |
|---|---|---|---|
| **StorageClass** | 클러스터 관리자(k3s가 기본 제공) | 어떤 방식으로 볼륨을 만들지 정의하는 "템플릿" | 저장소 자판기의 종류 |
| **PVC** (PersistentVolumeClaim) | 개발자(앱 담당) | "이만큼 저장소를 주세요"라는 **요청** | 자판기에 넣는 주문서 |
| **PV** (PersistentVolume) | StorageClass가 자동 생성 | 실제로 할당된 저장 공간 | 자판기에서 나온 실물 |

핵심 흐름은 이렇습니다.

1. 개발자가 **PVC** 로 저장소를 요청한다.
2. **StorageClass** 가 요청을 받아 **PV** 를 자동으로 만든다(동적 프로비저닝).
3. PVC 와 PV 가 **Bound(연결)** 된다.
4. Pod 가 PVC 를 볼륨으로 마운트해 데이터를 읽고 쓴다.

## 검증

- `kubectl get pvc` 의 `STATUS` 가 `Bound` 이다.
- `kubectl get pv` 에 `pvc-...` 이름의 PV 가 자동 생성되어 있다.
- Pod 를 삭제하고 새 Pod 에서 조회해도 `alice` 데이터가 유지된다.

## 직접 해 보기

1. `kubectl describe pvc mysql-pvc` 를 실행해, `Events` 에 `Provisioning` → `ProvisioningSucceeded` 흐름이 기록된 것을 찾아보세요.
2. `kubectl get pvc mysql-pvc -o yaml` 로 `spec.volumeName` 필드를 확인해, PVC 가 어떤 PV 에 바인딩되었는지 보세요. `kubectl get pv` 의 이름과 일치할 것입니다.
3. Deployment 를 삭제(`kubectl delete -f mysql-pv.yaml`)해도 PVC 와 데이터는 남습니다. 다시 `kubectl apply -f mysql-pv.yaml` 로 배포한 뒤 조회하면 데이터가 여전히 있는지 확인해 보세요.

!!! failure "자주 만나는 오류"
    **증상**: PVC 가 계속 `Pending` 이고 Pod 도 `Pending`
    **원인**: 기본 StorageClass 가 없거나, PVC 를 쓰는 Pod 가 아직 없음.
    **해결**: `kubectl get storageclass` 로 `(default)` 표시를 확인합니다. `local-path` 는 Pod 가 스케줄되어야 PV 를 만드므로, Deployment 가 정상 배포됐는지 `kubectl describe pod` 로 확인하세요.

    ---

    **증상**: Pod 가 `CrashLoopBackOff`, 로그에 `Data Dictionary initialization` 오류
    **원인**: 다른 MySQL 버전이 쓰던 볼륨을 재사용하는 등 데이터 디렉터리 충돌.
    **해결**: 실습용이라면 `kubectl delete pvc mysql-pvc` 로 볼륨을 비우고 처음부터 다시 만듭니다. (실무에서는 절대 함부로 지우면 안 됩니다.)

## 정리

다음 실습(StatefulSet)에서는 새 리소스를 쓰므로 여기서 정리합니다. PVC 를 지우면 `local-path` 의 회수 정책(`Delete`)에 따라 PV 와 실제 데이터도 함께 삭제됩니다.

```bash
kubectl delete -f mysql-pv.yaml
kubectl delete -f mysql-pvc.yaml
```

```text
deployment.apps "mysql-pv" deleted
persistentvolumeclaim "mysql-pvc" deleted
```

PV 가 사라졌는지 확인합니다.

```bash
kubectl get pv
```

```text
No resources found
```

---

다음: [Lab 3 · StatefulSet](03-statefulset.md)
