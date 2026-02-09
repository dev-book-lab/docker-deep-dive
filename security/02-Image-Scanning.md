# 02. Image Scanning - 이미지 스캐닝

## 🎯 이 챕터에서 배울 것

- **이미지 취약점 스캐닝** 개념과 중요성
- **Trivy** - 오픈소스 취약점 스캐너
- **Clair** - 컨테이너 정적 분석
- **Anchore** - 정책 기반 스캐닝
- **CI/CD 통합** - 자동화된 보안 파이프라인

## 📌 왜 중요한가?

**"이미지 스캐닝은 컨테이너 보안의 첫 번째 방어선입니다."**

```
스캐닝 없는 환경 vs 스캐닝 환경:

스캐닝 없는 환경:
┌─────────────────────────────────────┐
│ Docker Hub / Registry               │
│ ┌─────────────────────────────────┐ │
│ │ ubuntu:20.04                    │ │
│ │ - 알려지지 않은 취약점 포함           │ │
│ │ - CVE-2021-XXXX (Critical)      │ │
│ │ - CVE-2022-YYYY (High)          │ │
│ └─────────────────────────────────┘ │
└────────────┬────────────────────────┘
             │ docker pull
             ↓
┌─────────────────────────────────────┐
│ Production Environment              │
│ ┌─────────────────────────────────┐ │
│ │ 취약한 컨테이너 실행                 │ │
│ │ → 공격 가능                       │ │
│ │ → 데이터 유출 위험                  │ │
│ │ → 랜섬웨어 감염                    │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
❌ 알려진 취약점 노출
❌ 공격 표면 확대
❌ 컴플라이언스 위반
❌ 사고 후 대응

스캐닝 환경:
┌─────────────────────────────────────┐
│ Image Build                         │
│ ┌─────────────────────────────────┐ │
│ │ Dockerfile                      │ │
│ │ FROM ubuntu:20.04               │ │
│ └─────────────────────────────────┘ │
└────────────┬────────────────────────┘
             │ docker build
             ↓
┌─────────────────────────────────────┐
│ Security Scan Pipeline              │
│ ┌─────────────────────────────────┐ │
│ │ 1. Trivy Scan                   │ │
│ │    ├─ OS 패키지 취약점             │ │
│ │    ├─ 애플리케이션 라이브러리         │ │
│ │    └─ 설정 파일                   │ │
│ │                                 │ │
│ │ 2. Policy Check                 │ │
│ │    ├─ Critical: FAIL ❌         │ │
│ │    ├─ High: WARN ⚠️             │ │
│ │    └─ Medium: PASS ✅           │ │
│ └─────────────────────────────────┘ │
└────────────┬────────────────────────┘
             │ if PASS
             ↓
┌─────────────────────────────────────┐
│ Production (Clean Image)            │
│ ✅ 취약점 최소화                       │
│ ✅ 규정 준수                          │
│ ✅ 공격 표면 축소                      │
└─────────────────────────────────────┘

이미지 스캐닝의 핵심 가치:

1. 사전 예방:
   - 배포 전 취약점 발견
   - Critical/High 취약점 차단
   - 공격 표면 최소화
   - 보안 부채 관리

2. 컴플라이언스:
   - PCI-DSS 요구사항
   - HIPAA 규정 준수
   - SOC 2 인증
   - CIS 벤치마크

3. 자동화:
   CI/CD Pipeline:
   ┌────────┐   ┌──────┐   ┌──────┐   ┌────────┐
   │  Code  │──→│Build │──→│ Scan │──→│ Deploy │
   │ Commit │   │Image │   │ (CI) │   │  (CD)  │
   └────────┘   └──────┘   └──┬───┘   └────────┘
                              │
                              ↓ if FAIL
                          ┌─────────┐
                          │ Reject  │
                          │ & Alert │
                          └─────────┘

4. 가시성:
   - 전체 인프라 취약점 추적
   - 우선순위 기반 패치
   - 보안 메트릭 대시보드
   - 규정 준수 보고서

실무 시나리오:

취약점 발견 프로세스:
┌─────────────────────────────────────┐
│ Step 1: Image Build                 │
│ docker build -t myapp:v1.0 .        │
└────────────┬────────────────────────┘
             │
             ↓
┌─────────────────────────────────────┐
│ Step 2: Trivy Scan                  │
│ trivy image myapp:v1.0              │
│                                     │
│ Results:                            │
│ ┌─────────────────────────────────┐ │
│ │ CRITICAL: 3                     │ │
│ │ - CVE-2023-1234 (libssl)        │ │
│ │ - CVE-2023-5678 (curl)          │ │
│ │ - CVE-2023-9012 (python)        │ │
│ │                                 │ │
│ │ HIGH: 12                        │ │
│ │ MEDIUM: 45                      │ │
│ │ LOW: 102                        │ │
│ └─────────────────────────────────┘ │
└────────────┬────────────────────────┘
             │
             ↓
┌─────────────────────────────────────┐
│ Step 3: Decision                    │
│ Policy: Fail on CRITICAL            │
│ → ❌ Build FAILED                   │
│ → 📧 Alert to Security Team         │
└────────────┬────────────────────────┘
             │
             ↓
┌─────────────────────────────────────┐
│ Step 4: Fix                         │
│ Dockerfile:                         │
│ FROM ubuntu:20.04                   │
│ → FROM ubuntu:22.04 (패치됨)          │
│                                     │
│ RUN apt-get update && \             │
│     apt-get upgrade -y              │
└────────────┬────────────────────────┘
             │
             ↓
┌─────────────────────────────────────┐
│ Step 5: Re-scan                     │
│ trivy image myapp:v1.1              │
│                                     │
│ Results:                            │
│ ┌─────────────────────────────────┐ │
│ │ CRITICAL: 0 ✅                  │ │
│ │ HIGH: 2                         │ │
│ │ MEDIUM: 15                      │ │
│ │ LOW: 50                         │ │
│ └─────────────────────────────────┘ │
│ → ✅ Build PASSED                   │
└────────────┬────────────────────────┘
             │
             ↓
┌─────────────────────────────────────┐
│ Step 6: Deploy                      │
│ docker push myapp:v1.1              │
│ kubectl apply -f deployment.yaml    │
└─────────────────────────────────────┘

Shift Left Security:
┌──────────────────────────────────────┐
│ Traditional Security (Shift Right)   │
├──────────────────────────────────────┤
│ Dev → Build → Test → Deploy → Scan   │
│                              ↑       │
│                       취약점 발견 늦음   │
│                       수정 비용 높음     │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ Modern Security (Shift Left)         │
├──────────────────────────────────────┤
│ Dev → Scan → Build → Test → Deploy   │
│        ↑                             │
│     취약점 조기 발견                     │
│     수정 비용 낮음                      │
│     빠른 피드백                         │
└──────────────────────────────────────┘
```

**실무 영향:**
- 취약점 조기 발견 → 수정 비용 90% 감소
- 자동화된 스캔 → 인적 오류 제거
- 정책 기반 통제 → 일관된 보안 수준
- 지속적 모니터링 → 제로데이 대응

---

## 🔧 실습 1: Trivy로 이미지 스캔

### Step 1: Trivy 설치

```bash
# Ubuntu/Debian
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# macOS
brew install trivy

# Docker 컨테이너로 실행
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest image nginx:latest

# 버전 확인
trivy --version
```

### Step 2: 기본 이미지 스캔

```bash
# Public 이미지 스캔
trivy image nginx:latest

# 출력 예시:
# nginx:latest (debian 11.8)
# ==========================
# Total: 145 (CRITICAL: 0, HIGH: 15, MEDIUM: 47, LOW: 83)
#
# ┌───────────────┬────────────────┬──────────┬───────────────────┬───────────────┬────────────────────────────────────┐
# │   Library     │ Vulnerability  │ Severity │ Installed Version │ Fixed Version │              Title                 │
# ├───────────────┼────────────────┼──────────┼───────────────────┼───────────────┼────────────────────────────────────┤
# │ libssl1.1     │ CVE-2023-0464  │   HIGH   │ 1.1.1n-0+deb11u3  │ 1.1.1n-0+...  │ openssl: X.509 policy check...     │
# │ libssl1.1     │ CVE-2023-0465  │   HIGH   │ 1.1.1n-0+deb11u3  │ 1.1.1n-0+...  │ openssl: Invalid pointer...        │
# └───────────────┴────────────────┴──────────┴───────────────────┴───────────────┴────────────────────────────────────┘

# 심각도 필터링
trivy image --severity CRITICAL,HIGH nginx:latest

# JSON 형식 출력
trivy image --format json nginx:latest > scan-results.json

# 특정 취약점 타입만
trivy image --vuln-type os nginx:latest          # OS 패키지만
trivy image --vuln-type library nginx:latest     # 애플리케이션 라이브러리만
```

### Step 3: 로컬 이미지 스캔

```bash
# 테스트용 취약한 이미지 빌드
cat > Dockerfile.vulnerable <<EOF
FROM ubuntu:20.04

RUN apt-get update && apt-get install -y \
    curl \
    wget \
    libssl1.1 \
    python3-pip

# 오래된 Python 패키지 (취약점 있음)
RUN pip3 install flask==1.0.0 requests==2.20.0

COPY app.py /app/
WORKDIR /app

CMD ["python3", "app.py"]
EOF

# 간단한 앱
cat > app.py <<EOF
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello World!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# 빌드
docker build -t vulnerable-app:v1.0 -f Dockerfile.vulnerable .

# 스캔
trivy image vulnerable-app:v1.0
```

**출력 분석:**
```
vulnerable-app:v1.0 (ubuntu 20.04)
==================================

Total: 234 (CRITICAL: 8, HIGH: 45, MEDIUM: 98, LOW: 83)

CRITICAL:
┌───────────────┬────────────────┬───────────────────┬───────────────┬───────────────────────┐
│ CVE-2021-3177 │ python3.8      │ 3.8.5-1~20.04.2   │ 3.8.5-1~...   │ ctypes buffer overflow│
│ CVE-2022-0391 │ python3-pip    │ 20.0.2-5ubuntu1.6 │ 20.0.2-...    │ Arbitrary code exec   │
└───────────────┴────────────────┴───────────────────┴───────────────┴───────────────────────┘

Python (pip):
┌───────────────────┬────────────────┬───────────────────┬───────────────┬──────────────────────┐
│ CVE-2018-1000656  │ flask          │ 1.0.0             │ 1.0.1         │ Denial of Service    │
│ CVE-2018-18074    │ requests       │ 2.20.0            │ 2.20.1        │ Improper cert valid  │
└───────────────────┴────────────────┴───────────────────┴───────────────┴──────────────────────┘
```

### Step 4: Exit Code로 CI/CD 통합

```bash
# Critical이 있으면 실패 (exit code 1)
trivy image --exit-code 1 --severity CRITICAL vulnerable-app:v1.0

# 출력:
# ...
# exit code: 1

# High 이상이 있으면 실패
trivy image --exit-code 1 --severity CRITICAL,HIGH vulnerable-app:v1.0

# CI/CD 스크립트 예시
#!/bin/bash
IMAGE_NAME="myapp:${CI_COMMIT_SHA}"

# 이미지 빌드
docker build -t ${IMAGE_NAME} .

# 스캔
trivy image --exit-code 1 --severity CRITICAL,HIGH ${IMAGE_NAME}

if [ $? -eq 0 ]; then
  echo "✅ Security scan passed"
  docker push ${IMAGE_NAME}
else
  echo "❌ Security scan failed - critical vulnerabilities found"
  exit 1
fi
```

### Step 5: 보고서 생성

```bash
# HTML 보고서
trivy image --format template --template "@contrib/html.tpl" \
  -o report.html vulnerable-app:v1.0

# SARIF 형식 (GitHub Security 탭 연동)
trivy image --format sarif -o results.sarif vulnerable-app:v1.0

# CycloneDX SBOM (Software Bill of Materials)
trivy image --format cyclonedx -o sbom.json vulnerable-app:v1.0

# SPDX SBOM
trivy image --format spdx-json -o sbom-spdx.json vulnerable-app:v1.0

# Table 형식으로 저장
trivy image --format table -o scan-report.txt vulnerable-app:v1.0
```

### Step 6: 취약점 수정

```bash
# 개선된 Dockerfile
cat > Dockerfile.fixed <<EOF
# 최신 Ubuntu 사용
FROM ubuntu:22.04

# 패키지 업데이트 및 최소 설치
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    python3 \
    python3-pip && \
    rm -rf /var/lib/apt/lists/*

# 최신 버전 패키지 설치
RUN pip3 install --no-cache-dir \
    flask==2.3.0 \
    requests==2.31.0

COPY app.py /app/
WORKDIR /app

# 비특권 사용자 실행
RUN useradd -m -u 1000 appuser
USER appuser

CMD ["python3", "app.py"]
EOF

# 재빌드
docker build -t vulnerable-app:v1.1 -f Dockerfile.fixed .

# 재스캔
trivy image vulnerable-app:v1.1

# 비교
echo "=== Before ==="
trivy image --severity CRITICAL,HIGH vulnerable-app:v1.0 | grep Total
echo "=== After ==="
trivy image --severity CRITICAL,HIGH vulnerable-app:v1.1 | grep Total
```

**개선 결과:**
```
=== Before ===
Total: 234 (CRITICAL: 8, HIGH: 45, MEDIUM: 98, LOW: 83)

=== After ===
Total: 15 (CRITICAL: 0, HIGH: 2, MEDIUM: 8, LOW: 5)

✅ CRITICAL: 8 → 0
✅ HIGH: 45 → 2
✅ 취약점 94% 감소
```

---

## 🔧 실습 2: Clair로 정적 분석

### Step 1: Clair 설치 (Docker Compose)

```bash
# Clair 구성 파일
mkdir -p ~/clair-demo && cd ~/clair-demo

cat > docker-compose.yml <<EOF
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: clair
      POSTGRES_USER: clair
      POSTGRES_PASSWORD: clair
    volumes:
      - clair-db:/var/lib/postgresql/data
    networks:
      - clair-net

  clair:
    image: quay.io/projectquay/clair:4.7
    depends_on:
      - postgres
    ports:
      - "6060:6060"
      - "6061:6061"
    environment:
      CLAIR_CONF: /config/config.yaml
      CLAIR_MODE: combo
    volumes:
      - ./config.yaml:/config/config.yaml:ro
    networks:
      - clair-net

volumes:
  clair-db:

networks:
  clair-net:
EOF

# Clair 설정 파일
cat > config.yaml <<EOF
http_listen_addr: :6060
introspection_addr: :6061
log_level: info

indexer:
  connstring: host=postgres port=5432 dbname=clair user=clair password=clair sslmode=disable
  scanlock_retry: 10
  layer_scan_concurrency: 5
  migrations: true

matcher:
  connstring: host=postgres port=5432 dbname=clair user=clair password=clair sslmode=disable
  migrations: true
  indexer_addr: http://clair:6060

notifier:
  connstring: host=postgres port=5432 dbname=clair user=clair password=clair sslmode=disable
  migrations: true
  indexer_addr: http://clair:6060
  matcher_addr: http://clair:6060
EOF

# 시작
docker-compose up -d

# 상태 확인
docker-compose ps
curl http://localhost:6060/openapi/v1
```

### Step 2: clairctl로 이미지 스캔

```bash
# clairctl 설치
curl -L https://github.com/quay/clair/releases/download/v4.7.0/clairctl-linux-amd64 \
  -o /usr/local/bin/clairctl
chmod +x /usr/local/bin/clairctl

# 이미지 manifest 생성
clairctl manifest nginx:latest > nginx-manifest.json

# 스캔 요청
clairctl report nginx-manifest.json

# JSON 결과
clairctl report --format json nginx-manifest.json > clair-report.json
```

### Step 3: Clair 결과 분석

```bash
# 결과 확인
cat clair-report.json | jq .

# 취약점 개수 확인
cat clair-report.json | jq '.vulnerabilities | length'

# Critical 취약점만
cat clair-report.json | jq '.vulnerabilities | map(select(.normalized_severity == "Critical"))'

# 패키지별 취약점
cat clair-report.json | jq '.vulnerabilities | group_by(.package.name) | map({package: .[0].package.name, count: length})'
```

---

## 🔧 실습 3: Anchore로 정책 기반 스캐닝

### Step 1: Anchore Engine 설치

```bash
# Anchore Compose 파일
mkdir -p ~/anchore-demo && cd ~/anchore-demo

curl -O https://engine.anchore.io/docs/quickstart/docker-compose.yaml

# 시작
docker-compose up -d

# 초기화 대기 (2-3분)
docker-compose exec api anchore-cli system wait

# 상태 확인
docker-compose exec api anchore-cli system status
```

### Step 2: 이미지 추가 및 스캔

```bash
# CLI 별칭 설정
alias anchore-cli='docker-compose exec api anchore-cli'

# 이미지 추가
anchore-cli image add nginx:latest

# 스캔 진행 상황 확인
anchore-cli image wait nginx:latest

# 스캔 완료 확인
anchore-cli image list

# 출력:
# Full Tag                        Image Digest                                        Analysis Status
# docker.io/nginx:latest          sha256:... analyzed
```

### Step 3: 취약점 조회

```bash
# 전체 취약점
anchore-cli image vuln nginx:latest all

# OS 취약점만
anchore-cli image vuln nginx:latest os

# 심각도별 필터링
anchore-cli image vuln nginx:latest all | grep Critical
anchore-cli image vuln nginx:latest all | grep High

# JSON 형식
anchore-cli --json image vuln nginx:latest all > anchore-vulns.json
```

**출력 예시:**
```
Vulnerability ID        Package                   Severity   Fix         CVE Refs        
CVE-2023-0464          libssl1.1                 High       None        CVE-2023-0464
CVE-2023-0465          libssl1.1                 High       None        CVE-2023-0465
CVE-2023-0466          libssl1.1                 Medium     1.1.1n-... CVE-2023-0466
```

### Step 4: 정책 생성

```bash
# 기본 정책 확인
anchore-cli policy list

# 커스텀 정책 생성
cat > custom-policy.json <<EOF
{
  "id": "custom-security-policy",
  "name": "Custom Security Policy",
  "version": "1.0.0",
  "rules": [
    {
      "gate": "vulnerabilities",
      "trigger": "package",
      "action": "stop",
      "params": [
        {
          "name": "package_type",
          "value": "all"
        },
        {
          "name": "severity_comparison",
          "value": ">="
        },
        {
          "name": "severity",
          "value": "high"
        }
      ],
      "id": "rule-high-vulns"
    },
    {
      "gate": "dockerfile",
      "trigger": "instruction",
      "action": "warn",
      "params": [
        {
          "name": "instruction",
          "value": "RUN"
        },
        {
          "name": "check",
          "value": "=~"
        },
        {
          "name": "value",
          "value": ".*sudo.*"
        }
      ],
      "id": "rule-no-sudo"
    },
    {
      "gate": "dockerfile",
      "trigger": "effective_user",
      "action": "warn",
      "params": [
        {
          "name": "users",
          "value": "root"
        },
        {
          "name": "type",
          "value": "blacklist"
        }
      ],
      "id": "rule-no-root"
    }
  ],
  "whitelists": [],
  "mappings": [
    {
      "registry": "*",
      "repository": "*",
      "image": {
        "type": "tag",
        "value": "*"
      },
      "policy_id": "custom-security-policy",
      "whitelist_ids": []
    }
  ]
}
EOF

# 정책 추가
anchore-cli policy add custom-policy.json

# 정책 활성화
anchore-cli policy activate custom-security-policy
```

### Step 5: 정책 평가

```bash
# 이미지에 대해 정책 평가
anchore-cli evaluate check nginx:latest

# 출력:
# Image Digest: sha256:...
# Full Tag: docker.io/nginx:latest
# Status: fail
# Last Eval: 2024-02-10T10:00:00Z
# Policy ID: custom-security-policy
#
# Gate               Trigger              Detail                                Status
# vulnerabilities    package              HIGH Vulnerability found in package   stop
# dockerfile         effective_user       User root found                       warn

# 상세 결과
anchore-cli evaluate check nginx:latest --detail

# JSON 형식
anchore-cli --json evaluate check nginx:latest > policy-result.json
```

### Step 6: CI/CD 통합

```bash
# CI/CD 스크립트 예시
cat > scan-and-check.sh <<'EOF'
#!/bin/bash

IMAGE=$1
TIMEOUT=300

echo "Adding image to Anchore..."
anchore-cli image add ${IMAGE}

echo "Waiting for analysis..."
anchore-cli image wait ${IMAGE} --timeout ${TIMEOUT}

echo "Checking policy..."
anchore-cli evaluate check ${IMAGE}

if [ $? -eq 0 ]; then
    echo "✅ Policy evaluation passed"
    exit 0
else
    echo "❌ Policy evaluation failed"
    anchore-cli evaluate check ${IMAGE} --detail
    exit 1
fi
EOF

chmod +x scan-and-check.sh

# 실행
./scan-and-check.sh nginx:latest
```

---

## 🔧 실습 4: CI/CD 파이프라인 통합

### GitLab CI/CD

```yaml
# .gitlab-ci.yml
stages:
  - build
  - scan
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  TRIVY_VERSION: 0.48.0

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

trivy-scan:
  stage: scan
  image: aquasec/trivy:$TRIVY_VERSION
  variables:
    GIT_STRATEGY: none
  script:
    # Container image scan
    - trivy image --exit-code 0 --severity LOW,MEDIUM $IMAGE_TAG
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE_TAG
    # Generate reports
    - trivy image --format json -o trivy-report.json $IMAGE_TAG
    - trivy image --format sarif -o trivy-results.sarif $IMAGE_TAG
  artifacts:
    reports:
      container_scanning: trivy-results.sarif
    paths:
      - trivy-report.json
    expire_in: 30 days
  allow_failure: false

deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG
  only:
    - main
  needs:
    - trivy-scan
```

### GitHub Actions

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Build Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'
    
    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Generate HTML report
      if: always()
      run: |
        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy:latest image \
          --format template --template "@contrib/html.tpl" \
          -o trivy-report.html \
          myapp:${{ github.sha }}
    
    - name: Upload HTML report
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: trivy-report
        path: trivy-report.html
```

### Jenkins Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        IMAGE_NAME = "myapp"
        IMAGE_TAG = "${BUILD_NUMBER}"
        REGISTRY = "my-registry.com"
    }
    
    stages {
        stage('Build') {
            steps {
                script {
                    docker.build("${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    // Trivy scan
                    sh """
                        docker run --rm \
                          -v /var/run/docker.sock:/var/run/docker.sock \
                          aquasec/trivy:latest image \
                          --exit-code 1 \
                          --severity CRITICAL,HIGH \
                          --format json \
                          -o trivy-report.json \
                          ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
            post {
                always {
                    // Archive scan results
                    archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                }
                failure {
                    // Send notification
                    emailext (
                        subject: "Security Scan Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                        body: "Critical or High vulnerabilities found. Check ${env.BUILD_URL}",
                        to: 'security-team@company.com'
                    )
                }
            }
        }
        
        stage('Deploy') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'registry-credentials') {
                        docker.image("${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}").push()
                    }
                }
            }
        }
    }
}
```

### CircleCI

```yaml
# .circleci/config.yml
version: 2.1

orbs:
  docker: circleci/docker@2.2.0

jobs:
  build-and-scan:
    docker:
      - image: cimg/base:2023.01
    steps:
      - checkout
      
      - setup_remote_docker:
          version: 20.10.24
      
      - run:
          name: Build Docker image
          command: |
            docker build -t myapp:${CIRCLE_SHA1} .
      
      - run:
          name: Install Trivy
          command: |
            wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
            echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
            sudo apt-get update
            sudo apt-get install -y trivy
      
      - run:
          name: Scan image
          command: |
            trivy image \
              --exit-code 1 \
              --severity CRITICAL,HIGH \
              --format json \
              -o /tmp/trivy-report.json \
              myapp:${CIRCLE_SHA1}
      
      - store_artifacts:
          path: /tmp/trivy-report.json

workflows:
  build-scan-deploy:
    jobs:
      - build-and-scan
```

---

## 💡 주요 명령어 정리

### Trivy

```bash
# 기본 스캔
trivy image IMAGE_NAME

# 심각도 필터
trivy image --severity CRITICAL,HIGH IMAGE_NAME

# Exit code (CI/CD)
trivy image --exit-code 1 --severity CRITICAL IMAGE_NAME

# 형식 지정
trivy image --format json IMAGE_NAME
trivy image --format sarif IMAGE_NAME
trivy image --format template --template "@contrib/html.tpl" IMAGE_NAME

# SBOM 생성
trivy image --format cyclonedx IMAGE_NAME
trivy image --format spdx-json IMAGE_NAME

# 특정 취약점 타입
trivy image --vuln-type os IMAGE_NAME
trivy image --vuln-type library IMAGE_NAME

# 결과 저장
trivy image IMAGE_NAME > report.txt
trivy image --format json -o report.json IMAGE_NAME
```

### Clair

```bash
# Manifest 생성
clairctl manifest IMAGE_NAME > manifest.json

# 보고서 생성
clairctl report manifest.json
clairctl report --format json manifest.json

# API 직접 호출
curl -X POST http://localhost:6060/indexer/api/v1/index_report
```

### Anchore

```bash
# 이미지 추가
anchore-cli image add IMAGE_NAME

# 스캔 대기
anchore-cli image wait IMAGE_NAME

# 취약점 조회
anchore-cli image vuln IMAGE_NAME all
anchore-cli image vuln IMAGE_NAME os

# 정책 평가
anchore-cli evaluate check IMAGE_NAME
anchore-cli evaluate check --detail IMAGE_NAME

# 정책 관리
anchore-cli policy list
anchore-cli policy add policy.json
anchore-cli policy activate POLICY_ID
```

---

## 🎓 연습 문제

### 문제 1: 취약한 이미지 수정

다음 Dockerfile의 취약점을 찾고 수정하세요:

```dockerfile
FROM ubuntu:18.04

RUN apt-get update && apt-get install -y \
    python \
    python-pip \
    curl

RUN pip install flask==0.12.0 requests==2.6.0

COPY app.py /app/
WORKDIR /app

CMD ["python", "app.py"]
```

요구사항:
- Trivy로 스캔하여 CRITICAL/HIGH 취약점 확인
- 모든 CRITICAL 취약점 제거
- HIGH 취약점 최소화

<details>
<summary>힌트 보기</summary>

- 최신 Ubuntu 사용 (22.04)
- 패키지 업그레이드
- 최신 Python 및 라이브러리
- 비특권 사용자 추가

</details>

<details>
<summary>정답 보기</summary>

```dockerfile
# 개선된 Dockerfile
FROM ubuntu:22.04

# 패키지 업데이트 및 최소 설치
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    python3 \
    python3-pip && \
    rm -rf /var/lib/apt/lists/*

# 최신 버전 설치
RUN pip3 install --no-cache-dir \
    flask==2.3.0 \
    requests==2.31.0

# 비특권 사용자
RUN useradd -m -u 1000 appuser

COPY app.py /app/
WORKDIR /app
RUN chown -R appuser:appuser /app

USER appuser

CMD ["python3", "app.py"]
```

```bash
# 검증
docker build -t fixed-app:v1.0 -f Dockerfile.fixed .
trivy image --severity CRITICAL,HIGH fixed-app:v1.0
```

</details>

### 문제 2: CI/CD 파이프라인 구성

GitHub Actions를 사용하여 다음 요구사항을 만족하는 보안 스캔 파이프라인을 작성하세요:

1. PR 생성 시 자동 스캔
2. CRITICAL 취약점 발견 시 빌드 실패
3. HIGH 취약점은 경고만
4. SARIF 결과를 GitHub Security 탭에 업로드
5. HTML 보고서를 Artifact로 저장

<details>
<summary>정답 보기</summary>

```yaml
name: Security Scan

on:
  pull_request:
    branches: [ main, develop ]
  push:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
      
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Build image
      uses: docker/build-push-action@v4
      with:
        context: .
        load: true
        tags: myapp:${{ github.sha }}
    
    - name: Run Trivy (CRITICAL only - fail build)
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:${{ github.sha }}'
        format: 'table'
        severity: 'CRITICAL'
        exit-code: '1'
    
    - name: Run Trivy (HIGH - warn only)
      uses: aquasecurity/trivy-action@master
      if: always()
      with:
        image-ref: 'myapp:${{ github.sha }}'
        format: 'table'
        severity: 'HIGH'
        exit-code: '0'
    
    - name: Generate SARIF report
      uses: aquasecurity/trivy-action@master
      if: always()
      with:
        image-ref: 'myapp:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH,MEDIUM'
    
    - name: Upload to GitHub Security
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Generate HTML report
      if: always()
      run: |
        docker run --rm \
          -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy:latest image \
          --format template \
          --template "@contrib/html.tpl" \
          -o trivy-report.html \
          myapp:${{ github.sha }}
    
    - name: Upload HTML report
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: security-report
        path: trivy-report.html
        retention-days: 30
```

</details>

### 문제 3: Anchore 정책 작성

다음 요구사항을 만족하는 Anchore 정책을 작성하세요:

1. HIGH 이상 취약점 발견 시 실패
2. root 사용자로 실행 시 경고
3. EXPOSE 80, 443 외 포트 사용 시 경고
4. apt-get, yum 명령어 사용 시 경고 (캐시 미삭제 검사)

<details>
<summary>정답 보기</summary>

```json
{
  "id": "strict-security-policy",
  "name": "Strict Security Policy",
  "version": "1.0.0",
  "rules": [
    {
      "gate": "vulnerabilities",
      "trigger": "package",
      "action": "stop",
      "params": [
        {
          "name": "package_type",
          "value": "all"
        },
        {
          "name": "severity_comparison",
          "value": ">="
        },
        {
          "name": "severity",
          "value": "high"
        }
      ],
      "id": "rule-high-vulns"
    },
    {
      "gate": "dockerfile",
      "trigger": "effective_user",
      "action": "warn",
      "params": [
        {
          "name": "users",
          "value": "root"
        },
        {
          "name": "type",
          "value": "blacklist"
        }
      ],
      "id": "rule-no-root"
    },
    {
      "gate": "dockerfile",
      "trigger": "exposed_ports",
      "action": "warn",
      "params": [
        {
          "name": "ports",
          "value": "80,443"
        },
        {
          "name": "type",
          "value": "whitelist"
        }
      ],
      "id": "rule-allowed-ports"
    },
    {
      "gate": "dockerfile",
      "trigger": "instruction",
      "action": "warn",
      "params": [
        {
          "name": "instruction",
          "value": "RUN"
        },
        {
          "name": "check",
          "value": "=~"
        },
        {
          "name": "value",
          "value": ".*(apt-get|yum).*"
        }
      ],
      "id": "rule-package-cleanup-check"
    }
  ],
  "whitelists": [],
  "mappings": [
    {
      "registry": "*",
      "repository": "*",
      "image": {
        "type": "tag",
        "value": "*"
      },
      "policy_id": "strict-security-policy",
      "whitelist_ids": []
    }
  ]
}
```

```bash
# 정책 추가
anchore-cli policy add strict-security-policy.json

# 활성화
anchore-cli policy activate strict-security-policy

# 테스트
anchore-cli evaluate check myapp:latest
```

</details>

---

## 📌 핵심 요약

### 이미지 스캐닝 도구 비교

| 도구 | 장점 | 단점 | 사용 케이스 |
|-----|------|------|----------|
| **Trivy** | - 빠르고 간단<br>- 다양한 출력 형식<br>- CI/CD 통합 쉬움<br>- 무료 오픈소스 | - 정책 기능 부족<br>- 대규모 환경 관리 어려움 | 개인/소규모 프로젝트<br>CI/CD 파이프라인 |
| **Clair** | - Red Hat 지원<br>- Quay 통합<br>- 정적 분석 강력 | - 설정 복잡<br>- 리소스 사용 높음 | 엔터프라이즈<br>Quay 사용 환경 |
| **Anchore** | - 강력한 정책 엔진<br>- 상세한 분석<br>- SBOM 생성 | - 무거움<br>- 학습 곡선 높음 | 대규모 엔터프라이즈<br>규정 준수 필요 |

### 스캔 전략

**Shift Left Security:**
```
개발자 로컬 →  CI 빌드  →  스테이징  →  프로덕션
    ↓          ↓         ↓          ↓
  Trivy      Trivy    Anchore    정기 스캔
(빠른 피드백)  (게이트)    (정책)     (모니터링)
```

**심각도별 대응:**
| 심각도 | 대응 | SLA |
|--------|------|-----|
| CRITICAL | 즉시 빌드 차단 | 24시간 내 패치 |
| HIGH | 빌드 차단 또는 승인 필요 | 7일 내 패치 |
| MEDIUM | 경고, 추적 | 30일 내 패치 |
| LOW | 모니터링만 | 분기별 검토 |

### CI/CD 통합 체크리스트

- [ ] 빌드 단계에 스캔 추가
- [ ] Exit code로 빌드 실패 제어
- [ ] SARIF/JSON 보고서 생성
- [ ] 아티팩트 저장 (30일)
- [ ] GitHub/GitLab Security 탭 통합
- [ ] Slack/이메일 알림 설정
- [ ] 정기 스캔 스케줄 (cron)
- [ ] 베이스 이미지 자동 업데이트

### 실무 Best Practices

1. **다층 스캔**
   ```
   로컬 개발 (Trivy) → CI (Trivy) → 스테이징 (Anchore) → 프로덕션 (정기 스캔)
   ```

2. **베이스 이미지 관리**
   - 승인된 베이스 이미지 목록 유지
   - 정기적 업데이트 (월 1회)
   - Private registry에 캐시

3. **취약점 관리**
   - 취약점 데이터베이스 정기 업데이트
   - False positive 추적
   - Whitelist 최소화

4. **메트릭 추적**
   - 스캔한 이미지 수
   - 발견된 취약점 추이
   - MTTR (평균 해결 시간)
   - 컴플라이언스 점수

---

<div align="center">

**[⬅️ 이전: Security Principles](./01-Security-Principles.md)** | **[다음: Runtime Security ➡️](./03-Runtime-Security.md)**

</div>
