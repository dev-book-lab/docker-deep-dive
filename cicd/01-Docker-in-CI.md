# 01. Docker in CI - GitHub Actions, GitLab CI

## 🎯 이 챕터에서 배울 것

- **CI/CD 기초**: Continuous Integration/Deployment 개념
- **GitHub Actions**: Docker 빌드 및 푸시 워크플로우
- **GitLab CI/CD**: .gitlab-ci.yml 파이프라인
- **Multi-stage Build**: 최적화된 이미지 빌드
- **Cache 전략**: 빌드 시간 단축
- **실전 구현**: 완전한 CI/CD 파이프라인

## 📌 왜 중요한가?

**"CI/CD는 코드 커밋부터 프로덕션 배포까지의 전 과정을 자동화합니다."**

```
Docker in CI/CD의 핵심:

Without CI/CD (수동 배포):
┌─────────────────────────────────────────────────┐
│ Manual Process                                  │
│                                                 │
│ 1. 개발자가 코드 작성                               │
│    ↓                                            │
│ 2. 로컬에서 docker build                          │
│    ↓                                            │
│ 3. 수동으로 테스트 실행                              │
│    ↓                                            │
│ 4. docker tag myapp:v1.2.3                      │
│    ↓                                            │
│ 5. docker push registry/myapp:v1.2.3            │
│    ↓                                            │
│ 6. 서버 SSH 접속                                  │
│    ↓                                            │
│ 7. kubectl apply -f deployment.yaml             │
│    ↓                                            │
│ 8. 배포 확인                                      │
└─────────────────────────────────────────────────┘

문제점:
❌ 시간 소모 (30분 이상)
❌ 사람 실수 (태그 오타, 잘못된 환경)
❌ 일관성 부족 (개발자마다 다른 절차)
❌ 롤백 어려움
❌ 배포 이력 추적 어려움

With CI/CD (자동화):
┌─────────────────────────────────────────────────┐
│ Automated Pipeline                              │
│                                                 │
│ 1. git push                                     │
│    ↓                                            │
│ 2. GitHub Actions Triggered ⚡                   │
│    ├─ Checkout code                             │
│    ├─ Run tests                                 │
│    ├─ Build Docker image                        │
│    ├─ Security scan                             │
│    ├─ Push to registry                          │
│    └─ Deploy to Kubernetes                      │
│    ↓                                            │
│ 3. Slack notification 📢                        │
│    ↓                                            │
│ 4. ✅ Done! (5분)                               │
└─────────────────────────────────────────────────┘

장점:
✅ 빠른 배포 (5분)
✅ 일관성 (항상 동일한 절차)
✅ 자동 테스트
✅ 배포 이력 추적
✅ 쉬운 롤백

CI/CD 파이프라인 단계:
┌───────────────────────────────────────────────┐
│ 1. Source (코드 변경)                           │
│    - git push                                 │
│    - Pull Request                             │
│    ↓                                          │
│ 2. Build (이미지 빌드)                           │
│    - docker build                             │
│    - Multi-stage build                        │
│    - Layer caching                            │
│    ↓                                          │
│ 3. Test (자동 테스트)                            │
│    - Unit tests                               │
│    - Integration tests                        │
│    - Security scan                            │
│    ↓                                          │
│ 4. Push (레지스트리 푸시)                         │
│    - docker tag                               │
│    - docker push                              │
│    - Multi-platform build                     │
│    ↓                                          │
│ 5. Deploy (배포)                               │
│    - kubectl apply                            │
│    - Rolling update                           │
│    - Health check                             │
│    ↓                                          │
│ 6. Verify (검증)                               │
│    - Smoke tests                              │
│    - Monitoring                               │
│    - Rollback if needed                       │
└───────────────────────────────────────────────┘

GitHub Actions vs GitLab CI:
┌──────────────┬──────────────┬──────────────┐
│ 특성          │ GitHub       │ GitLab CI    │
├──────────────┼──────────────┼──────────────┤
│ 설정 파일      │ .github/     │ .gitlab-     │
│              │ workflows/   │ ci.yml       │
├──────────────┼──────────────┼──────────────┤
│ Runner       │ GitHub-      │ Self-hosted  │
│              │ hosted       │ or GitLab    │
├──────────────┼──────────────┼──────────────┤
│ 무료 분        │ 2000분/월     │ 400분/월      │
├──────────────┼──────────────┼──────────────┤
│ Docker       │ 기본 지원      │ 기본 지원      │
└──────────────┴──────────────┴──────────────┘
```

**실무 영향:**
- **생산성**: 수동 작업 제거로 개발에 집중
- **품질**: 자동 테스트로 버그 조기 발견
- **속도**: 하루 수십 번 배포 가능
- **신뢰성**: 일관된 배포 프로세스

---

## 🔬 Deep Dive

### 1. GitHub Actions 기초

#### Workflow 구조

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

# 트리거
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

# 환경 변수
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

# Jobs
jobs:
  build:
    runs-on: ubuntu-latest
    
    # Steps
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Build image
      run: docker build -t myapp .
    
    - name: Run tests
      run: docker run myapp npm test
```

#### Actions Marketplace

```yaml
# 재사용 가능한 Actions
steps:
  # Docker 빌드 및 푸시
  - uses: docker/build-push-action@v5
  
  # 보안 스캔
  - uses: aquasecurity/trivy-action@master
  
  # Slack 알림
  - uses: 8398a7/action-slack@v3
```

---

### 2. GitLab CI/CD 기초

#### Pipeline 구조

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

test:
  stage: test
  script:
    - docker run $IMAGE_TAG npm test

deploy:
  stage: deploy
  script:
    - kubectl apply -f k8s/
```

---

## 🔧 실습 1: GitHub Actions - 기본 Docker 빌드

### Step 1: Workflow 파일 생성

```yaml
# .github/workflows/docker-build.yml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
    
    steps:
      # 1. 코드 체크아웃
      - name: Checkout repository
        uses: actions/checkout@v4
      
      # 2. Docker Buildx 설정 (multi-platform)
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      # 3. 레지스트리 로그인
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      # 4. 메타데이터 추출 (태그, 라벨)
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
      
      # 5. 빌드 및 푸시
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Step 2: Dockerfile

```dockerfile
# Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Production
FROM node:18-alpine

WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Step 3: 테스트

```bash
# 1. 변경 사항 푸시
git add .
git commit -m "Add GitHub Actions workflow"
git push origin main

# 2. GitHub Actions 탭에서 확인
# https://github.com/USER/REPO/actions

# 3. 빌드 로그 확인
# - Checkout repository ✅
# - Set up Docker Buildx ✅
# - Log in to registry ✅
# - Build and push ✅

# 4. 이미지 확인
docker pull ghcr.io/USER/REPO:main
docker run -p 3000:3000 ghcr.io/USER/REPO:main
```

---

## 🔧 실습 2: GitHub Actions - 멀티 스테이지 빌드 with 테스트

### Step 1: 테스트 포함 Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm test
      
      - name: Run linter
        run: npm run lint
      
      - name: Upload test coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  build:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: test  # 테스트 성공 후 빌드
    
    if: github.event_name == 'push'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}

  deploy:
    name: Deploy to Kubernetes
    runs-on: ubuntu-latest
    needs: build
    
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Set Kubernetes context
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG }}
      
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.sha }}
          
          kubectl rollout status deployment/myapp
```

---

## 🔧 실습 3: GitLab CI/CD - 완전한 파이프라인

### Step 1: GitLab CI 설정

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - security
  - push
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: ""
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  LATEST_TAG: $CI_REGISTRY_IMAGE:latest

# Build stage
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_TAG -t $LATEST_TAG .
    - docker push $IMAGE_TAG
  only:
    - main
    - develop
  tags:
    - docker

# Test stage
test:unit:
  stage: test
  image: node:18-alpine
  script:
    - npm ci
    - npm test
  coverage: '/Statements\s*:\s*(\d+\.\d+)%/'
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

test:integration:
  stage: test
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker run --rm $IMAGE_TAG npm run test:integration

# Security scanning
security:trivy:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 0 --no-progress $IMAGE_TAG
    - trivy image --exit-code 1 --severity HIGH,CRITICAL --no-progress $IMAGE_TAG
  allow_failure: true

security:container:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --format json --output trivy-report.json $IMAGE_TAG
  artifacts:
    reports:
      container_scanning: trivy-report.json

# Push latest tag (main only)
push:latest:
  stage: push
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker pull $IMAGE_TAG
    - docker tag $IMAGE_TAG $LATEST_TAG
    - docker push $LATEST_TAG
  only:
    - main

# Deploy to staging
deploy:staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context staging
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG
    - kubectl rollout status deployment/myapp
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

# Deploy to production
deploy:production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context production
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG
    - kubectl rollout status deployment/myapp
  environment:
    name: production
    url: https://example.com
  when: manual  # 수동 승인 필요
  only:
    - main
```

---

## 🔧 실습 4: Docker Layer Caching

### Step 1: 최적화된 Dockerfile

```dockerfile
# Dockerfile - Layer Caching 최적화
FROM node:18-alpine AS deps

WORKDIR /app

# 의존성만 먼저 설치 (변경 빈도 낮음)
COPY package*.json ./
RUN npm ci --only=production

# Builder stage
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

# 소스 코드 복사 (변경 빈도 높음)
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine AS production

WORKDIR /app

ENV NODE_ENV=production

# 의존성 복사
COPY --from=deps /app/node_modules ./node_modules

# 빌드 결과 복사
COPY --from=builder /app/dist ./dist
COPY package*.json ./

USER node

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### Step 2: GitHub Actions Cache

```yaml
# .github/workflows/optimized-build.yml
name: Optimized Docker Build

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      # Build with cache
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          
          # GitHub Actions Cache 사용
          cache-from: |
            type=gha
            type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
          cache-to: type=gha,mode=max
          
          # Build arguments
          build-args: |
            BUILDKIT_INLINE_CACHE=1
```

### Step 3: 빌드 시간 비교

```bash
# Without cache
# Build time: 5분

# With cache (첫 번째)
# Build time: 5분 (캐시 생성)

# With cache (두 번째, 코드만 변경)
# Build time: 30초 (의존성 레이어 재사용)
```

---

## 🔧 실습 5: Multi-Platform 빌드

### Step 1: ARM64 + AMD64 빌드

```yaml
# .github/workflows/multi-platform.yml
name: Multi-Platform Build

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ secrets.DOCKERHUB_USERNAME }}/myapp
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
      
      - name: Build and push multi-platform
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ CI/CD 도구            │ 용도                        │
├──────────────────────┼────────────────────────────┤
│ GitHub Actions       │ GitHub 호스팅                │
├──────────────────────┼────────────────────────────┤
│ GitLab CI/CD         │ Self-hosted, 통합 환경       │
├──────────────────────┼────────────────────────────┤
│ Jenkins              │ 커스터마이징                  │
├──────────────────────┼────────────────────────────┤
│ CircleCI             │ 클라우드 빌드                 │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. Multi-stage build
2. Layer caching
3. 자동 테스트
4. Security scanning
5. 버전 태깅
```

---

## 🎓 연습 문제

### 문제 1: GitHub Actions에서 Secret을 안전하게 관리하려면?

<details>
<summary>정답 보기</summary>

**1. Repository Secrets 설정:**
```
Settings → Secrets and variables → Actions

추가할 Secrets:
- DOCKERHUB_USERNAME
- DOCKERHUB_TOKEN
- KUBE_CONFIG
```

**2. Workflow에서 사용:**
```yaml
- name: Login to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}
```

**3. Environment Secrets:**
```yaml
deploy:
  environment: production
  steps:
    - name: Deploy
      run: |
        echo "Deploying to ${{ secrets.PROD_SERVER }}"
```

**4. OIDC (권장):**
```yaml
permissions:
  id-token: write
  contents: read

- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubAction
    aws-region: us-east-1
```

</details>

### 문제 2: 빌드 시간을 단축하는 방법은?

<details>
<summary>정답 보기</summary>

**1. Layer Caching:**
```dockerfile
# ❌ Bad
COPY . .
RUN npm install

# ✅ Good
COPY package*.json ./
RUN npm install
COPY . .
```

**2. GitHub Actions Cache:**
```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

**3. BuildKit:**
```yaml
- name: Build
  run: docker build .
  env:
    DOCKER_BUILDKIT: 1
```

**4. Parallel Builds:**
```yaml
strategy:
  matrix:
    platform: [linux/amd64, linux/arm64]
```

**5. 불필요한 파일 제외:**
```dockerignore
node_modules
.git
*.md
tests/
```

</details>

### 문제 3: Pull Request에서만 빌드하고 푸시는 안 하려면?

<details>
<summary>정답 보기</summary>

```yaml
on:
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build (no push)
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false  # PR에서는 푸시 안 함
          tags: myapp:pr-${{ github.event.number }}
      
      - name: Test image
        run: |
          docker run myapp:pr-${{ github.event.number }} npm test
```

**조건부 푸시:**
```yaml
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    push: ${{ github.event_name != 'pull_request' }}
```

</details>

---

## 📌 핵심 요약

```
Docker in CI/CD 핵심:
1. 자동화된 빌드
2. 자동 테스트 실행
3. 레지스트리 푸시
4. 배포 자동화
5. 롤백 전략

Best Practices:
✅ Multi-stage build
✅ Layer caching
✅ Secret 암호화
✅ 자동 테스트
✅ 버전 태깅
```

---

## 📚 참고 자료

- [GitHub Actions Documentation](https://docs.github.com/actions)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [Docker Build with Buildx](https://docs.docker.com/build/buildx/)

---

## 🤔 생각해볼 문제

1. Self-hosted runner vs GitHub-hosted runner - 언제 Self-hosted를 사용하는가?
2. 모든 커밋마다 Docker 이미지를 빌드하는 것이 효율적인가?
3. CI/CD 파이프라인이 실패했을 때 알림을 어떻게 받는가?

> 💡 **답변**:
> 
> **1) Self-hosted Runner:**
> 
> **GitHub-hosted (기본):**
> - 무료 (제한 있음)
> - 관리 불필요
> - 2000분/월
> 
> **Self-hosted 사용 시:**
> - 무제한 빌드 시간
> - 더 강력한 하드웨어
> - 사내 리소스 접근
> - 비용 절감 (대규모)
> 
> **2) 효율적인 빌드 전략:**
> ```yaml
> # Path filter 사용
> on:
>   push:
>     paths:
>       - 'src/**'
>       - 'Dockerfile'
>       - 'package*.json'
> 
> # Branch 제한
> on:
>   push:
>     branches:
>       - main
>       - 'release/**'
> ```
> 
> **3) 알림 설정:**
> ```yaml
> - name: Slack Notification
>   if: failure()
>   uses: 8398a7/action-slack@v3
>   with:
>     status: ${{ job.status }}
>     text: 'Build failed!'
>     webhook_url: ${{ secrets.SLACK_WEBHOOK }}
> ```

---

<div align="center">

**[다음: Image Tagging ➡️](./02-Image-Tagging.md)**

</div>
