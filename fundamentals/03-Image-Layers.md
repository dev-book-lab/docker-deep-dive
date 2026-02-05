# 03. Image Layers - Docker 이미지 레이어 시스템

## 🎯 이 챕터에서 배울 것

- Docker 이미지의 **레이어 구조**와 동작 원리
- **Copy-on-Write (CoW)** 메커니즘 완전 이해
- 레이어 **캐싱 전략**과 최적화
- 실제 레이어 구조 분석 및 디버깅

## 📌 왜 중요한가?

**"이미지 레이어를 이해하면 Docker의 80%를 이해한 것입니다."**

```
문제 상황:
- Dockerfile 한 줄만 수정했는데 전체 빌드가 다시 실행됨
- 이미지 크기가 예상보다 훨씬 큼
- 빌드 시간이 너무 오래 걸림

원인: 레이어 시스템을 이해하지 못함
해결: 레이어 구조를 알면 모든 것이 명확해짐
```

**실전 영향:**
- 빌드 시간: 20분 → 2분 (10배 단축)
- 이미지 크기: 2GB → 200MB (10배 감소)
- 네트워크 비용: 레이어 재사용으로 대폭 절감

---

## 🔬 Deep Dive

### 1. 이미지 레이어란?

#### 개념

```
Docker 이미지 = 읽기 전용 레이어들의 스택

┌─────────────────────────────────┐
│   Layer 4: COPY app.js /app/    │  ← 가장 위 (최신)
├─────────────────────────────────┤
│   Layer 3: RUN npm install      │
├─────────────────────────────────┤
│   Layer 2: WORKDIR /app         │
├─────────────────────────────────┤
│   Layer 1: FROM node:18-alpine  │  ← 베이스
└─────────────────────────────────┘
        ↓ (컨테이너 실행 시)
┌─────────────────────────────────┐
│ Container Layer (R/W)           │  ← 쓰기 가능
├─────────────────────────────────┤
│   Layer 4: (읽기 전용)            │
│   Layer 3: (읽기 전용)            │
│   Layer 2: (읽기 전용)            │
│   Layer 1: (읽기 전용)            │
└─────────────────────────────────┘
```

#### 핵심 특징

1. **불변성 (Immutability)**
   - 한번 생성된 레이어는 절대 변경 안 됨
   - 재사용 가능
   - 안전한 공유

2. **증분 변경 (Incremental Changes)**
   - 각 레이어는 이전 레이어의 변경사항만 포함
   - 효율적인 저장

3. **레이어 공유 (Layer Sharing)**
   - 여러 이미지가 같은 레이어 공유
   - 디스크 공간 절약

---

### 2. 레이어 생성 메커니즘

#### Dockerfile 명령어와 레이어

**레이어를 생성하는 명령어:**
```dockerfile
FROM ubuntu:22.04        # Layer 1: 베이스 이미지
RUN apt-get update       # Layer 2: 패키지 업데이트
RUN apt-get install -y nginx  # Layer 3: nginx 설치
COPY index.html /var/www/html/  # Layer 4: 파일 복사
```

**레이어를 생성하지 않는 명령어 (메타데이터만):**
```dockerfile
ENV NODE_ENV=production  # 레이어 X, 메타데이터만
EXPOSE 3000             # 레이어 X, 문서화
CMD ["node", "app.js"]  # 레이어 X, 실행 명령어만
LABEL version="1.0"     # 레이어 X, 라벨
```

#### 실제 레이어 확인

```bash
# 이미지 빌드
cat > Dockerfile <<EOF
FROM alpine:3.18
RUN echo "Layer 1" > /file1.txt
RUN echo "Layer 2" > /file2.txt
RUN echo "Layer 3" > /file3.txt
COPY app.js /app.js
EOF

docker build -t layer-test .

# 레이어 확인
docker history layer-test

# IMAGE          CREATED BY                                      SIZE
# abc123...      COPY app.js /app.js                            1.2kB
# def456...      RUN echo "Layer 3" > /file3.txt                45B
# ghi789...      RUN echo "Layer 2" > /file2.txt                45B
# jkl012...      RUN echo "Layer 1" > /file1.txt                45B
# alpine:3.18    ...                                            7.04MB
```

#### 레이어 내부 확인

```bash
# 이미지를 tar로 추출
docker save layer-test -o layer-test.tar
tar -xf layer-test.tar

# 구조 확인
tree
# .
# ├── abc123.../
# │   ├── layer.tar    ← 실제 레이어 데이터
# │   └── json         ← 레이어 메타데이터
# ├── def456.../
# ├── manifest.json
# └── repositories

# 특정 레이어 내용 확인
cd abc123...
tar -tf layer.tar
# app.js  ← 이 레이어에 추가된 파일
```

---

### 3. Copy-on-Write (CoW) 메커니즘

#### 동작 원리

```
시나리오: 컨테이너에서 /etc/nginx/nginx.conf 수정

┌─────────────────────────────────┐
│ Container Layer (R/W)           │
│ ┌─────────────────────────────┐ │
│ │ nginx.conf (modified)       │ │ ← 수정된 파일 복사
│ └─────────────────────────────┘ │
├─────────────────────────────────┤
│ Image Layer 3                   │
│ ┌─────────────────────────────┐ │
│ │ nginx.conf (original)       │ │ ← 원본 유지
│ └─────────────────────────────┘ │
├─────────────────────────────────┤
│ Image Layer 2                   │
├─────────────────────────────────┤
│ Image Layer 1                   │
└─────────────────────────────────┘

읽기: 위에서 아래로 검색, 첫 번째 발견된 파일 사용
쓰기: Container Layer에 복사 후 수정 (원본 보존)
```

#### CoW 실습

```bash
# 1. 컨테이너 실행
docker run -it --name cow-test ubuntu bash

# 2. 파일 수정 (컨테이너 내부)
echo "original content" > /test.txt
cat /test.txt
# original content

# 3. 새 터미널에서 컨테이너 레이어 확인
docker diff cow-test
# A /test.txt  ← Added

# 4. 기존 시스템 파일 수정
echo "modified" >> /etc/hosts

docker diff cow-test
# C /etc          ← Changed (디렉토리)
# C /etc/hosts    ← Changed (파일)
# A /test.txt

# 5. 파일 삭제
rm /etc/hostname

docker diff cow-test
# C /etc
# C /etc/hosts
# D /etc/hostname  ← Deleted (실제로는 숨김)
# A /test.txt
```

#### CoW의 성능 영향

**파일 크기와 성능**
```bash
# 큰 파일 수정 테스트
docker run -it --name cow-perf ubuntu bash

# 1GB 파일 생성
dd if=/dev/zero of=/bigfile bs=1M count=1024

# 파일 수정 (1바이트만)
echo "x" >> /bigfile

# → 전체 1GB가 Container Layer로 복사됨!
# → 첫 번째 쓰기 작업이 느림

docker diff cow-perf
# A /bigfile  (1GB)
```

**Best Practice:**
```dockerfile
# ❌ 나쁜 예: 큰 파일을 이미지에 포함 후 수정
COPY large-database.sql /data/
RUN sed -i 's/old/new/g' /data/large-database.sql

# ✅ 좋은 예: 수정 후 복사 or Volume 사용
RUN sed 's/old/new/g' /tmp/large-database.sql > /data/large-database.sql
# 또는
# Volume 사용으로 CoW 우회
```

---

### 4. 레이어 캐싱

#### 캐시 메커니즘

```
Docker 빌드 과정:

1. Dockerfile 명령어 순차 실행
2. 각 명령어마다:
   ├─ 캐시 체크 (동일한 명령어 이전에 실행했나?)
   │  ├─ Yes → 캐시 사용 (빠름!)
   │  └─ No  → 실행 (느림)
   └─ 한번 캐시 미스 → 이후 모든 명령어 재실행

캐시 키 = 명령어 + 컨텍스트
```

#### 캐시 무효화 조건

```dockerfile
# 예제 Dockerfile
FROM node:18-alpine          # Cache HIT: 베이스 이미지
WORKDIR /app                 # Cache HIT: 명령어 동일
COPY package*.json ./        # Cache 체크: 파일 checksum 비교
RUN npm install              # 이전 단계 캐시 여부에 따라
COPY . .                     # Cache 체크: 파일 내용 비교
CMD ["npm", "start"]         # 메타데이터만, 영향 없음
```

**캐시 무효화 시나리오:**

1. **명령어 변경**
```dockerfile
# Before
RUN npm install

# After  
RUN npm ci              # ← 명령어 변경 → 캐시 MISS
```

2. **파일 내용 변경**
```dockerfile
COPY package.json ./    # package.json 내용 변경 → 캐시 MISS
```

3. **파일 권한/타임스탬프 변경**
```dockerfile
COPY app.js ./          # app.js 내용은 동일하지만
                        # chmod +x app.js 실행 → 캐시 MISS
```

#### 캐시 최적화 전략

**❌ 비효율적인 레이어 배치**
```dockerfile
FROM node:18-alpine

# 소스 코드를 먼저 복사
COPY . .                     # 소스 코드 변경 시마다 캐시 MISS

# 의존성 설치
RUN npm install              # ← 항상 재실행 (느림!)

CMD ["npm", "start"]

# 문제: 소스 코드만 변경해도 npm install 재실행
# → 빌드 시간: 5분
```

**✅ 최적화된 레이어 배치**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# 의존성 파일만 먼저 복사
COPY package*.json ./        # package.json 변경 시에만 캐시 MISS

# 의존성 설치
RUN npm install              # ← 대부분 캐시 HIT! (빠름)

# 소스 코드 복사
COPY . .                     # 소스 코드 변경해도 위는 캐시 유지

CMD ["npm", "start"]

# 장점: 소스 코드 변경 시 npm install 건너뜀
# → 빌드 시간: 10초
```

---

## 💻 실습

### 실습 1: 레이어 구조 완전 분석

```bash
# 복잡한 Dockerfile 작성
cat > Dockerfile <<'EOF'
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y curl
RUN apt-get install -y vim
RUN echo "Hello Layer 1" > /layer1.txt
RUN echo "Hello Layer 2" > /layer2.txt

COPY app.sh /app.sh
RUN chmod +x /app.sh

ENV APP_VERSION=1.0
EXPOSE 8080
CMD ["/app.sh"]
EOF

echo '#!/bin/bash\necho "App running"' > app.sh

# 빌드
docker build -t layer-analysis .

# 1. 히스토리 확인 (레이어 목록)
docker history layer-analysis

# 2. 상세 정보
docker history --no-trunc layer-analysis

# 3. 레이어 크기 확인
docker history --format "{{.Size}}\t{{.CreatedBy}}" layer-analysis

# 4. 이미지 인스펙트
docker image inspect layer-analysis

# 5. 레이어 ID 확인
docker image inspect layer-analysis | jq '.[0].RootFS.Layers'
# [
#   "sha256:abc123...",  ← Layer 1
#   "sha256:def456...",  ← Layer 2
#   ...
# ]
```

### 실습 2: 캐시 동작 확인

```bash
# Dockerfile 작성
cat > Dockerfile <<'EOF'
FROM alpine:3.18
RUN echo "Step 1" && sleep 2
RUN echo "Step 2" && sleep 2
RUN echo "Step 3" && sleep 2
RUN echo "Step 4" && sleep 2
EOF

# 첫 번째 빌드 (캐시 없음)
time docker build -t cache-test:v1 .
# Step 1: sleep 2초
# Step 2: sleep 2초
# Step 3: sleep 2초
# Step 4: sleep 2초
# Total: ~8초

# 두 번째 빌드 (캐시 사용)
time docker build -t cache-test:v2 .
# Step 1: Using cache  ← 즉시
# Step 2: Using cache  ← 즉시
# Step 3: Using cache  ← 즉시
# Step 4: Using cache  ← 즉시
# Total: ~0.5초

# Dockerfile 중간 수정
cat > Dockerfile <<'EOF'
FROM alpine:3.18
RUN echo "Step 1" && sleep 2
RUN echo "Step 2 MODIFIED" && sleep 2  # ← 수정
RUN echo "Step 3" && sleep 2
RUN echo "Step 4" && sleep 2
EOF

# 세 번째 빌드 (중간부터 캐시 무효화)
time docker build -t cache-test:v3 .
# Step 1: Using cache     ← 캐시 HIT
# Step 2: sleep 2초       ← 캐시 MISS (재실행)
# Step 3: sleep 2초       ← 이후 모두 재실행
# Step 4: sleep 2초       ← 이후 모두 재실행
# Total: ~6초
```

### 실습 3: 레이어 공유 확인

```bash
# 베이스 이미지 pull
docker pull nginx:alpine
docker pull nginx:latest

# 두 이미지의 레이어 확인
docker history nginx:alpine --format "{{.ID}}\t{{.Size}}"
docker history nginx:latest --format "{{.ID}}\t{{.Size}}"

# 공통 레이어 확인 (Alpine 기반이면 공유)
docker image inspect nginx:alpine | jq '.[0].RootFS.Layers'
docker image inspect nginx:latest | jq '.[0].RootFS.Layers'

# 전체 이미지 크기 vs 실제 디스크 사용량
docker images | grep nginx
# nginx  alpine  50MB
# nginx  latest  150MB
# → 200MB 예상?

docker system df -v
# Images space usage: 180MB  ← 실제로는 20MB 절약 (레이어 공유)
```

### 실습 4: CoW 성능 측정

```bash
# 테스트 이미지 생성
cat > Dockerfile <<'EOF'
FROM ubuntu:22.04
RUN dd if=/dev/zero of=/largefile bs=1M count=512  # 512MB 파일
EOF

docker build -t cow-test .

# 시나리오 1: 파일 읽기 (빠름)
time docker run --rm cow-test cat /largefile > /dev/null
# real: 0.5초

# 시나리오 2: 파일 수정 (느림 - CoW 발생)
time docker run --rm cow-test sh -c 'echo "x" >> /largefile'
# real: 2.5초  ← 512MB 전체 복사!

# 시나리오 3: 새 파일 생성 (빠름)
time docker run --rm cow-test sh -c 'echo "new" > /newfile'
# real: 0.1초  ← 작은 파일만 생성
```

---

## 🔥 실전 적용

### 시나리오 1: Python 애플리케이션 최적화

**Before: 비효율적**
```dockerfile
FROM python:3.11

WORKDIR /app

# 소스 코드 전체 복사
COPY . .                    # ← 소스 변경 시마다 캐시 MISS

# 의존성 설치
RUN pip install -r requirements.txt  # ← 항상 재실행

CMD ["python", "app.py"]

# 빌드 시간: 3분 (매번)
```

**After: 최적화**
```dockerfile
FROM python:3.11-slim       # slim 버전 사용 (크기 감소)

WORKDIR /app

# 의존성 파일만 먼저 복사
COPY requirements.txt .     # ← 이것만 변경 시 아래만 재실행

# 의존성 설치
RUN pip install --no-cache-dir -r requirements.txt  # ← 캐시 HIT 가능

# 소스 코드 복사
COPY . .                    # ← 소스 변경해도 pip install 건너뜀

CMD ["python", "app.py"]

# 빌드 시간: 10초 (소스 변경 시)
```

**결과:**
- 빌드 시간: 3분 → 10초 (18배 빠름)
- 이미지 크기: 1GB → 200MB (5배 감소)

### 시나리오 2: Node.js 멀티 스테이지 빌드

```dockerfile
# Stage 1: 빌드 환경
FROM node:18-alpine AS builder

WORKDIR /app

# 의존성 파일 복사 및 설치
COPY package*.json ./
RUN npm ci --only=production

# 소스 코드 복사 및 빌드
COPY . .
RUN npm run build

# Stage 2: 실행 환경 (작음!)
FROM node:18-alpine

WORKDIR /app

# 빌드된 파일만 복사
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

USER node
CMD ["node", "dist/index.js"]

# 결과:
# Stage 1: 500MB (빌드 도구 포함)
# Stage 2: 150MB (실행 파일만)
# → 최종 이미지: 150MB
```

### 시나리오 3: 레이어 압축 전략

**문제: 불필요한 파일이 레이어에 남음**
```dockerfile
# ❌ 나쁜 예
FROM ubuntu:22.04

# 패키지 설치
RUN apt-get update                      # Layer 1: 50MB
RUN apt-get install -y build-essential  # Layer 2: 300MB
RUN apt-get clean                       # Layer 3: 0MB (효과 없음!)

# 문제: apt 캐시가 Layer 2에 남아있음
# → 이미지 크기: 350MB
```

**해결: 레이어 합치기**
```dockerfile
# ✅ 좋은 예
FROM ubuntu:22.04

# 한 레이어에서 설치 및 정리
RUN apt-get update && \
    apt-get install -y build-essential && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*         # ← 같은 레이어에서 삭제!

# → 이미지 크기: 250MB (100MB 절약)
```

### 시나리오 4: 개발 vs 프로덕션 이미지

**개발 환경: 빠른 반복**
```dockerfile
FROM node:18

WORKDIR /app

# 모든 의존성 설치 (dev 포함)
COPY package*.json ./
RUN npm install

# 소스 코드 (자주 변경)
COPY . .

# Hot reload 지원
CMD ["npm", "run", "dev"]
```

**프로덕션: 최적화**
```dockerfile
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
USER node
CMD ["node", "dist/index.js"]
```

---

## ⚡ 고급 기법

### 1. BuildKit 캐시 마운트

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine

WORKDIR /app

# BuildKit 캐시 마운트 사용
RUN --mount=type=cache,target=/root/.npm \
    npm install

# 장점: npm 캐시를 빌드 간 유지
# → 의존성 다운로드 건너뜀
```

### 2. 외부 캐시 사용

```bash
# 캐시를 레지스트리에 저장
docker buildx build \
  --cache-to type=registry,ref=myregistry/cache:latest \
  --cache-from type=registry,ref=myregistry/cache:latest \
  -t myapp .

# CI/CD에서 캐시 재사용
# → 클린 빌드 환경에서도 캐시 활용
```

### 3. 병렬 레이어 빌드

```dockerfile
FROM alpine AS base
RUN apk add --no-cache curl

# 병렬 스테이지
FROM base AS stage1
RUN curl -O https://example.com/file1

FROM base AS stage2
RUN curl -O https://example.com/file2

# 최종 스테이지
FROM base
COPY --from=stage1 /file1 .
COPY --from=stage2 /file2 .

# stage1과 stage2가 병렬 실행!
```

---

## 🚫 안티패턴

### 1. 레이어 폭발

```dockerfile
# ❌ 나쁜 예: 불필요하게 많은 레이어
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get install -y git
RUN apt-get install -y wget
# → 5개 레이어, 느린 빌드

# ✅ 좋은 예: 레이어 최소화
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    git \
    wget \
  && rm -rf /var/lib/apt/lists/*
# → 1개 레이어, 빠른 빌드
```

### 2. 캐시 무시 패턴

```dockerfile
# ❌ 나쁜 예: 타임스탬프로 캐시 무효화
FROM alpine
RUN apk add --no-cache curl
RUN date > /timestamp.txt  # ← 매번 다른 값 → 캐시 항상 MISS
RUN echo "Done"            # ← 항상 재실행

# ✅ 좋은 예: 캐시 활용
FROM alpine
RUN apk add --no-cache curl
RUN echo "Done"
# 타임스탬프는 런타임에 생성
```

### 3. 비밀 정보 레이어에 포함

```dockerfile
# ❌ 매우 위험: 비밀 정보가 레이어에 남음
FROM alpine
COPY secret.key /tmp/
RUN use-secret /tmp/secret.key
RUN rm /tmp/secret.key    # ← 삭제해도 레이어에는 남아있음!

# ✅ 안전: BuildKit secrets 사용
FROM alpine
RUN --mount=type=secret,id=mysecret \
    use-secret /run/secrets/mysecret
# 빌드 중에만 마운트, 레이어에 포함 안 됨
```

### 4. 큰 파일 수정

```dockerfile
# ❌ 비효율: 큰 파일을 복사 후 수정
FROM ubuntu
COPY large-db.sql /data/
RUN sed -i 's/old/new/g' /data/large-db.sql
# → large-db.sql이 두 레이어에 중복 저장

# ✅ 효율: 수정 후 복사 or 빌드 시 처리
FROM ubuntu
COPY large-db.sql /tmp/
RUN sed 's/old/new/g' /tmp/large-db.sql > /data/large-db.sql
# → 최종 파일만 레이어에 저장
```

---

## 📊 레이어 최적화 체크리스트

```
✅ 베이스 이미지
   ├─ alpine 또는 slim 버전 사용
   └─ 공식 이미지 사용

✅ 레이어 순서
   ├─ 자주 변경되는 것은 뒤로
   ├─ 의존성 파일 먼저 복사
   └─ 소스 코드는 나중에 복사

✅ 레이어 합치기
   ├─ RUN 명령어 && 로 연결
   └─ 같은 레이어에서 설치 및 정리

✅ 불필요한 파일 제거
   ├─ apt/yum 캐시 삭제
   ├─ 임시 파일 삭제
   └─ 빌드 도구 제거 (멀티 스테이지)

✅ .dockerignore 사용
   ├─ node_modules, .git 제외
   └─ 불필요한 파일 복사 방지

✅ 멀티 스테이지 빌드
   ├─ 빌드 환경 분리
   └─ 최종 이미지 최소화
```

---

## 🎓 핵심 정리

### 레이어 시스템

```
1. 이미지 = 레이어 스택
   ├─ 각 레이어는 불변
   ├─ 위에서 아래로 쌓임
   └─ 레이어 공유로 효율성

2. 컨테이너 = 이미지 + R/W 레이어
   ├─ 이미지 레이어: 읽기 전용
   └─ 컨테이너 레이어: 읽기/쓰기

3. Copy-on-Write
   ├─ 수정 시 레이어로 복사
   ├─ 원본 레이어 보존
   └─ 큰 파일 수정 주의
```

### 최적화 전략

```
1. 캐시 활용
   ├─ 의존성 먼저 설치
   ├─ 소스 코드는 나중에
   └─ 캐시 무효화 최소화

2. 레이어 최소화
   ├─ RUN 명령어 합치기
   ├─ 멀티 스테이지 빌드
   └─ 불필요한 파일 제거

3. 이미지 크기 감소
   ├─ alpine/slim 베이스
   ├─ 빌드 도구 제거
   └─ 압축 및 정리
```

---

## 🔗 다음 단계

레이어 시스템을 마스터했습니다! 다음 챕터:

- **[04. Union Filesystem](./04-Union-Filesystem.md)**: OverlayFS 동작 원리
- **[images/01-Dockerfile-Best-Practices](../images/01-Dockerfile-Best-Practices.md)**: Dockerfile 최적화
- **[images/03-Image-Optimization](../images/03-Image-Optimization.md)**: 이미지 최적화 심화

---

## 📚 참고 자료

- [Docker Image Specification](https://github.com/moby/moby/blob/master/image/spec/v1.2.md)
- [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [BuildKit Documentation](https://github.com/moby/buildkit)
- [Docker Layer Caching](https://docs.docker.com/build/cache/)

---

**🤔 생각해볼 문제**

1. `RUN apt-get update && apt-get install` vs `RUN apt-get update` + `RUN apt-get install` 중 어느 것이 나을까?
2. `.dockerignore`에 `node_modules`를 추가하면 왜 빌드가 빨라질까?
3. 멀티 스테이지 빌드에서 중간 스테이지는 최종 이미지 크기에 영향을 줄까?

> 💡 **답변**: 1) 전자 (캐시 문제 방지), 2) 불필요한 파일 복사 방지 → 빌드 컨텍스트 감소, 3) 아니오 (최종 스테이지만 포함)
