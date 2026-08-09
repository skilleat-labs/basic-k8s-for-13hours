# 기초 쿠버네티스 13시간 실습 가이드

3일(총 13시간) 과정으로 도커의 한계부터 쿠버네티스 배포·네트워킹·보안까지 **직접 손으로 따라 하는** 실습 가이드입니다. 모든 실습은 **Windows(PowerShell)** 와 **macOS/Linux** 명령을 함께 제공합니다.

- **1일차** — 입문과 워크로드 (환경 구축 · Pod · Deployment)
- **2일차** — 서비스와 네트워킹 (Service · Ingress · 3-tier 배포 · ConfigMap/Secret)
- **3일차** — 영속성과 보안 (PV/PVC · securityContext · RBAC · NetworkPolicy)

## 실습 환경

- [Rancher Desktop](https://rancherdesktop.io) (내장 k3s 단일 노드 클러스터)
- `kubectl` (Rancher Desktop 설치 시 자동 포함)

자세한 준비 방법은 [실습 환경 준비](docs/setup/prerequisites.md) 문서를 참고하세요.

## 로컬에서 문서 미리보기

```bash
pip install -r requirements.txt
mkdocs serve
```

브라우저에서 `http://127.0.0.1:8000` 접속.

## 배포

`main` 브랜치에 푸시하면 GitHub Actions가 자동으로 GitHub Pages(`gh-pages` 브랜치)에 배포합니다.
저장소 **Settings → Pages → Source** 를 `gh-pages` 브랜치로 설정하세요.

---

자료 제작 · **스킬잇 김누리**
