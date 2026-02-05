# 01. Dockerfile Best Practices - 효율적인 Dockerfile 작성법

## 🎯 이 챕터에서 배울 것

- **효율적인 레이어 구성**과 캐시 활용 전략
- **빌드 컨텍스트 최적화**로 빌드 시간 단축
- **베스트 프랙티스**로 유지보수하기 쉬운 Dockerfile
- **보안과 성능**을 고려한 실전 패턴

## 📌 왜 중요한가?

**"Dockerfile 작성법에 따라 빌드 시간과 이미지 크기가 10배 차이날 수 있습니다."**

```
잘못된 Dockerfile:
- 빌드 시간: 10분
- 이미지 크기: 2GB
- 캐시 히트율: 20%

최적화된 Dockerfile:
- 빌드 시간: 1분 (10배 빠름)
- 이미지 크기: 200MB (10배 작음)
- 캐시 히트율: 80%
```

**실무 영향:**
- CI/CD 파이프라인 속도 대폭 향상
- 네트워크 비용 절감 (이미지 크기 감소)
- 개발 생산성 증가 (빠른 피드백)

---

## 🔬 Deep Dive

### 1. 레이어 순서 최적화

#### 원칙: 변경 빈도 순서대로 배치

```
자주 변경되지 않는 것 → 위쪽 (캐시 유지)
자주 변경되는 것 → 아래쪽 (캐시 무효화 최소화)

이상적인 순서:
1. 베이스 이미지
2. 시스템 패키지 설치
3. 의존성 파일 복사 (package.json, requirements.txt)
4. 의존성 설치
5. 소스 코드 복사
6. 애플리케이션 빌드
```

#### Before: 비효율적인 순서

```dockerfile
FROM node:18-alpine

# ❌ 소스 코드를 먼저 복사
COPY . /app
WORKDIR /app

# 의존성 설치
RUN npm install

# 빌드
RUN npm run build

CMD ["npm", "start"]

# 문제:
# - 소스 코드 변경 시마다 npm install 재실행
# - 캐시 활용 불가
# - 빌드 시간: 5분 (매번)
```

#### After: 최적화된 순서

```dockerfile
FROM node:18-alpine

WORKDIR /app

# ✅ 의존성 파일만 먼저 복사
COPY package*.json ./

# 의존성 설치 (소스 변경 시 캐시 유지!)
RUN npm ci --only=production

# 소스 코드 복사
COPY . .

# 빌드
RUN npm run build

CMD ["npm", "start"]

# 개선:
# - 소스 변경 시 npm ci 건너뜀
# - 캐시 히트율 80%
# - 빌드 시간: 30초
```

---

### 2. 레이어 최소화

#### RUN 명령어 합치기

**❌ 나쁜 예: 불필요한 레이어**
```dockerfile
FROM ubuntu:22.04

RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get install -y git
RUN apt-get clean

# 문제:
# - 5개의 레이어 생성
# - apt 캐시가 중간 레이어에 남음 (크기 증가)
# - 빌드 느림
```

**✅ 좋은 예: 레이어 합치기**
```dockerfile
FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y \
        curl \
        vim \
        git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 개선:
# - 1개의 레이어
# - apt 캐시 즉시 제거 (크기 감소)
# - 빌드 빠름
```

#### 파이프라인 최적화

```dockerfile
# ✅ 한 줄로 연결
RUN wget -O - https://example.com/script.sh | sh

# ✅ 다운로드 후 즉시 삭제
RUN wget https://example.com/large-file.tar.gz && \
    tar -xzf large-file.tar.gz && \
    rm large-file.tar.gz

# ❌ 나쁜 예: 파일이 레이어에 남음
RUN wget https://example.com/large-file.tar.gz
RUN tar -xzf large-file.tar.gz
RUN rm large-file.tar.gz  # 이미 늦음! 이전 레이어에 파일 존재
```

---

### 3. 빌드 컨텍스트 최적화

#### .dockerignore 활용

**문제: 불필요한 파일까지 복사**
```dockerfile
COPY . /app

# 복사되는 파일:
# - node_modules/ (500MB)
# - .git/ (200MB)
# - dist/ (100MB)
# - *.log (50MB)
# → 총 850MB 전송!
```

**해결: .dockerignore 작성**
```dockerignore
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.env.*
dist
build
coverage
.vscode
.idea
*.log
*.tmp
*.swp
.DS_Store
Dockerfile
docker-compose.yml

# 결과: 10MB만 전송 (85배 감소!)
```

#### 빌드 컨텍스트 크기 확인

```bash
# 빌드 컨텍스트 크기 측정
docker build --no-cache -t test . 2>&1 | grep "Sending build context"
# Sending build context to Docker daemon  850MB

# .dockerignore 추가 후
# Sending build context to Docker daemon  10MB
```

---

### 4. 베이스 이미지 선택

#### 크기 비교

```dockerfile
# ❌ 매우 큼: 1GB+
FROM ubuntu:22.04

# ⚠️ 보통: 200-500MB
FROM node:18

# ✅ 작음: 50-100MB
FROM node:18-slim

# ✅ 매우 작음: 20-50MB
FROM node:18-alpine

# ✅ 극소: 5-10MB (앱 바이너리만)
FROM scratch
```

#### 실전 선택 가이드

```dockerfile
# 개발/디버깅: Full 버전
FROM node:18
# 장점: 모든 도구 포함
# 단점: 크기 큼

# 일반 프로덕션: Slim 버전
FROM node:18-slim
# 장점: 균형잡힌 크기와 호환성
# 단점: 일부 도구 없음

# 최적화 프로덕션: Alpine 버전
FROM node:18-alpine
# 장점: 매우 작음
# 단점: musl libc (호환성 주의)

# 바이너리만: Distroless/Scratch
FROM gcr.io/distroless/nodejs18
# 장점: 최소 공격 표면
# 단점: 디버깅 어려움
```

---

### 5. ARG vs ENV

#### ARG: 빌드 타임 변수

```dockerfile
# 빌드 시에만 사용
ARG NODE_VERSION=18
ARG BUILD_DATE
ARG GIT_COMMIT

FROM node:${NODE_VERSION}-alpine

# 빌드 정보 저장
LABEL build_date=${BUILD_DATE}
LABEL git_commit=${GIT_COMMIT}

# 빌드 시 전달
# docker build --build-arg NODE_VERSION=20 --build-arg BUILD_DATE=$(date) .
```

#### ENV: 런타임 변수

```dockerfile
# 컨테이너 실행 중에도 유지
ENV NODE_ENV=production
ENV PORT=3000
ENV LOG_LEVEL=info

# 런타임에 오버라이드 가능
# docker run -e NODE_ENV=development myapp
```

#### 조합 사용

```dockerfile
# ARG로 받아서 ENV로 설정
ARG APP_VERSION=1.0.0
ENV APP_VERSION=${APP_VERSION}

# 빌드 시: docker build --build-arg APP_VERSION=2.0.0 .
# 런타임에: echo $APP_VERSION → 2.0.0
```

---

### 6. COPY vs ADD

#### COPY: 단순 복사 (권장)

```dockerfile
# ✅ 명확하고 예측 가능
COPY package.json /app/
COPY src/ /app/src/

# 권한 설정
COPY --chown=node:node app.js /app/
```

#### ADD: 자동 압축 해제 (특수 기능)

```dockerfile
# ⚠️ tar 파일 자동 압축 해제
ADD archive.tar.gz /app/

# ⚠️ URL에서 다운로드 (비권장, RUN wget 사용 권장)
ADD https://example.com/file.txt /app/

# 문제: 예상치 못한 동작 가능
```

**권장사항:**
```dockerfile
# ✅ 압축 해제가 필요 없으면 COPY 사용
COPY archive.tar.gz /app/

# ✅ 압축 해제가 필요하면 명시적으로
RUN tar -xzf archive.tar.gz && rm archive.tar.gz

# ✅ URL 다운로드는 RUN으로
RUN wget https://example.com/file.txt
```

---

## 💻 실습

### 실습 1: Before/After 비교

**Before: 비효율적인 Python Dockerfile**
```dockerfile
FROM python:3.11

COPY . /app
WORKDIR /app

RUN pip install -r requirements.txt

CMD ["python", "app.py"]

# 문제:
# - 소스 변경 시마다 pip install 재실행
# - 큰 베이스 이미지
# - 캐시 활용 불가
```

**빌드 시간 측정**
```bash
# 첫 빌드
time docker build -t python-app-before .
# real: 2m30s

# 소스 코드 변경 후 재빌드
echo "# comment" >> app.py
time docker build -t python-app-before .
# real: 2m30s (캐시 안 됨!)

# 이미지 크기
docker images python-app-before
# 1.2GB
```

**After: 최적화된 Dockerfile**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 의존성 파일만 먼저 복사
COPY requirements.txt .

# 의존성 설치 (캐시 가능!)
RUN pip install --no-cache-dir -r requirements.txt

# 소스 코드 복사
COPY . .

CMD ["python", "app.py"]
```

**개선 결과**
```bash
# 첫 빌드
time docker build -t python-app-after .
# real: 1m20s (베이스 이미지 작아짐)

# 소스 코드 변경 후 재빌드
echo "# comment" >> app.py
time docker build -t python-app-after .
# real: 0m5s (캐시 활용!)

# 이미지 크기
docker images python-app-after
# 180MB (1/6 크기)
```

### 실습 2: 레이어 분석

```bash
# dive 도구 설치
brew install dive
# or
wget https://github.com/wagoodman/dive/releases/download/v0.11.0/dive_0.11.0_linux_amd64.tar.gz

# 이미지 레이어 분석
dive python-app-before

# 결과:
# Layer 1: FROM python:3.11 (900MB)
# Layer 2: COPY . /app (300MB) ← 불필요한 파일 포함
# Layer 3: pip install (100MB)

dive python-app-after

# 결과:
# Layer 1: FROM python:3.11-slim (150MB)
# Layer 2: COPY requirements.txt (1KB)
# Layer 3: pip install (30MB)
# Layer 4: COPY app.py (10KB)

# 개선:
# - 불필요한 파일 제거
# - 레이어 순서 최적화
# - 총 크기 감소
```

### 실습 3: 빌드 캐시 확인

```bash
# 캐시 동작 확인
docker build --progress=plain -t test .

# 출력:
# #5 [2/4] COPY requirements.txt .
# #5 CACHED

# #6 [3/4] RUN pip install -r requirements.txt
# #6 CACHED ← 캐시 사용!

# #7 [4/4] COPY . .
# #7 0.5s ← 소스만 복사

# 캐시 비활성화 빌드
docker build --no-cache -t test .
# 모든 단계 재실행
```

---

## 🔥 실전 적용

### 시나리오 1: Node.js 애플리케이션

**완벽한 Node.js Dockerfile**
```dockerfile
# 1. 빌드 스테이지
FROM node:18-alpine AS builder

WORKDIR /app

# 의존성 파일 복사
COPY package*.json ./

# 의존성 설치 (dev dependencies 포함)
RUN npm ci

# 소스 코드 복사
COPY . .

# 빌드
RUN npm run build

# 2. 프로덕션 스테이지
FROM node:18-alpine

# 보안: 비 root 사용자
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# 의존성 파일 복사
COPY package*.json ./

# 프로덕션 의존성만 설치
RUN npm ci --only=production && \
    npm cache clean --force

# 빌드 결과물 복사
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist

USER nodejs

EXPOSE 3000

CMD ["node", "dist/index.js"]

# 결과:
# - 빌드 스테이지와 프로덕션 분리
# - dev dependencies 제외
# - 비 root 실행
# - 최종 이미지 크기: 80MB
```

### 시나리오 2: Java Spring Boot

```dockerfile
FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

# pom.xml만 먼저 복사
COPY pom.xml .

# 의존성 다운로드 (캐시 가능!)
RUN mvn dependency:go-offline

# 소스 코드 복사
COPY src ./src

# 빌드
RUN mvn package -DskipTests

# 프로덕션 이미지
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

# JAR 파일만 복사
COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]

# 개선:
# - Maven 의존성 캐싱
# - JRE만 사용 (JDK 불필요)
# - 이미지 크기: 200MB (from 700MB)
```

### 시나리오 3: Python FastAPI

```dockerfile
FROM python:3.11-slim AS builder

WORKDIR /app

# 시스템 의존성
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        gcc \
        python3-dev \
    && rm -rf /var/lib/apt/lists/*

# 의존성 파일 복사
COPY requirements.txt .

# 가상환경에 설치
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir -r requirements.txt

# 프로덕션 이미지
FROM python:3.11-slim

WORKDIR /app

# 가상환경 복사
COPY --from=builder /opt/venv /opt/venv

# 소스 코드 복사
COPY . .

# 가상환경 활성화
ENV PATH="/opt/venv/bin:$PATH"

# 비 root 사용자
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

# 특징:
# - 빌드 도구 분리
# - 가상환경 사용
# - 비 root 실행
```

---

## ⚡ 고급 팁

### 1. BuildKit 기능 활용

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine

WORKDIR /app

# BuildKit 캐시 마운트
RUN --mount=type=cache,target=/root/.npm \
    npm install

# Secret 마운트 (빌드 로그에 남지 않음)
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm install

# SSH 마운트 (private repo 접근)
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git
```

### 2. 헬스체크 추가

```dockerfile
FROM nginx:alpine

COPY nginx.conf /etc/nginx/nginx.conf
COPY html /usr/share/nginx/html

# 헬스체크 설정
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1

EXPOSE 80
```

### 3. 메타데이터 레이블

```dockerfile
FROM node:18-alpine

# OCI 표준 레이블
LABEL org.opencontainers.image.title="My App"
LABEL org.opencontainers.image.description="Production-ready Node.js app"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.authors="team@example.com"
LABEL org.opencontainers.image.source="https://github.com/example/myapp"

# 확인:
# docker inspect myapp | jq '.[0].Config.Labels'
```

---

## 🚫 안티패턴

### 1. 루트 사용자로 실행

```dockerfile
# ❌ 위험: root로 실행
FROM node:18-alpine
COPY . /app
CMD ["node", "app.js"]

# ✅ 안전: 비 root 사용자
FROM node:18-alpine
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser
COPY --chown=appuser:appgroup . /app
USER appuser
CMD ["node", "app.js"]
```

### 2. 비밀 정보 포함

```dockerfile
# ❌ 매우 위험: 비밀 정보가 레이어에 남음
FROM alpine
ARG API_KEY=secret123
ENV API_KEY=${API_KEY}
RUN echo "API_KEY=${API_KEY}" > /config

# ✅ 안전: 런타임에 주입
FROM alpine
# 환경 변수로 전달
# docker run -e API_KEY=secret123 myapp
```

### 3. 최신 태그 사용

```dockerfile
# ❌ 불안정: 예측 불가능
FROM node:latest

# ✅ 안정: 특정 버전 고정
FROM node:18.17.0-alpine3.18

# 또는 메이저 버전만
FROM node:18-alpine
```

---

## 📊 체크리스트

```
이미지 빌드 체크리스트:

✅ 베이스 이미지
   ├─ slim/alpine 버전 사용
   ├─ 특정 버전 태그 지정
   └─ 공식 이미지 사용

✅ 레이어 최적화
   ├─ 변경 빈도 순서로 배치
   ├─ RUN 명령어 합치기
   └─ 불필요한 파일 삭제 (같은 레이어)

✅ 빌드 컨텍스트
   ├─ .dockerignore 작성
   ├─ 필요한 파일만 COPY
   └─ 빌드 컨텍스트 크기 확인

✅ 보안
   ├─ 비 root 사용자 실행
   ├─ 비밀 정보 제외
   └─ 최소 권한 원칙

✅ 캐시 활용
   ├─ 의존성 파일 먼저 복사
   ├─ 소스 코드는 나중에 복사
   └─ 캐시 히트율 확인

✅ 메타데이터
   ├─ HEALTHCHECK 추가
   ├─ LABEL 정보 기입
   └─ EXPOSE 포트 명시
```

---

## 🎓 핵심 정리

### Dockerfile 최적화 원칙

```
1. 레이어 순서
   변경 빈도 낮음 → 위
   변경 빈도 높음 → 아래

2. 베이스 이미지
   Full > Slim > Alpine > Distroless

3. 캐시 전략
   의존성 → 먼저
   소스 코드 → 나중

4. 보안
   비 root 사용자 필수
   비밀 정보 제외

5. 크기 최적화
   레이어 합치기
   불필요한 파일 제거
   멀티 스테이지 빌드
```

---

## 🔗 다음 단계

Dockerfile 최적화를 마스터했습니다! 다음 챕터:

- **[02. Multi-Stage Builds](./02-Multi-Stage-Builds.md)**: 빌드/실행 환경 분리
- **[03. Image Optimization](./03-Image-Optimization.md)**: 이미지 크기 극한 최적화
- **[04. Cache Mechanism](./04-Cache-Mechanism.md)**: 캐시 시스템 심화

---

## 📚 참고 자료

- [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker BuildKit](https://docs.docker.com/build/buildkit/)
- [Dive - Image Layer Explorer](https://github.com/wagoodman/dive)
- [Hadolint - Dockerfile Linter](https://github.com/hadolint/hadolint)

---

**🤔 생각해볼 문제**

1. `COPY . .`를 맨 위에 두면 어떤 문제가 생길까?
2. Alpine과 Ubuntu 중 어느 것을 선택해야 할까?
3. ARG와 ENV의 차이가 중요한 이유는?

> 💡 **답변**: 1) 소스 변경 시마다 모든 레이어 재빌드, 2) 상황에 따라 (호환성 vs 크기), 3) 빌드 타임 vs 런타임 보안
