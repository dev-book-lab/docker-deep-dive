# 06. Image Security - 이미지 보안

## 🎯 이 챕터에서 배울 것

- **취약점 스캔**으로 보안 문제 사전 탐지
- **이미지 서명**으로 무결성 보장
- **최소 권한 원칙**과 보안 강화 기법
- **프로덕션 보안 체크리스트**

## 📌 왜 중요한가?

**"컨테이너 이미지는 공격의 첫 진입점입니다."**

```
취약한 이미지:
- 알려진 CVE: 500+
- Root 권한 실행
- 불필요한 도구 포함
- 검증되지 않은 이미지
→ 해킹 위험 높음

보안 강화 이미지:
- CVE: 5개 이하
- Non-root 실행
- 최소 패키지만 포함
- 서명 및 검증
→ 공격 표면 최소화
```

**실무 영향:**
- 보안 사고 방지: 데이터 유출, 랜섬웨어
- 컴플라이언스: SOC2, ISO27001, PCI-DSS
- 신뢰성: 공급망 보안 강화
- 비용: 보안 사고 대응 비용 절감

---

## 🔬 Deep Dive

### 1. 취약점 스캔

#### 취약점(CVE)이란?

```
CVE (Common Vulnerabilities and Exposures):
- 공개된 보안 취약점
- 고유 ID로 식별 (예: CVE-2024-1234)
- 심각도 등급: LOW, MEDIUM, HIGH, CRITICAL

예시:
CVE-2024-3094 - XZ Utils backdoor
- 영향: 원격 코드 실행
- 심각도: CRITICAL
- 영향받는 버전: xz 5.6.0, 5.6.1
```

#### 주요 스캐너 비교

```
┌──────────────┬────────────┬──────────┬─────────┬──────────┐
│   도구        │   무료      │  정확도    │  속도    │  통합     │
├──────────────┼────────────┼──────────┼─────────┼──────────┤
│ Trivy        │ ✅         │ ★★★★★    │ 빠름     │ 쉬움      │
│ (Aqua)       │            │          │         │          │
├──────────────┼────────────┼──────────┼─────────┼──────────┤
│ Grype        │ ✅         │ ★★★★☆    │ 빠름     │ 쉬움      │
│ (Anchore)    │            │          │         │          │
├──────────────┼────────────┼──────────┼─────────┼──────────┤
│ Clair        │ ✅         │ ★★★☆☆    │ 보통     │ 복잡      │
│ (Red Hat)    │            │          │         │          │
├──────────────┼────────────┼──────────┼─────────┼──────────┤
│ Snyk         │ 제한적       │ ★★★★★    │ 빠름     │ 쉬움      │
│              │            │          │         │          │
├──────────────┼────────────┼──────────┼─────────┼──────────┤
│ Docker Scout │ 제한적       │ ★★★★☆    │ 빠름     │ 매우쉬움   │
│              │            │          │         │          │
└──────────────┴────────────┴──────────┴─────────┴──────────┘

권장: Trivy (무료, 빠름, 정확)
```

---

### 2. Trivy로 취약점 스캔

#### 설치 및 기본 사용

```bash
# Trivy 설치
# Linux
wget https://github.com/aquasecurity/trivy/releases/download/v0.48.0/trivy_0.48.0_Linux-64bit.tar.gz
tar zxvf trivy_0.48.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/

# macOS
brew install aquasecurity/trivy/trivy

# Docker로 실행
docker run aquasec/trivy --version

# 기본 스캔
trivy image nginx:latest

# 출력:
# nginx:latest (debian 12.4)
# ==========================
# Total: 145 (CRITICAL: 0, HIGH: 2, MEDIUM: 57, LOW: 86)
```

#### 상세 스캔 옵션

```bash
# 심각도 필터링 (HIGH 이상만)
trivy image --severity HIGH,CRITICAL nginx:latest

# 특정 취약점 타입
trivy image --vuln-type os nginx:latest      # OS 패키지만
trivy image --vuln-type library nginx:latest # 라이브러리만

# JSON 출력
trivy image -f json -o results.json nginx:latest

# 테이블 형식
trivy image -f table nginx:latest

# 무시 파일 사용
trivy image --ignorefile .trivyignore nginx:latest

# 오프라인 스캔 (DB 다운로드 후)
trivy image --download-db-only
trivy image --skip-db-update --offline-scan nginx:latest
```

#### 결과 해석

```bash
trivy image node:18-alpine

# 출력 예시:
# node:18-alpine (alpine 3.19.0)
# ================================
# Total: 2 (CRITICAL: 0, HIGH: 0, MEDIUM: 2, LOW: 0)
# 
# ┌─────────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┐
# │  Library    │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │
# ├─────────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┤
# │ libcrypto3  │ CVE-2024-0727  │ MEDIUM   │ fixed  │ 3.1.4-r0          │ 3.1.4-r5      │
# │ libssl3     │ CVE-2024-0727  │ MEDIUM   │ fixed  │ 3.1.4-r0          │ 3.1.4-r5      │
# └─────────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┘

해석:
- 총 2개 취약점
- OpenSSL 관련 (libcrypto3, libssl3)
- 수정 가능 (3.1.4-r5로 업데이트)
- 조치: 베이스 이미지 업데이트
```

#### .trivyignore 사용

```bash
# .trivyignore - 특정 CVE 무시
# 예: 오탐이거나 영향 없는 경우

# 특정 CVE 무시
CVE-2024-1234

# 만료일 지정
CVE-2024-5678 exp:2024-12-31

# 이유 주석
# False positive - not using affected feature
CVE-2024-9012
```

---

### 3. 취약점 수정 전략

#### 1단계: 베이스 이미지 업데이트

```dockerfile
# ❌ Before: 오래된 이미지
FROM node:18-alpine3.17
# Trivy: 50 vulnerabilities

# ✅ After: 최신 이미지
FROM node:18-alpine3.19
# Trivy: 5 vulnerabilities

# 개선: 90% 취약점 감소
```

#### 2단계: 시스템 패키지 업데이트

```dockerfile
FROM node:18-alpine

# ✅ 시스템 패키지 업데이트
RUN apk update && \
    apk upgrade && \
    apk add --no-cache \
        libssl3>=3.1.4-r5 \
        libcrypto3>=3.1.4-r5 && \
    rm -rf /var/cache/apk/*

WORKDIR /app
COPY . .
CMD ["node", "server.js"]
```

#### 3단계: Distroless 전환

```dockerfile
# ✅ 최소 패키지 = 최소 취약점
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM gcr.io/distroless/nodejs18-debian11
WORKDIR /app
COPY --from=builder /app .
CMD ["server.js"]

# Trivy 결과:
# builder 스테이지: 50 vulnerabilities
# 최종 이미지: 2 vulnerabilities
# 개선: 96% 감소
```

---

### 4. 이미지 서명 및 검증

#### Docker Content Trust (DCT)

```bash
# DCT 활성화
export DOCKER_CONTENT_TRUST=1

# 이미지 push (자동 서명)
docker push myregistry/myapp:latest
# Enter passphrase for root key:
# Enter passphrase for new repository key:

# 서명된 이미지 pull
docker pull myregistry/myapp:latest
# Pull by digest: sha256:abc123...

# 서명 안 된 이미지는 거부됨
docker pull untrusted/image:latest
# Error: image untrusted/image:latest not signed

# 서명 정보 확인
docker trust inspect myregistry/myapp:latest
```

#### Cosign (Sigstore)

```bash
# Cosign 설치
wget https://github.com/sigstore/cosign/releases/download/v2.2.0/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# 키 생성
cosign generate-key-pair
# Private key: cosign.key
# Public key: cosign.pub

# 이미지 서명
cosign sign --key cosign.key myregistry/myapp:latest
# Enter password for private key:
# Pushing signature to: myregistry/myapp

# 서명 검증
cosign verify --key cosign.pub myregistry/myapp:latest

# Keyless 서명 (추천)
cosign sign myregistry/myapp:latest
# Opens browser for OIDC authentication
```

#### Notary (TUF)

```bash
# Notary 서버 설정
docker run -d -p 4443:4443 --name notary-server notary:server

# 리포지토리 초기화
notary init myregistry/myapp

# 이미지 서명
notary add myregistry/myapp latest sha256:abc123...
notary publish myregistry/myapp

# 서명 확인
notary list myregistry/myapp
```

---

### 5. 최소 권한 원칙

#### Non-root 사용자

```dockerfile
# ❌ Root로 실행 (위험)
FROM node:18-alpine
WORKDIR /app
COPY . .
CMD ["node", "server.js"]
# 컨테이너 내부에서 root 권한

# ✅ Non-root 사용자
FROM node:18-alpine

# 사용자 생성
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# 파일 소유권 설정
COPY --chown=appuser:appgroup . .

# Non-root로 전환
USER appuser

CMD ["node", "server.js"]

# 검증
# docker exec <container> whoami
# appuser
```

#### 읽기 전용 루트 파일시스템

```dockerfile
FROM node:18-alpine

RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# 쓰기 가능한 디렉토리 생성
RUN mkdir -p /app/tmp /app/logs && \
    chown -R appuser:appgroup /app/tmp /app/logs

COPY --chown=appuser:appgroup . .

USER appuser

CMD ["node", "server.js"]
```

```bash
# 읽기 전용으로 실행
docker run --read-only \
  --tmpfs /app/tmp \
  --tmpfs /app/logs \
  myapp:latest

# 테스트
docker exec <container> touch /test.txt
# touch: /test.txt: Read-only file system ✅
```

#### Capabilities 제한

```bash
# 기본 실행 (많은 capabilities)
docker run myapp:latest

# ✅ 모든 capabilities 제거
docker run --cap-drop=ALL myapp:latest

# 필요한 것만 추가
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  myapp:latest

# 현재 capabilities 확인
docker exec <container> capsh --print
```

---

### 6. 보안 스캔 자동화

#### Dockerfile에서 스캔

```dockerfile
# syntax=docker/dockerfile:1

FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app .
USER node
CMD ["node", "server.js"]

# ✅ 빌드 시 자동 스캔
# docker buildx build --output type=docker .
# trivy image <image-id>
```

#### GitHub Actions 통합

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      
      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Upload results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Fail on HIGH/CRITICAL
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
```

#### GitLab CI 통합

```yaml
# .gitlab-ci.yml
stages:
  - build
  - security-scan
  - deploy

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

trivy-scan:
  stage: security-scan
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: false

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main
```

---

## 💻 실습

### 실습 1: 취약점 스캔 및 수정

#### 준비

```bash
mkdir security-scan-demo
cd security-scan-demo

# 취약한 이미지 생성
cat > Dockerfile.vulnerable << 'EOF'
FROM node:16-alpine3.15

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

CMD ["node", "server.js"]
EOF

cat > package.json << 'EOF'
{
  "name": "demo",
  "version": "1.0.0",
  "dependencies": {
    "express": "4.17.1"
  }
}
EOF

cat > server.js << 'EOF'
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('Hello!'));
app.listen(3000, () => console.log('Server running'));
EOF
```

#### Step 1: 취약점 스캔

```bash
# 빌드
docker build -f Dockerfile.vulnerable -t demo:vulnerable .

# Trivy 스캔
trivy image demo:vulnerable

# 결과:
# Total: 147 (CRITICAL: 2, HIGH: 15, MEDIUM: 78, LOW: 52)
# 
# Critical Issues:
# CVE-2023-44487 - HTTP/2 Rapid Reset Attack
# CVE-2024-0727 - OpenSSL vulnerability
```

#### Step 2: 베이스 이미지 업데이트

```dockerfile
# Dockerfile.updated
FROM node:18-alpine3.19  # ← 버전 업데이트

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

CMD ["node", "server.js"]
```

```bash
docker build -f Dockerfile.updated -t demo:updated .
trivy image demo:updated

# 결과:
# Total: 45 (CRITICAL: 0, HIGH: 2, MEDIUM: 28, LOW: 15)
# 개선: 70% 감소
```

#### Step 3: 의존성 업데이트

```json
// package.json
{
  "name": "demo",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2"  // ← 최신 버전
  }
}
```

```bash
docker build -f Dockerfile.updated -t demo:deps-fixed .
trivy image demo:deps-fixed

# 결과:
# Total: 38 (CRITICAL: 0, HIGH: 0, MEDIUM: 25, LOW: 13)
# 개선: HIGH 완전 제거
```

#### Step 4: 멀티 스테이지 + Distroless

```dockerfile
# Dockerfile.secure
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

FROM gcr.io/distroless/nodejs18-debian11

WORKDIR /app
COPY --from=builder /app .

USER nonroot:nonroot

CMD ["server.js"]
```

```bash
docker build -f Dockerfile.secure -t demo:secure .
trivy image demo:secure

# 결과:
# Total: 5 (CRITICAL: 0, HIGH: 0, MEDIUM: 3, LOW: 2)
# 개선: 96% 감소!
```

#### 비교 요약

```
┌──────────────────┬─────────┬──────┬────────┬─────┐
│   Image          │ CRITICAL│ HIGH │ MEDIUM │ LOW │
├──────────────────┼─────────┼──────┼────────┼─────┤
│ vulnerable       │    2    │  15  │   78   │  52 │
├──────────────────┼─────────┼──────┼────────┼─────┤
│ updated          │    0    │   2  │   28   │  15 │
├──────────────────┼─────────┼──────┼────────┼─────┤
│ deps-fixed       │    0    │   0  │   25   │  13 │
├──────────────────┼─────────┼──────┼────────┼─────┤
│ secure           │    0    │   0  │    3   │   2 │
└──────────────────┴─────────┴──────┴────────┴─────┘

개선율: 96.6% (147개 → 5개)
```

---

### 실습 2: Non-root 사용자 적용

#### Before: Root로 실행

```dockerfile
# Dockerfile.root
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

CMD ["node", "server.js"]
```

```bash
docker build -f Dockerfile.root -t demo:root .
docker run -d --name test-root demo:root

# 사용자 확인
docker exec test-root whoami
# root ← 위험!

docker exec test-root id
# uid=0(root) gid=0(root) groups=0(root)

# 파일 생성 테스트
docker exec test-root touch /test.txt
# 성공 (root 권한)

docker stop test-root && docker rm test-root
```

#### After: Non-root 사용자

```dockerfile
# Dockerfile.nonroot
FROM node:18-alpine

# 사용자 생성
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# 소유권 설정하며 복사
COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --only=production

COPY --chown=appuser:appgroup . .

# Non-root로 전환
USER appuser

CMD ["node", "server.js"]
```

```bash
docker build -f Dockerfile.nonroot -t demo:nonroot .
docker run -d --name test-nonroot demo:nonroot

# 사용자 확인
docker exec test-nonroot whoami
# appuser ✅

docker exec test-nonroot id
# uid=1001(appuser) gid=1001(appgroup)

# 파일 생성 테스트
docker exec test-nonroot touch /test.txt
# touch: /test.txt: Permission denied ✅

docker stop test-nonroot && docker rm test-nonroot
```

---

### 실습 3: 읽기 전용 파일시스템

#### Dockerfile 준비

```dockerfile
# Dockerfile.readonly
FROM node:18-alpine

RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# 쓰기 가능한 디렉토리 생성
RUN mkdir -p /app/tmp /app/logs && \
    chown -R appuser:appgroup /app/tmp /app/logs

COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --only=production

COPY --chown=appuser:appgroup . .

USER appuser

CMD ["node", "server.js"]
```

```bash
docker build -f Dockerfile.readonly -t demo:readonly .

# 일반 실행
docker run -d --name test-normal demo:readonly
docker exec test-normal touch /app/newfile.txt
# 성공 (쓰기 가능)

# ✅ 읽기 전용으로 실행
docker run -d --name test-readonly \
  --read-only \
  --tmpfs /app/tmp \
  --tmpfs /app/logs \
  demo:readonly

# 루트에 쓰기 시도
docker exec test-readonly touch /test.txt
# touch: /test.txt: Read-only file system ✅

# /app에 쓰기 시도
docker exec test-readonly touch /app/test.txt
# touch: /app/test.txt: Read-only file system ✅

# tmpfs에 쓰기 (허용됨)
docker exec test-readonly touch /app/tmp/test.txt
# 성공 ✅

docker stop test-normal test-readonly
docker rm test-normal test-readonly
```

---

### 실습 4: 이미지 서명 (Cosign)

#### 설치

```bash
# Cosign 설치
wget https://github.com/sigstore/cosign/releases/download/v2.2.0/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

cosign version
```

#### 키 생성 및 서명

```bash
# 키 쌍 생성
cosign generate-key-pair
# Enter password for private key: ********
# Enter password again: ********
# Private key written to cosign.key
# Public key written to cosign.pub

# 이미지 빌드 및 push (로컬 레지스트리)
docker build -t localhost:5000/myapp:latest .
docker push localhost:5000/myapp:latest

# 이미지 서명
cosign sign --key cosign.key localhost:5000/myapp:latest
# Enter password for private key: ********
# Pushing signature to: localhost:5000/myapp

# 서명 검증
cosign verify --key cosign.pub localhost:5000/myapp:latest

# 성공 출력:
# Verification for localhost:5000/myapp:latest --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key
```

#### 서명 정보 확인

```bash
# 서명 정보 조회
cosign tree localhost:5000/myapp:latest

# 출력:
# 📦 Supply Chain Security Related artifacts for an image: localhost:5000/myapp:latest
# └── 💾 Attestations for an image tag: localhost:5000/myapp:sha256-abc123...
#     └── 🍒 sha256:def456...

# 서명 다운로드
cosign download signature localhost:5000/myapp:latest
```

---

## 🔥 실전 적용

### 시나리오 1: 취약점 Zero 목표

**상황:**
- 금융 서비스 컨테이너
- CRITICAL/HIGH 취약점: 25개
- 컴플라이언스 요구: 0개

**단계별 수정:**

```dockerfile
# Step 1: 최신 베이스 이미지
FROM python:3.11-alpine3.19

# Step 2: 시스템 패키지 업데이트
RUN apk update && \
    apk upgrade && \
    rm -rf /var/cache/apk/*

# Step 3: 최소 의존성
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    pip check

# Step 4: Distroless 전환
FROM gcr.io/distroless/python3-debian11

COPY --from=0 /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY app.py .

# Step 5: Non-root
USER nonroot:nonroot

CMD ["python3", "app.py"]
```

**결과:**
```bash
trivy image finance-app:final --severity CRITICAL,HIGH
# Total: 0 (CRITICAL: 0, HIGH: 0)

# 개선:
# CRITICAL: 5 → 0
# HIGH: 20 → 0
# 컴플라이언스: 통과 ✅
```

---

### 시나리오 2: CI/CD 파이프라인 보안 강화

**상황:**
- 배포 시 보안 검증 없음
- 취약한 이미지 프로덕션 배포
- 보안 사고 발생

**해결: 자동화된 보안 게이트:**

```yaml
# .github/workflows/secure-pipeline.yml
name: Secure Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  security-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # 1. 이미지 빌드
      - name: Build image
        run: |
          docker build -t myapp:${{ github.sha }} .
      
      # 2. 취약점 스캔
      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
      
      # 3. 이미지 서명
      - name: Install Cosign
        uses: sigstore/cosign-installer@v3
      
      - name: Sign image
        run: |
          cosign sign --yes myapp:${{ github.sha }}
      
      # 4. SBOM 생성
      - name: Generate SBOM
        run: |
          trivy image --format cyclonedx myapp:${{ github.sha }} > sbom.json
      
      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          name: sbom
          path: sbom.json
      
      # 5. 프로덕션 배포 (보안 통과 시)
      - name: Deploy
        if: github.ref == 'refs/heads/main'
        run: |
          kubectl set image deployment/myapp myapp=myapp:${{ github.sha }}
```

**효과:**
```
배포 전 자동 검증:
✅ CRITICAL/HIGH 취약점 0개
✅ 이미지 서명 완료
✅ SBOM 생성

보안 사고:
- Before: 월 2-3건
- After: 0건
```

---

### 시나리오 3: 공급망 보안

**상황:**
- 써드파티 베이스 이미지 사용
- 출처 불명 이미지
- 공급망 공격 위험

**해결: 검증된 이미지만 사용:**

```dockerfile
# Dockerfile
# syntax=docker/dockerfile:1

# ✅ 공식 이미지만 사용
FROM node:18-alpine AS builder

# ✅ 체크섬 검증
RUN apk add --no-cache \
    curl=8.5.0-r0 \
    ca-certificates=20230506-r0

# ✅ 패키지 서명 검증
RUN npm ci --only=production && \
    npm audit signatures

COPY . .
RUN npm run build

# ✅ Distroless (Google 서명)
FROM gcr.io/distroless/nodejs18-debian11

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

USER nonroot:nonroot

CMD ["dist/server.js"]
```

```bash
# 빌드 시 서명 검증
export DOCKER_CONTENT_TRUST=1
docker build -t myapp:latest .

# Cosign으로 최종 서명
cosign sign --key cosign.key myapp:latest

# 배포 시 검증
cosign verify --key cosign.pub myapp:latest
kubectl apply -f deployment.yaml
```

---

## ⚡ 보안 체크리스트

### 이미지 빌드

```
□ 최신 베이스 이미지 사용
□ 시스템 패키지 업데이트
□ 불필요한 패키지 제거
□ 멀티 스테이지 빌드
□ Distroless/Alpine 고려
```

### 취약점 관리

```
□ Trivy 스캔 자동화
□ CRITICAL/HIGH 0개 목표
□ 정기적 스캔 (weekly)
□ .trivyignore 문서화
□ 취약점 수정 프로세스
```

### 실행 보안

```
□ Non-root 사용자
□ 읽기 전용 파일시스템
□ Capabilities 최소화
□ Seccomp/AppArmor 프로파일
□ 리소스 제한
```

### 이미지 무결성

```
□ 이미지 서명 (Cosign)
□ Docker Content Trust
□ 서명 검증 자동화
□ SBOM 생성
□ 프라이빗 레지스트리
```

### CI/CD

```
□ 빌드 시 자동 스캔
□ 보안 게이트 설정
□ 실패 시 배포 중단
□ 서명 자동화
□ 감사 로그
```

---

## 🚫 안티패턴

### 1. Root 권한 실행

```dockerfile
# ❌ 위험
FROM node:18-alpine
COPY . /app
WORKDIR /app
CMD ["node", "server.js"]
# 컨테이너 탈출 시 호스트 root 권한

# ✅ 안전
FROM node:18-alpine
RUN adduser -D -u 1001 appuser
WORKDIR /app
COPY --chown=appuser . .
USER appuser
CMD ["node", "server.js"]
```

### 2. 취약점 무시

```dockerfile
# ❌ 오래된 이미지
FROM node:14-alpine3.13
# 수많은 알려진 취약점

# ✅ 최신 이미지
FROM node:18-alpine3.19
# 최신 보안 패치 적용
```

### 3. Secrets 하드코딩

```dockerfile
# ❌ 이미지에 노출
ENV DB_PASSWORD=secretpassword123
# docker history로 확인 가능!

# ✅ 런타임 주입
# Dockerfile에 secret 없음
# docker run -e DB_PASSWORD=$DB_PASSWORD ...
```

### 4. 불필요한 도구 포함

```dockerfile
# ❌ 공격 도구 포함
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    curl wget git vim nano build-essential \
    python3 gcc make
# 공격자가 악용 가능

# ✅ 최소 패키지
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

---

## 🎓 핵심 정리

### 1. 취약점 관리

```
스캔 도구:
Trivy (추천) → 빠르고 정확

목표:
CRITICAL: 0개
HIGH: 0개
MEDIUM: 최소화

주기:
빌드마다 자동 스캔
주 1회 정기 점검
```

### 2. 이미지 강화

```
베이스 이미지:
Alpine → 작고 보안 강화
Distroless → 최소 패키지

실행 권한:
Non-root 필수
읽기 전용 파일시스템
Capabilities 제한
```

### 3. 서명 및 검증

```
서명:
Cosign (Sigstore)
Docker Content Trust

검증:
배포 전 자동 검증
서명 없으면 배포 차단
```

### 4. 보안 지표

```
취약점:
CRITICAL/HIGH: 0개 목표

이미지 크기:
작을수록 공격 표면 작음

스캔 주기:
빌드마다 + 주 1회
```

---

## 📚 참고 자료

- [Trivy](https://github.com/aquasecurity/trivy)
- [Cosign](https://github.com/sigstore/cosign)
- [Docker Security](https://docs.docker.com/engine/security/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [OWASP Container Security](https://owasp.org/www-project-docker-top-10/)

---

## 🤔 생각해볼 문제

1. Alpine과 Distroless 중 보안 관점에서 어떤 것이 더 나을까?
2. 취약점 스캔에서 false positive를 어떻게 처리해야 할까?
3. 이미지 서명은 어떤 공격을 방어할 수 있을까?

> 💡 **답변**: 1) Distroless가 더 우수 - 쉘이 없어 공격자가 명령 실행 불가, 패키지 매니저도 없어 런타임 변조 불가능, Alpine은 musl libc와 apk가 있어 상대적으로 공격 표면이 큼, 2) .trivyignore로 문서화하되 정기적으로 재검토, 만료일 설정, 보안팀 승인 프로세스 필요, 3) 이미지 변조 공격(man-in-the-middle), 악의적 이미지 배포, 공급망 공격을 방어하며 무결성과 출처 검증 가능

---

<div align="center">

**[⬅️ 이전: BuildKit](./05-BuildKit.md)** | **[다음: Custom Base Images ➡️](./07-Custom-Base-Images.md)**

</div>
