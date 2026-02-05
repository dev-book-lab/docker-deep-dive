# 04. Cache Mechanism - 빌드 캐시 메커니즘

## 🎯 이 챕터에서 배울 것

- **Docker 빌드 캐시**의 동작 원리와 내부 구조
- **캐시 무효화 조건**과 최적화 전략
- **원격 캐시**를 활용한 CI/CD 성능 향상
- **BuildKit 캐시** 고급 기능

## 📌 왜 중요한가?

**"캐시를 이해하면 빌드 시간이 10배 빨라집니다."**

```
캐시 미활용:
- 매 빌드마다 전체 재실행
- 빌드 시간: 10분
- CI/CD 파이프라인: 느림
- 개발 생산성: 낮음

캐시 최적화:
- 변경된 부분만 재빌드
- 빌드 시간: 1분 (10배 빠름)
- CI/CD 파이프라인: 빠름
- 개발 생산성: 높음
```

**실무 영향:**
- 개발자 시간 절약: 하루 1시간+ 절약
- CI/CD 비용 절감: 빌드 시간 90% 감소
- 빠른 피드백: 버그 수정 속도 향상
- 팀 생산성: 대폭 증가

---

## 🔬 Deep Dive

### 1. 빌드 캐시 동작 원리

#### 캐시 레이어 구조

```
Docker 빌드 캐시는 레이어 단위로 동작

┌─────────────────────────────────────┐
│ Layer 5: CMD ["node", "server.js"]  │ ← 캐시됨
├─────────────────────────────────────┤
│ Layer 4: COPY . .                   │ ← 소스 변경 시 재빌드
├─────────────────────────────────────┤
│ Layer 3: RUN npm install            │ ← package.json 변경 시 재빌드
├─────────────────────────────────────┤
│ Layer 2: COPY package*.json ./      │ ← 변경 없으면 캐시 사용
├─────────────────────────────────────┤
│ Layer 1: FROM node:18-alpine        │ ← 항상 캐시 사용
└─────────────────────────────────────┘

캐시 키 = 명령어 + 파일 체크섬
- FROM: 이미지 ID
- COPY/ADD: 파일 내용 체크섬
- RUN: 명령어 문자열
```

#### 캐시 검색 과정

```
1. 명령어 비교
   이전 빌드와 동일한 명령어?
   ↓ YES
   
2. 입력 체크
   파일이나 변수가 동일?
   ↓ YES
   
3. 캐시 히트!
   기존 레이어 재사용
   
   ↓ NO (어느 단계든)
   
4. 캐시 미스
   해당 레이어부터 재빌드
   하위 모든 레이어도 재빌드
```

---

### 2. 캐시 무효화 조건

#### FROM 명령어

```dockerfile
# 캐시 키: 이미지 ID + 태그
FROM node:18-alpine

# 캐시 무효화 조건:
# - 베이스 이미지 업데이트
# - 태그 변경
# - 이미지 삭제

# 예시:
FROM node:18-alpine  # 캐시 히트
# node:18-alpine이 업데이트되면
FROM node:18-alpine  # 캐시 미스 (새 이미지 ID)
```

#### COPY/ADD 명령어

```dockerfile
# 캐시 키: 명령어 + 파일 체크섬(SHA256)
COPY package.json ./

# 캐시 무효화 조건:
# - 파일 내용 변경 (체크섬 변경)
# - 파일 메타데이터 변경 (모드, 타임스탬프 제외)
# - 복사할 파일 경로 변경

# 주의: 타임스탬프는 무시됨!
COPY app.js ./
# app.js 터치만 해도 (내용 동일)
# → 캐시 히트 (체크섬 동일)
```

#### RUN 명령어

```dockerfile
# 캐시 키: 명령어 문자열
RUN npm install

# 캐시 무효화 조건:
# - 명령어 문자열 변경
# - 상위 레이어 변경 (캐스케이드)
# - 외부 의존성 변경 (감지 불가!)

# ⚠️ 주의: 외부 변경 감지 안 됨
RUN apt-get update && apt-get install curl
# 레포지토리 업데이트되어도 → 캐시 히트
# (명령어 문자열 동일)

# 해결: --no-cache 플래그 사용
RUN apt-get update && apt-get install curl
# 또는 날짜 포함
RUN apt-get update && apt-get install curl && echo "2024-02-05"
```

#### ENV, ARG 명령어

```dockerfile
# ENV: 캐시 키에 값 포함
ENV NODE_ENV=production
RUN npm install
# NODE_ENV 변경 시 → RUN도 재실행

# ARG: 빌드 타임에만 영향
ARG VERSION=1.0.0
RUN echo $VERSION > version.txt
# VERSION 변경 시 → RUN 재실행

# ⚠️ 주의: ARG는 FROM 전/후 범위 다름
ARG BASE_IMAGE=node:18
FROM ${BASE_IMAGE}  # 여기서 사용
ARG VERSION         # 여기서 다시 선언 필요
RUN echo $VERSION   # 여기서 사용
```

---

### 3. 캐시 최적화 전략

#### 전략 1: 변경 빈도 순서 배치

```dockerfile
# ❌ Before: 소스 변경마다 전체 재빌드
FROM node:18-alpine
WORKDIR /app
COPY . .                    # 소스 변경 → 캐시 무효화
RUN npm install             # 매번 재실행 (5분)
RUN npm run build           # 매번 재실행 (2분)
CMD ["npm", "start"]

# ✅ After: 의존성 먼저, 소스는 나중에
FROM node:18-alpine
WORKDIR /app

# 1. 의존성 파일만 복사 (변경 적음)
COPY package*.json ./
RUN npm ci --only=production  # 캐시 히트율 90%

# 2. 소스 복사 (변경 많음)
COPY . .
RUN npm run build             # 소스 변경 시만 재실행

CMD ["npm", "start"]

# 개선:
# - 의존성 변경 없으면: 10초 (npm install 스킵)
# - 의존성 변경 있으면: 5분 10초
```

#### 전략 2: 멀티 스테이지 캐시 활용

```dockerfile
# ✅ 각 스테이지가 독립적으로 캐시됨
FROM node:18-alpine AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
# → dependencies 스테이지 캐시

FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
# → builder 스테이지 캐시

FROM node:18-alpine
WORKDIR /app
COPY --from=dependencies /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./
CMD ["node", "dist/server.js"]

# 장점:
# - dependencies 변경 없으면 builder만 재빌드
# - 캐시 히트율 극대화
```

#### 전략 3: .dockerignore 활용

```dockerignore
# .dockerignore - 불필요한 파일 제외로 캐시 안정화

# 개발 파일
node_modules/
.git/
.gitignore

# 테스트
tests/
**/*.test.js
coverage/

# 문서
*.md
docs/

# 로그
logs/
*.log

# 환경 설정
.env
.env.local

# 효과:
# - 빌드 컨텍스트 크기 감소
# - 불필요한 파일 변경으로 인한 캐시 무효화 방지
# - package.json만 변경 시 캐시 유지
```

#### 전략 4: 명령어 분리

```dockerfile
# ❌ Before: 하나라도 변경되면 전체 재실행
RUN apt-get update && \
    apt-get install -y curl wget git vim && \
    apt-get clean

# ✅ After: 변경 빈도에 따라 분리
# 자주 변경되지 않는 것
RUN apt-get update && \
    apt-get install -y curl wget && \
    apt-get clean

# 가끔 추가/제거 되는 것
RUN apt-get install -y git vim && \
    apt-get clean

# 장점:
# - git, vim 추가/제거 시 curl, wget 캐시 유지
# 단점:
# - 레이어 수 증가

# 균형점 찾기:
# - 자주 변경: 분리
# - 거의 불변: 통합
```

---

### 4. 원격 캐시 (Remote Cache)

#### 원격 캐시란?

```
로컬 캐시의 한계:
- 개발자 A의 빌드 캐시
  → 개발자 B가 사용 불가
- CI/CD 서버의 캐시
  → 다른 러너가 사용 불가

원격 캐시:
- 중앙 레지스트리에 캐시 저장
- 팀 전체가 공유
- CI/CD 러너 간 공유

┌──────────┐     push cache     ┌──────────────┐
│ Dev A    │ ─────────────────→ │              │
└──────────┘                    │  Registry    │
                                │  (Harbor,    │
┌──────────┐     pull cache     │   ECR, etc)  │
│ Dev B    │ ←───────────────── │              │
└──────────┘                    └──────────────┘

┌──────────┐     share cache         ↑
│ CI/CD    │ ←───────────────────────┘
└──────────┘
```

#### BuildKit 인라인 캐시

```dockerfile
# Dockerfile (변경 없음)
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
CMD ["npm", "start"]
```

```bash
# 빌드 시 캐시 메타데이터 포함
docker buildx build \
  --cache-to type=inline \
  --tag myapp:latest \
  --push \
  .

# 다른 머신에서 캐시 활용
docker buildx build \
  --cache-from myapp:latest \
  --tag myapp:latest \
  .

# 동작:
# 1. myapp:latest 이미지 pull
# 2. 이미지 내 캐시 메타데이터 읽기
# 3. 캐시 가능한 레이어 재사용
```

#### Registry 캐시

```bash
# 전용 캐시 이미지 사용
docker buildx build \
  --cache-to type=registry,ref=myregistry/myapp:cache \
  --cache-from type=registry,ref=myregistry/myapp:cache \
  --tag myapp:latest \
  --push \
  .

# 장점:
# - 실제 이미지와 캐시 분리
# - 캐시 이미지 관리 용이
# - 더 많은 레이어 저장 가능
```

#### 로컬 캐시 디렉토리

```bash
# 로컬 디렉토리에 캐시 저장
docker buildx build \
  --cache-to type=local,dest=/tmp/cache \
  --cache-from type=local,src=/tmp/cache \
  --tag myapp:latest \
  .

# CI/CD에서 활용:
# .gitlab-ci.yml
build:
  script:
    - docker buildx build 
        --cache-to type=local,dest=./cache
        --cache-from type=local,src=./cache
        --tag myapp:latest .
  cache:
    paths:
      - cache/

# 효과:
# - 파이프라인 간 캐시 공유
# - 빌드 시간 대폭 감소
```

---

### 5. BuildKit 고급 캐시 기능

#### 캐시 마운트

```dockerfile
# 기존 방식: 매번 다운로드
FROM golang:1.21-alpine
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download  # 매번 다운로드
COPY . .
RUN go build -o app

# BuildKit 캐시 마운트
# syntax=docker/dockerfile:1
FROM golang:1.21-alpine
WORKDIR /app
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download  # 캐시에서 재사용!
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o app

# 효과:
# - go mod download: 5분 → 5초
# - go build: 3분 → 30초
# - 총: 8분 → 35초 (13배 빠름)
```

#### 언어별 캐시 마운트 예시

```dockerfile
# Node.js - npm 캐시
# syntax=docker/dockerfile:1
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# Python - pip 캐시
# syntax=docker/dockerfile:1
FROM python:3.11-alpine
WORKDIR /app
COPY requirements.txt ./
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Rust - cargo 캐시
# syntax=docker/dockerfile:1
FROM rust:1.74-alpine
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    --mount=type=cache,target=/app/target \
    cargo build --release
```

---

## 💻 실습

### 실습 1: 캐시 동작 확인

#### 준비

```bash
mkdir cache-demo
cd cache-demo

cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

# Step 1: 의존성 파일 복사
COPY package*.json ./

# Step 2: 의존성 설치
RUN npm ci --only=production

# Step 3: 소스 복사
COPY . .

CMD ["node", "app.js"]
EOF

cat > package.json << 'EOF'
{
  "name": "cache-demo",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

cat > app.js << 'EOF'
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello from cache demo!');
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF
```

#### 테스트 1: 첫 빌드 (캐시 없음)

```bash
# 시간 측정
time docker build -t cache-demo:v1 .

# 출력:
# Step 2/5 : COPY package*.json ./
#  ---> 8a3f2c1b4e5d
# Step 3/5 : RUN npm ci --only=production
#  ---> Running in 123abc456def
# added 57 packages in 45s      ← 45초 소요
#  ---> 9b4e3f2a1c8d
# Step 4/5 : COPY . .
#  ---> 5c6d7e8f9a0b
# 
# real    0m50.234s              ← 총 50초
```

#### 테스트 2: 재빌드 (캐시 히트)

```bash
# 변경 없이 재빌드
time docker build -t cache-demo:v2 .

# 출력:
# Step 2/5 : COPY package*.json ./
#  ---> Using cache              ← 캐시 사용!
#  ---> 8a3f2c1b4e5d
# Step 3/5 : RUN npm ci --only=production
#  ---> Using cache              ← 캐시 사용!
#  ---> 9b4e3f2a1c8d
# Step 4/5 : COPY . .
#  ---> Using cache              ← 캐시 사용!
#  ---> 5c6d7e8f9a0b
#
# real    0m1.234s               ← 1초! (50배 빠름)
```

#### 테스트 3: 소스 변경 (부분 캐시)

```bash
# 소스 코드만 변경
cat > app.js << 'EOF'
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello from UPDATED cache demo!');  // 변경
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

time docker build -t cache-demo:v3 .

# 출력:
# Step 2/5 : COPY package*.json ./
#  ---> Using cache              ← 캐시 사용
#  ---> 8a3f2c1b4e5d
# Step 3/5 : RUN npm ci --only=production
#  ---> Using cache              ← 캐시 사용
#  ---> 9b4e3f2a1c8d
# Step 4/5 : COPY . .
#  ---> 5c6d7e8f9a0b            ← 새로 빌드 (소스 변경)
#
# real    0m2.567s               ← 2.5초 (npm install 스킵!)
```

#### 테스트 4: 의존성 변경 (캐시 무효화)

```bash
# package.json 변경 (의존성 추가)
cat > package.json << 'EOF'
{
  "name": "cache-demo",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2",
    "lodash": "^4.17.21"
  }
}
EOF

time docker build -t cache-demo:v4 .

# 출력:
# Step 2/5 : COPY package*.json ./
#  ---> 1a2b3c4d5e6f            ← 새로 빌드 (파일 변경)
# Step 3/5 : RUN npm ci --only=production
#  ---> Running in 789ghi012jkl
# added 59 packages in 48s       ← 재실행 (48초)
#  ---> 2b3c4d5e6f7a
# Step 4/5 : COPY . .
#  ---> 3c4d5e6f7a8b            ← 새로 빌드
#
# real    0m52.345s              ← 52초 (캐시 무효화)
```

---

### 실습 2: 캐시 최적화 비교

#### Before: 비최적화 Dockerfile

```dockerfile
# Dockerfile.bad
FROM node:18-alpine

WORKDIR /app

# ❌ 모든 파일을 먼저 복사
COPY . .

# 의존성 설치
RUN npm install

# 빌드
RUN npm run build

CMD ["npm", "start"]
```

```bash
# 첫 빌드
time docker build -f Dockerfile.bad -t demo:bad .
# real: 0m45s

# 소스 변경 후 재빌드
echo "// updated" >> app.js
time docker build -f Dockerfile.bad -t demo:bad .
# real: 0m45s (캐시 없음, 매번 전체 재빌드!)
```

#### After: 최적화 Dockerfile

```dockerfile
# Dockerfile.good
FROM node:18-alpine

WORKDIR /app

# ✅ 의존성 파일만 먼저
COPY package*.json ./
RUN npm ci --only=production

# 소스는 나중에
COPY . .
RUN npm run build

CMD ["npm", "start"]
```

```bash
# 첫 빌드
time docker build -f Dockerfile.good -t demo:good .
# real: 0m45s

# 소스 변경 후 재빌드
echo "// updated" >> app.js
time docker build -f Dockerfile.good -t demo:good .
# real: 0m3s (15배 빠름! npm install 캐시 히트)
```

#### 비교 결과

```
┌──────────────────┬─────────────┬──────────────┬──────────┐
│   시나리오         │ Before (bad)│ After (good) │   개선    │
├──────────────────┼─────────────┼──────────────┼──────────┤
│ 첫 빌드            │    45초     │     45초      │    -     │
├──────────────────┼─────────────┼──────────────┼──────────┤
│ 소스만 변경         │    45초     │     3초       │  15배↑   │
├──────────────────┼─────────────┼──────────────┼──────────┤
│ 의존성 변경         │    45초     │     45초      │    -     │
├──────────────────┼─────────────┼──────────────┼──────────┤
│ 변경 없음          │    45초      │     1초      │  45배↑    │
└──────────────────┴─────────────┴──────────────┴──────────┘

하루 10번 빌드 시:
- Before: 450초 (7.5분)
- After: 75초 (1.25분)
- 절감: 375초 (6.25분, 83% 감소)
```

---

### 실습 3: BuildKit 캐시 마운트

#### 준비

```bash
# BuildKit 활성화
export DOCKER_BUILDKIT=1

mkdir buildkit-cache-demo
cd buildkit-cache-demo

cat > go.mod << 'EOF'
module demo

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/go-redis/redis/v8 v8.11.5
)
EOF

cat > main.go << 'EOF'
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()
    r.GET("/", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "hello"})
    })
    r.Run(":8080")
}
EOF
```

#### Before: 캐시 마운트 없음

```dockerfile
# Dockerfile.nocache
FROM golang:1.21-alpine

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o server

CMD ["./server"]
```

```bash
# 첫 빌드
time docker build -f Dockerfile.nocache -t go-demo:nocache .
# go mod download: 2m30s

# go.mod 변경 없이 재빌드
time docker build -f Dockerfile.nocache -t go-demo:nocache .
# Using cache → 즉시 완료

# go.mod 변경 후 재빌드
echo "// comment" >> go.mod
time docker build -f Dockerfile.nocache -t go-demo:nocache .
# go mod download: 2m30s (다시 다운로드!)
```

#### After: 캐시 마운트 사용

```dockerfile
# Dockerfile.cache
# syntax=docker/dockerfile:1
FROM golang:1.21-alpine

WORKDIR /app

COPY go.mod go.sum ./

# ✅ 캐시 마운트 사용
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o server

CMD ["./server"]
```

```bash
# 첫 빌드
time docker build -f Dockerfile.cache -t go-demo:cache .
# go mod download: 2m30s (첫 번째는 동일)

# go.mod 변경 후 재빌드
echo "// comment" >> go.mod
time docker build -f Dockerfile.cache -t go-demo:cache .
# go mod download: 5s (캐시에서 재사용!)

# 개선:
# 2m30s → 5s (30배 빠름)
```

---

### 실습 4: 원격 캐시 (CI/CD)

#### GitHub Actions 예제

```yaml
# .github/workflows/docker-build.yml
name: Docker Build

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: myapp:latest
          # ✅ 원격 캐시 사용
          cache-from: type=registry,ref=myapp:cache
          cache-to: type=registry,ref=myapp:cache,mode=max

# 효과:
# - 첫 실행: 5분
# - 이후 실행: 30초 (캐시 활용)
# - 다른 러너에서도 캐시 공유
```

#### GitLab CI 예제

```yaml
# .gitlab-ci.yml
build:
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_BUILDKIT: 1
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      docker buildx build \
        --cache-from type=registry,ref=$CI_REGISTRY_IMAGE:cache \
        --cache-to type=registry,ref=$CI_REGISTRY_IMAGE:cache,mode=max \
        --tag $CI_REGISTRY_IMAGE:latest \
        --push \
        .
  # 로컬 캐시도 저장
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - cache/
```

---

## 🔥 실전 적용

### 시나리오 1: 모노레포 빌드 최적화

**상황:**
- 10개 마이크로서비스 (모노레포)
- 공통 의존성 많음
- 빌드 시간: 서비스당 5분

**문제:**
```dockerfile
# 기존 Dockerfile (각 서비스)
FROM node:18-alpine
WORKDIR /app
COPY . .                    # 전체 레포 복사
RUN npm install             # 전체 의존성 설치
RUN npm run build:service-a # 해당 서비스만 빌드
```

**최적화:**
```dockerfile
# 1단계: 공통 의존성 레이어
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
COPY packages/shared/package*.json ./packages/shared/
RUN npm ci --only=production

# 2단계: 서비스별 빌드
FROM node:18-alpine AS service-a-builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY packages/service-a ./packages/service-a
COPY packages/shared ./packages/shared
RUN npm run build:service-a

# 3단계: 최종 이미지
FROM node:18-alpine
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=service-a-builder /app/dist ./dist
CMD ["node", "dist/service-a/main.js"]
```

**결과:**
```
빌드 시간:
- Before: 50분 (10개 × 5분)
- After: 10분 (공통 의존성 1회 + 서비스별 1분)
- 절감: 80%

캐시 효과:
- 공통 의존성 변경: 10분 (전체 재빌드)
- 특정 서비스만 변경: 1분 (해당 서비스만)
- 캐시 히트율: 90%
```

---

### 시나리오 2: CI/CD 파이프라인 가속화

**상황:**
- 하루 100번 빌드
- 평균 빌드 시간: 10분
- 총 소요 시간: 1,000분 (16.6시간)

**원격 캐시 적용:**

```dockerfile
# Dockerfile (변경 없음)
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build
CMD ["npm", "start"]
```

```yaml
# .github/workflows/build.yml
- name: Build with cache
  uses: docker/build-push-action@v4
  with:
    context: .
    push: true
    tags: myapp:${{ github.sha }}
    cache-from: |
      type=registry,ref=myapp:cache
      type=registry,ref=myapp:latest
    cache-to: type=registry,ref=myapp:cache,mode=max
```

**결과:**
```
빌드 시간 분포:
- 첫 빌드: 10분 (1%)
- 의존성 변경: 10분 (10%)
- 소스만 변경: 1분 (89%)

하루 평균:
- Before: 1,000분 (16.6시간)
- After: 199분 (3.3시간)
  = 10분×1 + 10분×10 + 1분×89
- 절감: 801분 (80%)

월 비용 (GitHub Actions):
- Before: $500
- After: $100
- 절감: $400/월
```

---

### 시나리오 3: 멀티 아키텍처 빌드

**상황:**
- ARM64, AMD64 지원 필요
- 각 아키텍처별 빌드: 15분
- 총 빌드 시간: 30분

**캐시 최적화:**

```bash
# buildx 빌더 생성
docker buildx create --name multiarch --use

# 멀티 아키텍처 빌드 + 캐시
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --cache-from type=registry,ref=myapp:cache-amd64 \
  --cache-from type=registry,ref=myapp:cache-arm64 \
  --cache-to type=registry,ref=myapp:cache-amd64,mode=max \
  --cache-to type=registry,ref=myapp:cache-arm64,mode=max \
  --tag myapp:latest \
  --push \
  .
```

**결과:**
```
첫 빌드:
- 시간: 30분
- 캐시: 생성

이후 빌드 (소스만 변경):
- AMD64: 15분 → 2분
- ARM64: 15분 → 2분
- 총: 30분 → 4분 (87% 감소)

의존성 변경 없는 빌드:
- 총: 30분 → 4분
- 하루 10번: 300분 → 40분
- 절감: 260분/일
```

---

## ⚡ 캐시 최적화 체크리스트

### Dockerfile 구조

```
□ 변경 빈도 낮은 것부터 배치
□ COPY는 필요한 파일만 명시적으로
□ .dockerignore 작성
□ 멀티 스테이지 활용
□ RUN 명령어 적절히 분리/병합
```

### 의존성 관리

```
□ package.json 등 의존성 파일 먼저 COPY
□ 의존성 설치 후 소스 COPY
□ --only=production 사용
□ 캐시 정리 명령어 추가
□ lock 파일 활용 (package-lock.json, go.sum 등)
```

### BuildKit 활용

```
□ DOCKER_BUILDKIT=1 환경변수 설정
□ 캐시 마운트 활용 (--mount=type=cache)
□ 언어별 캐시 경로 확인
□ syntax=docker/dockerfile:1 선언
□ 병렬 빌드 가능하도록 구조화
```

### CI/CD 최적화

```
□ 원격 캐시 설정
□ cache-from/cache-to 파라미터 사용
□ 캐시 이미지 태그 전략 수립
□ 빌드 시간 모니터링
□ 캐시 히트율 추적
```

---

## 🚫 안티패턴

### 1. 과도한 COPY . .

```dockerfile
# ❌ 모든 것을 한 번에
FROM node:18-alpine
COPY . .
RUN npm install
# 작은 파일 하나 변경해도 전체 캐시 무효화

# ✅ 선택적 복사
FROM node:18-alpine
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/
COPY public/ ./public/
# 각 디렉토리별로 캐시 가능
```

### 2. 외부 의존성 캐시 무시

```dockerfile
# ❌ 항상 최신 버전 가져옴 (캐시 무효화)
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl
# apt 레포지토리 변경 시 캐시 무효화

# ✅ 날짜를 명시하여 의도적 무효화
FROM ubuntu:22.04
ARG CACHE_DATE=2024-02-05
RUN apt-get update && apt-get install -y curl
# CACHE_DATE 변경 시에만 재실행
```

### 3. 불필요한 캐시 무효화

```dockerfile
# ❌ 타임스탬프가 포함된 파일 생성
FROM alpine
RUN date > /tmp/build-date.txt
# 매번 다른 결과 → 캐시 무효화

# ✅ 빌드 인자로 제어
FROM alpine
ARG BUILD_DATE
RUN echo $BUILD_DATE > /tmp/build-date.txt
# BUILD_DATE 변경 시에만 무효화
```

### 4. 캐시 마운트 오용

```dockerfile
# ❌ 최종 이미지에 필요한 파일 캐시에 저장
# syntax=docker/dockerfile:1
FROM node:18-alpine
RUN --mount=type=cache,target=/app/node_modules \
    npm install
COPY . .
CMD ["npm", "start"]
# node_modules가 캐시에만 있고 이미지에 없음!

# ✅ 캐시는 중간 다운로드용
# syntax=docker/dockerfile:1
FROM node:18-alpine
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production
# npm 캐시만 마운트, node_modules는 이미지에 포함
```

---

## 🎓 핵심 정리

### 1. 캐시 동작 원리

```
캐시 키 구성:
- FROM: 이미지 ID
- COPY/ADD: 파일 체크섬
- RUN: 명령어 문자열
- ENV/ARG: 변수 값

캐시 체인:
레이어 N 캐시 미스 → N+1부터 모두 재빌드

최적화 원칙:
변경 적음(상위) → 변경 많음(하위)
```

### 2. 최적화 우선순위

```
1순위: 의존성 파일 먼저 복사
→ 가장 큰 효과

2순위: .dockerignore 작성
→ 불필요한 캐시 무효화 방지

3순위: 멀티 스테이지 활용
→ 각 스테이지별 캐시

4순위: BuildKit 캐시 마운트
→ 패키지 매니저 캐시

5순위: 원격 캐시
→ 팀/CI 간 공유
```

### 3. 캐시 효율성 측정

```
캐시 히트율:
- 목표: 70% 이상
- 우수: 85% 이상
- 최적: 90% 이상

빌드 시간 개선:
- 목표: 50% 감소
- 우수: 80% 감소
- 최적: 90% 감소

측정 방법:
docker build --progress=plain
→ Using cache 개수 확인
```

### 4. 언어별 캐시 전략

```
Node.js:
COPY package*.json → npm ci → COPY src

Python:
COPY requirements.txt → pip install → COPY src

Go:
COPY go.mod go.sum → go mod download → COPY src

Java:
COPY pom.xml → mvn dependency:go-offline → COPY src

Rust:
COPY Cargo.* → cargo fetch → COPY src
```

---

## 📚 참고 자료

- [Docker Build Cache](https://docs.docker.com/build/cache/)
- [BuildKit](https://docs.docker.com/build/buildkit/)
- [Cache Storage Backends](https://docs.docker.com/build/cache/backends/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

---

## 🤔 생각해볼 문제

1. RUN 명령어를 여러 개로 나누는 것과 하나로 합치는 것, 캐시 관점에서 언제 어떤 것이 유리할까?
2. 원격 캐시를 사용할 때 inline 방식과 registry 방식의 차이는?
3. BuildKit 캐시 마운트를 사용할 때 주의해야 할 점은?

> 💡 **답변**: 1) 자주 변경되는 부분은 분리(부분 캐시), 거의 불변인 부분은 병합(레이어 수 감소), 2) inline은 실제 이미지에 포함되어 크기 증가, registry는 별도 관리로 더 많은 레이어 저장 가능, 3) 최종 이미지에 필요한 파일을 캐시에만 두면 안 됨, 빌드 시간 의존성(npm 캐시 등)만 마운트

---

<div align="center">

**[⬅️ 이전: Image Optimization](./03-Image-Optimization.md)** | **[다음: BuildKit ➡️](./05-BuildKit.md)**

</div>
