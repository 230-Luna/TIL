# CI/CD Pipeline Architecture Design Log

## v0.1 - 초기 구조

![v0.1](./img/CICD1.png)

### 구성 요소

- `Developer -> Source Repository (GitHub)`: 개발자가 코드 Push 및 PR Open
- `GitHub Action (CI)`:
  - Job 1: Checkout -> ESLint/Format Check -> Type Check
  - Job 2: Docker Image Build -> Trivy (보안 스캔) -> ECR 푸시
- `GitOps Repository`: ECR 이미지 태그 자동 업데이트
- `Argo CD`: GitOps Repo 변경 감지 -> Helm으로 매니페스트 생성 -> Kubernetes 배포

### 문제점

- 테스트 단계 없음
- PR Open과 main 머지 후 트리거 구분 불명확

## v0.2 - CI 구조 개선

![v0.2](./img/CICD2.png)

### 구성 요소

- GitHub Action CI에서 `테스트 단계` 추가
- `GitHub Action (CI)` 트리거 및 job 구분 명확화:
  - `job - PR Open`: Checkout → ESLint/Format Check → Type Check → Test
  - `job - Branch Merge`: Docker Image Build → Trivy → ECR 푸시

### 문제점

- dev / staging / prod 환경 분리 없음

## v0.3 - 배포 환경 분리

![v0.3](./img/CICD3.png)

### 구성 요소

- `staging 환경 미포함`: 서비스 규모가 작아 dev / prod 2단계로만 구성

### 문제점

- 파이프라인 실패 시 알림(Notification) 없음
