# Lab 6 · RBAC 최소 권한

클러스터 안의 앱과 사용자는 저마다 **필요한 만큼의 권한만** 가져야 합니다. 로그 조회만 하면 되는 앱이 Secret 을 읽거나 Pod 를 삭제할 수 있다면, 그 앱이 탈취됐을 때 피해가 걷잡을 수 없이 커집니다. **RBAC(Role-Based Access Control)** 는 "누가 무엇을 할 수 있는가"를 정밀하게 통제하는 쿠버네티스의 권한 체계입니다. 이번 실습에서는 Pod 조회만 허용하는 ServiceAccount 를 만들고, `kubectl auth can-i` 로 권한이 딱 그만큼만 부여됐는지 검증합니다.

!!! abstract "이 실습에서 배우는 것"
    - ServiceAccount·Role·RoleBinding 세 리소스의 관계
    - `kubectl auth can-i ... --as=...` 로 특정 주체의 권한을 테스트하는 법
    - 최소 권한 원칙(least privilege)과 와일드카드(`*`)의 위험
    - Role vs ClusterRole, RoleBinding vs ClusterRoleBinding 의 차이

## 사전 조건

- 네임스페이스와 Pod 개념에 익숙해야 합니다.
- 클러스터가 정상인지 확인합니다.

```bash
kubectl get nodes
```

## 1. 네임스페이스와 ServiceAccount 만들기

실습용 네임스페이스와, 앱이 사용할 ServiceAccount 를 만듭니다. **ServiceAccount** 는 Pod(앱)의 신원, 즉 "누구"에 해당합니다.

`rbac-setup.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rbac-demo
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: rbac-demo
```

```bash
kubectl apply -f rbac-setup.yaml
```

```text
namespace/rbac-demo created
serviceaccount/app-sa created
```

## 2. Role 만들기 (무엇을 할 수 있나)

**Role** 은 "무엇을 할 수 있는가"의 목록, 즉 권한의 집합입니다. 여기서는 `rbac-demo` 네임스페이스에서 **Pod 를 조회(get/list/watch)** 하는 것만 허용합니다. 삭제·생성·수정이나 다른 리소스(Secret 등)는 전혀 넣지 않습니다.

`pod-reader-role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-demo
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

- `apiGroups: [""]`: 빈 문자열은 Pod·Service 같은 핵심(core) API 그룹을 뜻합니다.
- `resources: ["pods"]`: 대상 리소스는 Pod 뿐입니다.
- `verbs: ["get", "list", "watch"]`: 읽기 계열 동작만 허용합니다.

```bash
kubectl apply -f pod-reader-role.yaml
```

```text
role.rbac.authorization.k8s.io/pod-reader created
```

## 3. RoleBinding 만들기 (누구에게 줄까)

**RoleBinding** 은 Role(무엇을)과 주체(누구에게)를 연결합니다. `app-sa` ServiceAccount 에게 `pod-reader` Role 을 부여합니다.

`pod-reader-binding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-pod-reader
  namespace: rbac-demo
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: rbac-demo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

- `subjects`: 권한을 받을 주체(여기서는 `app-sa`).
- `roleRef`: 부여할 Role(여기서는 `pod-reader`).

```bash
kubectl apply -f pod-reader-binding.yaml
```

```text
rolebinding.rbac.authorization.k8s.io/app-sa-pod-reader created
```

!!! note "세 리소스의 관계"
    **ServiceAccount(누구)** ← **RoleBinding(연결)** → **Role(무엇을)**. RoleBinding 이 둘을 이어 줘야 비로소 권한이 실제로 적용됩니다. Role 만 만들고 바인딩하지 않으면 아무 효과가 없습니다.

## 4. 권한 검증하기 (auth can-i)

이제 `app-sa` 가 정확히 의도한 권한만 갖는지 테스트합니다. `--as=system:serviceaccount:<네임스페이스>:<이름>` 으로 그 주체인 척 흉내 내어(impersonate) 물어봅니다. 실제로 Pod 를 만들지 않고도 권한만 확인할 수 있습니다.

허용되어야 하는 동작 — Pod 조회:

```bash
kubectl auth can-i list pods --as=system:serviceaccount:rbac-demo:app-sa -n rbac-demo
```

```text
yes
```

거부되어야 하는 동작 — Pod 삭제:

```bash
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-demo:app-sa -n rbac-demo
```

```text
no
```

거부되어야 하는 동작 — Secret 조회:

```bash
kubectl auth can-i list secrets --as=system:serviceaccount:rbac-demo:app-sa -n rbac-demo
```

```text
no
```

!!! success "딱 필요한 권한만 부여됐습니다"
    Pod 조회는 `yes`, Pod 삭제와 Secret 조회는 `no`. `app-sa` 가 탈취되더라도 공격자는 이 네임스페이스의 Pod 목록을 보는 것 말고는 아무것도 할 수 없습니다. 이것이 **최소 권한 원칙**입니다. 각 주체에게 자기 일에 꼭 필요한 권한만 주면, 침해 시 피해 범위(폭발 반경)가 최소화됩니다.

`app-sa` 가 가진 모든 권한을 한눈에 보려면 `--list` 를 씁니다.

```bash
kubectl auth can-i --list --as=system:serviceaccount:rbac-demo:app-sa -n rbac-demo
```

```text
Resources   Non-Resource URLs   Resource Names   Verbs
pods        []                  []               [get list watch]
...
```

## 와일드카드는 위험 신호

Role 을 만들 때 아래처럼 쓰고 싶은 유혹이 있습니다.

```yaml
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

!!! danger "`*` 는 최소 권한의 반대입니다"
    이 Role 은 "모든 리소스에 모든 동작"을 허용합니다. 사실상 관리자 권한이며, 최소 권한 원칙에 정면으로 위배됩니다. 코드 리뷰나 보안 점검에서 `verbs: ["*"]`, `resources: ["*"]` 같은 와일드카드를 보면 **경고 신호**로 받아들여야 합니다. 정말 필요한 동사(`get`, `list` 등)와 리소스만 명시하세요.

## Role vs ClusterRole, RoleBinding vs ClusterRoleBinding

권한의 **적용 범위**에 따라 두 쌍의 리소스가 있습니다.

| 리소스 | 범위 | 용도 |
|---|---|---|
| **Role** | 특정 **네임스페이스** 안 | 한 네임스페이스의 리소스 권한 |
| **ClusterRole** | **클러스터 전체** | 모든 네임스페이스, 또는 노드처럼 네임스페이스에 속하지 않는 리소스 |
| **RoleBinding** | 특정 **네임스페이스** 안에서 부여 | Role 또는 ClusterRole 을 한 네임스페이스에 적용 |
| **ClusterRoleBinding** | **클러스터 전체**에서 부여 | ClusterRole 을 모든 네임스페이스에 적용 |

!!! tip "가능한 한 좁은 범위를 고르세요"
    한 네임스페이스에서만 필요하면 **Role + RoleBinding** 을 씁니다. 노드 조회처럼 클러스터 전역 리소스가 필요할 때만 ClusterRole 을 씁니다. 재미있는 조합으로, **ClusterRole 을 RoleBinding 으로 특정 네임스페이스에만** 부여할 수도 있습니다(공통 권한 정의를 재사용하되 범위는 좁히기).

## 검증

- `list pods` 는 `yes` 다.
- `delete pods` 와 `list secrets` 는 모두 `no` 다.
- `auth can-i --list` 에 `pods [get list watch]` 만 보인다.

## 직접 해 보기

1. Role 의 `verbs` 에 `create` 를 추가하고 다시 적용한 뒤, `kubectl auth can-i create pods --as=...` 가 `yes` 로 바뀌는지 확인해 보세요. 권한이 실시간으로 반영되는 것을 체감할 수 있습니다.
2. `app-sa` 를 실제 Pod 에 붙여(`spec.serviceAccountName: app-sa`) 배포한 뒤, 그 Pod 안에서 `kubectl` 로 Pod 목록은 되고 Secret 조회는 안 되는지 확인해 보세요. (다음 Lab 7 에서 이 토큰 마운트를 더 깊이 다룹니다.)

!!! failure "자주 만나는 오류"
    **증상**: `auth can-i list pods` 가 예상과 달리 `no`
    **원인**: RoleBinding 의 `subjects.namespace` 나 `--as` 의 네임스페이스가 틀림, 또는 `-n rbac-demo` 를 빼먹음.
    **해결**: 세 곳(RoleBinding subject, `--as` 문자열, `-n` 플래그)의 네임스페이스가 모두 `rbac-demo` 로 일치하는지 확인합니다.

    ---

    **증상**: `error: You must be logged in to the server (Unauthorized)`
    **원인**: `--as` 문자열 형식 오류. 정확한 형식은 `system:serviceaccount:<ns>:<sa이름>`.
    **해결**: 콜론(`:`)으로 구분된 4개 요소(`system` `serviceaccount` `네임스페이스` `이름`)를 다시 확인합니다.

## 정리

```bash
kubectl delete namespace rbac-demo
```

```text
namespace "rbac-demo" deleted
```

네임스페이스를 지우면 그 안의 ServiceAccount·Role·RoleBinding 이 모두 함께 삭제됩니다.

---

다음: [Lab 7 · 토큰 마운트 차단](07-token-hardening.md)
