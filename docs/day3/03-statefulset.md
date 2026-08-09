# Lab 3 · StatefulSet

Deployment 로 여러 개의 복제본을 띄우면 Pod 이름은 무작위 해시가 붙고, 볼륨은 모두 같은 것을 공유하려 합니다. 하지만 데이터베이스 클러스터처럼 **각 Pod 가 고유한 신원(이름)과 전용 저장소**를 가져야 하는 워크로드가 있습니다. 이번 실습에서는 **StatefulSet** 으로 그런 워크로드를 배포하고, Pod 마다 안정적인 이름(`web-0`, `web-1`)과 전용 PVC 가 붙는 것을 확인합니다.

!!! abstract "이 실습에서 배우는 것"
    - StatefulSet 이 부여하는 **안정적이고 순차적인 Pod 이름**(`web-0`, `web-1`)
    - `volumeClaimTemplates` 로 Pod 마다 **전용 PVC** 를 자동 생성하는 법
    - StatefulSet 이 필요로 하는 **헤드리스 Service**(`clusterIP: None`)
    - Deployment 와 StatefulSet 의 차이

## 사전 조건

- Lab 2 를 마쳐 PV/PVC 개념에 익숙해야 합니다.
- 이전 실습 리소스가 정리된 상태인지 확인합니다.

```bash
kubectl get pods
kubectl get pvc
```

## 1. 헤드리스 Service 만들기

StatefulSet 은 각 Pod 에 고유한 네트워크 이름(DNS)을 주기 위해 **헤드리스(headless) Service** 를 함께 씁니다. 헤드리스 Service 는 `clusterIP: None` 으로 지정해, 하나의 대표 IP 대신 각 Pod 를 직접 가리키는 DNS 레코드를 만듭니다.

`web-headless.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None
  selector:
    app: web
  ports:
    - port: 80
      name: http
```

- `clusterIP: None` 이 이 Service 를 "헤드리스"로 만듭니다.
- 이 이름(`web`)은 뒤에서 StatefulSet 의 `serviceName` 과 일치시켜야 합니다.

적용합니다.

```bash
kubectl apply -f web-headless.yaml
```

```text
service/web created
```

## 2. StatefulSet 배포하기

nginx 를 2개 복제본으로 띄우되, `volumeClaimTemplates` 로 Pod 마다 전용 PVC 를 만들도록 합니다.

`web-statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: web
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
              name: http
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

- `serviceName: web`: 1단계에서 만든 헤드리스 Service 이름과 일치해야 합니다.
- `volumeClaimTemplates`: **Pod 마다** 이 템플릿으로 PVC 를 하나씩 자동 생성합니다. Deployment 의 `volumes` 와 달리, 복제본이 볼륨을 공유하지 않고 각자 전용 볼륨을 갖습니다.

적용합니다.

```bash
kubectl apply -f web-statefulset.yaml
```

```text
statefulset.apps/web created
```

## 3. 안정적인 Pod 이름 관찰하기

Pod 목록을 봅니다.

```bash
kubectl get pods -l app=web
```

```text
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          40s
web-1   1/1     Running   0          25s
```

!!! success "이름이 무작위 해시가 아닙니다"
    Deployment 의 Pod 는 `web-7d4c8f9b6d-abcde` 처럼 무작위 해시가 붙었습니다. 하지만 StatefulSet 은 `web-0`, `web-1` 처럼 **순번이 붙은 안정적인 이름**을 줍니다. Pod 가 죽었다 다시 떠도 이름은 그대로 `web-0` 입니다. 데이터베이스의 "1번 노드", "2번 노드"처럼 신원이 중요한 워크로드에 필수적인 특성입니다.

!!! info "순차 생성"
    StatefulSet 은 `web-0` 이 완전히 `Running` 이 된 뒤에야 `web-1` 을 만듭니다. Deployment 가 복제본을 한꺼번에 병렬로 만드는 것과 다릅니다. 순서가 중요한 클러스터형 앱(예: 마스터 먼저, 그다음 복제본)에 맞춰진 동작입니다.

## 4. Pod 별 전용 PVC 확인하기

이제 볼륨을 봅니다.

```bash
kubectl get pvc
```

```text
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pvc-aaaa1111-...                           1Gi        RWO            local-path     1m
www-web-1   Bound    pvc-bbbb2222-...                           1Gi        RWO            local-path     50s
```

!!! success "Pod 마다 전용 PVC 가 생겼습니다"
    `www-web-0`, `www-web-1` 처럼 **`<볼륨템플릿이름>-<Pod이름>`** 규칙으로 PVC 가 Pod 수만큼 자동 생성됐습니다. 각 Pod 는 자기 전용 볼륨만 사용합니다. 이 PVC 들은 Pod 나 StatefulSet 을 삭제해도 **자동으로 지워지지 않아**, 데이터가 안전하게 보존됩니다.

각 Pod 가 정말 다른 볼륨을 쓰는지 확인해 봅니다. `web-0` 에만 파일을 하나 만들어 보겠습니다.

```bash
kubectl exec web-0 -- sh -c "echo 'hello from web-0' > /usr/share/nginx/html/id.txt"
kubectl exec web-0 -- cat /usr/share/nginx/html/id.txt
```

```text
hello from web-0
```

`web-1` 에는 그 파일이 없어야 정상입니다.

```bash
kubectl exec web-1 -- cat /usr/share/nginx/html/id.txt
```

```text
cat: /usr/share/nginx/html/id.txt: No such file or directory
command terminated with exit code 1
```

`web-0` 의 볼륨과 `web-1` 의 볼륨이 완전히 분리되어 있음이 확인됩니다.

## Deployment vs StatefulSet 비교

| 항목 | Deployment | StatefulSet |
|---|---|---|
| Pod 이름 | 무작위 해시 (`web-7d4c...-abcde`) | 순번 고정 (`web-0`, `web-1`) |
| 생성·삭제 순서 | 병렬(순서 없음) | 순차(`0` → `1` → …) |
| 스토리지 | 복제본이 볼륨 공유 경향 | Pod 마다 전용 PVC (`volumeClaimTemplates`) |
| 네트워크 신원 | 없음(Service가 대표 IP 제공) | Pod별 안정적 DNS(헤드리스 Service) |
| 대표 용도 | 무상태 앱(웹 서버, API) | 상태 있는 앱(DB, 메시지 큐, 분산 저장소) |

!!! tip "언제 무엇을 쓰나"
    대부분의 웹·API 서버는 **Deployment** 면 충분합니다. StatefulSet 은 "각 인스턴스가 고유한 신원과 자기만의 데이터를 가져야 할 때"만 씁니다. 애매하면 Deployment + PVC 로 시작하는 게 간단합니다.

## 검증

- `kubectl get pods -l app=web` 에 `web-0`, `web-1` 이름이 보인다.
- `kubectl get pvc` 에 `www-web-0`, `www-web-1` 이 각각 `Bound` 다.
- `web-0` 에 만든 파일이 `web-1` 에는 없다(볼륨 분리).

## 직접 해 보기

1. `kubectl delete pod web-0` 로 `web-0` 을 지운 뒤 다시 목록을 보세요. 새로 뜬 Pod 이름도 여전히 `web-0` 이고, `id.txt` 파일이 그대로 남아 있는지 확인해 보세요(전용 PVC 재연결).
2. `kubectl scale statefulset web --replicas=3` 으로 늘려 보세요. `web-2` 와 `www-web-2` 가 새로 생기는 순서를 관찰합니다. 다시 `--replicas=2` 로 줄이면 `web-2` 는 사라지지만 **PVC `www-web-2` 는 남는지** 확인해 보세요.
3. `kubectl exec web-0 -- nslookup web-1.web` 을 실행해, 헤드리스 Service 덕분에 Pod 별 DNS 이름(`web-1.web`)이 해석되는지 살펴보세요. (`nslookup` 이 없으면 이미지에 따라 실패할 수 있습니다.)

!!! failure "자주 만나는 오류"
    **증상**: StatefulSet Pod 가 `Pending` 이고 `describe` 에 `pod has unbound immediate PersistentVolumeClaims`
    **원인**: 기본 StorageClass 가 없어 `volumeClaimTemplates` 의 PVC 가 바인딩되지 못함.
    **해결**: `kubectl get storageclass` 로 `local-path (default)` 를 확인합니다.

    ---

    **증상**: `web-1` 이 만들어지지 않고 `web-0` 만 계속 대기
    **원인**: `web-0` 이 아직 `Running`·`Ready` 가 아님. StatefulSet 은 순차 생성이라 앞 Pod 가 준비돼야 다음을 만듭니다.
    **해결**: `kubectl describe pod web-0` 으로 `web-0` 이 멈춘 이유를 먼저 해결합니다.

## 정리

StatefulSet 을 지워도 PVC 는 남으므로 **PVC 도 함께 삭제**해야 볼륨이 정리됩니다.

```bash
kubectl delete -f web-statefulset.yaml
kubectl delete -f web-headless.yaml
kubectl delete pvc www-web-0 www-web-1
```

```text
statefulset.apps "web" deleted
service "web" deleted
persistentvolumeclaim "www-web-0" deleted
persistentvolumeclaim "www-web-1" deleted
```

!!! note "3번 문제로 `www-web-2` 를 만들었다면"
    `kubectl get pvc` 로 남은 PVC 가 없는지 확인하고, 있다면 이름을 지정해 함께 삭제하세요.

---

다음: [Lab 4 · securityContext 하드닝](04-securitycontext.md)
