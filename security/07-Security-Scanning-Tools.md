# 07. Security Scanning Tools - 보안 스캐닝 도구

## 🎯 이 챕터에서 배울 것

- **자동화된 스캐닝** - CI/CD 파이프라인 통합
- **취약점 데이터베이스** - CVE, NVD 활용
- **정책 기반 게이팅** - 자동 승인/거부
- **지속적 모니터링** - 런타임 스캐닝
- **보안 대시보드** - 메트릭 및 리포팅

## 📌 왜 중요한가?

**"자동화된 보안 스캐닝은 취약점을 조기에 발견하고 배포 전 차단하는 핵심 방어선입니다."**

```
수동 보안 vs 자동화된 보안:

수동 보안 검토:
┌─────────────────────────────────────┐
│ 1. 개발 완료                          │
│    개발자가 코드 작성                   │
└────────────┬────────────────────────┘
             │ (1주일 후)
┌────────────▼────────────────────────┐
│ 2. 보안팀 검토 요청                     │
│    티켓 생성 및 대기                    │
└────────────┬────────────────────────┘
             │ (며칠 대기)
┌────────────▼────────────────────────┐
│ 3. 수동 검토                          │
│    보안팀이 코드/이미지 검토              │
└────────────┬────────────────────────┘
             │ (발견 시)
┌────────────▼────────────────────────┐
│ 4. 취약점 발견                         │
│    개발팀에 피드백                      │
└────────────┬────────────────────────┘
             │ (수정 후)
┌────────────▼────────────────────────┐
│ 5. 재검토 및 배포                      │
│    최종 승인                          │
└─────────────────────────────────────┘

문제점:
❌ 느린 피드백 (1-2주)
❌ 병목 현상 (보안팀 대기)
❌ 인적 오류 가능
❌ 불완전한 커버리지
❌ 비용 높음

자동화된 보안 파이프라인:
┌─────────────────────────────────────┐
│ 1. Git Push                         │
│    개발자가 코드 커밋                   │
└────────────┬────────────────────────┘
             │ (즉시)
┌────────────▼────────────────────────┐
│ 2. CI Pipeline 트리거                 │
│    ├─ Build Image                   │
│    ├─ Trivy Scan (취약점)             │
│    ├─ Hadolint (Dockerfile)         │
│    └─ git-secrets (시크릿)            │
└────────────┬────────────────────────┘
             │ (2-3분)
┌────────────▼────────────────────────┐
│ 3. 자동 정책 평가                      │
│    CRITICAL: 0                      │
│    HIGH: 2 (허용 임계값 5)             │
│    → ✅ PASS                        │
└────────────┬────────────────────────┘
             │ (즉시)
┌────────────▼────────────────────────┐
│ 4. 이미지 푸시                         │
│    Registry에 서명된 이미지 저장         │
└────────────┬────────────────────────┘
             │ (즉시)
┌────────────▼────────────────────────┐
│ 5. 자동 배포                          │
│    Production 환경                   │
└─────────────────────────────────────┘

장점:
✅ 빠른 피드백 (분 단위)
✅ 병목 제거
✅ 일관된 검사
✅ 100% 커버리지
✅ 비용 효율적

Shift Left Security:
┌──────────────────────────────────────┐
│ Traditional (Shift Right)            │
├──────────────────────────────────────┤
│ Dev → Build → Test → Deploy → Scan   │
│                              ↑       │
│                       늦은 발견        │
│                       높은 수정 비용    │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ Modern (Shift Left)                  │
├──────────────────────────────────────┤
│ Dev → Scan → Build → Test → Deploy   │
│        ↑                             │
│     조기 발견                          │
│     낮은 수정 비용                      │
│     빠른 피드백                         │
└──────────────────────────────────────┘

수정 비용 비교:
┌────────────────────────────────┐
│ Development  : $100            │
│ Testing      : $1,000          │
│ Staging      : $10,000         │
│ Production   : $100,000        │
└────────────────────────────────┘
→ 조기 발견 시 최대 1000배 절감

보안 스캐닝 계층:
┌──────────────────────────────────────┐
│ Layer 6: Runtime Monitoring          │
│ - Falco                              │
│ - Aqua Security                      │
│ - Sysdig Secure                      │
├──────────────────────────────────────┤
│ Layer 5: Registry Scanning           │
│ - Harbor                             │
│ - Quay                               │
│ - ECR                                │
├──────────────────────────────────────┤
│ Layer 4: Post-Build Scanning         │
│ - Trivy                              │
│ - Clair                              │
│ - Anchore                            │
├──────────────────────────────────────┤
│ Layer 3: Dockerfile Linting          │
│ - Hadolint                           │
│ - dockerfile-lint                    │
├──────────────────────────────────────┤
│ Layer 2: Dependency Scanning         │
│ - Snyk                               │
│ - npm audit                          │
│ - pip-audit                          │
├──────────────────────────────────────┤
│ Layer 1: Secret Detection            │
│ - git-secrets                        │
│ - TruffleHog                         │
│ - GitGuardian                        │
└──────────────────────────────────────┘

실무 시나리오:

취약점 발견 프로세스:
┌─────────────────────────────────────┐
│ Day 1: 개발                          │
│ $ git commit -m "Add new feature"   │
│ $ git push                          │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ CI Pipeline 자동 실행                 │
│ ┌─────────────────────────────────┐ │
│ │ Trivy Scan                      │ │
│ │ ────────────────────────────    │ │
│ │ CRITICAL: 3                     │ │
│ │ - CVE-2023-1234 (openssl)       │ │
│ │ - CVE-2023-5678 (curl)          │ │
│ │ - CVE-2023-9012 (libxml2)       │ │
│ │                                 │ │
│ │ HIGH: 12                        │ │
│ │ MEDIUM: 45                      │ │
│ └─────────────────────────────────┘ │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Policy Evaluation                   │
│ Rule: CRITICAL > 0 → FAIL           │
│ ❌ Build Failed                     │
│                                     │
│ Slack 알림:                          │
│ "🚨 Build #123 failed"              │
│ "3 Critical vulnerabilities found"  │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 개발자 수정                            │
│ Dockerfile:                         │
│ FROM ubuntu:20.04 → ubuntu:22.04    │
│ RUN apt-get upgrade -y              │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 재스캔                                │
│ CRITICAL: 0 ✅                      │
│ HIGH: 2                             │
│ → ✅ Build Passed                   │
│ → 자동 배포                           │
└─────────────────────────────────────┘

Total time: ~30분
vs 수동: ~1-2주

DevSecOps 파이프라인:
┌────────────────────────────────────┐
│ 1. Pre-commit                      │
│    - git-secrets                   │
│    - pre-commit hooks              │
├────────────────────────────────────┤
│ 2. Build                           │
│    - Dockerfile lint               │
│    - Dependency check              │
├────────────────────────────────────┤
│ 3. Image Scan                      │
│    - OS vulnerabilities            │
│    - Application libraries         │
├────────────────────────────────────┤
│ 4. Policy Gate                     │
│    - Severity thresholds           │
│    - License compliance            │
├────────────────────────────────────┤
│ 5. Sign & Push                     │
│    - Image signing                 │
│    - Registry scan                 │
├────────────────────────────────────┤
│ 6. Deploy                          │
│    - Admission control             │
│    - Runtime monitoring            │
└────────────────────────────────────┘
```

**실무 영향:**
- 취약점 조기 발견 → 수정 비용 90% 절감
- 배포 차단 → 보안 사고 95% 감소
- 자동화 → 인력 80% 절감
- 지속적 스캔 → 제로데이 대응

---

## 🔧 실습 1: 완전한 CI/CD 보안 파이프라인

### Step 1: GitLab CI/CD 통합

```yaml
# .gitlab-ci.yml
stages:
  - secrets
  - lint
  - build
  - scan
  - test
  - deploy

variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  TRIVY_VERSION: "0.48.0"

# Stage 1: Secret Detection
detect-secrets:
  stage: secrets
  image: python:3.11-slim
  before_script:
    - pip install detect-secrets
  script:
    - detect-secrets scan --baseline .secrets.baseline
    - |
      if [ $? -ne 0 ]; then
        echo "🚨 Secrets detected! Review and update baseline."
        exit 1
      fi
  allow_failure: false

git-secrets:
  stage: secrets
  image: alpine/git
  before_script:
    - apk add --no-cache bash wget
    - wget -O git-secrets.tar.gz https://github.com/awslabs/git-secrets/archive/master.tar.gz
    - tar -xzf git-secrets.tar.gz
    - cd git-secrets-master && make install
  script:
    - git secrets --scan
  allow_failure: false

# Stage 2: Dockerfile Linting
hadolint:
  stage: lint
  image: hadolint/hadolint:latest-alpine
  script:
    - hadolint Dockerfile
  allow_failure: false

dockerfile-lint:
  stage: lint
  image: node:18-alpine
  before_script:
    - npm install -g dockerfile_lint
  script:
    - dockerfile_lint -f Dockerfile
  allow_failure: true  # Warning only

# Stage 3: Build Image
build:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_NAME .
    - docker push $IMAGE_NAME
  only:
    - main
    - develop

# Stage 4: Security Scanning
trivy-scan:
  stage: scan
  image: aquasec/trivy:$TRIVY_VERSION
  variables:
    GIT_STRATEGY: none
  script:
    # Info scan (no failure)
    - trivy image --severity LOW,MEDIUM $IMAGE_NAME

    # Critical/High scan (fail on findings)
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE_NAME

    # Generate reports
    - trivy image --format json -o trivy-report.json $IMAGE_NAME
    - trivy image --format sarif -o trivy-results.sarif $IMAGE_NAME
    - trivy image --format cyclonedx -o sbom.json $IMAGE_NAME
    
    # HTML report
    - trivy image --format template --template "@contrib/html.tpl" -o trivy-report.html $IMAGE_NAME

  artifacts:
    when: always
    reports:
      container_scanning: trivy-results.sarif
    paths:
      - trivy-report.json
      - trivy-report.html
      - sbom.json
    expire_in: 30 days
  allow_failure: false

grype-scan:
  stage: scan
  image: anchore/grype:latest
  variables:
    GIT_STRATEGY: none
  script:
    - grype $IMAGE_NAME --fail-on critical
    - grype $IMAGE_NAME -o json > grype-report.json
  artifacts:
    paths:
      - grype-report.json
    expire_in: 30 days
  allow_failure: true

snyk-scan:
  stage: scan
  image: snyk/snyk:docker
  variables:
    GIT_STRATEGY: none
  script:
    - snyk auth $SNYK_TOKEN
    - snyk container test $IMAGE_NAME --severity-threshold=high
    - snyk container monitor $IMAGE_NAME
  allow_failure: true
  only:
    - main

# Stage 5: Integration Tests
integration-test:
  stage: test
  image: docker:24-dind
  services:
    - docker:24-dind
  script:
    - docker run --rm $IMAGE_NAME /app/tests/integration.sh
  only:
    - main
    - develop

# Stage 6: Deploy
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp myapp=$IMAGE_NAME -n staging
    - kubectl rollout status deployment/myapp -n staging
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp myapp=$IMAGE_NAME -n production
    - kubectl rollout status deployment/myapp -n production
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
  needs:
    - trivy-scan
    - integration-test
```

### Step 2: GitHub Actions 워크플로우

```yaml
# .github/workflows/security-scan.yml
name: Security Scan Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    # Daily scan at 2 AM
    - cron: '0 2 * * *'

env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}:${{ github.sha }}

jobs:
  secret-detection:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: TruffleHog Scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

      - name: GitGuardian Scan
        uses: GitGuardian/ggshield-action@v1
        env:
          GITHUB_PUSH_BEFORE_SHA: ${{ github.event.before }}
          GITHUB_PUSH_BASE_SHA: ${{ github.event.base }}
          GITHUB_DEFAULT_BRANCH: ${{ github.event.repository.default_branch }}
          GITGUARDIAN_API_KEY: ${{ secrets.GITGUARDIAN_API_KEY }}

  dockerfile-lint:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Hadolint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          failure-threshold: warning

  build-and-scan:
    runs-on: ubuntu-latest
    needs: [secret-detection, dockerfile-lint]
    permissions:
      contents: read
      packages: write
      security-events: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build image
        uses: docker/build-push-action@v4
        with:
          context: .
          load: true
          tags: ${{ env.IMAGE_NAME }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run Trivy (CRITICAL only - fail build)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}
          format: 'table'
          severity: 'CRITICAL'
          exit-code: '1'

      - name: Run Trivy (HIGH - warn only)
        uses: aquasecurity/trivy-action@master
        if: always()
        with:
          image-ref: ${{ env.IMAGE_NAME }}
          format: 'table'
          severity: 'HIGH'
          exit-code: '0'

      - name: Generate Trivy SARIF report
        uses: aquasecurity/trivy-action@master
        if: always()
        with:
          image-ref: ${{ env.IMAGE_NAME }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH,MEDIUM'

      - name: Upload Trivy SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

      - name: Generate SBOM
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}
          format: 'cyclonedx'
          output: 'sbom.json'

      - name: Generate HTML Report
        if: always()
        run: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image \
            --format template \
            --template "@contrib/html.tpl" \
            -o trivy-report.html \
            ${{ env.IMAGE_NAME }}

      - name: Upload scan results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: security-reports
          path: |
            trivy-results.sarif
            trivy-report.html
            sbom.json
          retention-days: 30

      - name: Snyk Container Scan
        uses: snyk/actions/docker@master
        continue-on-error: true
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          image: ${{ env.IMAGE_NAME }}
          args: --severity-threshold=high

      - name: Push image (if passed)
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE_NAME }}

  policy-check:
    runs-on: ubuntu-latest
    needs: build-and-scan
    steps:
      - name: Download scan results
        uses: actions/download-artifact@v3
        with:
          name: security-reports

      - name: Policy Evaluation
        run: |
          #!/bin/bash
          
          # Extract vulnerability counts from SARIF
          CRITICAL=$(jq '[.runs[].results[] | select(.level=="error")] | length' trivy-results.sarif)
          HIGH=$(jq '[.runs[].results[] | select(.level=="warning")] | length' trivy-results.sarif)
          
          echo "Vulnerability Summary:"
          echo "CRITICAL: $CRITICAL"
          echo "HIGH: $HIGH"
          
          # Policy rules
          if [ "$CRITICAL" -gt 0 ]; then
            echo "❌ Policy FAILED: Critical vulnerabilities found"
            exit 1
          fi
          
          if [ "$HIGH" -gt 5 ]; then
            echo "⚠️  Policy WARNING: More than 5 High vulnerabilities"
            exit 1
          fi
          
          echo "✅ Policy PASSED"

      - name: Comment PR with results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const sarif = JSON.parse(fs.readFileSync('trivy-results.sarif', 'utf8'));
            
            const critical = sarif.runs[0].results.filter(r => r.level === 'error').length;
            const high = sarif.runs[0].results.filter(r => r.level === 'warning').length;
            
            const comment = `## 🔒 Security Scan Results
            
            | Severity | Count |
            |----------|-------|
            | CRITICAL | ${critical} |
            | HIGH | ${high} |
            
            ${critical > 0 ? '❌ **Build will fail due to critical vulnerabilities**' : '✅ **No critical vulnerabilities found**'}
            
            [View detailed report](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
```

### Step 3: Jenkins Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        IMAGE_NAME = "myapp"
        IMAGE_TAG = "${BUILD_NUMBER}"
        REGISTRY = "registry.example.com"
        TRIVY_VERSION = "0.48.0"
    }
    
    stages {
        stage('Secret Detection') {
            parallel {
                stage('git-secrets') {
                    steps {
                        sh '''
                            git secrets --scan || {
                                echo "Secrets detected!"
                                exit 1
                            }
                        '''
                    }
                }
                
                stage('TruffleHog') {
                    steps {
                        sh '''
                            docker run --rm \
                              -v $(pwd):/proj \
                              trufflesecurity/trufflehog:latest \
                              filesystem /proj --fail
                        '''
                    }
                }
            }
        }
        
        stage('Dockerfile Lint') {
            steps {
                sh '''
                    docker run --rm -i \
                      hadolint/hadolint:latest \
                      < Dockerfile
                '''
            }
        }
        
        stage('Build Image') {
            steps {
                script {
                    docker.build("${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Security Scan') {
            parallel {
                stage('Trivy') {
                    steps {
                        sh """
                            docker run --rm \
                              -v /var/run/docker.sock:/var/run/docker.sock \
                              aquasec/trivy:${TRIVY_VERSION} image \
                              --exit-code 1 \
                              --severity CRITICAL,HIGH \
                              --format json \
                              -o trivy-report.json \
                              ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        """
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                        }
                    }
                }
                
                stage('Grype') {
                    steps {
                        sh """
                            docker run --rm \
                              -v /var/run/docker.sock:/var/run/docker.sock \
                              anchore/grype:latest \
                              ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                              --fail-on critical \
                              -o json > grype-report.json
                        """
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'grype-report.json', allowEmptyArchive: true
                        }
                    }
                }
            }
        }
        
        stage('Policy Gate') {
            steps {
                script {
                    def trivyReport = readJSON file: 'trivy-report.json'
                    def critical = 0
                    def high = 0
                    
                    trivyReport.Results.each { result ->
                        result.Vulnerabilities.each { vuln ->
                            if (vuln.Severity == 'CRITICAL') critical++
                            if (vuln.Severity == 'HIGH') high++
                        }
                    }
                    
                    echo "Critical: ${critical}, High: ${high}"
                    
                    if (critical > 0) {
                        error("Build failed: ${critical} critical vulnerabilities")
                    }
                    
                    if (high > 10) {
                        error("Build failed: ${high} high vulnerabilities (threshold: 10)")
                    }
                }
            }
        }
        
        stage('Push Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry("https://${REGISTRY}", 'registry-credentials') {
                        docker.image("${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}").push()
                        docker.image("${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}").push('latest')
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    kubectl set image deployment/myapp \
                      myapp=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                      -n production
                    
                    kubectl rollout status deployment/myapp -n production
                """
            }
        }
    }
    
    post {
        always {
            // Generate HTML report
            sh """
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  aquasec/trivy:${TRIVY_VERSION} image \
                  --format template \
                  --template "@contrib/html.tpl" \
                  -o trivy-report.html \
                  ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
            """
            
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'trivy-report.html',
                reportName: 'Trivy Security Report'
            ])
        }
        
        failure {
            emailext (
                subject: "Build Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: """
                    Build failed due to security vulnerabilities.
                    
                    View details: ${env.BUILD_URL}
                    
                    Trivy Report: ${env.BUILD_URL}Trivy_20Security_20Report/
                """,
                to: 'security-team@example.com'
            )
            
            // Slack notification
            slackSend(
                color: 'danger',
                message: """
                    🚨 Security Scan Failed
                    Job: ${env.JOB_NAME}
                    Build: ${env.BUILD_NUMBER}
                    Details: ${env.BUILD_URL}
                """
            )
        }
        
        success {
            slackSend(
                color: 'good',
                message: """
                    ✅ Security Scan Passed
                    Job: ${env.JOB_NAME}
                    Build: ${env.BUILD_NUMBER}
                    Image: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            )
        }
    }
}
```

---

## 🔧 실습 2: 정책 기반 게이팅

### Step 1: OPA (Open Policy Agent) 정책

```bash
# OPA 정책 파일
cat > image-policy.rego <<'EOF'
package imageadmission

# 기본 거부
default allow = false

# 허용 조건
allow {
    not has_critical_vulnerabilities
    not has_excessive_high_vulnerabilities
    has_required_labels
    from_trusted_registry
}

# Critical 취약점 체크
has_critical_vulnerabilities {
    input.scan_result.vulnerabilities[_].severity == "CRITICAL"
}

# High 취약점 임계값 체크
has_excessive_high_vulnerabilities {
    high_count := count([v | v := input.scan_result.vulnerabilities[_]; v.severity == "HIGH"])
    high_count > 5
}

# 필수 레이블 체크
has_required_labels {
    input.image.labels["maintainer"]
    input.image.labels["version"]
}

# 신뢰할 수 있는 레지스트리
from_trusted_registry {
    startswith(input.image.name, "registry.example.com/")
}

# 거부 이유
deny[msg] {
    has_critical_vulnerabilities
    msg := "Image contains critical vulnerabilities"
}

deny[msg] {
    has_excessive_high_vulnerabilities
    msg := "Image contains more than 5 high vulnerabilities"
}

deny[msg] {
    not has_required_labels
    msg := "Image missing required labels (maintainer, version)"
}

deny[msg] {
    not from_trusted_registry
    msg := "Image not from trusted registry"
}
EOF

# OPA 테스트 데이터
cat > test-input.json <<'EOF'
{
  "image": {
    "name": "registry.example.com/myapp:v1.0",
    "labels": {
      "maintainer": "devops@example.com",
      "version": "1.0.0"
    }
  },
  "scan_result": {
    "vulnerabilities": [
      {
        "id": "CVE-2023-1234",
        "severity": "HIGH",
        "package": "openssl"
      },
      {
        "id": "CVE-2023-5678",
        "severity": "MEDIUM",
        "package": "curl"
      }
    ]
  }
}
EOF

# OPA 평가
opa eval -d image-policy.rego -i test-input.json "data.imageadmission.allow"
# true

opa eval -d image-policy.rego -i test-input.json "data.imageadmission.deny"
# []
```

### Step 2: Conftest를 사용한 Dockerfile 정책

```bash
# Conftest 정책
mkdir -p policy
cat > policy/dockerfile.rego <<'EOF'
package main

# 거부 규칙

# Root 사용자 금지
deny[msg] {
    input[i].Cmd == "user"
    input[i].Value[_] == "root"
    msg := "Running as root is not allowed"
}

# ADD 대신 COPY 사용
deny[msg] {
    input[i].Cmd == "add"
    msg := "Use COPY instead of ADD"
}

# 특정 포트 금지
deny[msg] {
    input[i].Cmd == "expose"
    port := input[i].Value[_]
    to_number(port) < 1024
    msg := sprintf("Exposing privileged port %s is not allowed", [port])
}

# 최신 태그 금지
deny[msg] {
    input[i].Cmd == "from"
    val := input[i].Value
    contains(val[0], ":latest")
    msg := "Using 'latest' tag is not allowed"
}

# apt-get update 후 설치 필수
deny[msg] {
    input[i].Cmd == "run"
    val := input[i].Value[_]
    contains(val, "apt-get install")
    not contains(val, "apt-get update")
    msg := "apt-get install must be preceded by apt-get update"
}

# 캐시 정리 필수
deny[msg] {
    input[i].Cmd == "run"
    val := input[i].Value[_]
    contains(val, "apt-get install")
    not contains(val, "rm -rf /var/lib/apt/lists")
    msg := "apt-get install must include cache cleanup"
}

# 경고 규칙

warn[msg] {
    input[i].Cmd == "run"
    val := input[i].Value[_]
    contains(val, "curl")
    not contains(val, "https://")
    msg := "Prefer HTTPS over HTTP for downloads"
}

warn[msg] {
    input[i].Cmd == "healthcheck"
    count(input[i]) == 0
    msg := "Consider adding a HEALTHCHECK instruction"
}
EOF

# Dockerfile 테스트
conftest test Dockerfile -p policy/

# 출력:
# FAIL - Dockerfile - Using 'latest' tag is not allowed
# FAIL - Dockerfile - Running as root is not allowed
# WARN - Dockerfile - Consider adding a HEALTHCHECK instruction
```

### Step 3: GitHub Actions Policy Gate

```yaml
# .github/workflows/policy-gate.yml
name: Policy Gate

on:
  pull_request:
    branches: [ main ]

jobs:
  policy-check:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Conftest - Dockerfile Policy
        uses: instrumenta/conftest-action@master
        with:
          files: Dockerfile
          policy: policy/dockerfile.rego

      - name: Build image
        run: docker build -t test-image .

      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: test-image
          format: 'json'
          output: 'trivy-results.json'

      - name: OPA - Image Policy
        run: |
          # Install OPA
          curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64
          chmod +x opa
          
          # Prepare input
          cat > input.json <<EOF
          {
            "image": {
              "name": "test-image",
              "labels": $(docker inspect test-image | jq '.[0].Config.Labels')
            },
            "scan_result": $(cat trivy-results.json)
          }
          EOF
          
          # Evaluate policy
          ./opa eval -d image-policy.rego -i input.json "data.imageadmission.allow" --format raw
          
          if [ $? -ne 0 ]; then
            ./opa eval -d image-policy.rego -i input.json "data.imageadmission.deny"
            exit 1
          fi

      - name: Policy Report
        if: always()
        run: |
          echo "## Policy Evaluation Results" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          ./opa eval -d image-policy.rego -i input.json "data.imageadmission" --format pretty >> $GITHUB_STEP_SUMMARY
```

---

## 🔧 실습 3: 지속적 모니터링

### Step 1: Scheduled Scanning

```yaml
# .github/workflows/scheduled-scan.yml
name: Scheduled Security Scan

on:
  schedule:
    # Daily at 2 AM UTC
    - cron: '0 2 * * *'
  workflow_dispatch:

jobs:
  scan-production-images:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        image:
          - webapp:latest
          - api:latest
          - worker:latest
          - database:latest

    steps:
      - name: Scan ${{ matrix.image }}
        run: |
          docker pull registry.example.com/${{ matrix.image }}
          
          trivy image \
            --severity CRITICAL,HIGH \
            --format json \
            -o scan-${{ matrix.image }}.json \
            registry.example.com/${{ matrix.image }}

      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: scan-results
          path: scan-*.json

  aggregate-results:
    needs: scan-production-images
    runs-on: ubuntu-latest
    steps:
      - name: Download scan results
        uses: actions/download-artifact@v3
        with:
          name: scan-results

      - name: Aggregate and report
        run: |
          #!/bin/bash
          
          total_critical=0
          total_high=0
          
          for file in scan-*.json; do
            image=$(basename $file .json | sed 's/scan-//')
            critical=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity=="CRITICAL")] | length' $file)
            high=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity=="HIGH")] | length' $file)
            
            echo "Image: $image"
            echo "  CRITICAL: $critical"
            echo "  HIGH: $high"
            
            total_critical=$((total_critical + critical))
            total_high=$((total_high + high))
          done
          
          echo ""
          echo "Total CRITICAL: $total_critical"
          echo "Total HIGH: $total_high"
          
          # Send to monitoring system
          curl -X POST https://monitoring.example.com/api/metrics \
            -H "Content-Type: application/json" \
            -d "{
              \"metric\": \"vulnerabilities\",
              \"critical\": $total_critical,
              \"high\": $total_high,
              \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
            }"

      - name: Create GitHub Issue if critical
        if: ${{ env.total_critical > 0 }}
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: '🚨 Critical Vulnerabilities in Production Images',
              body: `Critical vulnerabilities detected in scheduled scan.
              
              See workflow run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
              
              **Action Required:** Review and remediate immediately.`,
              labels: ['security', 'critical']
            })
```

### Step 2: Runtime Monitoring with Falco

```yaml
# falco-rules.yaml - 추가 규칙
- rule: Unexpected outbound connection
  desc: Detect unexpected outbound connections from containers
  condition: >
    container and fd.rport exists and
    fd.rip != "0.0.0.0" and
    not fd.rip in (allowed_outbound_ips)
  output: >
    Unexpected outbound connection
    (user=%user.name command=%proc.cmdline
    connection=%fd.name container_id=%container.id)
  priority: WARNING
  tags: [network, container]

- rule: Container privilege escalation
  desc: Detect privilege escalation in containers
  condition: >
    container and spawned_process and
    proc.name in (sudo, su) and
    not proc.args contains "healthcheck"
  output: >
    Privilege escalation attempt detected
    (user=%user.name command=%proc.cmdline
    container_id=%container.id container_name=%container.name)
  priority: CRITICAL
  tags: [container, privilege_escalation]

- rule: Suspicious file write
  desc: Detect writes to sensitive directories
  condition: >
    container and open_write and
    (fd.name startswith /etc or
     fd.name startswith /usr/bin or
     fd.name startswith /usr/sbin) and
    not container.image.repository in (allowed_images)
  output: >
    Suspicious file write detected
    (user=%user.name file=%fd.name command=%proc.cmdline
    container_id=%container.id)
  priority: ERROR
  tags: [container, filesystem]

- list: allowed_outbound_ips
  items: ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]

- list: allowed_images
  items: ["nginx", "postgres", "redis"]
```

```yaml
# Kubernetes Deployment with Falco
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: security
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      serviceAccountName: falco
      hostNetwork: true
      hostPID: true
      containers:
      - name: falco
        image: falcosecurity/falco:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: boot
          mountPath: /host/boot
          readOnly: true
        - name: modules
          mountPath: /host/lib/modules
          readOnly: true
        - name: usr
          mountPath: /host/usr
          readOnly: true
        - name: etc
          mountPath: /host/etc
          readOnly: true
        - name: config
          mountPath: /etc/falco
      volumes:
      - name: dev
        hostPath:
          path: /dev
      - name: proc
        hostPath:
          path: /proc
      - name: boot
        hostPath:
          path: /boot
      - name: modules
        hostPath:
          path: /lib/modules
      - name: usr
        hostPath:
          path: /usr
      - name: etc
        hostPath:
          path: /etc
      - name: config
        configMap:
          name: falco-config
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

# 형식
trivy image --format json IMAGE_NAME
trivy image --format sarif IMAGE_NAME
trivy image --format cyclonedx IMAGE_NAME  # SBOM

# HTML 리포트
trivy image --format template --template "@contrib/html.tpl" -o report.html IMAGE_NAME

# 파일시스템 스캔
trivy fs /path/to/project

# Git 저장소 스캔
trivy repo https://github.com/user/repo
```

### Grype

```bash
# 이미지 스캔
grype IMAGE_NAME

# 심각도 기반 실패
grype IMAGE_NAME --fail-on critical

# JSON 출력
grype IMAGE_NAME -o json

# 파일시스템 스캔
grype dir:/path/to/project
```

### Hadolint

```bash
# Dockerfile 린팅
hadolint Dockerfile

# 특정 룰 무시
hadolint --ignore DL3006 --ignore DL3008 Dockerfile

# JSON 출력
hadolint --format json Dockerfile
```

### Conftest

```bash
# 정책 테스트
conftest test Dockerfile -p policy/

# 모든 Dockerfile 테스트
conftest test -p policy/ --all-namespaces
```

### OPA

```bash
# 정책 평가
opa eval -d policy.rego -i input.json "data.package.rule"

# 서버 모드
opa run --server policy/
```

---

## 🎓 연습 문제

### 문제 1: GitHub Actions 파이프라인 구축

다음 요구사항을 만족하는 보안 파이프라인을 구축하세요:

1. Secret 감지 (TruffleHog)
2. Dockerfile 린팅 (Hadolint)
3. 이미지 스캔 (Trivy)
4. Critical 취약점 시 빌드 실패
5. SARIF 결과를 GitHub Security 탭에 업로드
6. PR에 스캔 결과 코멘트

<details>
<summary>힌트 보기</summary>

- `trufflesecurity/trufflehog` action 사용
- `hadolint/hadolint-action` 사용
- `aquasecurity/trivy-action` 사용
- `github/codeql-action/upload-sarif` 사용
- `actions/github-script` 로 PR 코멘트

</details>

<details>
<summary>정답 보기</summary>

위의 "Step 2: GitHub Actions 워크플로우" 참고

</details>

### 문제 2: OPA 정책 작성

다음 규칙을 OPA 정책으로 작성하세요:

1. Critical 취약점 0개
2. High 취약점 10개 이하
3. 이미지에 `maintainer` 레이블 필수
4. `registry.company.com`에서만 허용

<details>
<summary>정답 보기</summary>

위의 "Step 1: OPA 정책" 참고

</details>

### 문제 3: Scheduled 스캔 구현

매일 새벽 2시에 프로덕션 이미지를 스캔하고, Critical 취약점 발견 시 GitHub Issue를 자동 생성하세요.

<details>
<summary>정답 보기</summary>

위의 "Step 1: Scheduled Scanning" 참고

</details>

---

## 📌 핵심 요약

### 도구 비교

| 도구 | 타입 | 장점 | 단점 | 사용 |
|-----|------|------|------|------|
| **Trivy** | 이미지 스캔 | 빠름, 쉬움, 무료 | 정책 기능 약함 | CI/CD |
| **Grype** | 이미지 스캔 | 정확도 높음 | 느림 | 심층 분석 |
| **Snyk** | 종합 | 강력, 다양한 기능 | 유료 | 엔터프라이즈 |
| **Hadolint** | Dockerfile | 베스트 프랙티스 | 이미지 미스캔 | Pre-build |
| **Conftest** | 정책 | 유연함 | 학습 곡선 | Policy |
| **OPA** | 정책 엔진 | 강력함 | 복잡함 | 고급 정책 |

### 파이프라인 단계

```
1. Pre-commit
   ├─ git-secrets
   └─ pre-commit hooks

2. Build
   ├─ Hadolint (Dockerfile)
   └─ Dependency check

3. Scan
   ├─ Trivy (vulnerabilities)
   ├─ Grype (deep scan)
   └─ Snyk (licenses)

4. Policy
   ├─ OPA evaluation
   └─ Conftest rules

5. Deploy
   ├─ Sign image
   └─ Registry scan

6. Runtime
   ├─ Falco monitoring
   └─ Admission control
```

### Best Practices

**1. Shift Left:**
```
빠른 피드백 > 완벽한 검사
개발 단계에서 차단 > 프로덕션 사고
```

**2. 정책 설정:**
```yaml
Development:
  CRITICAL: warn
  HIGH: warn
  MEDIUM: info

Staging:
  CRITICAL: fail
  HIGH: warn (threshold: 10)
  MEDIUM: info

Production:
  CRITICAL: fail
  HIGH: fail (threshold: 5)
  MEDIUM: warn
```

**3. 자동화:**
```
Manual review → Automated scan
Weekly scan → Daily scan
Post-deploy → Pre-deploy
```

**4. 지속적 개선:**
```
1. 메트릭 수집
2. 트렌드 분석
3. 정책 조정
4. 도구 업데이트
```

---

<div align="center">

**[⬅️ 이전: User Namespaces](./06-User-Namespaces.md)** | **[다음: Compliance ➡️](./08-Compliance.md)**

</div>
