# 기초 쿠버네티스 13시간 실습 가이드

도커만 쓰던 사람이 **3일(총 13시간)** 만에 쿠버네티스로 애플리케이션을 배포·노출·보호하는 것까지 직접 해 보는 실습 과정입니다.

이 사이트는 강의 슬라이드와 짝을 이루는 **핸즈온(hands-on) 가이드**입니다. 위에서부터 순서대로 명령을 따라 치면 클러스터에서 실제로 무슨 일이 일어나는지 눈으로 확인할 수 있습니다.

!!! tip "이 가이드의 특징"
    - 모든 명령은 **Windows(PowerShell)** 와 **macOS / Linux** 버전을 함께 제공합니다. 자신의 OS 탭을 선택하세요.
    - 모든 실습은 `목표 → 단계 → 검증 → 정리` 흐름으로 구성되어 있습니다.
    - 각 실습 끝에는 **직접 해 보기**와 **자주 만나는 오류**가 있습니다.

## 3일 커리큘럼

<div class="grid cards" markdown>

-   :material-cube-outline: **1일차 — 입문과 워크로드**

    ---

    로컬 클러스터를 띄우고, Pod와 Deployment로 애플리케이션을 실행·복구·확장·업데이트합니다.

    [:octicons-arrow-right-24: 1일차 시작](day1/index.md)

-   :material-lan: **2일차 — 서비스와 네트워킹**

    ---

    Service·DNS로 Pod를 연결하고, NodePort·LoadBalancer·Ingress로 외부에 노출하며, ConfigMap/Secret으로 설정을 분리합니다.

    [:octicons-arrow-right-24: 2일차 시작](day2/index.md)

-   :material-shield-lock-outline: **3일차 — 영속성과 보안**

    ---

    PV/PVC로 데이터를 지키고, securityContext·PSA·RBAC·NetworkPolicy로 침입 사슬을 끊습니다.

    [:octicons-arrow-right-24: 3일차 시작](day3/index.md)

</div>

## 시작하기 전에

먼저 **[실습 환경 준비](setup/prerequisites.md)** 문서를 따라 Rancher Desktop을 설치하고 클러스터가 정상 동작하는지 확인하세요. 1일차 Lab 1부터는 클러스터가 떠 있다고 가정합니다.

!!! warning "이 클러스터는 학습용입니다"
    Rancher Desktop이 띄우는 것은 **학습·개발용 단일 노드 k3s 클러스터**입니다. 실제 운영 클러스터가 아니므로, 여기서 익힌 개념을 실무에 적용할 때는 노드 수·가용성·백업 등을 추가로 고려해야 합니다.

---

자료 제작 · **스킬잇 김누리**
