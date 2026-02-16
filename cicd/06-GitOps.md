# 06. GitOps - Git 기반 배포 자동화

## 🎯 이 챕터에서 배울 것

- **GitOps 개념**: Git을 배포의 Source of Truth로
- **ArgoCD**: Kubernetes GitOps 도구
- **Flux CD**: CNCF GitOps 엔진
- **워크플로우**: Pull vs Push 모델
- **자동 동기화**: Git → Kubernetes
- **실전 구현**: 프로덕션급 GitOps 파이프라인

## 📌 왜 중요한가?

**"GitOps는 Git을 통해 인프라와 애플리케이션을 선언적으로 관리합니다."**

```
GitOps의 핵심:

Traditional Deployment (Push 모델):
┌─────────────────────────────────────────────────┐
│ CI/CD Pipeline                                  │
│                                                 │
│ git push → Build → Test → kubectl apply         │
│                              │                  │
│                              └─→ Kubernetes     │
│                                   ↓             │
│                              직접 변경!           │
└─────────────────────────────────────────────────┘

문제점:
❌ kubectl 직접 실행 (위험)
❌ 누가 변경했는지 추적 어려움
❌ 실제 상태 vs 원하는 상태 불일치
❌ 롤백 복잡
❌ 여러 클러스터 관리 어려움

GitOps (Pull 모델):
┌─────────────────────────────────────────────────┐
│ GitOps Architecture                             │
│                                                 │
│  ┌──────────────┐                               │
│  │   Git Repo   │ ← Source of Truth             │
│  │              │                               │
│  │ deployment/  │                               │
│  │ - app.yaml   │                               │
│  │ - service... │                               │
│  └──────┬───────┘                               │
│         │                                       │
│         │ Watch & Sync                          │
│         ▼                                       │
│  ┌──────────────┐                               │
│  │   ArgoCD     │                               │
│  │   (Agent)    │                               │
│  └──────┬───────┘                               │
│         │                                       │
│         │ Apply Changes                         │
│         ▼                                       │
│  ┌──────────────┐                               │
│  │  Kubernetes  │                               │
│  │   Cluster    │                               │
│  └──────────────┘                               │
│                                                 │
│  변경 흐름:                                       │
│  1. 개발자 → Git에 YAML 커밋                       │
│  2. ArgoCD가 변경 감지                            │
│  3. 자동으로 Kubernetes 업데이트                    │
│  4. 상태 동기화 확인                                │
└─────────────────────────────────────────────────┘

장점:
✅ Git이 단일 진실 공급원
✅ 모든 변경 이력 추적
✅ 쉬운 롤백 (git revert)
✅ 선언적 관리
✅ 자동 동기화

GitOps Principles:
1. Declarative (선언적)
   - "무엇을 원하는지" 정의
   - "어떻게 하는지"가 아님

2. Versioned (버전 관리)
   - Git에 모든 상태 저장
   - 이력 추적 가능

3. Immutable (불변)
   - 직접 수정 금지
   - Git을 통해서만 변경

4. Pulled Automatically (자동 동기화)
   - Agent가 Git 감시
   - 자동으로 적용

Repository Structure:
┌─────────────────────────────────────────────────┐
│ gitops-repo/                                    │
│ ├── apps/                                       │
│ │   ├── frontend/                               │
│ │   │   ├── deployment.yaml                     │
│ │   │   ├── service.yaml                        │
│ │   │   └── ingress.yaml                        │
│ │   └── backend/                                │
│ │       ├── deployment.yaml                     │
│ │       └── service.yaml                        │
│ ├── infrastructure/                             │
│ │   ├── namespaces.yaml                         │
│ │   └── rbac.yaml                               │
│ └── environments/                               │
│     ├── dev/                                    │
│     │   └── kustomization.yaml                  │
│     ├── staging/                                │
│     │   └── kustomization.yaml                  │
│     └── prod/                                   │
│         └── kustomization.yaml                  │
└─────────────────────────────────────────────────┘

ArgoCD vs Flux CD:
┌──────────────┬──────────────┬──────────────┐
│ 특성          │ ArgoCD       │ Flux CD      │
├──────────────┼──────────────┼──────────────┤
│ UI           │ ✅ 풍부한 UI  │ ❌ CLI only  │
├──────────────┼──────────────┼──────────────┤
│ Multi-cluster│ ✅ 기본 지원   │ ⚠️ 설정 필요   │
├──────────────┼──────────────┼──────────────┤
│ CNCF         │ ❌           │ ✅ Graduated │
├──────────────┼──────────────┼──────────────┤
│ 학습 곡선      │ 낮음          │ 중간          │
└──────────────┴──────────────┴──────────────┘
```

**실무 영향:**
- **추적성**: 모든 변경 Git 이력으로
- **안정성**: 선언적 상태 관리
- **속도**: 자동 동기화 (수 분)
- **규모**: 수백 개 클러스터 관리

---

## 🔬 Deep Dive

### 1. GitOps 워크플로우

#### 배포 프로세스

```
1. 개발자가 코드 변경
   git commit -m "Update image version"
   git push origin main

2. CI 파이프라인
   - 테스트 실행
   - 이미지 빌드
   - 레지스트리 푸시
   - Git에 매니페스트 업데이트

3. ArgoCD
   - Git 변경 감지 (3분마다 또는 Webhook)
   - 현재 상태 vs 원하는 상태 비교
   - Kubernetes에 변경 적용
   - 동기화 상태 확인

4. 검증
   - Health Check
   - Rollout 확인
   - Slack 알림
```

#### Image Updater

```yaml
# 자동 이미지 업데이트
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=ghcr.io/user/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
spec:
  ...
```

---

## 🔧 실습 1: ArgoCD 설치 및 설정

### Step 1: ArgoCD 설치

```bash
# Namespace 생성
kubectl create namespace argocd

# ArgoCD 설치
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 초기 admin 비밀번호
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port Forward (로컬 접속)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 웹 UI 접속
# https://localhost:8080
# Username: admin
# Password: (위에서 확인한 비밀번호)
```

### Step 2: ArgoCD CLI 설치

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# 로그인
argocd login localhost:8080
# Username: admin
# Password: ...
```

### Step 3: 첫 애플리케이션 배포

```bash
# Git 저장소 준비
# gitops-demo/
# └── apps/
#     └── guestbook/
#         ├── deployment.yaml
#         └── service.yaml

# ArgoCD Application 생성
argocd app create guestbook \
  --repo https://github.com/user/gitops-demo \
  --path apps/guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# 동기화
argocd app sync guestbook

# 상태 확인
argocd app get guestbook

# 실시간 감시
argocd app wait guestbook
```

---

## 🔧 실습 2: Application 정의 (YAML)

### Step 1: Application Manifest

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  # 프로젝트
  project: default
  
  # Git 소스
  source:
    repoURL: https://github.com/user/gitops-demo
    targetRevision: main
    path: apps/myapp
    
    # Kustomize (선택)
    kustomize:
      namePrefix: prod-
      commonLabels:
        env: production
  
  # 대상 클러스터
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  
  # 동기화 정책
  syncPolicy:
    automated:
      prune: true        # 삭제된 리소스 정리
      selfHeal: true     # 수동 변경 자동 복구
      allowEmpty: false
    
    syncOptions:
      - CreateNamespace=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

```bash
# 적용
kubectl apply -f argocd/application.yaml

# 확인
argocd app list
```

---

## 🔧 실습 3: 멀티 환경 관리 (Kustomize)

### Step 1: Repository 구조

```bash
gitops-demo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

### Step 2: Base 정의

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1  # 기본값
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
```

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
```

### Step 3: Overlay (환경별)

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namePrefix: dev-

replicas:
  - name: myapp
    count: 1

images:
  - name: myapp
    newTag: dev-abc1234

configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=DEBUG
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
  - ../../base

namePrefix: prod-

replicas:
  - name: myapp
    count: 3

images:
  - name: myapp
    newTag: v1.2.3

configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=ERROR

# Resource Limits
patches:
  - target:
      kind: Deployment
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: "256Mi"
```

### Step 4: ArgoCD Application (환경별)

```yaml
# argocd/dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/user/gitops-demo
    path: overlays/dev
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# argocd/prod.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/user/gitops-demo
    path: overlays/prod
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 🔧 실습 4: CI/CD + GitOps 통합

### Step 1: CI 파이프라인 (이미지 빌드)

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      
      - name: Push to registry
        run: |
          docker tag myapp:${{ github.sha }} ghcr.io/user/myapp:${{ github.sha }}
          docker push ghcr.io/user/myapp:${{ github.sha }}
      
      # GitOps: 매니페스트 업데이트
      - name: Update manifest
        run: |
          git clone https://github.com/user/gitops-demo.git
          cd gitops-demo/overlays/dev
          
          # Kustomize로 이미지 태그 업데이트
          kustomize edit set image myapp=ghcr.io/user/myapp:${{ github.sha }}
          
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "Update image to ${{ github.sha }}"
          git push
```

### Step 2: Image Updater (자동)

```bash
# ArgoCD Image Updater 설치
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# Application에 Annotation 추가
kubectl annotate app myapp-dev -n argocd \
  argocd-image-updater.argoproj.io/image-list=myapp=ghcr.io/user/myapp \
  argocd-image-updater.argoproj.io/myapp.update-strategy=latest

# 5분마다 최신 이미지 확인 및 자동 업데이트
```

---

## 🔧 실습 5: Flux CD 사용

### Step 1: Flux 설치

```bash
# Flux CLI 설치
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap (GitHub)
flux bootstrap github \
  --owner=user \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/my-cluster \
  --personal

# 생성되는 구조:
# fleet-infra/
# └── clusters/
#     └── my-cluster/
#         └── flux-system/
```

### Step 2: GitRepository 정의

```yaml
# clusters/my-cluster/apps.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/user/gitops-demo
  ref:
    branch: main
```

### Step 3: Kustomization (배포)

```yaml
# clusters/my-cluster/apps-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 5m
  path: ./overlays/prod
  prune: true
  sourceRef:
    kind: GitRepository
    name: apps
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp
      namespace: prod
```

### Step 4: 이미지 자동 업데이트

```yaml
# Image Repository 감시
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: ghcr.io/user/myapp
  interval: 1m

---
# 이미지 정책
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: 1.x.x

---
# 자동 업데이트
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: apps
  git:
    commit:
      author:
        name: Flux
        email: flux@users.noreply.github.com
      messageTemplate: 'Update image to {{range .Updated.Images}}{{println .}}{{end}}'
  update:
    path: ./overlays/prod
    strategy: Setters
```

---

## 🔧 실습 6: Rollback 및 복구

### Step 1: Git을 통한 롤백

```bash
# 현재 상태 확인
kubectl get deployment myapp -n prod -o yaml | grep image:

# Git 이력 확인
git log --oneline apps/myapp/deployment.yaml

# 이전 버전으로 롤백
git revert HEAD
git push

# ArgoCD가 자동으로 이전 버전 배포
```

### Step 2: ArgoCD Rollback

```bash
# 이력 확인
argocd app history myapp-prod

# ID  DATE         REVISION
# 5   2024-01-15   abc1234  (현재)
# 4   2024-01-14   xyz5678
# 3   2024-01-13   def9012

# 롤백
argocd app rollback myapp-prod 4

# 확인
argocd app get myapp-prod
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 도구                  │ 용도                        │
├──────────────────────┼────────────────────────────┤
│ ArgoCD               │ UI, Multi-cluster          │
├──────────────────────┼────────────────────────────┤
│ Flux CD              │ CNCF, Native               │
├──────────────────────┼────────────────────────────┤
│ Kustomize            │ 환경별 오버레이                │
├──────────────────────┼────────────────────────────┤
│ Helm                 │ 패키징                       │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. Git = Single Source of Truth
2. 환경별 Repository 분리
3. 자동 동기화
4. Webhook 활용
5. RBAC 설정
```

---

## 🎓 연습 문제

### 문제 1: 수동으로 kubectl apply를 실행하면 어떻게 되는가?

<details>
<summary>정답 보기</summary>

**Self-Heal 설정 시:**
```yaml
syncPolicy:
  automated:
    selfHeal: true
```

```bash
# 1. 수동 변경
kubectl scale deployment myapp --replicas=5

# 2. ArgoCD가 감지 (수 초 내)
# "OutOfSync" 상태

# 3. 자동 복구 (selfHeal)
# Git의 상태로 되돌림
# replicas: 3 (Git에 정의된 값)

# 결과: 수동 변경 무시됨!
```

**Self-Heal 없을 때:**
```yaml
syncPolicy:
  automated:
    selfHeal: false
```

```bash
# 수동 변경 유지됨
# 하지만 "OutOfSync" 경고

# 수동 동기화 필요
argocd app sync myapp
```

**권장 사항:**
- 프로덕션: selfHeal: true
- 개발: selfHeal: false (디버깅 용이)

</details>

### 문제 2: 민감한 정보(Secret)를 Git에 어떻게 저장하는가?

<details>
<summary>정답 보기</summary>

**방법 1: Sealed Secrets**
```bash
# 설치
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Secret 생성
kubectl create secret generic mysecret \
  --from-literal=password=secret123 \
  --dry-run=client -o yaml > secret.yaml

# 암호화
kubeseal -f secret.yaml -w sealed-secret.yaml

# Git에 커밋 (암호화됨)
git add sealed-secret.yaml
git commit -m "Add sealed secret"

# 클러스터에서 자동 복호화
```

**방법 2: External Secrets**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secret
spec:
  secretStoreRef:
    name: vault-backend
  target:
    name: app-secret
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/myapp
        property: db_password
```

**방법 3: SOPS**
```bash
# 암호화
sops -e secret.yaml > secret.enc.yaml

# Git에 커밋
git add secret.enc.yaml
```

</details>

### 문제 3: 여러 클러스터를 어떻게 관리하는가?

<details>
<summary>정답 보기</summary>

**ArgoCD App of Apps:**
```yaml
# apps/root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
spec:
  source:
    path: apps/
  destination:
    server: https://kubernetes.default.svc

# apps/
# ├── cluster1.yaml
# ├── cluster2.yaml
# └── cluster3.yaml
```

**Cluster 등록:**
```bash
# 클러스터 추가
argocd cluster add cluster2-context

# 확인
argocd cluster list

# Application 배포
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-cluster2
spec:
  destination:
    server: https://cluster2.example.com
    namespace: prod
```

**ApplicationSet (여러 클러스터):**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-clusters
spec:
  generators:
    - list:
        elements:
          - cluster: cluster1
            url: https://cluster1.example.com
          - cluster: cluster2
            url: https://cluster2.example.com
  
  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      source:
        path: overlays/{{cluster}}
      destination:
        server: '{{url}}'
```

</details>

---

## 📌 핵심 요약

```
GitOps 핵심:
1. Git = Source of Truth
2. 선언적 상태 관리
3. 자동 동기화
4. Pull 모델
5. 이력 추적

Best Practices:
✅ ArgoCD/Flux 사용
✅ Kustomize로 환경 관리
✅ Sealed Secrets
✅ 자동 이미지 업데이트
✅ RBAC 설정
```

---

## 📚 참고 자료

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Flux CD Documentation](https://fluxcd.io/)
- [GitOps Principles](https://opengitops.dev/)

---

## 🤔 생각해볼 문제

1. GitOps와 전통적 CI/CD의 차이는?
2. Git Repository를 어떻게 구조화하는가?
3. 긴급 배포 시 GitOps를 우회해야 하는가?

> 💡 **답변**:
> 
> **1) GitOps vs Traditional CI/CD:**
> 
> ```
> Traditional (Push):
> CI/CD → kubectl apply
> - 직접 변경
> - 추적 어려움
> 
> GitOps (Pull):
> Git → ArgoCD → K8s
> - 간접 변경
> - Git 이력 보존
> 
> 장점:
> ✅ 감사 추적
> ✅ 쉬운 롤백
> ✅ 선언적 상태
> ```
> 
> **2) Repository 구조:**
> 
> ```
> 방법 1: Monorepo
> gitops-repo/
> ├── apps/
> └── infrastructure/
> 
> 방법 2: App별 분리
> app1-gitops/
> app2-gitops/
> infra-gitops/
> 
> 권장: Monorepo (소규모)
>       분리 (대규모, 팀별)
> ```
> 
> **3) 긴급 배포:**
> 
> ```
> ❌ GitOps 우회하지 말 것
> 
> ✅ 빠른 GitOps:
> 1. Git 커밋
> 2. Webhook 트리거 (즉시)
> 3. 수동 Sync
>    argocd app sync myapp --force
> 
> 또는:
> 1. Hotfix Branch
> 2. PR 없이 Main 머지
> 3. 자동 배포
> ```

---

<div align="center">

**[⬅️ 이전: Security Scanning](./05-Security-Scanning.md)** | **[다음: Deployment Strategies ➡️](./07-Deployment-Strategies.md)**

</div>
