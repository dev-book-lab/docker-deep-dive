# 07. Deployment Strategies - Blue/Green, Canary

## 🎯 이 챕터에서 배울 것

- **배포 전략**: Rolling, Blue/Green, Canary
- **무중단 배포**: Zero-Downtime Deployment
- **점진적 롤아웃**: 트래픽 분산
- **자동 롤백**: 에러 감지 및 복구
- **A/B 테스팅**: 프로덕션 실험
- **실전 구현**: Argo Rollouts, Flagger

## 📌 왜 중요한가?

**"배포 전략은 위험을 최소화하고 안정적으로 새 버전을 출시합니다."**

```
Deployment Strategies의 핵심:

Naive Deployment (위험):
┌─────────────────────────────────────────────────┐
│ All-at-Once Deployment                          │
│                                                 │
│ v1.0.0 (3 pods) → ❌ 모두 삭제                    │
│                  ↓                              │
│ v2.0.0 (3 pods) → 새로 생성                       │
│                                                 │
│ 문제:                                            │
│ ⏱️ 다운타임 (30초~1분)                             │
│ 💥 버그 발견 → 모든 사용자 영향                       │
│ 🔄 롤백 느림 (재배포 필요)                           │
└─────────────────────────────────────────────────┘

Safe Deployment Strategies:

1. Rolling Update (점진적):
┌─────────────────────────────────────────────────┐
│ Time:  t0      t1      t2      t3               │
│                                                 │
│ v1.0  [●][●][●]                                 │
│ v2.0           [●]     [●][●]  [●][●][●]        │
│                                                 │
│ 트래픽: 100%   75%     50%     0%                 │
│        v1     v1     v1/v2    v2                │
│                                                 │
│ 장점: 무중단, 리소스 효율적                           │
│ 단점: 느린 롤백, 버전 혼재                           │
└─────────────────────────────────────────────────┘

2. Blue/Green (즉시 전환):
┌─────────────────────────────────────────────────┐
│ Before:                                         │
│  Blue (v1.0)  [●][●][●] ← 100% 트래픽             │
│  Green (v2.0) [○][○][○] ← 0% 트래픽               │
│                                                 │
│ After (스위치):                                   │
│  Blue (v1.0)  [●][●][●] ← 0% 트래픽 (대기)         │
│  Green (v2.0) [●][●][●] ← 100% 트래픽             │
│                                                 │
│ 장점: 즉시 롤백, 버전 혼재 없음                        │
│ 단점: 2배 리소스 필요                                │
└─────────────────────────────────────────────────┘

3. Canary (점진적 확인):
┌─────────────────────────────────────────────────┐
│ Stage 1 (5%):                                   │
│  Stable (v1.0)  [●][●][●][●] ← 95%              │
│  Canary (v2.0)  [●]          ← 5%               │
│                  ↓                              │
│  메트릭 확인 (에러율, 레이턴시)                        │
│                                                 │
│ Stage 2 (25%):                                  │
│  Stable (v1.0)  [●][●][●] ← 75%                 │
│  Canary (v2.0)  [●]       ← 25%                 │
│                  ↓                              │
│  메트릭 확인                                       │
│                                                 │
│ Stage 3 (100%):                                 │
│  Stable (v1.0)  [○][○][○] ← 0%                  │
│  Canary (v2.0)  [●][●][●] ← 100%                │
│                                                 │
│ 장점: 위험 최소화, 자동 롤백                          │
│ 단점: 복잡도, 시간 소요                              │
└─────────────────────────────────────────────────┘

전략 비교:
┌────────────┬────────┬────────┬────────┬────────┐
│ 전략        │ 속도    │ 안전성   │ 리소스   │ 복잡도  │
├────────────┼────────┼────────┼────────┼────────┤
│ Recreate   │ 빠름    │ 낮음    │ 낮음    │ 낮음     │
├────────────┼────────┼────────┼────────┼────────┤
│ Rolling    │ 중간    │ 중간    │ 낮음    │ 낮음    │
├────────────┼────────┼────────┼────────┼────────┤
│ Blue/Green │ 빠름    │ 높음    │ 높음    │ 중간     │
├────────────┼────────┼────────┼────────┼────────┤
│ Canary     │ 느림    │ 매우높음 │ 중간     │ 높음    │
└────────────┴────────┴────────┴────────┴────────┘

언제 어떤 전략?
┌─────────────────────┬──────────────────────┐
│ 상황                 │ 전략                  │
├─────────────────────┼──────────────────────┤
│ 개발/테스트            │ Recreate             │
├─────────────────────┼──────────────────────┤
│ 일반 서비스            │ Rolling Update       │
├─────────────────────┼──────────────────────┤
│ 중요 서비스            │ Blue/Green           │
├─────────────────────┼──────────────────────┤
│ 매우 중요 서비스        │ Canary               │
├─────────────────────┼──────────────────────┤
│ 신기능 실험            │ A/B Testing          │
└─────────────────────┴──────────────────────┘

자동 롤백 조건:
┌─────────────────────────────────────────────────┐
│ Canary with Auto-Rollback                       │
│                                                 │
│ 메트릭 수집:                                       │
│  - HTTP 에러율 (4xx, 5xx)                         │
│  - 응답 시간 (p95, p99)                           │
│  - 메모리/CPU 사용률                               │
│  - 커스텀 메트릭 (비즈니스 로직)                      │
│                                                 │
│ 임계값:                                          │
│  IF error_rate > 1% → 롤백                       │
│  IF p95_latency > 500ms → 롤백                   │
│  IF memory > 80% → 롤백                          │
│                                                 │
│ 자동 진행:                                        │
│  ✅ 메트릭 정상 → 다음 단계 (5% → 25% → 100%)        │
│  ❌ 메트릭 비정상 → 즉시 롤백 (100% → 0%)            │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **안정성**: 버그 영향 최소화 (5% 사용자만)
- **신뢰**: 자동 롤백으로 안심 배포
- **속도**: 문제없으면 빠른 확산
- **실험**: A/B 테스트로 최적화

---

## 🔬 Deep Dive

### 1. Kubernetes Rolling Update

#### 기본 Rolling Update

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # 동시에 1개까지 중단
      maxSurge: 1        # 동시에 1개까지 추가 생성
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2.0.0
```

#### 실행 흐름

```bash
# 배포
kubectl apply -f deployment.yaml

# 롤아웃 상태
kubectl rollout status deployment/myapp

# 실시간 확인
kubectl get pods -w

# Pod 생성/삭제 순서:
# myapp-v2-1 생성 → 준비 완료
# myapp-v1-1 삭제
# myapp-v2-2 생성 → 준비 완료
# myapp-v1-2 삭제
# ...
```

---

### 2. Argo Rollouts

#### 설치

```bash
# Argo Rollouts 설치
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# CLI 설치
brew install argoproj/tap/kubectl-argo-rollouts
```

---

## 🔧 실습 1: Rolling Update (Kubernetes 기본)

### Step 1: 배포 설정

```yaml
# deployment-rolling.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-rolling
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2   # 동시에 최대 2개 중단
      maxSurge: 2         # 동시에 최대 2개 추가
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
```

### Step 2: 배포 및 업데이트

```bash
# 초기 배포
kubectl apply -f deployment-rolling.yaml

# v2로 업데이트
kubectl set image deployment/myapp-rolling myapp=myapp:v2.0.0

# 롤아웃 진행 상황
kubectl rollout status deployment/myapp-rolling

# 히스토리
kubectl rollout history deployment/myapp-rolling

# 롤백
kubectl rollout undo deployment/myapp-rolling

# 특정 revision으로 롤백
kubectl rollout undo deployment/myapp-rolling --to-revision=2
```

---

## 🔧 실습 2: Blue/Green Deployment

### Step 1: Blue/Green 설정 (Argo Rollouts)

```yaml
# rollout-bluegreen.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-bluegreen
spec:
  replicas: 3
  revisionHistoryLimit: 2
  
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
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
  
  strategy:
    blueGreen:
      # Active Service (프로덕션 트래픽)
      activeService: myapp-active
      
      # Preview Service (테스트용)
      previewService: myapp-preview
      
      # 자동 승인 (false면 수동)
      autoPromotionEnabled: false
      
      # 자동 승인 대기 시간
      autoPromotionSeconds: 30
      
      # 이전 버전 유지 시간
      scaleDownDelaySeconds: 30
```

### Step 2: Service 정의

```yaml
# services.yaml
---
# Active Service (사용자 트래픽)
apiVersion: v1
kind: Service
metadata:
  name: myapp-active
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080

---
# Preview Service (테스트)
apiVersion: v1
kind: Service
metadata:
  name: myapp-preview
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

### Step 3: 배포 및 승인

```bash
# 배포
kubectl apply -f rollout-bluegreen.yaml
kubectl apply -f services.yaml

# 상태 확인
kubectl argo rollouts get rollout myapp-bluegreen

# 새 버전 배포
kubectl argo rollouts set image myapp-bluegreen myapp=myapp:v2.0.0

# 실시간 확인
kubectl argo rollouts get rollout myapp-bluegreen --watch

# Preview 테스트
curl http://myapp-preview/

# 승인 (Green → Active)
kubectl argo rollouts promote myapp-bluegreen

# 또는 자동 승인 (30초 후)
# autoPromotionEnabled: true

# 롤백
kubectl argo rollouts undo myapp-bluegreen
```

---

## 🔧 실습 3: Canary Deployment

### Step 1: Canary 설정

```yaml
# rollout-canary.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-canary
spec:
  replicas: 10
  
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
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
  
  strategy:
    canary:
      # Canary 단계
      steps:
      - setWeight: 10   # 10% 트래픽
      - pause: {duration: 5m}
      
      - setWeight: 25   # 25% 트래픽
      - pause: {duration: 5m}
      
      - setWeight: 50   # 50% 트래픽
      - pause: {duration: 5m}
      
      - setWeight: 75   # 75% 트래픽
      - pause: {duration: 5m}
      
      # 100% (완료)
      
      # Canary Service (Canary Pod만)
      canaryService: myapp-canary
      
      # Stable Service (Stable Pod만)
      stableService: myapp-stable
      
      # Traffic Routing (Istio/Nginx)
      trafficRouting:
        istio:
          virtualService:
            name: myapp
            routes:
            - primary
```

### Step 2: Istio VirtualService

```yaml
# virtualservice.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - name: primary
    route:
    - destination:
        host: myapp-stable
      weight: 100
    - destination:
        host: myapp-canary
      weight: 0
```

### Step 3: 배포

```bash
# 배포
kubectl apply -f rollout-canary.yaml

# 새 버전 배포
kubectl argo rollouts set image myapp-canary myapp=myapp:v2.0.0

# 실시간 확인
kubectl argo rollouts get rollout myapp-canary --watch

# 진행 상황:
# ├── ✔ Healthy - ReplicaSet "myapp-v2" (1/10 replicas)
# ├── ⊚ Paused - SetWeight 10% (5m remaining)
# 
# Stable: v1 (9 pods) - 90% traffic
# Canary: v2 (1 pod)  - 10% traffic

# 수동 승인 (다음 단계)
kubectl argo rollouts promote myapp-canary

# 중단 (롤백)
kubectl argo rollouts abort myapp-canary
```

---

## 🔧 실습 4: 자동 분석 및 롤백

### Step 1: AnalysisTemplate (메트릭 기반)

```yaml
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  # 성공률 체크
  - name: success-rate
    interval: 30s
    successCondition: result >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status!~"5.."}[1m]))
          /
          sum(rate(http_requests_total[1m]))
  
  # 응답 시간 체크
  - name: latency
    interval: 30s
    successCondition: result <= 0.5
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[1m])
          )
```

### Step 2: Canary with Analysis

```yaml
# rollout-canary-analysis.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-canary-auto
spec:
  replicas: 10
  
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
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080
  
  strategy:
    canary:
      steps:
      # 10% 배포
      - setWeight: 10
      
      # 분석 시작
      - analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: myapp-canary
      
      # 메트릭 정상 → 25%
      - setWeight: 25
      - pause: {duration: 2m}
      
      # 50%
      - setWeight: 50
      - analysis:
          templates:
          - templateName: success-rate
      
      # 100%
      - setWeight: 100
      
      # 자동 롤백 조건
      canaryService: myapp-canary
      stableService: myapp-stable
      
      # 분석 실패 시 자동 롤백
      abortScaleDownDelaySeconds: 30
```

---

## 🔧 실습 5: Flagger (Automated Canary)

### Step 1: Flagger 설치

```bash
# Flagger 설치 (Istio 사용)
kubectl apply -k github.com/fluxcd/flagger//kustomize/istio

# Prometheus 설치 (메트릭 수집)
kubectl apply -k github.com/fluxcd/flagger//kustomize/prometheus
```

### Step 2: Canary 정의

```yaml
# canary.yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: default
spec:
  # Target Deployment
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  
  # Service
  service:
    port: 80
    targetPort: 8080
  
  # Canary Analysis
  analysis:
    # 체크 간격
    interval: 1m
    
    # 임계값 (10번 성공 시 다음 단계)
    threshold: 10
    
    # 최대 가중치
    maxWeight: 50
    
    # 단계별 증가
    stepWeight: 5
    
    # 메트릭
    metrics:
    # HTTP 성공률
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 1m
    
    # 응답 시간
    - name: request-duration
      thresholdRange:
        max: 500
      interval: 1m
    
    # Webhook (커스텀 체크)
    webhooks:
    - name: load-test
      url: http://loadtester.default/
      timeout: 5s
      metadata:
        cmd: "hey -z 1m -q 10 -c 2 http://myapp-canary/"
```

### Step 3: 배포

```bash
# Deployment 생성
kubectl apply -f deployment.yaml

# Canary 생성
kubectl apply -f canary.yaml

# 상태 확인
kubectl get canary myapp -w

# 새 버전 배포 (이미지 변경)
kubectl set image deployment/myapp myapp=myapp:v2.0.0

# Flagger가 자동으로:
# 1. Canary 생성 (5%)
# 2. 메트릭 분석
# 3. 점진적 증가 (5% → 10% → ... → 50%)
# 4. 성공 → 100% 전환
# 5. 실패 → 자동 롤백
```

---

## 🔧 실습 6: A/B Testing

### Step 1: Header-based Routing

```yaml
# rollout-ab.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-ab
spec:
  replicas: 10
  
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
        image: myapp:v1.0.0
  
  strategy:
    canary:
      # A/B Testing (헤더 기반)
      trafficRouting:
        istio:
          virtualService:
            name: myapp
            routes:
            - primary
      
      steps:
      # Beta 사용자에게만 노출
      - setCanaryScale:
          replicas: 2
      
      # 무한 대기 (수동 제어)
      - pause: {}
```

### Step 2: VirtualService (조건부 라우팅)

```yaml
# virtualservice-ab.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - name: primary
    match:
    # Beta 사용자 (헤더)
    - headers:
        x-version:
          exact: "beta"
    route:
    - destination:
        host: myapp-canary
      weight: 100
  
  - name: default
    route:
    # 일반 사용자
    - destination:
        host: myapp-stable
      weight: 100
```

### Step 3: 테스트

```bash
# 일반 사용자 (Stable 버전)
curl http://myapp/
# Response: v1.0.0

# Beta 사용자 (Canary 버전)
curl -H "x-version: beta" http://myapp/
# Response: v2.0.0

# 메트릭 비교
# - v1 (Stable): 전환율 10%
# - v2 (Beta): 전환율 12%
# → v2가 우수 → 전체 배포
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 도구                  │ 용도                        │
├──────────────────────┼────────────────────────────┤
│ Kubernetes           │ Rolling Update (기본)       │
├──────────────────────┼────────────────────────────┤
│ Argo Rollouts        │ Blue/Green, Canary         │
├──────────────────────┼────────────────────────────┤
│ Flagger              │ 자동 Canary (Istio)         │
├──────────────────────┼────────────────────────────┤
│ Istio                │ Traffic Splitting          │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. Readiness Probe 필수
2. 점진적 롤아웃
3. 메트릭 기반 자동 롤백
4. 충분한 대기 시간
5. 모니터링 필수
```

---

## 🎓 연습 문제

### 문제 1: Canary가 실패하면 자동으로 롤백되는가?

<details>
<summary>정답 보기</summary>

**AnalysisTemplate 있을 때:**
```yaml
strategy:
  canary:
    steps:
    - setWeight: 10
    - analysis:
        templates:
        - templateName: success-rate
    # Analysis 실패 → 자동 롤백
```

**없을 때:**
```yaml
strategy:
  canary:
    steps:
    - setWeight: 10
    - pause: {duration: 5m}
    # 메트릭 확인 없음
    # 문제 있어도 계속 진행!
```

**권장 설정:**
```yaml
analysis:
  metrics:
  - name: error-rate
    successCondition: result < 0.01  # 1% 이하
    failureLimit: 3  # 3번 실패 시 롤백
```

</details>

### 문제 2: Blue/Green에서 리소스를 절약하려면?

<details>
<summary>정답 보기</summary>

**문제: 2배 리소스**
```
Blue (v1):  3 replicas
Green (v2): 3 replicas
총: 6 replicas (2배!)
```

**해결 1: 스케일 다운**
```yaml
strategy:
  blueGreen:
    activeService: myapp-active
    previewService: myapp-preview
    
    # 전환 후 이전 버전 즉시 삭제
    scaleDownDelaySeconds: 30
    
    # 또는 0 replicas로
    scaleDownDelayRevisionLimit: 0
```

**해결 2: Canary 사용**
```yaml
# 점진적 증가 (2배 리소스 순간만)
strategy:
  canary:
    steps:
    - setWeight: 50
    # Stable: 5 replicas (50%)
    # Canary: 5 replicas (50%)
    # 잠깐만 2배
```

</details>

### 문제 3: 배포가 멈췄을 때 어떻게 디버깅하는가?

<details>
<summary>정답 보기</summary>

**확인 순서:**

**1. Rollout 상태**
```bash
kubectl argo rollouts get rollout myapp

# Paused? 어디서?
# ├── ⊚ Paused - SetWeight 10%
```

**2. Analysis 결과**
```bash
kubectl get analysisrun -l rollout=myapp

# 실패한 메트릭?
kubectl describe analysisrun myapp-xxx
```

**3. Pod 상태**
```bash
kubectl get pods -l rollout=myapp

# CrashLoopBackOff?
# ImagePullBackOff?
```

**4. Readiness Probe**
```bash
kubectl describe pod myapp-xxx

# Readiness probe failed?
# Liveness probe failed?
```

**5. 로그**
```bash
kubectl logs -l rollout=myapp --tail=100
```

**일반적 원인:**
- Readiness Probe 실패
- Analysis 메트릭 임계값 초과
- 수동 승인 대기
- 이미지 Pull 실패

</details>

---

## 📌 핵심 요약

```
Deployment Strategies 핵심:
1. Rolling Update (기본)
2. Blue/Green (즉시 전환)
3. Canary (점진적, 안전)
4. 자동 분석 및 롤백
5. A/B Testing (실험)

Best Practices:
✅ Readiness Probe
✅ 점진적 롤아웃
✅ 메트릭 기반 자동 롤백
✅ Argo Rollouts/Flagger
✅ 충분한 대기 및 검증
```

---

## 📚 참고 자료

- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)
- [Flagger](https://flagger.app/)
- [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

---

## 🤔 생각해볼 문제

1. 모든 배포에 Canary를 사용해야 하는가?
2. Canary 단계를 어떻게 결정하는가?
3. 데이터베이스 스키마 변경은 어떻게 배포하는가?

> 💡 **답변**:
> 
> **1) Canary 사용 기준:**
> 
> ```
> ✅ Canary 사용:
> - 중요한 서비스
> - 트래픽 많음
> - 장애 영향 큼
> 
> ❌ Canary 불필요:
> - 개발/테스트 환경
> - 내부 도구
> - 트래픽 적음
> - 버그 영향 작음
> 
> 기본: Rolling Update
> 중요: Blue/Green
> 매우 중요: Canary
> ```
> 
> **2) Canary 단계 설정:**
> 
> ```yaml
> # 보수적 (안전)
> steps:
> - setWeight: 5
> - pause: {duration: 10m}
> - setWeight: 25
> - pause: {duration: 10m}
> - setWeight: 50
> - pause: {duration: 10m}
> 
> # 공격적 (빠름)
> steps:
> - setWeight: 25
> - pause: {duration: 2m}
> - setWeight: 50
> - pause: {duration: 2m}
> 
> 고려사항:
> - 트래픽 규모
> - 영향 범위
> - 비즈니스 중요도
> ```
> 
> **3) DB 스키마 변경:**
> 
> ```
> 방법 1: Expand-Contract
> 1. 새 컬럼 추가 (NULL 허용)
> 2. 앱 v2 배포 (새 컬럼 사용)
> 3. 데이터 마이그레이션
> 4. 구 컬럼 삭제
> 
> 방법 2: 버전 체크
> ALTER TABLE users ADD COLUMN email_v2;
> 
> v1: email 사용
> v2: email_v2 사용
> 
> Blue/Green 전환
> 
> 구 컬럼 삭제
> ```

---

<div align="center">

**[⬅️ 이전: GitOps](./06-GitOps.md)** | **[홈으로 🏠](../README.md)**

</div>
