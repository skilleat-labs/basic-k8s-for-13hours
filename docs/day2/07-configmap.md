# Lab 7 · ConfigMap

앞선 실습에서 환경변수와 비밀번호를 매니페스트에 직접 박아 넣었습니다. 설정이 코드(YAML)에 섞이면, 값 하나 바꾸려고 이미지를 다시 만들거나 매니페스트를 여기저기 고쳐야 합니다. **ConfigMap** 은 이런 **설정값을 애플리케이션에서 분리** 해 별도 객체로 관리하게 해 줍니다. 이번 실습에서는 ConfigMap을 만들고, 환경변수와 볼륨 세 가지 방식으로 Pod에 주입하며, 값이 바뀌었을 때 실행 중인 Pod에 어떻게(또는 왜 안) 반영되는지 확인합니다.

!!! abstract "이 실습에서 배우는 것"
    - ConfigMap을 리터럴과 파일 두 방식으로 만든다
    - Pod에 주입하는 세 가지 방법: `env`(키 지정) · `envFrom`(통째로) · 볼륨 마운트
    - ConfigMap 값을 바꿔도 실행 중 Pod에는 반영되지 않으며, `rollout restart` 가 필요함을 안다

## 사전 조건

- Rancher Desktop의 k3s 클러스터가 실행 중이다

## 1. ConfigMap 만들기 — 리터럴로

가장 간단한 방법은 명령줄에서 키-값을 직접 주는 것입니다.

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_COLOR=blue
```

```text
configmap/app-config created
```

내용을 확인합니다.

```bash
kubectl get configmap app-config -o yaml
```

```text
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_COLOR: blue
```

`data:` 아래에 키-값이 그대로 들어 있습니다. Secret과 달리 **평문** 으로 저장됩니다(민감하지 않은 설정용).

## 2. ConfigMap 만들기 — 파일로

설정 파일 전체를 ConfigMap으로 담을 수도 있습니다. 아래를 `app.properties` 로 저장하세요.

```text
log.level=info
feature.newUI=true
max.connections=100
```

이 파일로부터 ConfigMap을 만듭니다.

```bash
kubectl create configmap app-file-config --from-file=app.properties
kubectl get configmap app-file-config -o yaml
```

```text
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-file-config
data:
  app.properties: |
    log.level=info
    feature.newUI=true
    max.connections=100
```

파일 이름(`app.properties`)이 키가 되고, 파일 내용 전체가 값이 됩니다.

## 3. 주입 방식 (a) — env로 특정 키만

ConfigMap의 특정 키 하나를 환경변수로 꽂습니다. 아래를 `pod-env.yaml` 로 저장하세요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - name: app
          image: nginx
          env:
            - name: APP_ENV                 # 컨테이너 안에서 쓸 변수 이름
              valueFrom:
                configMapKeyRef:
                  name: app-config          # 어떤 ConfigMap에서
                  key: APP_ENV              # 어떤 키를
```

```bash
kubectl apply -f pod-env.yaml
```

Pod 안에서 환경변수를 확인합니다.

```bash
kubectl exec deploy/demo -- sh -c "env | grep APP"
```

```text
APP_ENV=production
```

`APP_ENV` 만 들어왔습니다. `configMapKeyRef` 는 원하는 키만 골라 꽂는 방식입니다.

## 4. 주입 방식 (b) — envFrom로 통째로

ConfigMap의 **모든 키** 를 한 번에 환경변수로 꽂으려면 `envFrom` 을 씁니다. `pod-env.yaml` 의 `env:` 블록을 아래 `envFrom:` 으로 교체하고 다시 apply하세요.

```yaml
      containers:
        - name: app
          image: nginx
          envFrom:
            - configMapRef:
                name: app-config
```

```bash
kubectl apply -f pod-env.yaml
kubectl exec deploy/demo -- sh -c "env | grep APP"
```

```text
APP_ENV=production
APP_COLOR=blue
```

이번엔 `APP_ENV`, `APP_COLOR` 둘 다 들어왔습니다. 키가 많을 때 편리합니다.

## 5. 주입 방식 (c) — 볼륨으로 마운트

파일형 ConfigMap은 볼륨으로 마운트해 **파일** 로 노출할 수 있습니다. 아래를 `pod-volume.yaml` 로 저장하세요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-vol
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-vol
  template:
    metadata:
      labels:
        app: demo-vol
    spec:
      containers:
        - name: app
          image: nginx
          volumeMounts:
            - name: config-vol
              mountPath: /etc/config      # 이 경로에 파일로 나타남
      volumes:
        - name: config-vol
          configMap:
            name: app-file-config
```

```bash
kubectl apply -f pod-volume.yaml
```

컨테이너 안에서 파일로 보이는지 확인합니다.

```bash
kubectl exec deploy/demo-vol -- sh -c "ls /etc/config && echo '---' && cat /etc/config/app.properties"
```

```text
app.properties
---
log.level=info
feature.newUI=true
max.connections=100
```

ConfigMap의 각 키가 `/etc/config/` 아래 파일로 만들어졌습니다. 설정 파일을 통째로 읽는 앱에 적합합니다.

## 6. 값 변경 → 반영 실험

ConfigMap 값을 바꾸면 실행 중인 Pod에 곧바로 반영될까요? 실험해 봅니다. `APP_COLOR` 를 `red` 로 바꿉니다.

```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_COLOR=red \
  --dry-run=client -o yaml | kubectl apply -f -
```

```text
configmap/app-config configured
```

`envFrom` 으로 주입한 `demo` Pod의 환경변수를 다시 봅니다.

```bash
kubectl exec deploy/demo -- sh -c "env | grep APP_COLOR"
```

```text
APP_COLOR=blue
```

바꿨는데 여전히 `blue` 입니다! **환경변수로 주입한 값은 Pod가 만들어질 때 한 번 결정되고, 이후 ConfigMap이 바뀌어도 실행 중 Pod에는 반영되지 않습니다.**

새 값을 적용하려면 Pod를 새로 만들어야 합니다. `rollout restart` 로 무중단 재생성합니다.

```bash
kubectl rollout restart deployment/demo
kubectl rollout status deployment/demo
kubectl exec deploy/demo -- sh -c "env | grep APP_COLOR"
```

```text
deployment "demo" successfully rolled out
APP_COLOR=red
```

새 Pod로 교체되자 `red` 가 반영됐습니다.

!!! info "env는 재시작 필요, 볼륨은 자동 갱신(지연)"
    - **환경변수(`env`/`envFrom`)** 로 넣은 값은 Pod 생성 시점에 고정 → 변경하려면 **`rollout restart` 필수**.
    - **볼륨 마운트** 로 넣은 파일은 시간이 지나면(수십 초~1분, kubelet 동기화 주기) **자동으로 갱신** 됩니다. 단, 앱이 그 파일을 다시 읽어야 반영되므로, 앱 자체가 파일 변경을 감지하지 못하면 역시 재시작이 필요합니다.

## 검증

- `kubectl get configmap app-config -o yaml` 에 키-값이 보인다
- `demo` Pod에서 `env | grep APP` 로 주입된 변수가 확인된다
- `demo-vol` Pod의 `/etc/config/app.properties` 에 내용이 파일로 있다
- ConfigMap 변경 후 `rollout restart` 를 해야 새 값이 반영된다

## 직접 해 보기

!!! question "도전 과제"
    1. `demo-vol`(볼륨 방식)에서 ConfigMap을 바꾼 뒤, `rollout restart` 없이 1분쯤 기다렸다가 `cat /etc/config/app.properties` 를 다시 보세요. 자동 갱신되나요? (env 방식과 비교)
    2. `configMapKeyRef` 에서 없는 키(`key: NOPE`)를 참조하면 Pod가 어떻게 되는지 관찰하세요.
    3. `kubectl edit configmap app-config` 로 값을 바꾼 뒤 `rollout restart` 로 반영해 보세요.

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **값을 바꿨는데 안 바뀜** — env로 주입했다면 정상입니다. `kubectl rollout restart deployment/<이름>` 을 하세요.
    - **`CreateContainerConfigError`** — 참조한 ConfigMap 이름이나 키가 존재하지 않습니다. `kubectl describe pod <파드>` 의 이벤트를 확인하세요.
    - **`create configmap` 이 `AlreadyExists`** — 이미 있는 ConfigMap입니다. 6단계처럼 `--dry-run=client -o yaml | kubectl apply -f -` 패턴으로 갱신하세요.

## 정리

```bash
kubectl delete -f pod-volume.yaml
kubectl delete -f pod-env.yaml
kubectl delete configmap app-config app-file-config
```

---

다음: [Lab 8 · Secret](08-secret.md)
