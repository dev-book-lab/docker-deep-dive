# 05. Security Scanning - 파이프라인 보안 스캔

## 🎯 이 챕터에서 배울 것

- **취약점 스캔**: Trivy, Clair, Snyk
- **보안 레이어**: Image, Code, Dependencies
- **CI/CD 통합**: 자동화된 보안 검사
- **정책 관리**: 임계값, 차단 규칙
- **이미지 서명**: Cosign, Notary
- **실전 구현**: 프로덕션급 보안 파이프라인

## 📌 왜 중요한가?

**"보안 스캔은 취약점을 조기에 발견하고 공격을 예방합니다."**

```
Security Scanning의 핵심:

Without Security Scanning:
┌─────────────────────────────────────────────────┐
│ Vulnerable Deployment                           │
│                                                 │
│ 개발 → 빌드 → 배포 → 프로덕션                         │
│                           │                     │
│                           ▼                     │
│                    ⚠️ 취약점 발견!                 │
│                    - CVE-2023-12345             │
│                    - Critical RCE               │
│                    - 이미 배포됨 💥               │
└─────────────────────────────────────────────────┘

문제점:
❌ 늦은 발견 (프로덕션)
❌ 긴급 패치 필요
❌ 서비스 중단 가능
❌ 보안 사고 위험

With Security Scanning:
┌─────────────────────────────────────────────────┐
│ Secure Pipeline                                 │
│                                                 │
│ git push                                        │
│    ↓                                            │
│ Build Image                                     │
│    ↓                                            │
│ Security Scan ⚡                                 │
│    ├─ Base Image (alpine:3.18)                  │
│    │   ✅ 0 Critical, 0 High                    │
│    │                                            │
│    ├─ Dependencies (npm packages)               │
│    │   ⚠️ 1 High (lodash 4.17.20)               │
│    │   → Auto upgrade to 4.17.21                │
│    │                                            │
│    ├─ Application Code                          │
│    │   ✅ No SQL Injection                      │
│    │   ✅ No Hardcoded Secrets                  │
│    │                                            │
│    └─ Container Config                          │
│        ✅ Non-root user                         │
│        ✅ Read-only filesystem                  │
│    ↓                                            │
│ All Pass → Sign Image → Deploy                  │
│ Any Critical → Block Deploy ❌                  │
└─────────────────────────────────────────────────┘

장점:
✅ 조기 발견 (빌드 단계)
✅ 자동 차단
✅ 안전한 배포
✅ 규정 준수

Security Layers:
┌─────────────────────────────────────────────────┐
│ 1. Base Image Vulnerabilities                   │
│    - alpine:3.18 → CVE 검사                      │
│    - OS packages                                │
│                                                 │
│ 2. Application Dependencies                     │
│    - npm, pip, maven                            │
│    - Known CVEs                                 │
│                                                 │
│ 3. Application Code                             │
│    - SAST (Static Analysis)                     │
│    - SQL Injection, XSS                         │
│    - Hardcoded Secrets                          │
│                                                 │
│ 4. Container Configuration                      │
│    - Dockerfile best practices                  │
│    - Root user check                            │
│    - Exposed secrets                            │
│                                                 │
│ 5. Runtime Security                             │
│    - DAST (Dynamic Analysis)                    │
│    - Behavioral monitoring                      │
└─────────────────────────────────────────────────┘

Vulnerability Severity:
CRITICAL (9.0-10.0)  → 즉시 차단
HIGH (7.0-8.9)       → 경고, 선택적 차단
MEDIUM (4.0-6.9)     → 경고
LOW (0.1-3.9)        → 정보
NONE (0.0)           → 무시

Scanning Tools:
┌──────────────┬──────────────┬──────────────┐
│ 도구          │ 장점          │ 용도          │
├──────────────┼──────────────┼──────────────┤
│ Trivy        │ 빠름, 정확     │ 범용          │
├──────────────┼──────────────┼──────────────┤
│ Clair        │ Harbor 통합   │ Registry     │
├──────────────┼──────────────┼──────────────┤
│ Snyk         │ 자동 수정      │ Dependencies │
├──────────────┼──────────────┼──────────────┤
│ Anchore      │ Policy       │ Enterprise   │
└──────────────┴──────────────┴──────────────┘
```

**실무 영향:**
- **안전성**: 취약점 사전 차단 (90% 이상)
- **규정 준수**: SOC2, HIPAA 등
- **신뢰**: 보안 인증 획득
- **비용**: 보안 사고 예방 (평균 수억 원)

---

## 🔬 Deep Dive

### 1. Trivy 기본

#### 설치 및 사용

```bash
# 설치
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# 이미지 스캔
trivy image nginx:latest

# 출력:
# nginx:latest (alpine 3.18.4)
# Total: 2 (CRITICAL: 0, HIGH: 0, MEDIUM: 2, LOW: 0, UNKNOWN: 0)
```

#### 심각도 필터

```bash
# Critical만
trivy image --severity CRITICAL nginx:latest

# Critical, High
trivy image --severity CRITICAL,HIGH nginx:latest

# Exit code 1 (CI/CD 차단)
trivy image --exit-code 1 --severity CRITICAL nginx:latest
```

---

### 2. 취약점 유형

#### CVE (Common Vulnerabilities and Exposures)

```
CVE-2023-12345
│    │    │
│    │    └─ 일련번호
│    └────── 연도
└─────────── CVE

예:
CVE-2021-44228 (Log4Shell)
- Severity: Critical (10.0)
- Apache Log4j RCE
- 긴급 패치 필요
```

#### CWE (Common Weakness Enumeration)

```
CWE-89: SQL Injection
CWE-79: Cross-site Scripting (XSS)
CWE-798: Hardcoded Credentials
```

---

## 🔧 실습 1: Trivy CI/CD 통합

### Step 1: GitHub Actions

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      
      # Trivy 스캔
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'  # Critical/High 발견 시 실패
      
      # GitHub Security tab에 업로드
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
      
      # HTML 리포트 생성
      - name: Generate HTML report
        if: always()
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'template'
          template: '@/contrib/html.tpl'
          output: 'trivy-report.html'
      
      - name: Upload HTML report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: trivy-report
          path: trivy-report.html
```

### Step 2: GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - scan
  - deploy

variables:
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

trivy-scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    # 스캔
    - trivy image --exit-code 0 --no-progress $IMAGE_TAG
    
    # Critical/High 발견 시 실패
    - trivy image --exit-code 1 --severity CRITICAL,HIGH --no-progress $IMAGE_TAG
    
    # JSON 리포트
    - trivy image --format json --output trivy-report.json $IMAGE_TAG
  
  artifacts:
    reports:
      container_scanning: trivy-report.json
    when: always
  
  allow_failure: false  # 스캔 실패 시 파이프라인 중단
```

---

## 🔧 실습 2: 다중 레이어 스캔

### Step 1: Dockerfile 스캔

```yaml
# .github/workflows/comprehensive-scan.yml
name: Comprehensive Security Scan

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      # 1. Dockerfile 스캔
      - name: Scan Dockerfile
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
      
      # 2. 파일시스템 스캔 (소스 코드)
      - name: Scan filesystem
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'table'
          exit-code: '0'  # 경고만
      
      # 3. 이미지 빌드
      - name: Build image
        run: docker build -t myapp:scan .
      
      # 4. 이미지 스캔
      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:scan
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
```

### Step 2: 의존성 스캔

```yaml
# Snyk 통합
- name: Run Snyk to check for vulnerabilities
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
    command: test

# 자동 수정 PR 생성
- name: Snyk fix
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    command: fix
```

---

## 🔧 실습 3: 정책 기반 스캔

### Step 1: Trivy 정책 파일

```yaml
# trivy-policy.yaml
policies:
  - id: CVE-2023-critical
    severity: CRITICAL
    action: reject
    message: "Critical CVE detected - deployment blocked"
  
  - id: high-severity
    severity: HIGH
    max-count: 5
    action: warn
    message: "Too many high severity vulnerabilities"
  
  - id: root-user
    type: dockerfile
    checks:
      - USER root
    action: reject
    message: "Container should not run as root"
  
  - id: outdated-base
    checks:
      - alpine:3.15  # 오래된 버전
      - ubuntu:18.04
    action: warn
    message: "Base image is outdated"
```

### Step 2: Policy 적용

```bash
# 정책 파일로 스캔
trivy image --policy trivy-policy.yaml myapp:latest

# OPA (Open Policy Agent) 통합
trivy image --policy opa/policy.rego myapp:latest
```

### Step 3: OPA Policy 예시

```rego
# policy.rego
package trivy

# Critical CVE 차단
deny[msg] {
    input.Vulnerabilities[_].Severity == "CRITICAL"
    msg := "Critical vulnerability found"
}

# Root user 차단
deny[msg] {
    input.Config.User == "root"
    msg := "Container should not run as root"
}

# 특정 패키지 버전 차단
deny[msg] {
    input.Packages[_].Name == "log4j"
    input.Packages[_].Version < "2.17.0"
    msg := "log4j version is vulnerable to Log4Shell"
}
```

---

## 🔧 실습 4: Harbor 통합 스캔

### Step 1: Harbor 자동 스캔 설정

```bash
# Harbor Web UI
# Project → myproject → Configuration

Automatically scan images on push: ✅
Prevent vulnerable images from running: ✅
Severity threshold: High
```

### Step 2: Harbor API로 스캔

```bash
# 이미지 푸시 후 자동 스캔
docker push harbor.example.com/myproject/myapp:v1.0.0

# 수동 스캔 트리거
curl -X POST \
  -u admin:password \
  https://harbor.example.com/api/v2.0/projects/myproject/repositories/myapp/artifacts/v1.0.0/scan

# 스캔 결과 확인
curl -u admin:password \
  https://harbor.example.com/api/v2.0/projects/myproject/repositories/myapp/artifacts/v1.0.0/additions/vulnerabilities
```

### Step 3: Webhook 통합

```yaml
# Harbor Webhook → Slack
{
  "event_type": "scanningCompleted",
  "occur_at": 1234567890,
  "event_data": {
    "repository": {
      "name": "myapp",
      "namespace": "myproject"
    },
    "scan_overview": {
      "total": 10,
      "summary": {
        "Critical": 0,
        "High": 2,
        "Medium": 8
      }
    }
  }
}

# Slack으로 알림
```

---

## 🔧 실습 5: 이미지 서명 (Cosign)

### Step 1: Cosign 설치 및 키 생성

```bash
# 설치
brew install cosign  # macOS
# 또는
curl -O -L https://github.com/sigstore/cosign/releases/download/v2.2.0/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# 키 생성
cosign generate-key-pair

# 생성: cosign.key (private), cosign.pub (public)
```

### Step 2: 이미지 서명

```bash
# 이미지 빌드 및 푸시
docker build -t myregistry/myapp:v1.0.0 .
docker push myregistry/myapp:v1.0.0

# 서명
cosign sign --key cosign.key myregistry/myapp:v1.0.0

# 서명 확인
cosign verify --key cosign.pub myregistry/myapp:v1.0.0
```

### Step 3: GitHub Actions 통합

```yaml
# .github/workflows/sign-image.yml
name: Build, Scan, Sign

on:
  push:
    tags: ['v*']

jobs:
  build-and-sign:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
      id-token: write  # OIDC
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image
        run: docker build -t ghcr.io/${{ github.repository }}:${{ github.ref_name }} .
      
      # Trivy 스캔
      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.ref_name }}
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
      
      # 로그인
      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      # 푸시
      - name: Push image
        run: docker push ghcr.io/${{ github.repository }}:${{ github.ref_name }}
      
      # Cosign 설치
      - name: Install Cosign
        uses: sigstore/cosign-installer@v3
      
      # Keyless 서명 (OIDC)
      - name: Sign image
        run: |
          cosign sign --yes ghcr.io/${{ github.repository }}:${{ github.ref_name }}
```

### Step 4: Kubernetes에서 서명 검증

```yaml
# admission-controller.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: cosign-validation
webhooks:
  - name: cosign.sigstore.dev
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    failurePolicy: Fail
    clientConfig:
      service:
        name: cosign-webhook
        namespace: cosign-system
        path: /validate

# 서명 없는 이미지 차단
```

---

## 🔧 실습 6: 취약점 자동 수정

### Step 1: Dependabot 설정

```yaml
# .github/dependabot.yml
version: 2
updates:
  # Docker
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
  
  # npm
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 5
    
    # 자동 병합 (minor, patch)
    allow:
      - dependency-type: "all"
    
    # 보안 업데이트 우선
    versioning-strategy: increase-if-necessary
```

### Step 2: Renovate Bot

```json
// renovate.json
{
  "extends": ["config:base"],
  "docker": {
    "enabled": true,
    "major": {
      "enabled": true
    }
  },
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"],
    "assignees": ["@security-team"]
  },
  "packageRules": [
    {
      "matchUpdateTypes": ["patch", "minor"],
      "matchCurrentVersion": "!/^0/",
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "matchPackagePatterns": ["*"],
      "matchUpdateTypes": ["major"],
      "dependencyDashboardApproval": true
    }
  ]
}
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 스캔 단계              │ 도구                        │
├──────────────────────┼────────────────────────────┤
│ Dockerfile           │ Hadolint, Trivy            │
├──────────────────────┼────────────────────────────┤
│ Dependencies         │ Snyk, Dependabot           │
├──────────────────────┼────────────────────────────┤
│ Image                │ Trivy, Clair, Anchore      │
├──────────────────────┼────────────────────────────┤
│ Code                 │ SonarQube, CodeQL          │
├──────────────────────┼────────────────────────────┤
│ Signature            │ Cosign, Notary             │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. 빌드 단계에서 스캔
2. Critical/High 차단
3. 자동 수정 (Dependabot)
4. 이미지 서명
5. 정기 재스캔
```

---

## 🎓 연습 문제

### 문제 1: 모든 취약점을 수정해야 하는가?

<details>
<summary>정답 보기</summary>

**현실적 접근:**

```bash
# ❌ 모든 취약점 수정 불가능
- 오래된 베이스 이미지
- 업스트림 패치 없음
- False positive

# ✅ 우선순위 기반
Priority 1: Critical (CVSS 9.0-10.0)
- 즉시 수정 또는 차단

Priority 2: High (CVSS 7.0-8.9)
- 7일 이내 수정
- Workaround 적용

Priority 3: Medium (CVSS 4.0-6.9)
- 30일 이내 수정
- 다음 릴리스

Priority 4: Low (CVSS 0.1-3.9)
- 필요 시 수정
- Backlog
```

**위험 수용:**
```yaml
# trivy.yaml
ignoredVulnerabilities:
  - CVE-2023-12345
    reason: "No fix available, workaround applied"
    expiration: "2024-12-31"
```

</details>

### 문제 2: 스캔이 CI/CD를 너무 느리게 만든다면?

<details>
<summary>정답 보기</summary>

**최적화 전략:**

**1. 병렬 실행:**
```yaml
jobs:
  test:
    # 테스트와 병렬
  scan:
    # 스캔 병렬
```

**2. 캐싱:**
```yaml
- name: Cache Trivy DB
  uses: actions/cache@v3
  with:
    path: ~/.cache/trivy
    key: trivy-db-${{ hashFiles('**/Dockerfile') }}
```

**3. 선택적 스캔:**
```yaml
# PR: 빠른 스캔
trivy image --severity CRITICAL

# Main: 전체 스캔
trivy image --severity CRITICAL,HIGH,MEDIUM
```

**4. 비동기 스캔:**
```yaml
# 빌드 → 배포
# 스캔은 백그라운드 (알림만)
```

</details>

### 문제 3: False Positive를 어떻게 처리하는가?

<details>
<summary>정답 보기</summary>

**검증 방법:**

```bash
# 1. 실제 영향 확인
# CVE 상세 정보 읽기
https://nvd.nist.gov/vuln/detail/CVE-2023-12345

# 2. 사용 여부 확인
# 취약한 함수를 실제로 사용하는가?

# 3. 패키지 버전 확인
npm ls lodash
# 실제론 4.17.21 사용 (안전)
# 하지만 Trivy는 4.17.20으로 인식
```

**처리 방법:**
```yaml
# .trivyignore
CVE-2023-12345  # False positive, actually not vulnerable

# 또는 정책 파일
policies:
  - id: CVE-2023-12345
    action: ignore
    reason: "False positive - not using vulnerable function"
    expires: "2024-12-31"
```

</details>

---

## 📌 핵심 요약

```
Security Scanning 핵심:
1. 빌드 단계 스캔
2. Critical/High 차단
3. 자동 수정 (Dependabot)
4. 이미지 서명 (Cosign)
5. 정기 재스캔

Best Practices:
✅ Trivy 통합
✅ 정책 기반 차단
✅ Harbor 자동 스캔
✅ Dependabot 활성화
✅ 취약점 대응 프로세스
```

---

## 📚 참고 자료

- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [OWASP Container Security](https://owasp.org/www-project-docker-top-10/)
- [Sigstore Cosign](https://docs.sigstore.dev/cosign/overview/)

---

## 🤔 생각해볼 문제

1. 프로덕션 이미지에서 취약점이 발견되면?
2. 베이스 이미지 선택 기준은?
3. 스캔 결과를 누가 검토하는가?

> 💡 **답변**:
> 
> **1) 프로덕션 취약점 대응:**
> 
> ```
> Critical 발견:
> 1. 즉시 평가 (실제 영향)
> 2. 긴급 패치 또는 롤백
> 3. 핫픽스 배포
> 4. 사후 분석
> 
> High 발견:
> 1. 7일 이내 수정
> 2. 다음 배포에 포함
> 3. WAF 규칙 추가 (임시)
> ```
> 
> **2) 베이스 이미지 선택:**
> 
> ```
> ✅ 권장:
> - alpine (최소, 빠름)
> - distroless (초소형)
> - slim variants
> 
> ❌ 피할 것:
> - latest (불안정)
> - ubuntu:18.04 (오래됨)
> - full images (불필요한 패키지)
> ```
> 
> **3) 책임 및 프로세스:**
> 
> ```
> 개발자: 빌드 시 스캔 확인
> 보안팀: 정책 관리, 심각한 취약점 검토
> DevOps: 스캔 도구 관리
> 
> 프로세스:
> 1. 자동 스캔 (CI/CD)
> 2. Critical → 자동 차단
> 3. High → 보안팀 검토
> 4. 주간 리포트 (요약)
> ```

---

<div align="center">

**[⬅️ 이전: Automated Testing](./04-Automated-Testing.md)** | **[다음: GitOps ➡️](./06-GitOps.md)**

</div>
