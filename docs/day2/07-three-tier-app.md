# Lab 7 · 3-tier 앱 배포

지금까지 배운 조각들 — Deployment, ClusterIP Service, Ingress — 을 모아 **프론트엔드 · 백엔드 · 데이터베이스** 로 이루어진 전형적인 3계층(3-tier) 애플리케이션을 통째로 배포합니다. Docker Compose로 여러 컨테이너를 함께 띄워 본 적이 있다면, 그것이 쿠버네티스에서 어떻게 표현되는지 매핑해 봅니다. 또 쿠버네티스에 `depends_on` 이 없는데도 앱이 어떻게 스스로 수렴하는지, readiness/liveness probe가 왜 필요한지 살펴봅니다.

!!! abstract "이 실습에서 배우는 것"
    - 프론트·백엔드·DB 3계층을 하나의 앱으로 배포한다
    - Docker Compose 구성을 쿠버네티스 리소스로 매핑하는 감각을 익힌다
    - 쿠버네티스에 `depends_on` 이 없는 이유(선언형 + 재시도 + 자가수렴)를 이해한다
    - `readinessProbe` / `livenessProbe` 의 역할을 맛본다

## 사전 조건

- Rancher Desktop의 k3s 클러스터가 실행 중이다
- 이전 Lab의 자원(`web-lb`, `app-ingress` 등)을 정리해 80 포트가 비어 있다

```bash
kubectl get svc,ingress
```

## 1. 계층별 역할 정리

만들 앱의 구성입니다.

| 계층 | 이미지 | 역할 | 노출 방식 |
|---|---|---|---|
| frontend | `nginx` | 사용자 진입 웹 | Ingress `/` 로 외부 노출 |
| backend | `hashicorp/http-echo` | API 응답 | Ingress `/api`, 내부 Service |
| db | `mysql:8.0` | 데이터 저장 | ClusterIP(내부 전용, 외부 노출 안 함) |

DB는 절대 외부로 열지 않고 클러스터 내부에서만 접근한다는 점이 중요합니다.

## 2. 데이터베이스(db) 배포

가장 안쪽 계층인 DB부터 만듭니다. 아래를 `db.yaml` 로 저장하세요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "temp-root-pw"     # 실습용 임시값. 실제로는 Secret으로!
            - name: MYSQL_DATABASE
              value: "appdb"
          ports:
            - containerPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  type: ClusterIP
  selector:
    app: db
  ports:
    - port: 3306
      targetPort: 3306
```

```bash
kubectl apply -f db.yaml
```

!!! warning "비밀번호를 YAML에 직접 넣지 마세요"
    여기서는 실습 단순화를 위해 `MYSQL_ROOT_PASSWORD` 를 평문으로 넣었지만, 이는 나쁜 습관입니다. 다음 Lab 8(ConfigMap)·Lab 9(Secret)에서 이런 값을 매니페스트 밖으로 분리하는 법을 배웁니다.

## 3. 백엔드(backend) 배포 — probe 포함

백엔드에는 **readinessProbe** 를 붙여, 컨테이너가 요청을 받을 준비가 됐을 때만 Endpoints에 등록되도록 합니다. 아래를 `backend.yaml` 로 저장하세요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: http-echo
          image: hashicorp/http-echo
          args:
            - "-text=hello from backend (connected to db)"
            - "-listen=:5678"
          ports:
            - containerPort: 5678
          readinessProbe:
            httpGet:
              path: /
              port: 5678
            initialDelaySeconds: 2
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 5678
      targetPort: 5678
```

```bash
kubectl apply -f backend.yaml
```

!!! info "readinessProbe와 livenessProbe"
    - **readinessProbe**: "지금 트래픽을 받아도 되나?"를 검사합니다. 실패하면 Pod는 살아 있어도 **Endpoints에서 빠져** 트래픽을 안 받습니다. 부팅에 시간이 걸리는 앱(DB 연결 대기 등)에 유용합니다.
    - **livenessProbe**: "이 컨테이너가 죽었나(멈췄나)?"를 검사합니다. 실패하면 쿠버네티스가 컨테이너를 **재시작** 합니다.

## 4. 프론트엔드(frontend)와 Ingress 배포

마지막으로 프론트엔드와 진입점 Ingress를 만듭니다. 아래를 `frontend.yaml` 로 저장하세요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: traefik
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 5678
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

```bash
kubectl apply -f frontend.yaml
```

## 5. Compose → 쿠버네티스 매핑

Docker Compose로 같은 앱을 정의했다면 이렇게 대응됩니다.

| Docker Compose | 쿠버네티스 | 설명 |
|---|---|---|
| `services:` 한 항목 | `Deployment` | 컨테이너를 몇 개 띄우고 관리 |
| `image:` | `spec.template...image` | 사용할 이미지 |
| `ports:` (호스트 노출) | `Service` + `Ingress` | 내부 주소 부여 + 외부 노출 |
| `environment:` | `env:` (뒤에선 ConfigMap/Secret) | 환경변수 주입 |
| 서비스 이름으로 통신 | Service 이름 = DNS 이름 | `backend`, `db` 로 서로 호출 |
| `depends_on:` | **없음** (아래 참고) | 기동 순서 강제 안 함 |
| `deploy.replicas` | `spec.replicas` | 복제본 수 |

!!! info "쿠버네티스에 depends_on이 없는 이유"
    Compose의 `depends_on` 은 "DB가 뜬 다음 백엔드를 띄워라"처럼 **기동 순서** 를 강제합니다. 쿠버네티스는 이런 명령형 순서 대신 **선언형 + 자가수렴** 방식을 씁니다.

    - 모든 리소스를 한꺼번에 만들고, 아직 준비 안 된 의존 대상이 있으면 컨테이너가 잠깐 실패합니다.
    - 실패한 Pod는 `CrashLoopBackOff` 로 표시되며 쿠버네티스가 **자동으로 재시도** 합니다.
    - 의존 대상(예: DB)이 준비되면 다음 재시도에서 성공하고, 시스템 전체가 **원하는 상태로 수렴** 합니다.

    즉 "순서를 맞춰 주는" 대신 "될 때까지 다시 시도"합니다. 그래서 순서를 신경 쓰지 않고도 결국 정상 상태에 도달합니다(단, DB 준비를 앱에서 우아하게 기다리려면 readinessProbe나 재시도 로직을 함께 쓰는 것이 좋습니다).

## 6. 전체 상태 확인

한 번에 전체 자원을 봅니다.

```bash
kubectl get all
```

```text
NAME                            READY   STATUS    RESTARTS   AGE
pod/backend-5c8f7d9b6-2xk9p     1/1     Running   0          1m
pod/db-6b7d8f9c4d-8vt4m         1/1     Running   0          2m
pod/frontend-79c4d8f6b-hp8kt    1/1     Running   0          1m

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/backend    ClusterIP   10.43.10.5      <none>        5678/TCP   1m
service/db         ClusterIP   10.43.22.18     <none>        3306/TCP   2m
service/frontend   ClusterIP   10.43.99.40     <none>        80/TCP     1m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend    1/1     1            1           1m
deployment.apps/db         1/1     1            1           2m
deployment.apps/frontend   1/1     1            1           1m
```

`mysql` 은 초기화에 몇십 초 걸릴 수 있으니 `db` Pod가 `Running` 이 될 때까지 기다리세요. 이제 외부에서 접근해 봅니다.

=== "Windows (PowerShell)"

    ```powershell
    curl.exe http://localhost/
    curl.exe http://localhost/api
    ```

=== "macOS / Linux"

    ```bash
    curl http://localhost/
    curl http://localhost/api
    ```

```text
<!DOCTYPE html>
<title>Welcome to nginx!</title>
...
hello from backend (connected to db)
```

## 7. 내부에서 DB 접근 확인

DB는 외부에 노출하지 않았습니다. 클러스터 내부에서만 `db` 라는 DNS 이름으로 닿는지 확인합니다.

```bash
kubectl run tmp --image=busybox -it --rm --restart=Never -- sh -c "nslookup db.default.svc.cluster.local"
```

```text
Name:      db.default.svc.cluster.local
Address:   10.43.22.18
```

DB Service 이름이 내부 IP로 해석됩니다. 백엔드 앱이라면 `db:3306` 으로 접속하게 됩니다 — IP가 아니라 이름으로요.

## 검증

- `kubectl get all` 에서 3개 Deployment/Pod가 모두 `Running`
- `http://localhost/` → nginx, `http://localhost/api` → 백엔드 응답
- `db` 는 `EXTERNAL-IP` 가 `<none>` 으로 외부 미노출, 내부 DNS `db` 로만 접근

## 직접 해 보기

!!! question "도전 과제"
    1. `db.yaml` 에서 `MYSQL_ROOT_PASSWORD` env를 통째로 지우고 apply해 보세요. `db` Pod가 `CrashLoopBackOff` 로 재시도하는 모습을 `kubectl get pods -w` 로 관찰하세요(비밀번호가 없어 mysql이 기동 거부). 그 뒤 다시 값을 넣어 apply하면 자가수렴하는지 보세요.
    2. `kubectl scale deployment frontend --replicas=3` 후 `kubectl get endpoints frontend` 가 3개로 늘었는지 확인하세요.
    3. backend의 `readinessProbe` 의 `port` 를 `9999`(안 듣는 포트)로 바꿔 apply하면, Pod는 `Running` 이지만 `READY 0/1` 이 되고 `/api` 가 안 됩니다. `kubectl get pods`, `kubectl get endpoints backend` 로 확인하세요.

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **`db` Pod가 `CrashLoopBackOff`** — `MYSQL_ROOT_PASSWORD`(또는 관련 env)가 없거나 잘못됐습니다. `kubectl logs -l app=db` 로 확인하세요.
    - **`/api` 가 502/503** — backend의 readinessProbe가 아직 통과 못 했거나 Pod가 준비 중입니다. 잠시 후 재시도하고 `kubectl get endpoints backend` 를 확인하세요.
    - **`http://localhost/` 연결 거부** — 이전 Lab의 LoadBalancer가 80 포트를 잡고 있는지 `kubectl get svc -A` 로 확인하세요.
    - **mysql이 느리게 뜸** — 첫 기동 시 DB 초기화로 20~40초 걸릴 수 있습니다. 정상입니다.

## 정리

```bash
kubectl delete -f frontend.yaml
kubectl delete -f backend.yaml
kubectl delete -f db.yaml
```

---

다음: [Lab 8 · ConfigMap](08-configmap.md)
