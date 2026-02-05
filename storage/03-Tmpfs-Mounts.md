# 03. Tmpfs Mounts - 메모리 기반 스토리지

## 🎯 이 챕터에서 배울 것

- **Tmpfs Mount**의 개념과 동작 원리
- 메모리 기반 스토리지의 **장단점**
- **임시 데이터** 처리 전략
- 실전 **사용 시나리오**와 성능 최적화

## 📌 왜 중요한가?

**"일부 데이터는 디스크에 쓰지 않는 것이 더 안전하고 빠릅니다."**

```
디스크 기반 (Volume/Bind Mount):
┌─────────────────────────────────┐
│ Container                       │
│ /app/data                       │
└────────┬────────────────────────┘
         │ I/O (느림)
┌────────▼────────────────────────┐
│ Disk Storage                    │
│ - HDD/SSD                       │
│ - 영속적                          │
│ - 느림 (상대적)                    │
│ - 보안 위험 (파일 복구 가능)          │
└─────────────────────────────────┘

메모리 기반 (Tmpfs):
┌─────────────────────────────────┐
│ Container                       │
│ /app/cache                      │
└────────┬────────────────────────┘
         │ 직접 접근 (빠름)
┌────────▼────────────────────────┐
│ RAM (Memory)                    │
│ - 초고속                          │
│ - 휘발성 (재시작 시 소실)            │
│ - 보안 (메모리만 사용)              │
│ - 스왑 방지 가능                   │
└─────────────────────────────────┘

성능 비교:
┌──────────────┬──────────┬──────────┐
│ 작업          │ Disk     │ Tmpfs    │
├──────────────┼──────────┼──────────┤
│ 읽기          │ 100 MB/s │ 5 GB/s   │
├──────────────┼──────────┼──────────┤
│ 쓰기          │ 80 MB/s  │ 4 GB/s   │
├──────────────┼──────────┼──────────┤
│ IOPS         │ 1000     │ 100000+  │
└──────────────┴──────────┴──────────┘
```

**실무 영향:**
- 성능: 디스크 I/O 병목 제거
- 보안: 민감 데이터 메모리에만 보관
- 임시 데이터: 세션, 캐시, 빌드 파일
- 리소스: 메모리 사용량 증가

---

## 🔬 Deep Dive

### 1. Tmpfs Mount 개념

#### 기본 구조

```
Tmpfs (Temporary File System):
- Linux 커널의 메모리 기반 파일시스템
- RAM을 파일시스템처럼 사용
- 휘발성 (재시작 시 소실)
- 컨테이너 종료 시 자동 삭제

구조:
┌──────────────────────────────────┐
│ Container                        │
│ ┌──────────────────────────────┐ │
│ │ /tmp                         │ │
│ │ /run                         │ │
│ │ /app/cache (tmpfs)           │ │
│ └────────────┬─────────────────┘ │
└──────────────┼───────────────────┘
               │ 직접 매핑
┌──────────────▼───────────────────┐
│ Host Memory (RAM)                │
│ ┌──────────────────────────────┐ │
│ │ Tmpfs Area                   │ │
│ │ - 빠른 접근                    │ │
│ │ - 크기 제한 가능                │ │
│ │ - 스왑 방지 옵션                │ │
│ └──────────────────────────────┘ │
└──────────────────────────────────┘

특징:
- 디스크 I/O 없음
- 매우 빠른 읽기/쓰기
- 메모리 부족 시 스왑 가능 (옵션)
- 컨테이너 독립적 (공유 불가)
```

#### 기본 사용법

```bash
# --tmpfs 플래그 (간단)
docker run --tmpfs /app/cache image

# --mount 플래그 (권장, 상세 옵션)
docker run --mount type=tmpfs,target=/app/cache image

# 옵션 지정
docker run --mount type=tmpfs,target=/app/cache,tmpfs-size=100m,tmpfs-mode=1777 image
```

---

### 2. 기본 사용

#### 단순 Tmpfs Mount

```bash
# Tmpfs 마운트
docker run -d --name test \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  alpine sleep 3600

# 마운트 확인
docker exec test df -h /tmp
# Filesystem      Size  Used Avail Use% Mounted on
# tmpfs           100M     0  100M   0% /tmp

# 파일 생성
docker exec test sh -c 'echo "test data" > /tmp/test.txt'

# 확인
docker exec test cat /tmp/test.txt
# test data

# 컨테이너 재시작
docker restart test

# 데이터 소실 확인
docker exec test ls /tmp
# (비어있음)
# ✅ 휘발성 확인!

# 정리
docker rm -f test
```

#### --mount 구문 (상세 옵션)

```bash
# 크기, 권한, 옵션 지정
docker run -d --name test \
  --mount type=tmpfs,target=/app/cache,tmpfs-size=256m,tmpfs-mode=1755 \
  alpine sleep 3600

# 확인
docker exec test mount | grep tmpfs
# tmpfs on /app/cache type tmpfs (rw,nosuid,nodev,noexec,size=262144k,mode=1755)

# 옵션 설명:
# - tmpfs-size: 최대 크기 (256MB)
# - tmpfs-mode: 권한 (0755 = rwxr-xr-x)
# - rw: 읽기/쓰기
# - nosuid: SUID 비트 무시 (보안)
# - nodev: 디바이스 파일 금지 (보안)
# - noexec: 실행 금지 (보안)

docker rm -f test
```

---

### 3. 성능 비교

#### 디스크 vs 메모리 벤치마크

```bash
# 준비
mkdir bench-test
cd bench-test

# 1. 디스크 기반 (Bind Mount)
docker run -d --name disk-test \
  -v $(pwd)/disk-data:/data \
  alpine sleep 3600

# 쓰기 성능 (1GB)
time docker exec disk-test sh -c \
  'dd if=/dev/zero of=/data/testfile bs=1M count=1024'
# 1024+0 records in
# 1024+0 records out
# real    0m8.234s  ← 디스크

# 읽기 성능
time docker exec disk-test sh -c \
  'dd if=/data/testfile of=/dev/null bs=1M'
# real    0m2.156s

# 2. 메모리 기반 (Tmpfs)
docker run -d --name tmpfs-test \
  --tmpfs /data:size=2g \
  alpine sleep 3600

# 쓰기 성능 (1GB)
time docker exec tmpfs-test sh -c \
  'dd if=/dev/zero of=/data/testfile bs=1M count=1024'
# 1024+0 records in
# 1024+0 records out
# real    0m0.324s  ← 메모리 (25배 빠름!)

# 읽기 성능
time docker exec tmpfs-test sh -c \
  'dd if=/data/testfile of=/dev/null bs=1M'
# real    0m0.156s  (14배 빠름!)

# 정리
docker rm -f disk-test tmpfs-test
cd ..
rm -rf bench-test
```

#### IOPS 비교

```bash
# fio 벤치마크 도구 사용

# 디스크
docker run --rm \
  -v $(pwd)/disk-data:/data \
  ljishen/fio \
  fio --name=randwrite --ioengine=libaio --iodepth=16 \
      --rw=randwrite --bs=4k --direct=1 --size=1G \
      --numjobs=4 --runtime=30 --group_reporting \
      --directory=/data

# 결과:
# IOPS=2,345

# Tmpfs
docker run --rm \
  --tmpfs /data:size=2g \
  ljishen/fio \
  fio --name=randwrite --ioengine=libaio --iodepth=16 \
      --rw=randwrite --bs=4k --direct=1 --size=1G \
      --numjobs=4 --runtime=30 --group_reporting \
      --directory=/data

# 결과:
# IOPS=156,789  (67배 빠름!)
```

---

### 4. 사용 시나리오

#### 세션 스토리지

```bash
# Redis 세션 스토어 (메모리 최적화)
docker run -d --name redis-session \
  --tmpfs /data:size=512m \
  redis:alpine redis-server --dir /data --save ""

# --save "": 디스크 저장 비활성화
# 순수 메모리 캐시로 사용

# 성능 테스트
docker exec redis-session redis-benchmark -t set,get -n 100000 -q
# SET: 125,000 requests per second
# GET: 142,857 requests per second
```

#### 빌드 캐시

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

# 의존성 설치
COPY package*.json ./
RUN npm ci

# 소스 복사
COPY . .

# 빌드 (tmpfs 사용)
RUN --mount=type=tmpfs,target=/app/.next \
    npm run build

# 실행
CMD ["npm", "start"]
```

```bash
# 빌드 시 .next 디렉토리를 tmpfs로
# 빌드 속도 향상 + 디스크 I/O 감소
docker build -t myapp .
```

#### 임시 파일 처리

```bash
# 이미지 리사이징 서비스
docker run -d --name image-processor \
  --tmpfs /tmp:size=1g,mode=1777 \
  myapp/image-processor

# 처리 흐름:
# 1. 업로드된 이미지 → /tmp에 저장
# 2. 리사이징 처리
# 3. S3/CDN 업로드
# 4. /tmp 자동 정리 (컨테이너 재시작 시)
```

---

### 5. 보안 고려사항

#### 민감 데이터 보호

```bash
# 시크릿 처리 (메모리에만 보관)
docker run -d --name secure-app \
  --tmpfs /run/secrets:ro,mode=0400 \
  myapp

# 애플리케이션에서:
# 1. 시크릿 로드 → /run/secrets에 저장
# 2. 사용
# 3. 컨테이너 종료 → 메모리에서 완전 삭제
# 디스크에 절대 쓰여지지 않음!
```

#### 스왑 방지

```bash
# 스왑 비활성화 (보안 강화)
docker run -d --name no-swap \
  --tmpfs /secure-data:size=256m \
  --memory=512m \
  --memory-swap=512m \
  alpine sleep 3600

# --memory-swap=512m: 스왑 사용 금지
# 메모리 초과 시 OOM killer 발동
# 민감 데이터가 스왑(디스크)에 기록되지 않음
```

---

## 💻 실습

### 실습 1: 성능 비교 실습

#### 준비

```bash
mkdir perf-test
cd perf-test

# 테스트 스크립트
cat > benchmark.sh << 'EOF'
#!/bin/sh

echo "=== Write Test ==="
time sh -c 'for i in $(seq 1 1000); do
  echo "test data $i" > /data/file_$i.txt
done'

echo ""
echo "=== Read Test ==="
time sh -c 'for i in $(seq 1 1000); do
  cat /data/file_$i.txt > /dev/null
done'

echo ""
echo "=== Delete Test ==="
time sh -c 'rm -f /data/file_*.txt'
EOF

chmod +x benchmark.sh
```

#### 디스크 테스트

```bash
# Volume 사용
docker volume create perf-vol

docker run --rm \
  -v perf-vol:/data \
  -v $(pwd)/benchmark.sh:/benchmark.sh \
  alpine /benchmark.sh

# 결과:
# === Write Test ===
# real    0m 2.134s
# === Read Test ===
# real    0m 1.234s
# === Delete Test ===
# real    0m 0.456s

docker volume rm perf-vol
```

#### 메모리 테스트

```bash
# Tmpfs 사용
docker run --rm \
  --tmpfs /data:size=100m \
  -v $(pwd)/benchmark.sh:/benchmark.sh \
  alpine /benchmark.sh

# 결과:
# === Write Test ===
# real    0m 0.234s  (9배 빠름!)
# === Read Test ===
# real    0m 0.123s  (10배 빠름!)
# === Delete Test ===
# real    0m 0.045s  (10배 빠름!)

cd ..
rm -rf perf-test
```

---

### 실습 2: Redis 캐시 최적화

#### 설정 비교

**디스크 기반 (기본):**

```bash
# RDB 저장 활성화
docker run -d --name redis-disk \
  -v redis-data:/data \
  redis:alpine redis-server --save 60 1 --dir /data

# 성능 테스트
docker exec redis-disk redis-benchmark -t set,get -n 100000 -q
# SET: 89,285 requests per second
# GET: 98,039 requests per second

docker rm -f redis-disk
docker volume rm redis-data
```

**메모리 기반 (Tmpfs):**

```bash
# RDB 저장 비활성화 + tmpfs
docker run -d --name redis-tmpfs \
  --tmpfs /data:size=256m \
  redis:alpine redis-server --save "" --dir /data

# 성능 테스트
docker exec redis-tmpfs redis-benchmark -t set,get -n 100000 -q
# SET: 142,857 requests per second  (60% 빠름!)
# GET: 166,666 requests per second  (70% 빠름!)

# 메모리 사용량 확인
docker stats redis-tmpfs --no-stream
# NAME          MEM USAGE
# redis-tmpfs   15.23MiB / 256MiB

docker rm -f redis-tmpfs
```

---

### 실습 3: 세션 관리

#### Express 세션 서버

```bash
# 프로젝트 생성
mkdir session-test
cd session-test

# package.json
cat > package.json << 'EOF'
{
  "name": "session-test",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.0",
    "express-session": "^1.17.0",
    "connect-redis": "^6.1.0",
    "redis": "^4.0.0"
  }
}
EOF

# app.js
cat > app.js << 'EOF'
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const app = express();

const redisClient = createClient({
  url: 'redis://redis:6379'
});
redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'secret',
  resave: false,
  saveUninitialized: false,
  cookie: { maxAge: 3600000 }
}));

app.get('/', (req, res) => {
  if (req.session.views) {
    req.session.views++;
  } else {
    req.session.views = 1;
  }
  res.json({
    views: req.session.views,
    sessionID: req.sessionID
  });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

# Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install
COPY app.js .
CMD ["node", "app.js"]
EOF

# docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - redis

  redis:
    image: redis:alpine
    # Tmpfs로 세션 데이터 메모리 저장
    tmpfs:
      - /data:size=512m
    command: redis-server --save ""

volumes:
  session-data:
EOF

# 실행
docker-compose build
docker-compose up -d

# 테스트
curl http://localhost:3000
# {"views":1,"sessionID":"..."}

curl http://localhost:3000
# {"views":2,"sessionID":"..."}

# 서비스 재시작 (세션 유지 안 됨)
docker-compose restart redis

curl http://localhost:3000
# {"views":1,"sessionID":"..."}
# ✅ 새 세션 (tmpfs 휘발성)

# 정리
docker-compose down
cd ..
rm -rf session-test
```

---

### 실습 4: 빌드 최적화

#### Next.js 빌드

```bash
mkdir nextjs-build
cd nextjs-build

# Dockerfile (tmpfs 사용)
cat > Dockerfile << 'EOF'
FROM node:18-alpine AS builder

WORKDIR /app

# 의존성
COPY package*.json ./
RUN npm ci

# 소스
COPY . .

# 빌드 (tmpfs로 .next 디렉토리)
RUN --mount=type=tmpfs,target=/app/.next \
    npm run build && \
    cp -r .next /tmp/.next

# 프로덕션
FROM node:18-alpine

WORKDIR /app

COPY --from=builder /app/package*.json ./
RUN npm ci --only=production

COPY --from=builder /tmp/.next ./.next
COPY --from=builder /app/public ./public

CMD ["npm", "start"]
EOF

# 비교: 일반 빌드 vs tmpfs 빌드
# 일반: 45초, 디스크 I/O 많음
# tmpfs: 28초, 메모리만 사용 (37% 빠름!)

cd ..
rm -rf nextjs-build
```

---

## 🔥 실전 적용

### 시나리오 1: 고성능 API 서버

**구조:**

```
┌─────────────────────────────────────┐
│ API Server                          │
│ ┌─────────────────────────────────┐ │
│ │ /app/cache (tmpfs)              │ │
│ │ - API 응답 캐시                   │ │
│ │ - 1분 TTL                       │ │
│ └─────────────────────────────────┘ │
│ ┌─────────────────────────────────┐ │
│ │ /tmp (tmpfs)                    │ │
│ │ - 임시 파일 처리                   │ │
│ │ - 업로드 임시 저장                  │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  api:
    build: ./api
    ports:
      - "8080:8080"
    tmpfs:
      # 캐시 (읽기/쓰기)
      - /app/cache:size=512m,mode=1777
      
      # 임시 파일
      - /tmp:size=1g,mode=1777
    environment:
      - CACHE_DIR=/app/cache
      - TMP_DIR=/tmp
      - NODE_ENV=production
    mem_limit: 2g
    mem_reservation: 1g

  # Redis (순수 메모리)
  redis:
    image: redis:alpine
    tmpfs:
      - /data:size=512m
    command: redis-server --save "" --maxmemory 256mb --maxmemory-policy allkeys-lru
    mem_limit: 512m
```

**성능 향상:**

```
Before (디스크 캐시):
- 평균 응답: 45ms
- 캐시 히트: 70%
- 디스크 I/O: 높음

After (tmpfs 캐시):
- 평균 응답: 12ms (73% 빠름!)
- 캐시 히트: 70%
- 디스크 I/O: 없음
```

---

### 시나리오 2: 이미지 처리 파이프라인

**요구사항:**
- 대량 이미지 업로드
- 리사이징/최적화
- S3 업로드
- 임시 파일 자동 정리

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  # 이미지 프로세서
  processor:
    build: ./processor
    tmpfs:
      # 업로드 임시 저장
      - /tmp/uploads:size=2g,mode=1777
      
      # 처리 중 파일
      - /tmp/processing:size=4g,mode=1777
      
      # 최적화 결과
      - /tmp/output:size=2g,mode=1777
    environment:
      - UPLOAD_DIR=/tmp/uploads
      - PROCESSING_DIR=/tmp/processing
      - OUTPUT_DIR=/tmp/output
      - S3_BUCKET=my-images
    mem_limit: 8g
    mem_reservation: 4g

  # 작업 큐 (Redis)
  queue:
    image: redis:alpine
    tmpfs:
      - /data:size=512m
    command: redis-server --save ""

  # 워커 (여러 개)
  worker:
    build: ./worker
    deploy:
      replicas: 4
    tmpfs:
      - /tmp:size=1g,mode=1777
    environment:
      - REDIS_URL=redis://queue:6379
    depends_on:
      - queue
```

**워크플로우:**

```python
# processor/app.py
import os
from PIL import Image

UPLOAD_DIR = os.getenv('UPLOAD_DIR')
PROCESSING_DIR = os.getenv('PROCESSING_DIR')
OUTPUT_DIR = os.getenv('OUTPUT_DIR')

def process_image(file):
    # 1. 업로드 → /tmp/uploads (tmpfs)
    upload_path = f"{UPLOAD_DIR}/{file.filename}"
    file.save(upload_path)
    
    # 2. 처리 → /tmp/processing (tmpfs)
    processing_path = f"{PROCESSING_DIR}/{file.filename}"
    img = Image.open(upload_path)
    img = img.resize((800, 600))
    img.save(processing_path, optimize=True)
    
    # 3. 최적화 → /tmp/output (tmpfs)
    output_path = f"{OUTPUT_DIR}/{file.filename}"
    # ... 추가 최적화
    
    # 4. S3 업로드
    s3_upload(output_path)
    
    # 5. 임시 파일 삭제 (선택적, 컨테이너 재시작 시 자동 삭제)
    os.remove(upload_path)
    os.remove(processing_path)
    os.remove(output_path)
```

**장점:**
```
- 디스크 I/O 제로 (SSD 수명 보호)
- 처리 속도 3-5배 향상
- 자동 정리 (컨테이너 재시작)
- 스케일 아웃 용이
```

---

### 시나리오 3: CI/CD 빌드 에이전트

**요구사항:**
- 빠른 빌드
- 동시 여러 작업
- 임시 파일 자동 정리
- 보안 (빌드 아티팩트 보호)

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  # Jenkins 빌드 에이전트
  jenkins-agent:
    image: jenkins/inbound-agent
    tmpfs:
      # 작업 디렉토리
      - /home/jenkins/agent:size=10g,mode=1777
      
      # npm/pip 캐시
      - /tmp/cache:size=5g,mode=1777
      
      # 빌드 아티팩트
      - /tmp/artifacts:size=5g,mode=1777
    environment:
      - JENKINS_URL=http://jenkins:8080
      - JENKINS_SECRET=${JENKINS_SECRET}
      - JENKINS_AGENT_NAME=agent-tmpfs
      - NPM_CONFIG_CACHE=/tmp/cache/npm
      - PIP_CACHE_DIR=/tmp/cache/pip
    mem_limit: 16g
    volumes:
      # Docker socket (Docker-in-Docker)
      - /var/run/docker.sock:/var/run/docker.sock
      
      # 최종 아티팩트만 영구 저장
      - artifacts:/artifacts

  # GitLab Runner
  gitlab-runner:
    image: gitlab/gitlab-runner:alpine
    tmpfs:
      # 빌드 디렉토리
      - /builds:size=10g,mode=1777
      
      # 캐시
      - /cache:size=5g,mode=1777
    environment:
      - RUNNER_BUILDS_DIR=/builds
      - RUNNER_CACHE_DIR=/cache
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - runner-config:/etc/gitlab-runner

volumes:
  artifacts:
  runner-config:
```

**성능 개선:**

```
Before (디스크):
- 빌드 시간: 8분
- npm install: 2분
- 테스트: 3분
- 빌드: 3분

After (tmpfs):
- 빌드 시간: 4분 (50% 빠름!)
- npm install: 45초
- 테스트: 1분 30초
- 빌드: 1분 45초
```

---

## ⚡ Tmpfs 사용 가이드

### 적합한 케이스

```
✅ 세션 데이터
✅ 캐시 (TTL 짧음)
✅ 임시 파일 처리
✅ 빌드 아티팩트
✅ 로그 버퍼
✅ 민감 데이터 (보안)
✅ 테스트 데이터
```

### 부적합한 케이스

```
❌ 영속적 데이터
❌ 데이터베이스 파일
❌ 사용자 업로드 (장기)
❌ 백업 필요한 데이터
❌ 대용량 파일 (메모리 부족)
❌ 공유 필요한 데이터
```

### 크기 계산

```bash
# 메모리 사용 패턴 분석
docker stats --no-stream

# Tmpfs 크기 권장:
# - 일반: 총 메모리의 10-20%
# - 캐시: 총 메모리의 20-30%
# - 빌드: 총 메모리의 30-50%

# 예시: 8GB 메모리 시스템
# - 세션: 512MB
# - 캐시: 1GB
# - 빌드: 2GB
# - 여유: 4.5GB
```

---

## 🚫 안티패턴

### 1. 과도한 Tmpfs 크기

```yaml
# ❌ 메모리 초과
services:
  app:
    image: myapp
    tmpfs:
      - /tmp:size=10g  # 시스템 메모리 8GB인데!
    mem_limit: 2g
# OOM Killer 발동

# ✅ 적절한 크기
services:
  app:
    image: myapp
    tmpfs:
      - /tmp:size=512m  # 메모리의 10%
    mem_limit: 2g
```

### 2. 영속 데이터에 Tmpfs

```yaml
# ❌ 데이터베이스 파일
services:
  db:
    image: postgres
    tmpfs:
      - /var/lib/postgresql/data  # 재시작 시 소실!

# ✅ Volume 사용
services:
  db:
    image: postgres
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

### 3. 스왑 무시

```bash
# ❌ 스왑 고려 안 함
docker run --tmpfs /tmp:size=8g myapp
# 메모리 부족 시 스왑 사용 → 디스크 쓰임!

# ✅ 스왑 방지
docker run \
  --tmpfs /tmp:size=2g \
  --memory=4g \
  --memory-swap=4g \
  myapp
# 메모리 초과 시 OOM (스왑 안 함)
```

### 4. 크기 제한 미설정

```bash
# ❌ 크기 무제한
docker run --tmpfs /tmp myapp
# 메모리 고갈 위험!

# ✅ 명시적 크기
docker run --tmpfs /tmp:size=512m myapp
```

---

## 🎓 핵심 정리

### 1. Tmpfs 특징

```
장점:
+ 초고속 (RAM 속도)
+ 디스크 I/O 제거
+ 보안 (메모리만 사용)
+ 자동 정리

단점:
- 휘발성 (재시작 시 소실)
- 메모리 사용
- 크기 제한
- 공유 불가
```

### 2. 성능

```
Tmpfs vs Disk:
- 읽기: 10-50배 빠름
- 쓰기: 10-50배 빠름
- IOPS: 50-100배 많음
- 지연시간: 1/100
```

### 3. 사용 패턴

```
세션:
- Redis + tmpfs
- 짧은 TTL
- 재시작해도 무방

캐시:
- 애플리케이션 캐시
- 빌드 캐시
- 임시 결과

보안:
- 시크릿 임시 저장
- 스왑 방지
- 메모리만 사용
```

### 4. 핵심 명령어

```bash
# Tmpfs 마운트
docker run --tmpfs /path:size=<s>,mode=<m>

# --mount 구문
docker run --mount type=tmpfs,target=/path,tmpfs-size=<s>

# Compose
services:
  app:
    tmpfs:
      - /path:size=<s>,mode=<m>

# 확인
docker exec <c> df -h
docker exec <c> mount | grep tmpfs
```

---

## 📚 참고 자료

- [Docker Tmpfs Mounts](https://docs.docker.com/storage/tmpfs/)
- [Linux Tmpfs](https://www.kernel.org/doc/html/latest/filesystems/tmpfs.html)
- [Memory Management](https://docs.docker.com/config/containers/resource_constraints/)

---

## 🤔 생각해볼 문제

1. Tmpfs가 메모리 부족 시 스왑을 사용하면 보안상 문제는?
2. 여러 컨테이너가 같은 tmpfs 경로를 사용하면 공유될까?
3. Tmpfs 크기를 시스템 메모리보다 크게 설정하면?

> 💡 **답변**: 1) 민감 데이터가 디스크(스왑)에 쓰여져 보안 위험 - 스왑 파일은 암호화되지 않음, 메모리 덤프 시 복구 가능, 해결: --memory-swap 플래그로 스왑 제한, cgroup memory.swappiness=0 설정, 중요 데이터는 mlock() 사용, 2) 아니오, 완전히 독립적 - 각 컨테이너는 자신의 tmpfs 인스턴스 가짐, 같은 경로여도 다른 메모리 영역, 공유 필요 시 Volume이나 IPC 네임스페이스 사용, 3) 설정은 가능하지만 매우 위험 - 실제 사용 시 메모리 부족으로 OOM Killer 발동, 컨테이너 강제 종료, 시스템 불안정, 권장: 총 메모리의 50% 이하, 여유 메모리 확보 필수

---

<div align="center">

**[⬅️ 이전: Bind Mounts](./02-Bind-Mounts.md)** | **[다음: Volume Drivers ➡️](./04-Volume-Drivers.md)**

</div>
