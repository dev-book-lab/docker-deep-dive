# 07. Custom Base Images - 맞춤형 베이스 이미지

## 🎯 이 챕터에서 배울 것

- **scratch부터** 완전히 새로운 이미지 만들기
- **맞춤형 베이스 이미지** 설계 및 제작
- **공통 레이어**로 조직 전체 최적화
- **베이스 이미지 관리** 전략

## 📌 왜 중요한가?

**"맞춤형 베이스 이미지로 조직 전체의 효율을 10배 높일 수 있습니다."**

```
공용 베이스 이미지 문제:
- 불필요한 패키지 많음
- 보안 취약점 상속
- 일관성 없는 설정
- 중복 레이어

맞춤형 베이스 이미지:
- 필요한 것만 포함
- 사전 취약점 제거
- 표준화된 설정
- 공통 레이어 공유
```

**실무 영향:**
- 효율: 10개 서비스 공통 레이어 공유 → 90% 스토리지 절감
- 보안: 베이스 한 곳만 패치 → 전체 서비스 보안 강화
- 일관성: 표준화된 설정 → 운영 복잡도 감소
- 속도: 공유 레이어 → 배포 시간 대폭 단축

---

## 🔬 Deep Dive

### 1. scratch 이미지란?

#### scratch의 특징

```
scratch:
- 완전히 비어있는 이미지
- 실제로는 존재하지 않는 특수 키워드
- 레이어 0개, 크기 0바이트
- FROM scratch = "아무것도 없음"

사용 가능한 경우:
✅ 정적으로 링크된 바이너리
✅ 외부 의존성 없음
✅ 최소 크기 필요
✅ 최고 보안 필요

사용 불가능한 경우:
❌ 동적 링크 라이브러리 필요
❌ 쉘 명령어 필요
❌ 시스템 라이브러리 필요
```

#### scratch 이미지 구조

```
일반 이미지:
┌─────────────────────────┐
│ 애플리케이션               │
├─────────────────────────┤
│ 런타임 라이브러리           │
├─────────────────────────┤
│ 시스템 패키지              │
├─────────────────────────┤
│ 베이스 OS (ubuntu, etc)  │
└─────────────────────────┘
크기: 500MB+

scratch 이미지:
┌─────────────────────────┐
│ 정적 바이너리              │
└─────────────────────────┘
크기: 5-20MB

차이: 25배 작음!
```

---

### 2. scratch로 Go 애플리케이션

#### 완전 정적 빌드

```dockerfile
# syntax=docker/dockerfile:1

# Stage 1: 빌드
FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

# 완전 정적 링크
RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
    -a \
    -installsuffix cgo \
    -ldflags="-w -s -extldflags '-static'" \
    -o server \
    .

# Stage 2: scratch
FROM scratch

# CA 인증서 (HTTPS 통신용)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 타임존 데이터
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# passwd, group 파일 (non-root 실행용)
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group

# 바이너리
COPY --from=builder /app/server /server

# Non-root 사용자
USER nobody:nobody

ENTRYPOINT ["/server"]

# 결과:
# - 크기: 8MB
# - 취약점: 0개
# - 공격 표면: 최소
```

#### 필수 파일 생성

```dockerfile
# scratch에 필요한 파일 직접 생성
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Go 빌드
COPY . .
RUN CGO_ENABLED=0 go build -o server .

# 필수 파일 생성
RUN echo "nobody:x:65534:65534:Nobody:/:" > /etc_passwd && \
    echo "nogroup:x:65534:" > /etc_group

# scratch
FROM scratch

# 생성한 파일 복사
COPY --from=builder /etc_passwd /etc/passwd
COPY --from=builder /etc_group /etc/group
COPY --from=builder /app/server /server

USER nobody:nobody

ENTRYPOINT ["/server"]
```

---

### 3. scratch로 Rust 애플리케이션

#### musl 타겟 정적 빌드

```dockerfile
# syntax=docker/dockerfile:1

FROM rust:1.74-alpine AS builder

# musl 타겟 추가
RUN rustup target add x86_64-unknown-linux-musl

# 빌드 도구
RUN apk add --no-cache musl-dev

WORKDIR /app

COPY Cargo.toml Cargo.lock ./
RUN cargo fetch

COPY src ./src

# 정적 링크로 빌드
RUN cargo build \
    --release \
    --target x86_64-unknown-linux-musl

# scratch
FROM scratch

# 바이너리만 복사
COPY --from=builder \
    /app/target/x86_64-unknown-linux-musl/release/myapp \
    /myapp

USER 1000:1000

ENTRYPOINT ["/myapp"]

# 결과:
# - 크기: 5MB
# - 완전 정적
# - 의존성 0개
```

---

### 4. 맞춤형 베이스 이미지 설계

#### 조직 표준 베이스 생성

```dockerfile
# base-images/python/Dockerfile
FROM python:3.11-slim AS builder

# 공통 시스템 패키지
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        && rm -rf /var/lib/apt/lists/*

# 공통 Python 패키지
RUN pip install --no-cache-dir \
    gunicorn==21.2.0 \
    prometheus-client==0.19.0 \
    structlog==23.2.0

# 표준 디렉토리 구조
RUN mkdir -p /app/logs /app/tmp && \
    useradd -m -u 1000 -s /bin/bash appuser && \
    chown -R appuser:appuser /app

# 표준 설정 파일
COPY logging.conf /app/config/
COPY gunicorn.conf.py /app/config/

# 최종 이미지
FROM python:3.11-slim

COPY --from=builder / /

WORKDIR /app

USER appuser

# 자식 이미지에서 사용할 훅
ONBUILD COPY requirements.txt .
ONBUILD RUN pip install --no-cache-dir -r requirements.txt
ONBUILD COPY . .

CMD ["gunicorn", "-c", "/app/config/gunicorn.conf.py", "app:app"]
```

#### 사용 예시

```dockerfile
# 각 마이크로서비스
FROM mycompany/python-base:3.11

# ONBUILD가 자동 실행:
# - requirements.txt 복사
# - pip install
# - 소스 복사

# 서비스별 설정만 추가
ENV SERVICE_NAME=user-service

# 끝!
```

---

### 5. 계층적 베이스 이미지

#### 3단계 베이스 구조

```
Level 1: OS 베이스
└── mycompany/base-os:ubuntu-22.04
    - 보안 강화
    - 필수 도구
    - 표준 설정

Level 2: 런타임 베이스
├── mycompany/python-base:3.11
│   - Python 런타임
│   - 공통 패키지
│   - 표준 구조
│
├── mycompany/node-base:18
│   - Node.js 런타임
│   - 공통 패키지
│   - 표준 구조
│
└── mycompany/go-base:1.21
    - Go 런타임
    - 공통 도구
    - 표준 구조

Level 3: 서비스 이미지
├── user-service
├── order-service
└── payment-service
```

#### Level 1: OS 베이스

```dockerfile
# base-images/os/Dockerfile
FROM ubuntu:22.04

# 보안 업데이트
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        tzdata \
        && rm -rf /var/lib/apt/lists/*

# 표준 사용자
RUN groupadd -g 1000 appgroup && \
    useradd -u 1000 -g appgroup -m -s /bin/bash appuser

# 타임존 설정
ENV TZ=Asia/Seoul
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && \
    echo $TZ > /etc/timezone

# 표준 디렉토리
RUN mkdir -p /app/logs /app/tmp /app/config && \
    chown -R appuser:appgroup /app

# 레이블
LABEL maintainer="platform-team@mycompany.com"
LABEL version="1.0"
LABEL description="Company standard OS base"

WORKDIR /app
USER appuser
```

#### Level 2: Python 베이스

```dockerfile
# base-images/python/Dockerfile
FROM mycompany/base-os:latest

# Python 설치
USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3.11 \
        python3-pip \
        && rm -rf /var/lib/apt/lists/*

# 공통 Python 패키지
RUN pip3 install --no-cache-dir \
    gunicorn==21.2.0 \
    prometheus-client==0.19.0 \
    structlog==23.2.0 \
    python-json-logger==2.0.7

# 표준 설정
COPY --chown=appuser:appgroup logging.json /app/config/
COPY --chown=appuser:appgroup gunicorn.conf.py /app/config/

USER appuser

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

LABEL python.version="3.11"
```

#### Level 3: 서비스 이미지

```dockerfile
# services/user-service/Dockerfile
FROM mycompany/python-base:3.11

# 서비스별 의존성
COPY requirements.txt .
RUN pip3 install --no-cache-dir -r requirements.txt

# 소스 복사
COPY . .

# 서비스 설정
ENV SERVICE_NAME=user-service
ENV SERVICE_PORT=8000

EXPOSE 8000

CMD ["gunicorn", \
     "-c", "/app/config/gunicorn.conf.py", \
     "-b", "0.0.0.0:8000", \
     "app:create_app()"]
```

---

### 6. 공통 레이어 최적화

#### 문제: 중복 레이어

```
Before: 각 서비스가 독립적으로 빌드

Service A:
┌──────────────────┐
│ Service A code   │ 10MB
├──────────────────┤
│ Dependencies     │ 200MB  ← 중복!
├──────────────────┤
│ Python runtime   │ 300MB  ← 중복!
├──────────────────┤
│ Base OS          │ 100MB  ← 중복!
└──────────────────┘
Total: 610MB

Service B, C, D... 각각 610MB
10개 서비스 = 6.1GB
```

#### 해결: 공통 베이스 사용

```
After: 공통 베이스 이미지 사용

Common Base:
┌──────────────────┐
│ Base OS          │ 100MB  } 한 번만 저장
├──────────────────┤
│ Python runtime   │ 300MB  } (공유 레이어)
├──────────────────┤
│ Dependencies     │ 200MB  }
└──────────────────┘

Service A, B, C, D... 각각:
┌──────────────────┐
│ Service code     │ 10MB   ← 이것만 추가
└──────────────────┘

10개 서비스 = 600MB (공통) + 100MB (각 서비스)
Total: 700MB

절감: 6.1GB → 700MB (88% 감소!)
```

---

### 7. 베이스 이미지 버전 관리

#### Semantic Versioning

```
이미지 태그 전략:

mycompany/python-base:3.11.1.5
                      │  │ │ │
                      │  │ │ └─ 베이스 이미지 패치 (5)
                      │  │ └─── Python 패치 버전 (1)
                      │  └───── Python 마이너 버전 (11)
                      └──────── Python 메이저 버전 (3)

태그 예시:
- latest (불안정, 프로덕션 사용 금지)
- 3.11 (마이너 버전 고정)
- 3.11.1 (패치 버전 고정)
- 3.11.1.5 (완전 고정, 권장)
```

#### 자동 업데이트 파이프라인

```yaml
# .github/workflows/base-image-update.yml
name: Update Base Images

on:
  schedule:
    # 매주 월요일 2am
    - cron: '0 2 * * 1'
  workflow_dispatch:

jobs:
  update-python-base:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build new base
        run: |
          cd base-images/python
          docker build -t mycompany/python-base:3.11.1.${{ github.run_number }} .
      
      - name: Security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: mycompany/python-base:3.11.1.${{ github.run_number }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
      
      - name: Push to registry
        run: |
          docker push mycompany/python-base:3.11.1.${{ github.run_number }}
          docker tag mycompany/python-base:3.11.1.${{ github.run_number }} \
                     mycompany/python-base:3.11
          docker push mycompany/python-base:3.11
      
      - name: Trigger service rebuilds
        run: |
          curl -X POST https://api.github.com/repos/mycompany/services/dispatches \
            -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -d '{"event_type":"base-image-updated"}'
```

---

## 💻 실습

### 실습 1: scratch로 최소 Go 애플리케이션

#### 준비

```bash
mkdir scratch-go-demo
cd scratch-go-demo

cat > main.go << 'EOF'
package main

import (
    "fmt"
    "log"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello from scratch container!")
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

#### Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.21-alpine AS builder

WORKDIR /app

COPY go.mod ./
RUN go mod download

COPY main.go ./

# 완전 정적 빌드
RUN CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    go build \
    -a \
    -installsuffix cgo \
    -ldflags="-w -s" \
    -o server \
    main.go

# scratch
FROM scratch

# CA 인증서
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 타임존
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
ENV TZ=Asia/Seoul

# 사용자 파일 생성
COPY --from=builder /etc/passwd /etc/passwd

# 바이너리
COPY --from=builder /app/server /server

USER nobody:nobody

EXPOSE 8080

ENTRYPOINT ["/server"]
```

#### 빌드 및 테스트

```bash
# 빌드
docker build -t go-scratch:latest .

# 크기 확인
docker images go-scratch:latest
# REPOSITORY    TAG       SIZE
# go-scratch    latest    8MB  ← 매우 작음!

# 실행
docker run -d -p 8080:8080 --name test-scratch go-scratch:latest

# 테스트
curl http://localhost:8080
# Hello from scratch container!

# 내부 확인
docker exec test-scratch ls /
# ls: exec /bin/sh: no such file or directory
# (쉘이 없음! scratch니까)

# 정리
docker stop test-scratch && docker rm test-scratch
```

#### 비교: Alpine vs scratch

```bash
# Alpine 버전
cat > Dockerfile.alpine << 'EOF'
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o server main.go

FROM alpine:3.19
COPY --from=builder /app/server /server
CMD ["/server"]
EOF

docker build -f Dockerfile.alpine -t go-alpine:latest .

# 크기 비교
docker images | grep go-
# go-scratch    latest    8MB
# go-alpine     latest    18MB

# scratch가 55% 작음!
```

---

### 실습 2: 맞춤형 Python 베이스 생성

#### Level 1: OS 베이스

```dockerfile
# base/os/Dockerfile
FROM ubuntu:22.04

# 최소 패키지
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        tzdata \
        && rm -rf /var/lib/apt/lists/*

# 표준 사용자
RUN groupadd -g 1000 appgroup && \
    useradd -u 1000 -g appgroup -m appuser

# 타임존
ENV TZ=Asia/Seoul
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime

# 디렉토리
RUN mkdir -p /app/logs /app/tmp && \
    chown -R appuser:appgroup /app

WORKDIR /app
USER appuser

LABEL org.opencontainers.image.source="https://github.com/mycompany/base-images"
LABEL org.opencontainers.image.description="Company standard OS base"
```

```bash
cd base/os
docker build -t mycompany/base-os:22.04 .
docker push mycompany/base-os:22.04
```

#### Level 2: Python 베이스

```dockerfile
# base/python/Dockerfile
FROM mycompany/base-os:22.04

USER root

# Python 설치
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3.11 \
        python3-pip \
        && rm -rf /var/lib/apt/lists/*

# 공통 패키지
COPY requirements.txt /tmp/
RUN pip3 install --no-cache-dir -r /tmp/requirements.txt && \
    rm /tmp/requirements.txt

# 설정 파일
COPY --chown=appuser:appgroup logging.json /app/config/

USER appuser

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

LABEL python.version="3.11"
```

```bash
# base/python/requirements.txt
gunicorn==21.2.0
prometheus-client==0.19.0
structlog==23.2.0
python-json-logger==2.0.7
```

```bash
cd base/python
docker build -t mycompany/python-base:3.11 .
docker push mycompany/python-base:3.11
```

#### Level 3: 서비스 사용

```dockerfile
# services/api-service/Dockerfile
FROM mycompany/python-base:3.11

# 서비스 의존성
COPY requirements.txt .
RUN pip3 install --no-cache-dir -r requirements.txt

# 소스
COPY . .

ENV SERVICE_NAME=api-service

EXPOSE 8000

CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

---

### 실습 3: 공통 레이어 절감 효과 측정

#### 시나리오 설정

```bash
mkdir layer-optimization-demo
cd layer-optimization-demo

# 3개 마이크로서비스 시뮬레이션
for service in service-a service-b service-c; do
  mkdir -p $service
  
  cat > $service/app.py << EOF
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return '$service'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

  cat > $service/requirements.txt << EOF
Flask==3.0.0
gunicorn==21.2.0
prometheus-client==0.19.0
EOF
done
```

#### Before: 개별 빌드

```dockerfile
# service-a/Dockerfile.separate
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]
```

```bash
# 각 서비스 빌드
for service in service-a service-b service-c; do
  docker build -f $service/Dockerfile.separate -t $service:separate $service
done

# 총 크기 확인
docker images | grep separate | awk '{sum+=$7} END {print "Total: " sum " MB"}'
# Total: 450 MB (3 × 150MB)
```

#### After: 공통 베이스

```dockerfile
# base/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 공통 의존성
RUN pip install --no-cache-dir \
    Flask==3.0.0 \
    gunicorn==21.2.0 \
    prometheus-client==0.19.0
```

```bash
docker build -t mybase:python base/
```

```dockerfile
# service-a/Dockerfile.shared
FROM mybase:python

COPY app.py .

CMD ["gunicorn", "-b", "0.0.0.0:5000", "app:app"]
```

```bash
# 각 서비스 빌드
for service in service-a service-b service-c; do
  docker build -f $service/Dockerfile.shared -t $service:shared $service
done

# 총 크기 확인
docker images | grep -E '(mybase|shared)' | awk '{sum+=$7} END {print "Total: " sum " MB"}'
# Total: 160 MB (150MB base + 3 × 3MB)

# 절감: 450MB → 160MB (64% 감소!)
```

---

### 실습 4: 베이스 이미지 자동 업데이트

#### 베이스 이미지 레포지토리 구조

```
base-images/
├── .github/
│   └── workflows/
│       ├── build-os-base.yml
│       └── build-python-base.yml
├── os/
│   ├── Dockerfile
│   └── README.md
├── python/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── logging.json
└── README.md
```

#### 자동 빌드 워크플로우

```yaml
# .github/workflows/build-python-base.yml
name: Build Python Base

on:
  push:
    paths:
      - 'python/**'
  schedule:
    - cron: '0 2 * * 1'  # 매주 월요일
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set version
        id: version
        run: |
          VERSION="3.11.0.${{ github.run_number }}"
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - name: Build image
        run: |
          cd python
          docker build -t mycompany/python-base:${{ steps.version.outputs.version }} .
          docker tag mycompany/python-base:${{ steps.version.outputs.version }} \
                     mycompany/python-base:3.11
      
      - name: Security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: mycompany/python-base:${{ steps.version.outputs.version }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
      
      - name: Login to registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Push image
        run: |
          docker push mycompany/python-base:${{ steps.version.outputs.version }}
          docker push mycompany/python-base:3.11
      
      - name: Create release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: python-${{ steps.version.outputs.version }}
          release_name: Python Base ${{ steps.version.outputs.version }}
          body: |
            Python base image updated
            - Python 3.11
            - Security patches applied
            - Trivy scan passed
```

---

## 🔥 실전 적용

### 시나리오 1: 마이크로서비스 조직 표준화

**상황:**
- 20개 마이크로서비스
- 각팀이 독자적 베이스 사용
- 일관성 없음, 중복 많음

**해결: 3단계 표준 베이스:**

```dockerfile
# Level 1: base-os (보안팀 관리)
FROM ubuntu:22.04
# + 보안 강화
# + 필수 도구
# + 표준 설정

# Level 2: language-base (플랫폼팀 관리)
FROM mycompany/base-os:22.04
# + Python/Node/Go 런타임
# + 공통 패키지
# + 모니터링 에이전트

# Level 3: service (각 팀 관리)
FROM mycompany/python-base:3.11
# + 서비스 코드
# + 서비스별 의존성
```

**효과:**
```
스토리지:
- Before: 20 × 500MB = 10GB
- After: 600MB (공통) + 20 × 20MB = 1GB
- 절감: 90%

배포 시간:
- Before: 서비스당 3분
- After: 서비스당 30초 (공통 레이어 캐시)
- 절감: 83%

보안:
- Before: 20개 이미지 개별 패치
- After: 베이스 1개만 패치 → 전체 적용

일관성:
- 모든 서비스가 동일한 설정
- 문제 해결 용이
```

---

### 시나리오 2: 멀티 리전 배포 최적화

**상황:**
- 3개 리전 (US, EU, Asia)
- 각 리전에 동일 서비스 배포
- 네트워크 비용 과다

**해결: 리전별 베이스 이미지 캐시:**

```yaml
# 각 리전에 베이스 이미지 복제
- name: Replicate base images
  run: |
    # US 레지스트리에 push
    docker tag mycompany/python-base:3.11 \
               us.gcr.io/mycompany/python-base:3.11
    docker push us.gcr.io/mycompany/python-base:3.11
    
    # EU 레지스트리에 push
    docker tag mycompany/python-base:3.11 \
               eu.gcr.io/mycompany/python-base:3.11
    docker push eu.gcr.io/mycompany/python-base:3.11
    
    # Asia 레지스트리에 push
    docker tag mycompany/python-base:3.11 \
               asia.gcr.io/mycompany/python-base:3.11
    docker push asia.gcr.io/mycompany/python-base:3.11
```

```dockerfile
# 서비스 Dockerfile (리전 인식)
ARG REGION=us
FROM ${REGION}.gcr.io/mycompany/python-base:3.11

COPY . .
CMD ["python", "app.py"]
```

**효과:**
```
네트워크 전송:
- Before: 500MB × 20 서비스 × 3 리전 = 30GB
- After: 500MB × 1 (베이스, 한 번만) + 20MB × 20 × 3 = 1.7GB
- 절감: 94%

배포 시간:
- Before: 리전당 10분
- After: 리전당 1분 (로컬 캐시)

비용:
- 네트워크 전송 비용: 월 $1,000 → $60
```

---

### 시나리오 3: 보안 패치 자동 전파

**상황:**
- OpenSSL 취약점 발견
- 20개 서비스 모두 영향
- 긴급 패치 필요

**해결: 베이스 이미지 패치 → 자동 재배포:**

```yaml
# .github/workflows/emergency-patch.yml
name: Emergency Security Patch

on:
  workflow_dispatch:
    inputs:
      cve:
        description: 'CVE number'
        required: true

jobs:
  patch-base:
    runs-on: ubuntu-latest
    steps:
      # 1. 베이스 이미지 패치
      - name: Rebuild base with patches
        run: |
          cd base-images/os
          docker build --no-cache -t mycompany/base-os:22.04-patched .
      
      # 2. 취약점 스캔
      - name: Verify patch
        run: |
          trivy image --severity CRITICAL mycompany/base-os:22.04-patched
      
      # 3. 배포
      - name: Push patched image
        run: |
          docker tag mycompany/base-os:22.04-patched mycompany/base-os:22.04
          docker push mycompany/base-os:22.04
      
      # 4. 모든 서비스 재빌드 트리거
      - name: Trigger service rebuilds
        run: |
          for service in service-{1..20}; do
            curl -X POST "https://api.github.com/repos/mycompany/$service/dispatches" \
              -H "Authorization: token ${{ secrets.PAT }}" \
              -d '{"event_type":"base-updated","client_payload":{"cve":"${{ github.event.inputs.cve }}"}}'
          done
```

**효과:**
```
대응 시간:
- Before: 20개 서비스 개별 패치 (8시간)
- After: 베이스 1개 패치 + 자동 재배포 (30분)
- 절감: 93.75%

리스크:
- 일관성 보장
- 누락 위험 0%
- 자동 검증
```

---

## ⚡ 베이스 이미지 관리 체크리스트

### 설계

```
□ 조직 요구사항 분석
□ 계층 구조 설계 (OS → 런타임 → 서비스)
□ 버전 관리 전략
□ 태그 명명 규칙
□ 리전별 복제 전략
```

### 빌드

```
□ 자동 빌드 파이프라인
□ 보안 스캔 통합
□ 테스트 자동화
□ 버전 태깅
□ 변경 로그
```

### 배포

```
□ 레지스트리 선택
□ 접근 권한 관리
□ 리전별 복제
□ 캐시 전략
□ 롤백 계획
```

### 유지보수

```
□ 정기 업데이트 (주/월)
□ 보안 패치 프로세스
□ 사용 현황 모니터링
□ 사용 중단 계획
□ 문서 업데이트
```

### 거버넌스

```
□ 소유권 명확화
□ 승인 프로세스
□ 변경 관리
□ SLA 정의
□ 지원 채널
```

---

## 🚫 안티패턴

### 1. 베이스에 너무 많이 포함

```dockerfile
# ❌ 모든 걸 다 넣음
FROM ubuntu:22.04
RUN apt-get install -y \
    python3 nodejs golang rust \
    mysql postgresql redis \
    nginx apache2 \
    vim nano emacs \
    git curl wget
# 크기: 2GB, 취약점 많음

# ✅ 필요한 것만
FROM ubuntu:22.04
RUN apt-get install -y \
    python3 \
    ca-certificates
# 크기: 200MB, 취약점 적음
```

### 2. latest 태그 사용

```dockerfile
# ❌ 불안정
FROM mycompany/python-base:latest
# 언제 바뀔지 모름!

# ✅ 명시적 버전
FROM mycompany/python-base:3.11.0.5
# 정확히 제어 가능
```

### 3. 베이스 미공유

```bash
# ❌ 각 팀이 독자적 베이스
team-a/python-base:3.11
team-b/python-base:3.11
team-c/python-base:3.11
# 중복, 일관성 없음

# ✅ 조직 표준 베이스
mycompany/python-base:3.11
# 한 곳에서 관리, 전체 공유
```

### 4. 문서화 부족

```dockerfile
# ❌ 설명 없음
FROM mycompany/base:v2
# 뭐가 들었는지 모름

# ✅ 명확한 문서
FROM mycompany/python-base:3.11.0.5
# README.md:
# - Python 3.11.0
# - gunicorn 21.2.0
# - prometheus-client 0.19.0
# - 변경 이력: CHANGELOG.md
```

---

## 🎓 핵심 정리

### 1. scratch 이미지

```
특징:
- 완전히 비어있음
- 크기 0바이트
- 정적 바이너리만

사용 케이스:
- Go, Rust 애플리케이션
- 최소 크기 필요
- 최고 보안 필요

장점:
- 최소 크기 (5-20MB)
- 취약점 0개
- 공격 표면 최소
```

### 2. 맞춤형 베이스

```
계층 구조:
Level 1: OS 베이스
Level 2: 런타임 베이스
Level 3: 서비스 이미지

관리 주체:
Level 1: 보안팀
Level 2: 플랫폼팀
Level 3: 개발팀
```

### 3. 공통 레이어 효과

```
절감:
스토리지: 90%
배포 시간: 80%
네트워크: 94%

보안:
한 곳만 패치
전체 자동 적용
일관성 보장
```

### 4. 버전 관리

```
태깅:
major.minor.patch.build
예: 3.11.0.5

전략:
- 완전 고정 (권장)
- latest 금지
- 변경 로그 유지
```

---

## 📚 참고 자료

- [Docker Scratch](https://hub.docker.com/_/scratch)
- [Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Google Distroless](https://github.com/GoogleContainerTools/distroless)
- [OCI Image Spec](https://github.com/opencontainers/image-spec)

---

## 🤔 생각해볼 문제

1. scratch 이미지는 어떤 경우에 사용할 수 없을까?
2. 베이스 이미지를 3단계 계층으로 나누는 이유는?
3. 공통 레이어 공유가 많을수록 항상 좋을까?

> 💡 **답변**: 1) 동적 링크 라이브러리가 필요하거나(glibc 등), 쉘 스크립트를 실행해야 하거나, 디버깅 도구가 필요한 경우 사용 불가 - 이 경우 Alpine이나 Distroless 고려, 2) 책임 분리(보안팀/플랫폼팀/개발팀), 변경 격리(OS 패치가 모든 서비스에 영향 주지 않도록), 재사용성(여러 언어가 동일 OS 베이스 공유), 3) 아니다 - 너무 많은 것을 공통 레이어에 넣으면 불필요한 패키지 증가, 보안 취약점 증가, 특정 서비스만의 최적화 불가 - 적절한 균형 필요

---

<div align="center">

**[⬅️ 이전: Image Security](./06-Image-Security.md)** | **[홈으로 🏠](../README.md)**

</div>
