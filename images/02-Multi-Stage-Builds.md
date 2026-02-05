# 02. Multi-Stage Builds - 멀티 스테이지 빌드

## 🎯 이 챕터에서 배울 것

- **멀티 스테이지 빌드**의 개념과 동작 원리
- **빌드 환경과 실행 환경 분리**로 이미지 크기 최소화
- 언어별 **최적 멀티 스테이지 패턴**
- 복잡한 빌드 파이프라인 구성

## 📌 왜 중요한가?

**"빌드 도구와 실행 파일을 분리하면 이미지 크기가 10배 줄어듭니다."**

```
단일 스테이지:
FROM golang:1.21
COPY . .
RUN go build
→ 이미지 크기: 800MB (Go 컴파일러 포함)

멀티 스테이지:
FROM golang:1.21 AS builder
RUN go build
FROM scratch
COPY --from=builder /app .
→ 이미지 크기: 8MB (바이너리만)
```

**실무 영향:**
- 이미지 크기: 800MB → 8MB (100배 감소)
- 배포 시간: 2분 → 10초
- 보안: 빌드 도구 제외로 공격 표면 축소
- 비용: 네트워크/스토리지 비용 대폭 절감

---

## 🔬 Deep Dive

### 1. 멀티 스테이지 빌드란?

#### 개념

```
하나의 Dockerfile에서 여러 FROM 사용

┌─────────────────────────────────┐
│  Stage 1: Build                 │
│  - 빌드 도구 포함                  │
│  - 소스 코드 컴파일                 │
│  - 바이너리 생성                   │
└─────────────────────────────────┘
         ↓ (아티팩트만 복사)
┌─────────────────────────────────┐
│  Stage 2: Production            │
│  - 최소 런타임만 포함               │
│  - 빌드 결과물만 복사               │
│  - 실행 환경 구성                  │
└─────────────────────────────────┘

최종 이미지 = Stage 2만!
```

#### 기본 구조

```dockerfile
# Stage 1: Builder
FROM build-image AS builder
# 빌드 작업
RUN ...

# Stage 2: Production
FROM runtime-image
# 빌드 결과물만 복사
COPY --from=builder /path/to/artifact .
```

---

### 2. 단일 vs 멀티 스테이지 비교

#### 단일 스테이지 (전통적 방식)

```dockerfile
FROM golang:1.21

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o myapp

CMD ["./myapp"]

# 문제:
# - Go 컴파일러 포함 (700MB)
# - 소스 코드 포함
# - go mod 캐시 포함
# - 총 크기: 800MB
```

#### 멀티 스테이지 (최적화)

```dockerfile
# ========== Stage 1: Build ==========
FROM golang:1.21 AS builder

WORKDIR /app

# 의존성 다운로드 (캐시 가능)
COPY go.mod go.sum ./
RUN go mod download

# 소스 코드 복사 및 빌드
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp

# ========== Stage 2: Production ==========
FROM scratch

# 빌드된 바이너리만 복사
COPY --from=builder /app/myapp /myapp

# CA 인증서 복사 (HTTPS 통신용)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

CMD ["/myapp"]

# 결과:
# - Go 컴파일러 제외
# - 소스 코드 제외
# - 바이너리만 포함
# - 총 크기: 8MB (100배 감소!)
```

---

### 3. 언어별 멀티 스테이지 패턴

#### Go: Scratch 베이스

```dockerfile
# Build Stage
FROM golang:1.21-alpine AS builder

WORKDIR /app

# 의존성
COPY go.mod go.sum ./
RUN go mod download

# 빌드 (정적 링크)
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Production Stage
FROM scratch

# 타임존 데이터
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# CA 인증서
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 바이너리
COPY --from=builder /app/main /main

EXPOSE 8080
CMD ["/main"]

# 최종: 10MB
```

#### Node.js: 의존성 분리

```dockerfile
# Build Stage
FROM node:18-alpine AS builder

WORKDIR /app

# 의존성 설치
COPY package*.json ./
RUN npm ci

# 소스 복사 및 빌드
COPY . .
RUN npm run build

# Production Stage
FROM node:18-alpine

WORKDIR /app

# 프로덕션 의존성만
COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# 빌드 결과물만
COPY --from=builder /app/dist ./dist

# 비 root 사용자
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
USER nodejs

EXPOSE 3000
CMD ["node", "dist/index.js"]

# 최종: 150MB (from 800MB)
```

#### Python: 가상환경 전달

```dockerfile
# Build Stage
FROM python:3.11-slim AS builder

WORKDIR /app

# 빌드 도구 설치
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        gcc \
        python3-dev \
    && rm -rf /var/lib/apt/lists/*

# 가상환경 생성
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 의존성 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Production Stage
FROM python:3.11-slim

WORKDIR /app

# 가상환경 복사
COPY --from=builder /opt/venv /opt/venv

# 소스 코드
COPY . .

# 가상환경 활성화
ENV PATH="/opt/venv/bin:$PATH"

# 비 root 사용자
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app
USER appuser

EXPOSE 8000
CMD ["python", "app.py"]

# 최종: 200MB (from 1GB)
```

#### Java: JRE만 사용

```dockerfile
# Build Stage
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# 의존성 캐싱
COPY pom.xml .
RUN mvn dependency:go-offline

# 소스 빌드
COPY src ./src
RUN mvn package -DskipTests

# Production Stage
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# JAR 파일만 복사
COPY --from=builder /app/target/*.jar app.jar

# 비 root 사용자
RUN addgroup -S spring && adduser -S spring -G spring
USER spring

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]

# 최종: 250MB (from 700MB)
```

#### Rust: 극한 최적화

```dockerfile
# Build Stage
FROM rust:1.73 AS builder

WORKDIR /app

# 의존성 캐싱 트릭
RUN cargo init --name myapp
COPY Cargo.toml Cargo.lock ./
RUN cargo build --release && \
    rm src/*.rs

# 소스 빌드
COPY src ./src
RUN cargo build --release

# Production Stage
FROM debian:bookworm-slim

# 런타임 의존성만
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# 바이너리 복사
COPY --from=builder /app/target/release/myapp /usr/local/bin/

CMD ["myapp"]

# 최종: 15MB
```

---

### 4. 고급 패턴

#### 패턴 1: 여러 빌드 스테이지

```dockerfile
# Stage 1: 의존성 다운로드
FROM node:18-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Stage 2: 개발 의존성 포함 빌드
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Stage 3: 프로덕션 의존성만
FROM node:18-alpine AS prod-deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 4: 최종 프로덕션 이미지
FROM node:18-alpine
WORKDIR /app

COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json ./

USER node
CMD ["node", "dist/index.js"]

# 장점:
# - 각 스테이지 독립적 캐싱
# - 명확한 책임 분리
```

#### 패턴 2: 병렬 빌드 스테이지

```dockerfile
# 공통 베이스
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./

# 병렬 스테이지 1: Frontend 빌드
FROM base AS frontend-builder
RUN npm ci
COPY frontend ./frontend
RUN npm run build:frontend

# 병렬 스테이지 2: Backend 빌드
FROM base AS backend-builder
RUN npm ci
COPY backend ./backend
RUN npm run build:backend

# 최종: 결과물 병합
FROM node:18-alpine
WORKDIR /app

COPY --from=frontend-builder /app/dist/frontend ./public
COPY --from=backend-builder /app/dist/backend ./

CMD ["node", "index.js"]

# BuildKit이 frontend와 backend를 병렬 빌드!
```

#### 패턴 3: 테스트 스테이지

```dockerfile
# Build Stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Test Stage (선택적)
FROM builder AS tester
RUN npm run test
RUN npm run lint

# Production Stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]

# 테스트 포함 빌드:
# docker build --target tester .

# 프로덕션 빌드:
# docker build .
```

---

## 💻 실습

### 실습 1: Go 애플리케이션 최적화

**Before: 단일 스테이지**
```dockerfile
FROM golang:1.21

WORKDIR /app
COPY . .
RUN go build -o server

CMD ["./server"]
```

```bash
docker build -t go-app-before .
docker images go-app-before
# REPOSITORY      SIZE
# go-app-before   826MB
```

**After: 멀티 스테이지**
```dockerfile
# Build
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o server

# Production
FROM alpine:latest

RUN apk --no-cache add ca-certificates

COPY --from=builder /app/server /server

EXPOSE 8080
CMD ["/server"]
```

```bash
docker build -t go-app-after .
docker images go-app-after
# REPOSITORY     SIZE
# go-app-after   15MB

# 55배 감소!
```

### 실습 2: React + Node.js 풀스택

```dockerfile
# ========== Frontend Build ==========
FROM node:18-alpine AS frontend-builder

WORKDIR /app/frontend

COPY frontend/package*.json ./
RUN npm ci

COPY frontend/ ./
RUN npm run build

# ========== Backend Build ==========
FROM node:18-alpine AS backend-builder

WORKDIR /app/backend

COPY backend/package*.json ./
RUN npm ci

COPY backend/ ./
RUN npm run build

# ========== Production ==========
FROM node:18-alpine

WORKDIR /app

# Backend 프로덕션 의존성
COPY backend/package*.json ./
RUN npm ci --only=production

# Backend 빌드 결과
COPY --from=backend-builder /app/backend/dist ./dist

# Frontend 빌드 결과 (static files)
COPY --from=frontend-builder /app/frontend/build ./public

EXPOSE 3000
CMD ["node", "dist/index.js"]

# 결과:
# - Frontend 빌드 도구 제외
# - Backend dev 의존성 제외
# - 정적 파일 + API 서버만
```

### 실습 3: 특정 스테이지만 빌드

```dockerfile
# Development
FROM node:18 AS development
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

# Test
FROM development AS test
RUN npm run test

# Build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

**각 스테이지별 빌드**
```bash
# 개발 환경
docker build --target development -t myapp:dev .

# 테스트만 실행
docker build --target test -t myapp:test .

# 프로덕션 (기본)
docker build -t myapp:prod .

# 또는 명시적으로
docker build --target production -t myapp:prod .
```

---

## 🔥 실전 적용

### 시나리오 1: 마이크로서비스 통합

```dockerfile
# ========== 공통 베이스 ==========
FROM node:18-alpine AS base
WORKDIR /app
RUN npm install -g pnpm

# ========== 의존성 ==========
FROM base AS deps
COPY pnpm-lock.yaml package.json ./
RUN pnpm install --frozen-lockfile

# ========== Service A ==========
FROM deps AS service-a-builder
COPY services/service-a ./
RUN pnpm run build

FROM base AS service-a
COPY --from=service-a-builder /app/dist ./
CMD ["node", "index.js"]

# ========== Service B ==========
FROM deps AS service-b-builder
COPY services/service-b ./
RUN pnpm run build

FROM base AS service-b
COPY --from=service-b-builder /app/dist ./
CMD ["node", "index.js"]

# 빌드:
# docker build --target service-a -t service-a .
# docker build --target service-b -t service-b .
```

### 시나리오 2: 데이터베이스 마이그레이션

```dockerfile
# Build Stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Migration Stage
FROM builder AS migrator
CMD ["npm", "run", "migrate"]

# Production Stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]

# 사용:
# docker build --target migrator -t myapp:migrate .
# docker run myapp:migrate  # 마이그레이션 실행
# docker build -t myapp:latest .
# docker run myapp:latest  # 앱 실행
```

### 시나리오 3: 문서 생성 포함

```dockerfile
# Build Stage
FROM rust:1.73 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# Documentation Stage
FROM builder AS docs
RUN cargo doc --no-deps

# Production Stage
FROM debian:bookworm-slim
COPY --from=builder /app/target/release/myapp /usr/local/bin/

# 문서 추출 (선택적)
# docker build --target docs -t myapp:docs .
# docker run -v $(pwd)/docs:/app/target/doc myapp:docs
```

---

## ⚡ 최적화 팁

### 1. 스테이지 순서 최적화

```dockerfile
# ❌ 비효율: 자주 변경되는 것을 먼저
FROM node:18 AS frontend
COPY frontend ./  # 자주 변경됨
RUN npm run build

FROM node:18 AS backend
COPY backend ./   # 자주 변경됨
RUN npm run build

# ✅ 효율: 의존성을 먼저
FROM node:18 AS deps
COPY package*.json ./
RUN npm ci  # 덜 자주 변경됨

FROM deps AS frontend
COPY frontend ./
RUN npm run build

FROM deps AS backend
COPY backend ./
RUN npm run build
```

### 2. 빌드 아티팩트 최소화

```dockerfile
# ❌ 불필요한 파일 포함
COPY --from=builder /app ./

# ✅ 필요한 것만 복사
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
```

### 3. 레이어 재사용

```dockerfile
# 여러 서비스가 같은 의존성 사용
FROM node:18-alpine AS deps
COPY package*.json ./
RUN npm ci
# → 이 레이어는 모든 서비스가 공유!

FROM deps AS service1
FROM deps AS service2
FROM deps AS service3
```

---

## 🚫 안티패턴

### 1. 중간 스테이지에 비밀 정보

```dockerfile
# ❌ 위험: 중간 스테이지에 토큰
FROM node:18 AS builder
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc
RUN npm install
# .npmrc가 이미지에 남음!

# ✅ 안전: BuildKit secret 사용
FROM node:18 AS builder
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install
```

### 2. 불필요한 중간 이미지

```dockerfile
# ❌ 비효율: 너무 많은 스테이지
FROM base AS step1
FROM step1 AS step2
FROM step2 AS step3
FROM step3 AS step4
# 캐시 관리 어려움

# ✅ 효율: 의미 있는 단위로
FROM base AS builder
FROM builder AS tester
FROM base AS production
```

### 3. 스테이지 간 불필요한 복사

```dockerfile
# ❌ 비효율: 모든 것 복사
COPY --from=builder /app ./

# ✅ 효율: 필요한 것만
COPY --from=builder /app/binary ./binary
```

---

## 📊 비교표

| 측면 | 단일 스테이지 | 멀티 스테이지 |
|------|--------------|--------------|
| **이미지 크기** | 매우 큼 (GB) | 작음 (MB) |
| **빌드 시간** | 보통 | 빠름 (캐시) |
| **보안** | 낮음 | 높음 |
| **복잡도** | 낮음 | 중간 |
| **유지보수** | 어려움 | 쉬움 |
| **적용** | 개발/테스트 | 프로덕션 |

---

## 🎓 핵심 정리

### 멀티 스테이지 빌드 핵심

```
1. 개념
   하나의 Dockerfile, 여러 FROM
   최종 이미지 = 마지막 스테이지만

2. 장점
   - 이미지 크기 극적 감소
   - 빌드 도구 제외
   - 보안 향상
   - 명확한 책임 분리

3. 패턴
   Builder → Production
   Builder → Tester → Production
   Deps → Builder → Production

4. 실전 팁
   - 언어별 최적 패턴 적용
   - 의존성 캐싱 최대화
   - scratch/alpine 활용
   - 필요한 것만 복사
```

---

## 📚 참고 자료

- [Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Distroless Images](https://github.com/GoogleContainerTools/distroless)
- [Scratch Base Image](https://hub.docker.com/_/scratch)

---

**🤔 생각해볼 문제**

1. 멀티 스테이지를 사용해도 이미지가 큰 이유는?
2. scratch vs alpine vs distroless 언제 사용할까?
3. 테스트 스테이지를 언제 사용해야 할까?

> 💡 **답변**: 1) 최종 스테이지에 불필요한 파일 포함, 2) 정적링크/최소런타임/보안중시, 3) CI/CD에서 빌드 전 검증

---

<div align="center">

**[⬅️ 이전: Dockerfile Best Practices](./01-Dockerfile-Best-Practices.md)** | **[다음: Image Optimization ➡️](./03-Image-Optimization.md)**

</div>
