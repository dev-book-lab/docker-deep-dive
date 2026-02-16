# 02. Image Tagging - 태깅 전략, 버저닝

## 🎯 이 챕터에서 배울 것

- **태깅 전략**: latest, semantic versioning, git-based
- **버전 관리**: SemVer, CalVer
- **자동 태깅**: CI/CD 통합
- **멀티 태그**: 하나의 이미지에 여러 태그
- **태그 관리**: 정리 및 보안
- **실전 구현**: 프로덕션 태깅 전략

## 📌 왜 중요한가?

**"태그는 이미지 버전을 식별하고 배포를 추적하는 핵심입니다."**

```
Image Tagging의 핵심:

Without Proper Tagging (latest만 사용):
┌─────────────────────────────────────────────────┐
│ Poor Tagging Strategy                           │
│                                                 │
│ docker pull myapp:latest                        │
│                                                 │
│ 문제:                                            │
│ - 어떤 버전인지 모름                                │
│ - 롤백 불가 (이전 버전 찾기 어려움)                    │
│ - 재현 불가 (같은 환경 구성 어려움)                    │
│ - 디버깅 어려움 (어떤 코드인지 불명확)                  │
└─────────────────────────────────────────────────┘

With Proper Tagging:
┌─────────────────────────────────────────────────┐
│ Good Tagging Strategy                           │
│                                                 │
│ myapp:v1.2.3          (Semantic Version)        │
│ myapp:1.2             (Major.Minor)             │
│ myapp:1               (Major)                   │
│ myapp:latest          (Latest stable)           │
│ myapp:abc1234         (Git commit SHA)          │
│ myapp:main-abc1234    (Branch + SHA)            │
│ myapp:pr-123          (Pull Request)            │
│ myapp:20240115        (Date-based)              │
│                                                 │
│ 장점:                                            │
│ ✅ 명확한 버전 식별                                 │
│ ✅ 쉬운 롤백                                      │
│ ✅ 재현 가능                                      │
│ ✅ 디버깅 용이                                     │
└─────────────────────────────────────────────────┘

태깅 전략 비교:
┌────────────────┬──────────────┬──────────────┐
│ 전략            │ 예시          │ 용도          │
├────────────────┼──────────────┼──────────────┤
│ latest         │ latest       │ 최신 stable   │
├────────────────┼──────────────┼──────────────┤
│ Semantic       │ v1.2.3       │ 릴리스 버전     │
├────────────────┼──────────────┼──────────────┤
│ Git SHA        │ abc1234      │ 정확한 추적     │
├────────────────┼──────────────┼──────────────┤
│ Branch         │ main         │ 브랜치별       │
├────────────────┼──────────────┼──────────────┤
│ Date           │ 20240115     │ 날짜 기반      │
├────────────────┼──────────────┼──────────────┤
│ PR             │ pr-123       │ 테스트         │
└────────────────┴──────────────┴──────────────┘

Semantic Versioning (SemVer):
v1.2.3
│ │ │
│ │ └─ PATCH (버그 픽스)
│ └─── MINOR (하위 호환 기능 추가)
└───── MAJOR (하위 호환 안 되는 변경)

예:
v1.0.0 → v1.0.1: 버그 수정
v1.0.1 → v1.1.0: 새 기능 (하위 호환)
v1.1.0 → v2.0.0: Breaking change

Multi-tagging:
┌─────────────────────────────────────────────────┐
│ 하나의 이미지, 여러 태그                              │
│                                                 │
│  Image SHA: sha256:abc123...                    │
│  ├─ myapp:v1.2.3                                │
│  ├─ myapp:1.2                                   │
│  ├─ myapp:1                                     │
│  ├─ myapp:latest                                │
│  └─ myapp:main-abc1234                          │
│                                                 │
│  모두 같은 이미지를 가리킴!                           │
│  레지스트리 저장 공간 추가 사용 안 함                   │
└─────────────────────────────────────────────────┘

태그 선택 가이드:
Development:
- main, develop (브랜치명)
- pr-123 (Pull Request)
- abc1234 (Git SHA)

Staging:
- v1.2.3-rc.1 (Release Candidate)
- staging
- abc1234

Production:
- v1.2.3 (정확한 버전)
- latest (최신 stable)
- 1.2, 1 (Major.Minor, Major)
```

**실무 영향:**
- **추적성**: 어떤 코드가 배포됐는지 명확히 파악
- **롤백**: 이전 버전으로 쉽게 복귀
- **안정성**: 프로덕션에 정확한 버전 배포
- **협업**: 팀원 간 명확한 소통

---

## 🔬 Deep Dive

### 1. Semantic Versioning (SemVer)

#### 버전 형식

```bash
MAJOR.MINOR.PATCH

# 예시
v1.0.0  # 초기 릴리스
v1.0.1  # 버그 수정
v1.1.0  # 새 기능 추가 (하위 호환)
v2.0.0  # Breaking change

# Pre-release
v1.2.3-alpha.1
v1.2.3-beta.1
v1.2.3-rc.1

# Build metadata
v1.2.3+20240115
v1.2.3+001
```

#### 버전 올리기 규칙

```bash
# PATCH (버그 수정)
- 하위 호환되는 버그 수정
- 1.0.0 → 1.0.1

# MINOR (기능 추가)
- 하위 호환되는 새 기능
- 1.0.1 → 1.1.0
- PATCH는 0으로 리셋

# MAJOR (Breaking change)
- 하위 호환 안 되는 변경
- 1.1.0 → 2.0.0
- MINOR, PATCH 모두 0으로 리셋
```

---

### 2. Git-based Tagging

#### Git SHA

```bash
# Short SHA (7자리)
myapp:abc1234

# Full SHA
myapp:abc1234567890abcdef1234567890abcdef12

# Branch + SHA
myapp:main-abc1234
myapp:develop-xyz5678
```

#### Git Tags

```bash
# Git tag 생성
git tag -a v1.2.3 -m "Release version 1.2.3"
git push origin v1.2.3

# CI/CD에서 자동 태깅
if [[ $GITHUB_REF == refs/tags/* ]]; then
  VERSION=${GITHUB_REF#refs/tags/}
  docker build -t myapp:$VERSION .
fi
```

---

## 🔧 실습 1: 기본 태깅 전략

### Step 1: 수동 태깅

```bash
# 1. 이미지 빌드
docker build -t myapp:latest .

# 2. 버전 태그 추가
docker tag myapp:latest myapp:v1.2.3
docker tag myapp:latest myapp:1.2
docker tag myapp:latest myapp:1

# 3. 확인
docker images myapp
# REPOSITORY   TAG       IMAGE ID
# myapp        latest    abc123...
# myapp        v1.2.3    abc123...
# myapp        1.2       abc123...
# myapp        1         abc123...

# 4. 레지스트리에 푸시
docker push myapp:v1.2.3
docker push myapp:1.2
docker push myapp:1
docker push myapp:latest
```

### Step 2: Git SHA 태깅

```bash
# 현재 Git SHA 가져오기
GIT_SHA=$(git rev-parse --short HEAD)

# 빌드 및 태깅
docker build -t myapp:$GIT_SHA .
docker tag myapp:$GIT_SHA myapp:latest

# Branch + SHA
GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
docker tag myapp:$GIT_SHA myapp:${GIT_BRANCH}-${GIT_SHA}

# 푸시
docker push myapp:$GIT_SHA
docker push myapp:${GIT_BRANCH}-${GIT_SHA}
docker push myapp:latest
```

---

## 🔧 실습 2: GitHub Actions 자동 태깅

### Step 1: SemVer 자동 태깅

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'  # v1.2.3 형식

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      # 태그 추출
      - name: Extract version
        id: version
        run: |
          # v1.2.3 → 1.2.3
          VERSION=${GITHUB_REF#refs/tags/v}
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          
          # 1.2.3 → 1.2
          MINOR=${VERSION%.*}
          echo "minor=$MINOR" >> $GITHUB_OUTPUT
          
          # 1.2.3 → 1
          MAJOR=${VERSION%%.*}
          echo "major=$MAJOR" >> $GITHUB_OUTPUT
      
      # 빌드 및 푸시 (여러 태그)
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:v${{ steps.version.outputs.version }}
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ steps.version.outputs.minor }}
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ steps.version.outputs.major }}
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:latest
```

### Step 2: 테스트

```bash
# 1. Git tag 생성
git tag -a v1.2.3 -m "Release 1.2.3"
git push origin v1.2.3

# 2. GitHub Actions 실행 확인

# 3. 생성된 태그 확인
docker pull username/myapp:v1.2.3
docker pull username/myapp:1.2
docker pull username/myapp:1
docker pull username/myapp:latest

# 모두 같은 이미지!
```

---

## 🔧 실습 3: Branch-based 태깅

### Step 1: Branch별 자동 태깅

```yaml
# .github/workflows/branch-tagging.yml
name: Branch Build

on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'

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
      
      - name: Generate tags
        id: tags
        run: |
          # Branch name (main, develop, feature-xxx)
          BRANCH=${GITHUB_REF#refs/heads/}
          BRANCH_SAFE=${BRANCH//\//-}  # / → -
          
          # Short SHA
          SHA_SHORT=${GITHUB_SHA::7}
          
          # 태그 생성
          TAGS="ghcr.io/${{ github.repository }}:${BRANCH_SAFE}"
          TAGS="${TAGS},ghcr.io/${{ github.repository }}:${BRANCH_SAFE}-${SHA_SHORT}"
          
          # main 브랜치는 latest도 추가
          if [ "$BRANCH" = "main" ]; then
            TAGS="${TAGS},ghcr.io/${{ github.repository }}:latest"
          fi
          
          echo "tags=$TAGS" >> $GITHUB_OUTPUT
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.tags.outputs.tags }}
```

### Step 2: 결과

```bash
# main 브랜치 푸시
git push origin main

# 생성되는 태그:
# - ghcr.io/user/repo:main
# - ghcr.io/user/repo:main-abc1234
# - ghcr.io/user/repo:latest

# feature 브랜치 푸시
git checkout -b feature/new-ui
git push origin feature/new-ui

# 생성되는 태그:
# - ghcr.io/user/repo:feature-new-ui
# - ghcr.io/user/repo:feature-new-ui-xyz5678
```

---

## 🔧 실습 4: Docker Metadata Action

### Step 1: 자동 메타데이터 생성

```yaml
# .github/workflows/metadata.yml
name: Docker Metadata

on:
  push:
    branches:
      - main
      - develop
    tags:
      - 'v*'
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: |
            docker.io/${{ secrets.DOCKERHUB_USERNAME }}/myapp
            ghcr.io/${{ github.repository }}
          
          tags: |
            # Branch
            type=ref,event=branch
            
            # PR
            type=ref,event=pr
            
            # Git tag
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            
            # Git SHA
            type=sha
            
            # Latest for main
            type=raw,value=latest,enable={{is_default_branch}}
          
          labels: |
            org.opencontainers.image.title=My App
            org.opencontainers.image.description=My awesome application
            org.opencontainers.image.vendor=${{ github.repository_owner }}
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

### Step 2: 생성되는 태그 예시

```bash
# main 브랜치 푸시
docker.io/username/myapp:main
docker.io/username/myapp:sha-abc1234
docker.io/username/myapp:latest
ghcr.io/user/repo:main
ghcr.io/user/repo:sha-abc1234
ghcr.io/user/repo:latest

# v1.2.3 태그 푸시
docker.io/username/myapp:v1.2.3
docker.io/username/myapp:1.2
docker.io/username/myapp:1
ghcr.io/user/repo:v1.2.3
ghcr.io/user/repo:1.2
ghcr.io/user/repo:1

# PR #123
docker.io/username/myapp:pr-123
ghcr.io/user/repo:pr-123
```

---

## 🔧 실습 5: Tag Cleanup (정리)

### Step 1: 오래된 태그 삭제

```bash
#!/bin/bash
# cleanup-old-tags.sh

# 변수 설정
REGISTRY="docker.io"
USERNAME="myusername"
IMAGE="myapp"
TOKEN="your-token"

# 30일 이상 된 PR 태그 삭제
docker run --rm \
  -e REGISTRY_USERNAME=$USERNAME \
  -e REGISTRY_PASSWORD=$TOKEN \
  registry-cleanup \
  --registry $REGISTRY \
  --repo $USERNAME/$IMAGE \
  --tag-pattern "^pr-.*" \
  --older-than 30
```

### Step 2: GitHub Actions로 자동 정리

```yaml
# .github/workflows/cleanup-tags.yml
name: Cleanup Old Tags

on:
  schedule:
    - cron: '0 0 * * 0'  # 매주 일요일
  workflow_dispatch:  # 수동 실행

jobs:
  cleanup:
    runs-on: ubuntu-latest
    
    steps:
      - name: Delete old PR tags
        uses: snok/container-retention-policy@v2
        with:
          image-names: myapp
          cut-off: 30 days ago UTC
          account-type: org
          org-name: ${{ github.repository_owner }}
          filter-tags: pr-*
          token: ${{ secrets.GITHUB_TOKEN }}
```

### Step 3: Docker Hub Retention Policy

```bash
# Docker Hub에서 설정
Settings → Repository → Automated Builds → Tag Retention Policy

규칙 예:
- latest: 영구 보관
- v*: 영구 보관
- pr-*: 7일 후 삭제
- *-*: 30일 후 삭제
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 환경                  │ 태그 전략                    │
├──────────────────────┼────────────────────────────┤
│ Development          │ branch, pr-123, sha        │
├──────────────────────┼────────────────────────────┤
│ Staging              │ v1.2.3-rc.1, staging       │
├──────────────────────┼────────────────────────────┤
│ Production           │ v1.2.3, 1.2, 1, latest     │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. 프로덕션: SemVer
2. 개발: Git SHA
3. Multi-tagging 사용
4. latest는 신중히
5. 정기적 정리
```

---

## 🎓 연습 문제

### 문제 1: latest 태그를 사용해야 하는가?

<details>
<summary>정답 보기</summary>

**latest의 문제:**
```bash
# ❌ 문제 상황
docker pull myapp:latest
# 어떤 버전인지 모름!

# Production에서
docker run myapp:latest
# 내일 다시 실행하면 다른 이미지!
```

**권장 사용:**
```bash
# ✅ Development
docker pull myapp:latest  # 최신 개발 버전

# ✅ CI/CD
FROM myapp:latest  # 빌드 시 최신 베이스

# ❌ Production
# 절대 latest 사용 금지!
# 정확한 버전 사용
docker run myapp:v1.2.3
```

**대안:**
```bash
# Stable latest
myapp:stable  # 최신 stable 버전

# Environment-specific
myapp:production  # 프로덕션 latest
myapp:staging     # 스테이징 latest
```

</details>

### 문제 2: 이미지 태그를 변경할 수 있는가?

<details>
<summary>정답 보기</summary>

**기술적으론 가능, 하지만...**

```bash
# ❌ 나쁜 예
docker tag myapp:v1.0.0 myapp:v1.0.1
docker push myapp:v1.0.1

# 문제:
# - v1.0.1이 실제론 v1.0.0 코드
# - 신뢰성 파괴
# - 디버깅 불가능
```

**Immutable Tags (권장):**
```bash
# ✅ 좋은 예
# 한 번 푸시된 태그는 절대 변경 금지

# 새 버전 생성
docker tag myapp:v1.0.0 myapp:v1.0.1
# 코드 변경 후 새로 빌드!
```

**레지스트리 설정:**
```yaml
# Docker Hub: Repository Settings
# → Tag Immutability: Enabled

# Harbor: Project Settings
# → Immutability: Enabled

# 효과: 같은 태그 재푸시 시 에러
```

</details>

### 문제 3: 멀티 플랫폼 이미지의 태그는?

<details>
<summary>정답 보기</summary>

**단일 태그, 여러 아키텍처:**
```bash
# 빌드
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myapp:v1.2.3 \
  --push .

# 사용 (자동으로 맞는 아키텍처 선택)
# AMD64 머신
docker pull myapp:v1.2.3
# → linux/amd64 버전

# ARM64 머신 (M1 Mac, Raspberry Pi)
docker pull myapp:v1.2.3
# → linux/arm64 버전
```

**Manifest 확인:**
```bash
docker manifest inspect myapp:v1.2.3

# 출력:
# {
#   "manifests": [
#     {
#       "platform": {
#         "architecture": "amd64",
#         "os": "linux"
#       }
#     },
#     {
#       "platform": {
#         "architecture": "arm64",
#         "os": "linux"
#       }
#     }
#   ]
# }
```

**아키텍처별 태그 (비권장):**
```bash
# ❌ 사용자가 수동으로 선택
myapp:v1.2.3-amd64
myapp:v1.2.3-arm64

# ✅ 자동 선택 (권장)
myapp:v1.2.3
```

</details>

---

## 📌 핵심 요약

```
Image Tagging 핵심:
1. Semantic Versioning (프로덕션)
2. Git SHA (추적)
3. Multi-tagging (유연성)
4. Immutable tags (안정성)
5. 정기적 정리 (관리)

Best Practices:
✅ v1.2.3 (정확한 버전)
✅ Git SHA 포함
✅ latest는 개발용만
✅ 태그 변경 금지
✅ 자동화된 태깅
```

---

## 📚 참고 자료

- [Semantic Versioning](https://semver.org/)
- [Docker Tagging Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Container Image Tagging Best Practices](https://cloud.google.com/architecture/best-practices-for-building-containers#tagging)

---

## 🤔 생각해볼 문제

1. Monorepo에서 여러 서비스를 어떻게 태깅하는가?
2. 이미지 크기가 크다면 태그 전략을 어떻게 조정하는가?
3. 롤백 시 어떤 태그를 사용하는가?

> 💡 **답변**:
> 
> **1) Monorepo 태깅:**
> 
> ```bash
> # 서비스별 버전
> frontend:v1.2.3
> backend:v1.2.3
> api:v1.2.3
> 
> # 또는 통합 버전
> myapp:v1.2.3-frontend
> myapp:v1.2.3-backend
> myapp:v1.2.3-api
> 
> # Git SHA (변경 추적)
> frontend:abc1234
> backend:abc1234  # 같은 커밋
> ```
> 
> **2) 큰 이미지 최적화:**
> 
> ```bash
> # Base 이미지 재사용
> myapp-base:v1.0.0  # 10GB
> myapp:v1.2.3       # +100MB (base 위에)
> 
> # Layer 공유
> - 같은 base → 한 번만 다운로드
> ```
> 
> **3) 롤백 전략:**
> 
> ```bash
> # 현재: v1.2.3
> # 문제 발생!
> 
> # 롤백
> kubectl set image deployment/myapp \
>   myapp=myapp:v1.2.2
> 
> # 또는 이전 SHA
> kubectl set image deployment/myapp \
>   myapp=myapp:abc1234
> ```

---

<div align="center">

**[⬅️ 이전: Docker in CI](./01-Docker-in-CI.md)** | **[다음: Registry Setup ➡️](./03-Registry-Setup.md)**

</div>
