# 05. BuildKit - 차세대 빌드 엔진

## 🎯 이 챕터에서 배울 것

- **BuildKit**의 개념과 기존 빌드 엔진과의 차이
- **병렬 빌드**로 빌드 시간 단축
- **Secrets 마운트**로 안전한 빌드
- **SSH 마운트**와 고급 기능들

## 📌 왜 중요한가?

**"BuildKit은 Docker 빌드의 성능과 보안을 혁신적으로 개선합니다."**

```
기존 빌드 엔진 vs BuildKit:

레거시 빌드:
- 순차적 실행
- 빌드 시간: 10분
- 민감 정보 이미지에 남음
- 제한적 캐시

BuildKit:
- 병렬 실행
- 빌드 시간: 3분 (3배 빠름)
- Secrets 안전하게 관리
- 고급 캐시 (마운트, 원격)
```

**실무 영향:**
- 성능: 병렬화로 2-5배 빌드 속도 향상
- 보안: Secrets가 이미지 히스토리에 남지 않음
- 효율: 스마트 캐시로 불필요한 재빌드 방지
- 기능: SSH, 캐시 마운트 등 강력한 기능

---

## 🔬 Deep Dive

### 1. BuildKit이란?

#### 기존 빌드 엔진의 한계

```
레거시 Docker 빌드:
┌──────────────────────────────────┐
│ Step 1: FROM node:18             │
│ ↓ (완료 후 다음)                    │
│ Step 2: COPY package.json        │
│ ↓ (완료 후 다음)                    │
│ Step 3: RUN npm install          │
│ ↓ (완료 후 다음)                    │
│ Step 4: COPY . .                 │
│ ↓ (완료 후 다음)                    │
│ Step 5: RUN npm build            │
└──────────────────────────────────┘

문제:
- 순차적 실행만 가능
- 병렬화 불가
- 비효율적인 캐시
- 민감 정보 누출 위험
```

#### BuildKit의 개선

```
BuildKit:
┌─────────────────────────────────────────┐
│         병렬 실행 가능                     │
│  ┌──────────────┐  ┌──────────────┐     │
│  │ Stage 1:     │  │ Stage 2:     │     │
│  │ Dependencies │  │ Build Tools  │     │
│  └──────────────┘  └──────────────┘     │
│         ↓                  ↓            │
│  ┌──────────────────────────────┐       │
│  │   Final Stage (병합)          │       │
│  └──────────────────────────────┘       │
└─────────────────────────────────────────┘

개선점:
✅ 병렬 빌드
✅ 스마트 의존성 그래프
✅ 고급 캐시 (마운트, 원격)
✅ Secrets 관리
✅ SSH 마운트
✅ 진행률 표시
```

#### BuildKit 활성화

```bash
# 방법 1: 환경변수
export DOCKER_BUILDKIT=1
docker build .

# 방법 2: 데몬 설정 (영구적)
# /etc/docker/daemon.json
{
  "features": {
    "buildkit": true
  }
}

sudo systemctl restart docker

# 방법 3: docker buildx (권장)
docker buildx build .

# 확인
docker buildx version
# github.com/docker/buildx v0.12.0
```

---

### 2. 병렬 빌드

#### 병렬화 원리

```dockerfile
# 기존: 순차적 실행
FROM node:18-alpine AS deps-prod
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS deps-dev
COPY package*.json ./
RUN npm ci

FROM node:18-alpine AS builder
COPY --from=deps-dev /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:18-alpine
COPY --from=deps-prod /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist

# BuildKit은 자동으로 병렬화:
# deps-prod와 deps-dev를 동시 실행!
```

#### 병렬화 예제

```dockerfile
# syntax=docker/dockerfile:1

# ✅ 이 두 스테이지는 병렬 실행됨
FROM golang:1.21-alpine AS backend
WORKDIR /app/backend
COPY backend/ .
RUN go build -o server

FROM node:18-alpine AS frontend
WORKDIR /app/frontend
COPY frontend/ .
RUN npm ci && npm run build

# 최종 조합
FROM nginx:alpine
COPY --from=backend /app/backend/server /usr/local/bin/
COPY --from=frontend /app/frontend/dist /usr/share/nginx/html/

# 효과:
# 순차: backend(5분) + frontend(3분) = 8분
# 병렬: max(backend(5분), frontend(3분)) = 5분
# 절감: 37.5%
```

#### 복잡한 의존성 그래프

```dockerfile
# syntax=docker/dockerfile:1

# Base 스테이지들 (병렬 실행)
FROM python:3.11-alpine AS python-base
RUN pip install --upgrade pip

FROM node:18-alpine AS node-base
RUN npm install -g pnpm

FROM golang:1.21-alpine AS go-base
RUN go install github.com/swaggo/swag/cmd/swag@latest

# 각 서비스 빌드 (병렬 실행)
FROM python-base AS api-python
COPY api-python/ /app
RUN pip install -r requirements.txt

FROM node-base AS api-node
COPY api-node/ /app
RUN pnpm install && pnpm build

FROM go-base AS api-go
COPY api-go/ /app
RUN go build -o server

# BuildKit은 의존성 그래프 분석:
# python-base, node-base, go-base → 병렬
# api-python, api-node, api-go → 병렬
```

---

### 3. Secrets 관리

#### 기존 방식의 문제

```dockerfile
# ❌ 문제 1: ARG에 노출
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc && \
    npm install && \
    rm .npmrc

# 문제:
# - 이미지 히스토리에 NPM_TOKEN 남음
docker history myapp
# 누구나 토큰 확인 가능!

# ❌ 문제 2: COPY로 전달
COPY .npmrc /root/.npmrc
RUN npm install
RUN rm /root/.npmrc

# 문제:
# - .npmrc가 레이어에 남음
# - docker export로 추출 가능
```

#### BuildKit Secrets (정답)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

# ✅ Secret 마운트 사용
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --only=production

COPY . .

CMD ["npm", "start"]

# 장점:
# - Secret이 이미지에 남지 않음
# - 빌드 시에만 임시 마운트
# - 히스토리에도 없음
```

#### 빌드 실행

```bash
# npmrc 파일을 secret으로 전달
docker buildx build \
  --secret id=npmrc,src=$HOME/.npmrc \
  -t myapp:latest \
  .

# 또는 환경변수에서
echo $NPM_TOKEN | docker buildx build \
  --secret id=npm_token \
  -t myapp:latest \
  .

# Dockerfile에서
# RUN --mount=type=secret,id=npm_token \
#     NPM_TOKEN=$(cat /run/secrets/npm_token) npm install
```

#### 여러 Secret 사용

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.11-alpine

WORKDIR /app

# AWS credentials
RUN --mount=type=secret,id=aws_key \
    --mount=type=secret,id=aws_secret \
    export AWS_ACCESS_KEY_ID=$(cat /run/secrets/aws_key) && \
    export AWS_SECRET_ACCESS_KEY=$(cat /run/secrets/aws_secret) && \
    aws s3 cp s3://my-bucket/data.zip . && \
    unzip data.zip

# PyPI token
RUN --mount=type=secret,id=pypi_token \
    pip config set global.index-url \
    "https://$(cat /run/secrets/pypi_token)@pypi.example.com/simple"

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

```bash
# 빌드 시 모든 secret 전달
docker buildx build \
  --secret id=aws_key,src=./secrets/aws_key.txt \
  --secret id=aws_secret,src=./secrets/aws_secret.txt \
  --secret id=pypi_token,env=PYPI_TOKEN \
  -t myapp:latest \
  .
```

---

### 4. SSH 마운트

#### 사용 사례

```
Private Git Repository 접근:
- GitHub, GitLab의 private repo
- go get으로 private module 다운로드
- pip install -e git+ssh://...
- npm install git+ssh://...

SSH 인증 필요:
- SSH 키가 필요
- 하지만 이미지에 키를 넣으면 안 됨!
```

#### SSH 마운트 사용

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.21-alpine

# Git과 SSH 클라이언트 설치
RUN apk add --no-cache git openssh-client

WORKDIR /app

COPY go.mod go.sum ./

# ✅ SSH 마운트로 private repo 접근
RUN --mount=type=ssh \
    git config --global url."git@github.com:".insteadOf "https://github.com/" && \
    go mod download

COPY . .

RUN go build -o server

CMD ["./server"]
```

```bash
# SSH agent 시작
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa

# SSH 마운트로 빌드
docker buildx build \
  --ssh default \
  -t myapp:latest \
  .

# 특정 SSH 키 사용
docker buildx build \
  --ssh default=$HOME/.ssh/github_rsa \
  -t myapp:latest \
  .
```

#### Known Hosts 처리

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine

RUN apk add --no-cache git openssh-client

WORKDIR /app

# Known hosts 설정
RUN mkdir -p /root/.ssh && \
    ssh-keyscan github.com >> /root/.ssh/known_hosts

COPY package*.json ./

# SSH로 private npm package 설치
RUN --mount=type=ssh \
    npm ci --only=production

COPY . .

CMD ["npm", "start"]
```

---

### 5. 캐시 마운트

#### 영구 캐시 디렉토리

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.21-alpine

WORKDIR /app

COPY go.mod go.sum ./

# ✅ Go modules 캐시
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

# ✅ Go build 캐시
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o server

CMD ["./server"]

# 효과:
# 첫 빌드: go mod download (2분)
# 재빌드: 캐시 사용 (5초)
# 40배 빠름!
```

#### 언어별 캐시 마운트

```dockerfile
# syntax=docker/dockerfile:1

# Node.js
FROM node:18-alpine
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# Python
FROM python:3.11-alpine
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Rust
FROM rust:1.74-alpine
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/usr/local/cargo/git \
    cargo build --release

# Maven
FROM maven:3.9-eclipse-temurin-17
RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:go-offline

# Gradle
FROM gradle:8.5-jdk17
RUN --mount=type=cache,target=/home/gradle/.gradle \
    gradle build --no-daemon
```

#### 캐시 옵션

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine

# sharing=locked (기본값)
# - 동시 빌드 시 대기
RUN --mount=type=cache,target=/root/.npm \
    npm install

# sharing=shared
# - 동시 빌드 허용 (읽기만)
RUN --mount=type=cache,target=/root/.npm,sharing=shared \
    npm install

# sharing=private
# - 각 빌드가 독립 캐시
RUN --mount=type=cache,target=/root/.npm,sharing=private \
    npm install

# mode 설정
RUN --mount=type=cache,target=/root/.npm,mode=0755 \
    npm install

# id 지정 (여러 빌드가 같은 캐시 공유)
RUN --mount=type=cache,target=/root/.npm,id=npm-cache \
    npm install
```

---

### 6. Bind 마운트

#### 빌드 시 임시 파일 마운트

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.21-alpine

WORKDIR /app

# ✅ 소스를 복사하지 않고 마운트
RUN --mount=type=bind,source=.,target=/app \
    --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /out/server

# 최종 이미지
FROM alpine:3.19
COPY --from=0 /out/server /server
CMD ["/server"]

# 장점:
# - 소스 코드가 이미지에 남지 않음
# - 빌드만 하고 바이너리만 복사
```

#### 설정 파일 임시 마운트

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

# ✅ .npmrc를 이미지에 넣지 않고 마운트
RUN --mount=type=bind,source=.npmrc,target=/root/.npmrc \
    --mount=type=cache,target=/root/.npm \
    npm ci --only=production

COPY . .

CMD ["npm", "start"]
```

---

### 7. 진행률 표시

#### 상세 진행률

```bash
# 기본 (간단한 진행률)
docker buildx build .

# 상세 진행률 (plain)
docker buildx build --progress=plain .

# 출력:
# #1 [internal] load build definition from Dockerfile
# #1 transferring dockerfile: 123B done
# #1 DONE 0.0s
# 
# #2 [internal] load .dockerignore
# #2 transferring context: 34B done
# #2 DONE 0.0s
# 
# #3 [internal] load metadata for docker.io/library/node:18-alpine
# #3 DONE 1.2s
# 
# #4 [1/4] FROM docker.io/library/node:18-alpine
# #4 CACHED
# 
# #5 [2/4] COPY package*.json ./
# #5 DONE 0.1s
# 
# #6 [3/4] RUN npm ci --only=production
# #6 0.500 npm WARN ...
# #6 45.23 added 57 packages in 45s
# #6 DONE 45.5s

# tty 모드 (대화형)
docker buildx build --progress=tty .

# 진행률 비활성화
docker buildx build --progress=false .
```

---

## 💻 실습

### 실습 1: BuildKit 병렬 빌드 효과 측정

#### 준비

```bash
mkdir buildkit-parallel-demo
cd buildkit-parallel-demo

# Backend (Go)
mkdir -p backend
cat > backend/main.go << 'EOF'
package main
import "fmt"
func main() {
    fmt.Println("Backend server")
}
EOF

cat > backend/go.mod << 'EOF'
module backend
go 1.21
require github.com/gin-gonic/gin v1.9.1
EOF

# Frontend (Node.js)
mkdir -p frontend
cat > frontend/package.json << 'EOF'
{
  "name": "frontend",
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  }
}
EOF
```

#### Dockerfile 작성

```dockerfile
# syntax=docker/dockerfile:1

# ✅ 병렬 실행될 스테이지들
FROM golang:1.21-alpine AS backend
WORKDIR /app/backend
COPY backend/ .
RUN go mod download && go build -o server

FROM node:18-alpine AS frontend
WORKDIR /app/frontend
COPY frontend/ .
RUN npm install && npm run build || true

# 최종 이미지
FROM alpine:3.19
COPY --from=backend /app/backend/server /usr/local/bin/
COPY --from=frontend /app/frontend/build /var/www/ || true
CMD ["server"]
```

#### 성능 측정

```bash
# 레거시 빌드 (순차)
export DOCKER_BUILDKIT=0
time docker build -t demo:legacy .

# 출력:
# Step 1: backend (5초)
# Step 2: frontend (3초)
# Total: 8초

# BuildKit (병렬)
export DOCKER_BUILDKIT=1
time docker build -t demo:buildkit .

# 출력:
# #4 [backend 2/3] COPY backend/ .
# #5 [frontend 2/3] COPY frontend/ .
# (동시 실행)
# Total: 5초 (가장 긴 것)

# 개선: 37.5% 빠름
```

---

### 실습 2: Secrets 안전하게 사용

#### 준비

```bash
mkdir buildkit-secrets-demo
cd buildkit-secrets-demo

# Private npm package 시뮬레이션
cat > .npmrc << 'EOF'
//registry.npmjs.org/:_authToken=npm_FAKE_TOKEN_12345
EOF

cat > package.json << 'EOF'
{
  "name": "secrets-demo",
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

cat > app.js << 'EOF'
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('Hello!'));
app.listen(3000);
EOF
```

#### Before: 잘못된 방법

```dockerfile
# Dockerfile.insecure
FROM node:18-alpine

WORKDIR /app

# ❌ Secret이 이미지에 남음
COPY .npmrc /root/.npmrc
COPY package*.json ./
RUN npm ci --only=production
RUN rm /root/.npmrc

COPY . .
CMD ["node", "app.js"]
```

```bash
docker build -f Dockerfile.insecure -t demo:insecure .

# Secret 확인 (취약!)
docker history demo:insecure
# Step X: COPY .npmrc /root/.npmrc  ← 레이어에 남음

# 추출 가능
docker save demo:insecure -o demo.tar
tar -xf demo.tar
# .npmrc 파일 찾을 수 있음!
```

#### After: BuildKit Secrets

```dockerfile
# Dockerfile.secure
# syntax=docker/dockerfile:1

FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

# ✅ Secret 안전하게 사용
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --only=production

COPY . .

CMD ["node", "app.js"]
```

```bash
# 빌드
docker buildx build \
  -f Dockerfile.secure \
  --secret id=npmrc,src=.npmrc \
  -t demo:secure \
  .

# 히스토리 확인
docker history demo:secure
# Secret 관련 내용 없음!

# 추출 시도
docker save demo:secure -o demo.tar
tar -xf demo.tar
# .npmrc 없음!
```

---

### 실습 3: SSH 마운트로 Private Repo 접근

#### 준비

```bash
mkdir buildkit-ssh-demo
cd buildkit-ssh-demo

# Go 프로젝트
cat > go.mod << 'EOF'
module demo

go 1.21

require (
    github.com/your-org/private-lib v1.0.0
)
EOF

cat > main.go << 'EOF'
package main

import (
    "fmt"
    // "github.com/your-org/private-lib"
)

func main() {
    fmt.Println("SSH mount demo")
}
EOF
```

#### Dockerfile with SSH

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.21-alpine

RUN apk add --no-cache git openssh-client

WORKDIR /app

# SSH known_hosts 설정
RUN mkdir -p /root/.ssh && \
    ssh-keyscan github.com >> /root/.ssh/known_hosts

COPY go.mod go.sum ./

# ✅ SSH 마운트로 private repo 접근
RUN --mount=type=ssh \
    git config --global url."git@github.com:".insteadOf "https://github.com/" && \
    go mod download

COPY . .

RUN go build -o server

CMD ["./server"]
```

```bash
# SSH agent 시작
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa

# SSH 키로 빌드
docker buildx build \
  --ssh default \
  -t demo:ssh \
  .

# 다른 키 사용
docker buildx build \
  --ssh default=$HOME/.ssh/github_key \
  -t demo:ssh \
  .
```

---

### 실습 4: 캐시 마운트 효과

#### 준비

```bash
mkdir buildkit-cache-mount-demo
cd buildkit-cache-mount-demo

cat > go.mod << 'EOF'
module demo

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/go-redis/redis/v8 v8.11.5
    gorm.io/gorm v1.25.5
)
EOF

cat > main.go << 'EOF'
package main
import "github.com/gin-gonic/gin"
func main() {
    r := gin.Default()
    r.Run(":8080")
}
EOF
```

#### Before: 캐시 마운트 없음

```dockerfile
# Dockerfile.no-mount
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
time docker build -f Dockerfile.no-mount -t demo:no-mount .
# go mod download: 2분 30초

# go.mod 한 글자 변경
echo "// comment" >> go.mod

# 재빌드
time docker build -f Dockerfile.no-mount -t demo:no-mount .
# go mod download: 2분 30초 (다시 다운로드!)
```

#### After: 캐시 마운트

```dockerfile
# Dockerfile.mount
# syntax=docker/dockerfile:1

FROM golang:1.21-alpine

WORKDIR /app

COPY go.mod go.sum ./

# ✅ 캐시 마운트
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
time docker build -f Dockerfile.mount -t demo:mount .
# go mod download: 2분 30초

# go.mod 한 글자 변경
echo "// comment" >> go.mod

# 재빌드
time docker build -f Dockerfile.mount -t demo:mount .
# go mod download: 5초 (캐시 재사용!)

# 개선: 30배 빠름 (2분 30초 → 5초)
```

---

## 🔥 실전 적용

### 시나리오 1: 모노레포 빌드 최적화

**상황:**
- 10개 마이크로서비스
- 공통 라이브러리 공유
- 순차 빌드: 50분

**BuildKit 병렬화 적용:**

```dockerfile
# syntax=docker/dockerfile:1

# 공통 라이브러리 (한 번만)
FROM node:18-alpine AS shared-lib
WORKDIR /app/shared
COPY packages/shared/ .
RUN npm ci && npm run build

# ✅ 모든 서비스 병렬 빌드
FROM node:18-alpine AS service-1
WORKDIR /app
COPY --from=shared-lib /app/shared /app/shared
COPY packages/service-1/ .
RUN npm ci && npm run build

FROM node:18-alpine AS service-2
WORKDIR /app
COPY --from=shared-lib /app/shared /app/shared
COPY packages/service-2/ .
RUN npm ci && npm run build

# ... (service-3 ~ service-10도 동일)

# 원하는 서비스만 빌드
FROM node:18-alpine
ARG SERVICE=service-1
COPY --from=${SERVICE} /app/dist /app/
CMD ["node", "/app/main.js"]
```

```bash
# 특정 서비스만 빌드
docker buildx build \
  --build-arg SERVICE=service-1 \
  -t service-1:latest \
  .

# 효과:
# - shared-lib: 5분
# - service-1~10: 병렬 5분
# - 총: 10분 (기존 50분 → 80% 단축)
```

---

### 시나리오 2: Private 의존성 안전하게 관리

**상황:**
- Private npm packages
- Private GitHub repositories
- SSH 키 필요
- 기존: 키가 이미지에 노출

**BuildKit Secrets + SSH:**

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine

RUN apk add --no-cache git openssh-client

WORKDIR /app

# SSH 설정
RUN mkdir -p /root/.ssh && \
    ssh-keyscan github.com >> /root/.ssh/known_hosts

COPY package*.json ./

# ✅ NPM token (secret)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# ✅ Private git repos (ssh)
RUN --mount=type=ssh \
    --mount=type=cache,target=/root/.npm \
    npm install git+ssh://git@github.com/your-org/private-pkg.git

COPY . .

CMD ["npm", "start"]
```

```bash
# 빌드
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa

docker buildx build \
  --secret id=npmrc,src=$HOME/.npmrc \
  --ssh default \
  -t myapp:latest \
  .

# 보안 검증
docker history myapp:latest  # Secret 없음
docker save myapp:latest -o myapp.tar
tar -tf myapp.tar | grep -E '(npmrc|id_rsa)'  # 없음
```

---

### 시나리오 3: CI/CD 빌드 시간 단축

**상황:**
- Go 마이크로서비스
- go mod download: 3분
- 하루 100번 빌드
- CI 비용 과다

**캐시 마운트 적용:**

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./

# ✅ Go modules 캐시
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go mod download

COPY . .

# ✅ Build 캐시
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -o server

FROM alpine:3.19
COPY --from=builder /app/server /server
CMD ["/server"]
```

```yaml
# .github/workflows/build.yml
name: Build

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Build
        uses: docker/build-push-action@v4
        with:
          context: .
          # ✅ 캐시는 GitHub Actions 캐시에 저장
          cache-from: type=gha
          cache-to: type=gha,mode=max
          push: true
          tags: myapp:latest
```

**결과:**
```
빌드 시간:
- 첫 빌드: 3분
- 의존성 변경 없음: 30초 (6배 빠름)
- 소스만 변경: 30초

하루 비용:
- Before: 100 × 3분 = 300분
- After: 1 × 3분 + 99 × 30초 = 52.5분
- 절감: 82.5%

월 CI 비용:
- Before: $300
- After: $50
- 절감: $250/월
```

---

## ⚡ BuildKit 최적화 체크리스트

### 기본 설정

```
□ DOCKER_BUILDKIT=1 또는 buildx 사용
□ syntax=docker/dockerfile:1 선언
□ BuildKit 버전 최신화
□ buildx 플러그인 설치
```

### 병렬화

```
□ 독립적인 스테이지 분리
□ 의존성 최소화
□ 멀티 스테이지 활용
□ 불필요한 순차 의존성 제거
```

### Secrets

```
□ --mount=type=secret 사용
□ ARG로 secret 전달 금지
□ COPY로 secret 전달 금지
□ 빌드 후 secret 검증
```

### 캐시

```
□ --mount=type=cache로 패키지 캐시
□ 원격 캐시 설정 (CI/CD)
□ 캐시 키 최적화
□ 캐시 히트율 모니터링
```

### SSH

```
□ --mount=type=ssh 사용
□ known_hosts 설정
□ SSH agent 실행
□ git config 설정
```

---

## 🚫 안티패턴

### 1. Secret을 ARG로 전달

```dockerfile
# ❌ 이미지 히스토리에 남음
ARG NPM_TOKEN
RUN npm config set //registry.npmjs.org/:_authToken ${NPM_TOKEN}

# ✅ Secret 마운트
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npm_token \
    npm config set //registry.npmjs.org/:_authToken $(cat /run/secrets/npm_token)
```

### 2. 캐시 마운트 미사용

```dockerfile
# ❌ 매번 다운로드
FROM golang:1.21
RUN go mod download

# ✅ 캐시 재사용
# syntax=docker/dockerfile:1
FROM golang:1.21
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
```

### 3. 병렬화 기회 무시

```dockerfile
# ❌ 순차 의존성
FROM node:18 AS deps
RUN npm install

FROM node:18 AS build
COPY --from=deps /app/node_modules ./node_modules
RUN npm run build

# ✅ 병렬 가능하도록 분리
# syntax=docker/dockerfile:1
FROM node:18 AS deps-prod
RUN npm ci --only=production

FROM node:18 AS deps-dev
RUN npm ci

# deps-prod와 deps-dev는 병렬 실행!
```

### 4. syntax 선언 누락

```dockerfile
# ❌ BuildKit 기능 사용 불가
FROM node:18
RUN --mount=type=cache,target=/root/.npm \
    npm install
# 에러: unknown flag: --mount

# ✅ syntax 선언
# syntax=docker/dockerfile:1
FROM node:18
RUN --mount=type=cache,target=/root/.npm \
    npm install
```

---

## 🎓 핵심 정리

### 1. BuildKit vs 레거시

```
레거시 빌드:
- 순차 실행만 가능
- 제한적 캐시
- Secret 관리 위험
- 느린 빌드

BuildKit:
- 병렬 실행
- 고급 캐시 (마운트)
- 안전한 Secret
- 2-5배 빠른 빌드
```

### 2. 핵심 기능

```
병렬화:
멀티 스테이지 → 독립 실행 → 속도 향상

Secrets:
--mount=type=secret → 이미지에 안 남음 → 보안

SSH:
--mount=type=ssh → Private repo 접근 → 키 노출 없음

캐시 마운트:
--mount=type=cache → 패키지 재사용 → 빠른 빌드
```

### 3. 활성화 방법

```bash
# 환경변수
export DOCKER_BUILDKIT=1

# buildx (권장)
docker buildx build .

# Dockerfile
# syntax=docker/dockerfile:1
```

### 4. 측정 지표

```
빌드 시간:
- 목표: 50% 단축
- 우수: 70% 단축
- 최적: 80%+ 단축

캐시 효율:
- go mod download: 3분 → 5초 (30배)
- npm install: 2분 → 10초 (12배)
- pip install: 1분 → 5초 (12배)
```

---

## 📚 참고 자료

- [BuildKit Documentation](https://docs.docker.com/build/buildkit/)
- [Dockerfile Frontend](https://docs.docker.com/engine/reference/builder/)
- [Build Secrets](https://docs.docker.com/build/building/secrets/)
- [SSH Mounts](https://docs.docker.com/build/building/secrets/#ssh-mounts)
- [Cache Mounts](https://docs.docker.com/build/guide/mounts/)

---

## 🤔 생각해볼 문제

1. 멀티 스테이지에서 어떤 구조가 가장 병렬화에 유리할까?
2. Secret 마운트와 ARG의 근본적인 차이는?
3. 캐시 마운트를 사용할 때 공유 모드(shared/locked/private)는 언제 선택해야 할까?

> 💡 **답변**: 1) 상호 의존성이 없는 독립적인 스테이지들(예: frontend와 backend 별도 빌드), 2) ARG는 빌드 히스토리에 남아 docker history로 노출되지만, Secret 마운트는 빌드 시에만 임시로 마운트되어 이미지에 전혀 남지 않음, 3) locked(기본)는 안전하지만 느림, shared는 읽기만 하는 경우 빠름, private는 각 빌드가 독립적일 때

---

<div align="center">

**[⬅️ 이전: Cache Mechanism](./04-Cache-Mechanism.md)** | **[다음: Image Security ➡️](./06-Image-Security.md)**

</div>
