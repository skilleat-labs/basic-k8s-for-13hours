# Lab 8 · Secret

ConfigMap은 평문 설정에 적합하지만, DB 비밀번호나 API 키 같은 **민감한 값** 을 담기에는 적절하지 않습니다. 쿠버네티스는 이런 값을 위해 **Secret** 이라는 별도 객체를 제공합니다. Secret은 ConfigMap과 거의 똑같이 쓰지만, 값이 base64로 인코딩되어 저장되고 접근을 더 통제할 수 있습니다. 이번 실습에서는 Secret을 만들어 환경변수로 주입하고, **"base64는 암호화가 아니다"** 라는 중요한 사실을 직접 확인합니다.

!!! abstract "이 실습에서 배우는 것"
    - Secret을 만들어 DB 자격증명 같은 민감한 값을 매니페스트에서 분리한다
    - `secretKeyRef` 로 Secret 값을 환경변수로 주입한다
    - Secret의 `data` 가 base64일 뿐 **암호화가 아님** 을 이해한다
    - 진짜 방어는 etcd 암호화 + RBAC임을 알고, 값 변경 시 `rollout restart` 를 한다

## 사전 조건

- Rancher Desktop의 k3s 클러스터가 실행 중이다

## 1. Secret 만들기

리터럴로 Secret을 만듭니다. ConfigMap과 명령 형태가 거의 같습니다(`generic` 타입).

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=s3cr3t
```

```text
secret/db-secret created
```

kubectl이 값을 자동으로 base64로 인코딩해 저장하므로, 우리가 직접 인코딩할 필요는 없습니다.

## 2. base64 인코딩·디코딩 이해

Secret이 값을 어떻게 저장하는지 감을 잡기 위해, 같은 값을 손으로 base64 인코딩·디코딩해 봅니다. **이 명령은 OS마다 다릅니다.**

=== "Windows (PowerShell)"

    ```powershell
    # 인코딩: s3cr3t → base64
    [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('s3cr3t'))

    # 디코딩: base64 → 원문
    [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String('czNjcjN0'))
    ```

=== "macOS / Linux"

    ```bash
    # 인코딩: s3cr3t → base64
    echo -n 's3cr3t' | base64

    # 디코딩: base64 → 원문
    echo 'czNjcjN0' | base64 -d
    ```

```text
czNjcjN0
s3cr3t
```

!!! warning "핵심: base64는 암호화가 아니라 인코딩이다"
    방금 봤듯 `czNjcjN0` 는 **누구나 키 없이** 원문 `s3cr3t` 로 되돌릴 수 있습니다. base64는 데이터를 안전하게 만드는 게 아니라 단지 텍스트로 표현하는 방식일 뿐입니다. Secret에 넣었다고 값이 암호화되는 게 아닙니다.

## 3. Secret 내용 확인 — 정말 base64뿐

방금 만든 Secret의 실제 저장 형태를 봅니다.

```bash
kubectl get secret db-secret -o yaml
```

```text
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: czNjcjN0
```

`data.DB_PASSWORD` 에 `czNjcjN0` — 앞서 우리가 손으로 만든 것과 똑같은 base64 문자열입니다. 즉 Secret에 접근할 수 있는 사람은 아래처럼 값을 그대로 꺼내 볼 수 있습니다.

=== "Windows (PowerShell)"

    ```powershell
    $b64 = kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}'
    [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($b64))
    ```

=== "macOS / Linux"

    ```bash
    kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
    ```

```text
s3cr3t
```

## 4. Secret을 환경변수로 주입

이제 Secret 값을 Pod에 꽂습니다. ConfigMap의 `configMapKeyRef` 와 짝을 이루는 `secretKeyRef` 를 씁니다. 아래를 `pod-secret.yaml` 로 저장하세요.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-consumer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-consumer
  template:
    metadata:
      labels:
        app: db-consumer
    spec:
      containers:
        - name: app
          image: nginx
          env:
            - name: DB_PASSWORD              # 컨테이너 안 변수 이름
              valueFrom:
                secretKeyRef:
                  name: db-secret            # 어떤 Secret에서
                  key: DB_PASSWORD           # 어떤 키를
```

```bash
kubectl apply -f pod-secret.yaml
kubectl exec deploy/db-consumer -- sh -c "echo \$DB_PASSWORD"
```

```text
s3cr3t
```

컨테이너 안에서는 원문 `s3cr3t` 로 복원되어 환경변수로 들어옵니다. 앱은 base64를 신경 쓸 필요 없이 그대로 씁니다.

## 5. 값 변경 → rollout restart

ConfigMap과 마찬가지로, **env로 주입한 Secret 값도 실행 중 Pod에는 자동 반영되지 않습니다.** 비밀번호를 바꿔 봅니다.

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=newP@ss \
  --dry-run=client -o yaml | kubectl apply -f -
```

```text
secret/db-secret configured
```

```bash
kubectl exec deploy/db-consumer -- sh -c "echo \$DB_PASSWORD"
```

```text
s3cr3t
```

여전히 옛 값입니다. 새 Pod로 교체해야 반영됩니다.

```bash
kubectl rollout restart deployment/db-consumer
kubectl rollout status deployment/db-consumer
kubectl exec deploy/db-consumer -- sh -c "echo \$DB_PASSWORD"
```

```text
deployment "db-consumer" successfully rolled out
newP@ss
```

## 6. 그럼 Secret은 어떻게 지키나

base64가 암호화가 아니라면, 민감한 값을 실제로 지키는 것은 무엇일까요?

| 방어 수단 | 무엇을 하나 |
|---|---|
| **etcd 저장 시 암호화(Encryption at Rest)** | Secret이 저장되는 클러스터 DB(etcd)를 암호화해, 디스크가 유출돼도 값을 못 읽게 함 |
| **RBAC 접근 통제** | "누가 어떤 Secret을 읽을 수 있는가"를 역할 기반으로 제한. 이게 실질적인 1차 방어선 |
| **매니페스트에 평문 금지** | Secret 값을 Git에 커밋된 YAML에 평문으로 넣지 않기(외부 Secret 관리 도구 연동 등) |

!!! info "핵심 정리"
    Secret은 값을 "숨겨 주는" 마법이 아닙니다. base64 인코딩 + **접근 통제(RBAC)** + **저장소 암호화** 가 함께 있어야 실제로 안전합니다. Secret의 진짜 가치는 "민감한 값을 별도 객체로 분리해, 접근 권한을 따로 통제할 수 있게 하는 것"입니다.

!!! note "3일차 예고 — RBAC"
    "누가 무엇에 접근할 수 있는가"를 정하는 RBAC는 Secret 보호의 핵심입니다. 3일차 보안 파트에서 Role·RoleBinding으로 접근 권한을 직접 다뤄 봅니다.

## 검증

- `kubectl get secret db-secret -o yaml` 의 `data` 값이 base64 문자열이다
- base64 디코딩으로 원문을 그대로 되돌릴 수 있다(= 암호화가 아님)
- `db-consumer` Pod의 `DB_PASSWORD` 환경변수에 원문이 주입된다
- Secret 변경 후 `rollout restart` 를 해야 새 값이 반영된다

## 직접 해 보기

!!! question "도전 과제"
    1. ConfigMap의 `envFrom` 처럼 Secret도 `envFrom: - secretRef: { name: db-secret }` 로 통째 주입할 수 있습니다. 바꿔서 시도해 보세요.
    2. Secret을 볼륨으로 마운트(`/etc/secret`)하면 각 키가 파일로 나타납니다. Lab 7의 볼륨 방식을 참고해 `cat /etc/secret/DB_PASSWORD` 로 원문을 확인해 보세요.
    3. `kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}'` 로 base64를 꺼내, OS별 디코딩 명령으로 원문을 복원해 보세요.

## 자주 만나는 오류

!!! failure "자주 만나는 오류"
    - **값이 안 바뀜** — env 주입은 정상 동작입니다. `kubectl rollout restart deployment/<이름>` 을 하세요.
    - **`CreateContainerConfigError`** — 참조한 Secret 이름/키가 없습니다. `kubectl describe pod <파드>` 이벤트를 확인하세요.
    - **base64에 개행이 섞임** — macOS/Linux에서 `echo` 에 `-n` 을 빼면 끝에 개행이 들어가 인코딩 결과가 달라집니다. 인코딩할 땐 `echo -n` 을 쓰세요.
    - **`create secret` 이 `AlreadyExists`** — 5단계처럼 `--dry-run=client -o yaml | kubectl apply -f -` 로 갱신하세요.

## 정리

```bash
kubectl delete -f pod-secret.yaml
kubectl delete secret db-secret
```

수고하셨습니다. 2일차 서비스와 네트워킹 실습을 모두 마쳤습니다. 이어서 3일차에서는 데이터를 영구히 보존하는 **볼륨(PV·PVC)** 과 **StatefulSet**, 그리고 **securityContext·RBAC** 로 보안을 다룹니다.

---

다음: [3일차 개요](../day3/index.md)
