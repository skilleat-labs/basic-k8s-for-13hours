# 3일차 · 영속성과 보안

앱을 배포했으니 이제 **데이터를 지키고** 클러스터를 **안전하게** 만듭니다. Pod가 사라져도 데이터가 남도록 **PV/PVC** 를 붙이고, 공격자의 침입 사슬(침입→권한상승→정찰→내부이동)을 **securityContext·PSA·RBAC·NetworkPolicy** 로 한 단계씩 끊습니다.

## 오늘의 실습

| # | 실습 | 무엇을 배우나 |
|---|---|---|
| 1 | [데이터 소실 재현](01-data-loss.md) | 볼륨 없는 Pod 삭제 → 데이터 소실 체감 |
| 2 | [PV·PVC·StorageClass](02-pv-pvc.md) | 영속 볼륨 연결로 데이터 보존 |
| 3 | [StatefulSet](03-statefulset.md) | 안정적 이름·전용 스토리지가 필요한 워크로드 |
| 4 | [securityContext 하드닝](04-securitycontext.md) | root 컨테이너 위험 재현 → 방어 설정 |
| 5 | [Pod Security Admission](05-pod-security-admission.md) | 네임스페이스 단위로 위반 Pod 거부 |
| 6 | [RBAC 최소 권한](06-rbac.md) | ServiceAccount·Role·RoleBinding, `auth can-i` |
| 7 | [토큰 마운트 차단](07-token-hardening.md) | SA 토큰 자동 마운트 차단 |
| 8 | [NetworkPolicy](08-networkpolicy.md) | 전면 차단 후 선별 허용, 내부 이동 차단 |

!!! warning "보안 실습의 색 규칙"
    이 과정에서는 **공격·위협은 빨강**, **방어·차단은 파랑** 으로 구분합니다. 실습 문서에서도 :material-sword: 공격 재현과 :material-shield: 방어 적용 단계를 구분해 표시합니다.

시작: **[Lab 1 · 데이터 소실 재현](01-data-loss.md)**
