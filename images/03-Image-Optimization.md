# 03. Image Optimization - 이미지 최적화 전략

## 🎯 이 챕터에서 배울 것

- **베이스 이미지 선택 전략**: Alpine vs Distroless vs Scratch
- **이미지 크기 최소화** 기법과 실전 적용
- **보안과 성능**을 동시에 고려한 최적화
- 언어별 **최적 베이스 이미지**와 최적화 패턴

## 📌 왜 중요한가?

**"이미지 크기는 보안, 성능, 비용에 직접적인 영향을 미칩니다."**

```
일반 이미지 vs 최적화 이미지:

기본 Node.js:
- 베이스: node:18
- 크기: 1.1GB
- 취약점: 500+
- 배포 시간: 3분

최적화 Node.js:
- 베이스: node:18-alpine
- 크기: 180MB (6배 작음)
- 취약점: 50개
- 배포 시간: 30초 (6배 빠름)
```

**실무 영향:**
- 보안: 공격 표면 최소화, 취약점 감소
- 성능: 빠른 배포, 적은 메모리 사용
- 비용: 스토리지/네트워크 비용 절감
- 운영: CI/CD 파이프라인 속도 향상

---

## 🔬 Deep Dive

### 1. 베이스 이미지 선택 전략

#### 주요 베이스 이미지 비교

```
┌──────────────┬──────────┬────────────┬────────────┬──────────┐
│   타입        │   크기    │  패키지 수    │  보안 수준   │  난이도   │
├──────────────┼──────────┼────────────┼────────────┼──────────┤
│ Full OS      │ 500MB+   │ 1000+      │ 낮음        │ 쉬움      │
│ (ubuntu)     │          │            │            │          │
├──────────────┼──────────┼────────────┼────────────┼──────────┤
│ Slim         │ 200-300MB│ 500+       │ 중간        │ 쉬움      │
│ (node:slim)  │          │            │            │          │
├──────────────┼──────────┼────────────┼────────────┼──────────┤
│ Alpine       │ 5-50MB   │ 50-100     │ 높음        │ 중간      │
│              │          │            │            │          │
├──────────────┼──────────┼────────────┼────────────┼──────────┤
│ Distroless   │ 10-80MB  │ 10-30      │ 매우 높음    │ 높음      │
│              │          │            │            │          │
├──────────────┼──────────┼────────────┼────────────┼──────────┤
│ Scratch      │ 1-20MB   │ 0          │ 최고        │ 매우 높음  │
│ (정적 빌드)    │          │            │            │          │
└──────────────┴──────────┴────────────┴────────────┴──────────┘

선택 기준:
- 개발 환경: Full OS (디버깅 도구 필요)
- 프로덕션: Alpine/Distroless (보안+크기)
- 정적 바이너리: Scratch (최소 크기)
```

---

### 2. Alpine Linux 최적화

#### Alpine의 장점과 특징

```
Alpine Linux:
- musl libc 기반 (glibc 아님)
- apk 패키지 매니저
- 기본 크기: 5MB
- 보안 강화 (stack smashing protection)

주의사항:
- glibc 의존 프로그램 호환성 문제
- 일부 네이티브 확장 재컴파일 필요
- DNS 이슈 (musl libc 특성)
```

#### Before: 기본 이미지

```dockerfile
FROM node:18

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

CMD ["node", "server.js"]

# 결과:
# - 이미지 크기: 1.1GB
# - 레이어 수: 15개
# - 취약점: 500+
```

#### After: Alpine 최적화

```dockerfile
FROM node:18-alpine

# 필요한 빌드 도구만 임시 설치
RUN apk add --no-cache --virtual .build-deps \
    python3 \
    make \
    g++

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production && \
    # 네이티브 모듈 재빌드
    npm rebuild && \
    # 빌드 도구 제거
    apk del .build-deps && \
    # npm 캐시 정리
    npm cache clean --force

COPY . .

CMD ["node", "server.js"]

# 개선:
# - 이미지 크기: 180MB (6배 감소)
# - 레이어 수: 8개
# - 취약점: 50개 (10배 감소)
```

---

### 3. Distroless 이미지

#### Distroless란?

```
Google이 제공하는 초경량 컨테이너 이미지
- OS 패키지 매니저 없음
- 쉘 없음
- 런타임만 포함
- 보안 최적화

사용 가능한 Distroless:
- gcr.io/distroless/static        (정적 바이너리)
- gcr.io/distroless/base           (glibc + openssl)
- gcr.io/distroless/java17         (Java 17)
- gcr.io/distroless/nodejs18       (Node.js 18)
- gcr.io/distroless/python3        (Python 3)
```

#### Node.js with Distroless

```dockerfile
# Stage 1: 빌드
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Stage 2: Production with Distroless
FROM gcr.io/distroless/nodejs18-debian11

WORKDIR /app

# 빌드 결과물만 복사
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

# Distroless는 쉘이 없어서 CMD를 JSON 배열로
CMD ["dist/server.js"]

# 장점:
# - 이미지 크기: 150MB
# - 쉘 없음 → 공격 표면 최소화
# - 패키지 매니저 없음 → 런타임 변조 불가
# - CVE 대폭 감소
```

#### Go with Distroless

```dockerfile
# Stage 1: 빌드
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o myapp .

# Stage 2: Distroless
FROM gcr.io/distroless/static-debian11

COPY --from=builder /app/myapp /

USER nonroot:nonroot

CMD ["/myapp"]

# 결과:
# - 이미지 크기: 15MB
# - 취약점: 0개
# - 쉘 없음 → 최고 보안
```

---

### 4. Scratch 이미지

#### Scratch란?

```
완전히 비어있는 베이스 이미지
- 아무것도 없음 (문자 그대로 scratch)
- 정적 링크된 바이너리만 실행 가능
- 최소 크기, 최고 보안

사용 조건:
✅ 정적으로 링크된 바이너리
✅ 외부 의존성 없음
✅ 라이브러리 정적 포함
❌ 동적 링크 라이브러리 필요 시 불가
❌ 쉘 명령어 필요 시 불가
```

#### Go 정적 빌드

```dockerfile
# Stage 1: 빌드
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# 완전 정적 빌드 (중요!)
RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
    -a \
    -installsuffix cgo \
    -ldflags="-w -s" \
    -o myapp \
    .

# Stage 2: Scratch
FROM scratch

# CA 인증서 복사 (HTTPS 통신용)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 타임존 데이터 복사 (필요시)
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# non-root 사용자 정보 복사
COPY --from=builder /etc/passwd /etc/passwd

# 바이너리 복사
COPY --from=builder /app/myapp /myapp

# non-root 실행
USER nobody:nobody

ENTRYPOINT ["/myapp"]

# 결과:
# - 이미지 크기: 8MB
# - 취약점: 0개
# - 공격 표면: 최소
```

#### Rust 정적 빌드

```dockerfile
FROM rust:1.74-alpine AS builder

WORKDIR /app

# musl 타겟 추가
RUN rustup target add x86_64-unknown-linux-musl

COPY Cargo.toml Cargo.lock ./
RUN cargo fetch

COPY src ./src

# 정적 링크로 빌드
RUN cargo build --release --target x86_64-unknown-linux-musl

# Scratch
FROM scratch

COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /

USER 1000:1000

CMD ["/myapp"]

# 결과:
# - 이미지 크기: 5MB
# - 취약점: 0개
```

---

### 5. 불필요한 파일 제거

#### .dockerignore 활용

```dockerignore
# .dockerignore

# Git 관련
.git
.gitignore
.gitattributes

# 문서
README.md
CHANGELOG.md
docs/
*.md

# 테스트
tests/
**/*.test.js
**/*.spec.js
coverage/
.nyc_output/

# 개발 도구
.vscode/
.idea/
*.swp
*.swo
*~

# 환경 설정
.env
.env.*
!.env.example

# 로그
logs/
*.log

# 캐시
node_modules/
.npm/
.cache/
dist/
build/

# OS 관련
.DS_Store
Thumbs.db

# CI/CD
.github/
.gitlab-ci.yml
.travis.yml
Jenkinsfile

# 효과:
# - 빌드 컨텍스트 크기: 500MB → 50MB
# - 빌드 속도: 2분 → 30초
# - 레이어 크기 감소
```

#### 멀티 스테이지에서 선택적 복사

```dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# 의존성만 설치
COPY package*.json ./
RUN npm ci

# 소스 전체 복사 (빌드용)
COPY . .
RUN npm run build

# Production
FROM node:18-alpine

WORKDIR /app

# ✅ 필요한 것만 복사
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist

# ❌ 불필요한 것 제외:
# - 소스 코드 (src/)
# - 테스트 (tests/)
# - 빌드 설정 파일
# - 개발 도구

CMD ["node", "dist/server.js"]

# 개선:
# - 최종 이미지: 200MB → 150MB
# - 보안: 소스 코드 노출 방지
```

---

### 6. 레이어 최적화

#### RUN 명령어 체이닝

```dockerfile
# ❌ Before: 여러 레이어
FROM alpine:3.19

RUN apk add --no-cache curl
RUN apk add --no-cache wget
RUN apk add --no-cache ca-certificates
RUN rm -rf /var/cache/apk/*

# 문제:
# - 4개의 레이어
# - apk 캐시가 3개 레이어에 중복
# - 총 크기: 15MB

# ✅ After: 단일 레이어
FROM alpine:3.19

RUN apk add --no-cache \
    curl \
    wget \
    ca-certificates \
    && rm -rf /var/cache/apk/*

# 개선:
# - 1개의 레이어
# - apk 캐시 즉시 제거
# - 총 크기: 8MB
```

#### 임시 파일 정리

```dockerfile
# ❌ Before: 캐시가 레이어에 남음
FROM node:18-alpine

COPY package*.json ./
RUN npm install

# npm 캐시가 레이어에 포함됨
# 크기: 200MB

# ✅ After: 캐시 즉시 정리
FROM node:18-alpine

COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# 캐시 제거됨
# 크기: 150MB
```

---

## 💻 실습

### 실습 1: 베이스 이미지 비교

#### 준비

```bash
# 테스트용 Node.js 앱
mkdir image-optimization-demo
cd image-optimization-demo

cat > app.js << 'EOF'
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from optimized container!\n');
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

cat > package.json << 'EOF'
{
  "name": "demo",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
EOF
```

#### 1. 기본 이미지

```dockerfile
# Dockerfile.default
FROM node:18

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

CMD ["npm", "start"]
```

```bash
docker build -f Dockerfile.default -t demo:default .
docker images demo:default

# REPOSITORY   TAG      SIZE
# demo         default  1.1GB
```

#### 2. Slim 이미지

```dockerfile
# Dockerfile.slim
FROM node:18-slim

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

CMD ["npm", "start"]
```

```bash
docker build -f Dockerfile.slim -t demo:slim .
docker images demo:slim

# REPOSITORY   TAG    SIZE
# demo         slim   250MB (4.4배 작음)
```

#### 3. Alpine 이미지

```dockerfile
# Dockerfile.alpine
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force
COPY . .

CMD ["npm", "start"]
```

```bash
docker build -f Dockerfile.alpine -t demo:alpine .
docker images demo:alpine

# REPOSITORY   TAG     SIZE
# demo         alpine  180MB (6.1배 작음)
```

#### 4. Distroless 이미지

```dockerfile
# Dockerfile.distroless
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM gcr.io/distroless/nodejs18-debian11

WORKDIR /app
COPY --from=builder /app .

CMD ["app.js"]
```

```bash
docker build -f Dockerfile.distroless -t demo:distroless .
docker images demo:distroless

# REPOSITORY   TAG          SIZE
# demo         distroless   150MB (7.3배 작음)
```

#### 비교 분석

```bash
# 크기 비교
docker images demo --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# 취약점 스캔 (Trivy 필요)
docker run aquasec/trivy image demo:default
docker run aquasec/trivy image demo:alpine
docker run aquasec/trivy image demo:distroless

# 결과:
# default:     500+ vulnerabilities
# alpine:      50 vulnerabilities (10배 감소)
# distroless:  10 vulnerabilities (50배 감소)
```

---

### 실습 2: Go 애플리케이션 최적화

#### 준비

```bash
mkdir go-optimization-demo
cd go-optimization-demo

cat > main.go << 'EOF'
package main

import (
    "fmt"
    "log"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello from Go container!")
}

func main() {
    http.HandleFunc("/", handler)
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
EOF

cat > go.mod << 'EOF'
module demo

go 1.21
EOF
```

#### 1. 기본 이미지

```dockerfile
# Dockerfile.basic
FROM golang:1.21

WORKDIR /app
COPY . .
RUN go build -o server .

CMD ["./server"]
```

```bash
docker build -f Dockerfile.basic -t go-demo:basic .
docker images go-demo:basic

# REPOSITORY   TAG     SIZE
# go-demo      basic   850MB
```

#### 2. 멀티 스테이지 + Alpine

```dockerfile
# Dockerfile.alpine
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/server /server

CMD ["/server"]
```

```bash
docker build -f Dockerfile.alpine -t go-demo:alpine .
docker images go-demo:alpine

# REPOSITORY   TAG      SIZE
# go-demo      alpine   15MB (56배 작음!)
```

#### 3. Distroless

```dockerfile
# Dockerfile.distroless
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM gcr.io/distroless/static-debian11

COPY --from=builder /app/server /server

USER nonroot:nonroot

CMD ["/server"]
```

```bash
docker build -f Dockerfile.distroless -t go-demo:distroless .
docker images go-demo:distroless

# REPOSITORY   TAG          SIZE
# go-demo      distroless   10MB (85배 작음!)
```

#### 4. Scratch (최소)

```dockerfile
# Dockerfile.scratch
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY . .

# 완전 정적 빌드
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -a -installsuffix cgo \
    -ldflags="-w -s" \
    -o server .

FROM scratch

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server

USER 1000:1000

ENTRYPOINT ["/server"]
```

```bash
docker build -f Dockerfile.scratch -t go-demo:scratch .
docker images go-demo:scratch

# REPOSITORY   TAG       SIZE
# go-demo      scratch   8MB (106배 작음!)
```

#### 최종 비교

```bash
docker images go-demo --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# REPOSITORY   TAG          SIZE      감소율
# go-demo      basic        850MB     -
# go-demo      alpine       15MB      98.2%
# go-demo      distroless   10MB      98.8%
# go-demo      scratch      8MB       99.1%

# 실행 테스트
docker run -d -p 8080:8080 --name test-scratch go-demo:scratch
curl http://localhost:8080
# Hello from Go container!

docker stop test-scratch && docker rm test-scratch
```

---

### 실습 3: Python 애플리케이션 최적화

#### 준비

```bash
mkdir python-optimization-demo
cd python-optimization-demo

cat > app.py << 'EOF'
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from optimized Python container!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

cat > requirements.txt << 'EOF'
Flask==3.0.0
gunicorn==21.2.0
EOF
```

#### 1. 기본 이미지

```dockerfile
# Dockerfile.default
FROM python:3.11

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]
```

```bash
docker build -f Dockerfile.default -t py-demo:default .
docker images py-demo:default

# REPOSITORY   TAG       SIZE
# py-demo      default   1GB
```

#### 2. Slim 이미지

```dockerfile
# Dockerfile.slim
FROM python:3.11-slim

WORKDIR /app

# 빌드 의존성 설치 및 정리
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    apt-get purge -y --auto-remove gcc

COPY . .

CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]
```

```bash
docker build -f Dockerfile.slim -t py-demo:slim .
docker images py-demo:slim

# REPOSITORY   TAG    SIZE
# py-demo      slim   200MB (5배 작음)
```

#### 3. Alpine 이미지

```dockerfile
# Dockerfile.alpine
FROM python:3.11-alpine

WORKDIR /app

# 빌드 도구 임시 설치
RUN apk add --no-cache --virtual .build-deps \
    gcc \
    musl-dev \
    linux-headers

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    apk del .build-deps

COPY . .

CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]
```

```bash
docker build -f Dockerfile.alpine -t py-demo:alpine .
docker images py-demo:alpine

# REPOSITORY   TAG      SIZE
# py-demo      alpine   80MB (12.5배 작음)
```

#### 4. Distroless

```dockerfile
# Dockerfile.distroless
FROM python:3.11-slim AS builder

WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

FROM gcr.io/distroless/python3-debian11

WORKDIR /app

# Python 패키지 복사
COPY --from=builder /root/.local /root/.local
COPY . .

# PATH 설정
ENV PATH=/root/.local/bin:$PATH

CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]
```

```bash
docker build -f Dockerfile.distroless -t py-demo:distroless .
docker images py-demo:distroless

# REPOSITORY   TAG          SIZE
# py-demo      distroless   120MB (8.3배 작음)
```

---

## 🔥 실전 적용

### 시나리오 1: 마이크로서비스 최적화

**상황:**
- 20개의 마이크로서비스
- 각 서비스 평균 이미지 크기: 800MB
- 총 스토리지: 16GB
- 배포 시간: 서비스당 2분

**최적화 적용:**

```dockerfile
# Before: 단일 스테이지
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]
# 크기: 800MB

# After: 멀티 스테이지 + Alpine
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./
CMD ["node", "dist/server.js"]
# 크기: 150MB
```

**결과:**
```
이미지 크기:
- Before: 16GB (20 × 800MB)
- After: 3GB (20 × 150MB)
- 절감: 13GB (81% 감소)

배포 시간:
- Before: 40분 (20 × 2분)
- After: 10분 (20 × 30초)
- 절감: 30분 (75% 감소)

비용 절감:
- 네트워크: 월 $500 → $100
- 스토리지: 월 $200 → $40
- 총 절감: 월 $560
```

---

### 시나리오 2: CI/CD 파이프라인 최적화

**상황:**
- 하루 100번의 빌드
- 평균 빌드 시간: 5분
- 이미지 크기: 1.2GB

**최적화 전략:**

```dockerfile
# 1. .dockerignore 최적화
cat > .dockerignore << 'EOF'
node_modules
.git
tests
docs
*.md
.env*
EOF

# 2. 멀티 스테이지 + 캐시 최적화
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./
CMD ["node", "dist/server.js"]
```

**결과:**
```
빌드 시간:
- 첫 빌드: 5분
- 캐시 히트 빌드: 30초 (10배 빠름)
- 하루 총 시간: 500분 → 80분

이미지 크기:
- Before: 1.2GB
- After: 180MB
- 절감: 85%

CI/CD 비용:
- 빌드 시간 비용: 월 $300 → $50
- 네트워크 비용: 월 $200 → $30
- 총 절감: 월 $420
```

---

### 시나리오 3: 보안 강화

**상황:**
- 취약점 스캔 결과: HIGH 50개, CRITICAL 10개
- 컴플라이언스 요구사항 미충족
- 정기 보안 감사 실패

**최적화 적용:**

```dockerfile
# Before: Ubuntu 기반
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    curl wget git vim python3 ...
# 취약점: 500+

# After: Distroless
FROM python:3.11-slim AS builder
RUN pip install --user flask gunicorn

FROM gcr.io/distroless/python3-debian11
COPY --from=builder /root/.local /root/.local
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
USER nonroot:nonroot
CMD ["gunicorn", "app:app"]
# 취약점: 5개
```

**보안 스캔 결과:**

```bash
# Before
trivy image app:ubuntu
Total: 500 (HIGH: 50, CRITICAL: 10)

# After
trivy image app:distroless
Total: 5 (HIGH: 0, CRITICAL: 0)
```

**개선 사항:**
```
취약점:
- CRITICAL: 10 → 0
- HIGH: 50 → 0
- 총 취약점: 500 → 5 (99% 감소)

공격 표면:
- 쉘: 있음 → 없음
- 패키지 매니저: 있음 → 없음
- 불필요한 도구: 50+ → 0

컴플라이언스:
- CIS 벤치마크: 60점 → 95점
- 보안 감사: 실패 → 통과
```

---

## ⚡ 최적화 체크리스트

### 베이스 이미지 선택

```
□ Alpine/Slim/Distroless 검토
□ 언어별 최적 이미지 선택
□ Scratch 가능 여부 확인
□ 취약점 스캔 결과 확인
□ 크기 vs 호환성 트레이드오프 평가
```

### 빌드 최적화

```
□ 멀티 스테이지 빌드 적용
□ .dockerignore 작성
□ 레이어 수 최소화
□ RUN 명령어 체이닝
□ 빌드 캐시 최적화
```

### 런타임 최적화

```
□ 프로덕션 의존성만 포함
□ 개발 도구 제외
□ 소스 코드 제외 (가능한 경우)
□ 임시 파일 정리
□ 불필요한 파일 제거
```

### 보안 강화

```
□ non-root 사용자 실행
□ 최소 권한 원칙
□ 취약점 스캔
□ 쉘 제거 검토
□ 읽기 전용 파일시스템
```

---

## 🚫 안티패턴

### 1. 과도한 최적화

```dockerfile
# ❌ 너무 복잡한 최적화
FROM alpine AS deps
RUN apk add --no-cache python3 && ...
FROM alpine AS builder
RUN apk add --no-cache gcc && ...
FROM alpine AS minifier
RUN apk add --no-cache upx && ...
FROM scratch
COPY --from=deps ...
COPY --from=builder ...
COPY --from=minifier ...

# 문제:
# - 빌드 시간 증가
# - 유지보수 어려움
# - 크기 절감은 미미
# - 디버깅 복잡

# ✅ 적절한 최적화
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]

# 장점:
# - 간단하고 명확
# - 유지보수 용이
# - 충분한 최적화
# - 디버깅 가능
```

### 2. 호환성 무시

```dockerfile
# ❌ Alpine에서 glibc 의존 앱
FROM node:18-alpine
COPY . .
RUN npm install node-sass
# 에러: node-sass가 musl에서 안 됨

# ✅ 호환성 확인 후 선택
# Option 1: Alpine + 재빌드
FROM node:18-alpine
RUN apk add --no-cache python3 make g++
COPY . .
RUN npm install node-sass
RUN npm rebuild node-sass

# Option 2: Slim 사용 (glibc)
FROM node:18-slim
COPY . .
RUN npm install node-sass
# 호환성 문제 없음
```

### 3. 캐시 무효화

```dockerfile
# ❌ COPY . 을 너무 일찍
FROM node:18-alpine
WORKDIR /app
COPY . .  # 소스 변경 시마다 아래 캐시 무효화
RUN npm install  # 매번 재실행

# ✅ 의존성 파일 먼저
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./  # 의존성만 먼저
RUN npm ci --only=production  # 캐시 유지
COPY . .  # 소스는 나중에
```

### 4. 보안 무시

```dockerfile
# ❌ root로 실행
FROM node:18-alpine
COPY . /app
WORKDIR /app
CMD ["node", "server.js"]
# root 권한으로 실행 (위험)

# ✅ non-root 실행
FROM node:18-alpine
RUN addgroup -g 1001 appgroup && \
    adduser -D -u 1001 -G appgroup appuser
WORKDIR /app
COPY --chown=appuser:appgroup . .
USER appuser
CMD ["node", "server.js"]
```

---

## 🎓 핵심 정리

### 1. 베이스 이미지 선택 기준

```
개발:
→ Full OS (ubuntu, debian)
  - 디버깅 도구 필요
  - 호환성 최우선

프로덕션 (일반):
→ Alpine / Slim
  - 크기와 보안 균형
  - 대부분의 앱에 적합

프로덕션 (고보안):
→ Distroless
  - 최소 공격 표면
  - 쉘 불필요

정적 바이너리:
→ Scratch
  - 최소 크기
  - Go, Rust 등
```

### 2. 최적화 우선순위

```
1순위: 멀티 스테이지 빌드
→ 가장 큰 효과, 비교적 쉬움

2순위: Alpine/Slim 이미지
→ 큰 효과, 호환성 주의

3순위: .dockerignore
→ 빌드 속도 향상

4순위: 레이어 최적화
→ 캐시 활용 극대화

5순위: Distroless
→ 최고 보안, 난이도 높음
```

### 3. 측정 지표

```
크기:
- 목표: 500MB 이하
- 우수: 200MB 이하
- 최적: 100MB 이하

취약점:
- 목표: CRITICAL 0개
- 우수: HIGH 10개 이하
- 최적: 총 50개 이하

빌드 시간:
- 목표: 5분 이하
- 우수: 2분 이하
- 최적: 1분 이하
```

### 4. 최적화 프로세스

```
1. 현재 상태 측정
   docker images
   trivy scan
   빌드 시간 측정

2. 목표 설정
   크기 목표
   보안 목표
   빌드 시간 목표

3. 순차 적용
   멀티 스테이지
   → Alpine/Distroless
   → .dockerignore
   → 레이어 최적화

4. 측정 및 검증
   크기 비교
   취약점 스캔
   성능 테스트

5. 반복
   추가 최적화
   성능 모니터링
```

---

## 📚 참고 자료

- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Alpine Linux](https://alpinelinux.org/)
- [Distroless Images](https://github.com/GoogleContainerTools/distroless)
- [Trivy Scanner](https://github.com/aquasecurity/trivy)
- [Dive Tool](https://github.com/wagoodman/dive)

---

## 🤔 생각해볼 문제

1. Alpine 이미지를 사용했는데도 크기가 큰 이유는?
2. Distroless vs Alpine, 언제 어떤 것을 선택할까?
3. 멀티 스테이지에서 불필요한 파일이 복사되는 것을 방지하려면?

> 💡 **답변**: 1) node_modules나 빌드 아티팩트가 과도하게 포함, npm 캐시 미정리, 2) Distroless는 보안 최우선(쉘 없음), Alpine은 크기와 호환성 균형, 3) COPY --from=builder로 필요한 파일만 명시적으로 복사, .dockerignore 활용

---

<div align="center">

**[⬅️ 이전: Multi-Stage Builds](./02-Multi-Stage-Builds.md)** | **[다음: Cache Mechanism ➡️](./04-Cache-Mechanism.md)**

</div>
