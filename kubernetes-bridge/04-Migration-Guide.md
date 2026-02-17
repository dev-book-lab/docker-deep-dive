# 04. Migration Guide - Docker에서 Kubernetes로 마이그레이션

## 🎯 이 챕터에서 배울 것

- **마이그레이션 전략**: 점진적 vs 전면 전환
- **준비 단계**: 사전 점검 및 계획
- **변환 프로세스**: docker-compose → K8s
- **도구 활용**: Kompose, Helm
- **실전 마이그레이션**: 단계별 실행
- **검증 및 최적화**: 배포 후 점검
- **문제 해결**: 흔한 이슈와 해결책

## 📌 왜 중요한가?

**"마이그레이션은 단순히 YAML 변환이 아닙니다. 아키텍처 전환입니다."**

```
마이그레이션 필요성:

현재 (Docker Compose):
┌─────────────────────────────────────────────────┐
│ 한계:                                            │
│ ❌ 단일 서버 (확장 제한)                            │
│ ❌ 수동 복구                                      │
│ ❌ 제한적 로드 밸런싱                               │
│ ❌ 롤링 업데이트 어려움                              │
│ ❌ 자동 스케일링 없음                               │
│                                                 │
│ 신호:                                            │
│ ⚠️  트래픽 증가 (확장 필요)                          │
│ ⚠️  다운타임 비용 높음                              │
│ ⚠️  복잡한 마이크로서비스                            │
│ ⚠️  글로벌 배포 필요                                │
└─────────────────────────────────────────────────┘

미래 (Kubernetes):
┌─────────────────────────────────────────────────┐
│ 장점:                                            │
│ ✅ 무한 확장 (클러스터)                             │
│ ✅ 자동 복구 (Self-healing)                       │
│ ✅ 내장 로드 밸런싱                                 │
│ ✅ 롤링 업데이트 / 롤백                             │
│ ✅ HPA (자동 스케일링)                             │
│                                                 │
│ 트레이드오프:                                      │
│ ⚠️  복잡도 증가                                   │
│ ⚠️  학습 곡선                                     │
│ ⚠️  운영 오버헤드                                  │
│ ⚠️  비용 (초기)                                   │
└─────────────────────────────────────────────────┘

마이그레이션 전략:
┌─────────────────────────────────────────────────┐
│ 1. Big Bang (전면 전환)                           │
│    모든 서비스 동시 이동                             │
│    위험: 높음 | 속도: 빠름                           │
│                                                 │
│ 2. Strangler Fig (점진적 전환)                     │
│    서비스 하나씩 이동                                │
│    위험: 낮음 | 속도: 느림                           │
│                                                 │
│ 3. Hybrid (혼합)                                 │
│    일부 K8s, 일부 Docker                          │
│    위험: 중간 | 속도: 중간                          │
└─────────────────────────────────────────────────┘

마이그레이션 로드맵:
┌─────────────────────────────────────────────────┐
│ Phase 1: 준비 (1-2주)                             │
│  - 현황 파악                                       │
│  - K8s 클러스터 구축                                │
│  - 팀 교육                                        │
│                                                 │
│ Phase 2: 변환 (1-2주)                             │
│  - YAML 작성                                     │
│  - CI/CD 수정                                    │
│  - 테스트 환경 배포                                 │
│                                                 │
│ Phase 3: 검증 (1주)                              │
│  - 기능 테스트                                    │
│  - 성능 테스트                                    │
│  - 부하 테스트                                    │
│                                                │
│ Phase 4: 배포 (1-2주)                            │
│  - Canary 배포                                  │
│  - 모니터링                                      │
│  - 점진적 확대                                    │
│                                                │
│ Phase 5: 최적화 (지속)                            │
│  - 리소스 튜닝                                    │
│  - 비용 최적화                                    │
│  - SRE 구축                                      │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **리스크 관리**: 점진적 전환으로 위험 최소화
- **다운타임 최소화**: 무중단 마이그레이션
- **팀 준비**: 충분한 학습 시간
- **비용 효율**: 클라우드 리소스 최적화

---

## 🔧 실습 1: 사전 준비 - 현황 파악

### Step 1: Docker Compose 분석

```bash
# 현재 docker-compose.yml 확인
cat docker-compose.yml

# 실행 중인 컨테이너
docker-compose ps

# 리소스 사용
docker stats --no-stream

# 네트워크 구성
docker network ls
docker network inspect <network-name>

# 볼륨 확인
docker volume ls
```

### Step 2: 체크리스트

```yaml
# migration-checklist.yaml
준비 사항:
  인프라:
    - [ ] Kubernetes 클러스터 (GKE, EKS, AKS, 또는 Self-hosted)
    - [ ] kubectl 설치 및 설정
    - [ ] Container Registry (Docker Hub, GCR, ECR)
    - [ ] Ingress Controller (Nginx, Traefik)
    - [ ] Storage Class (PV 프로비저닝)
  
  애플리케이션:
    - [ ] 모든 이미지 Registry에 Push
    - [ ] 환경 변수 목록 작성
    - [ ] Secret 목록 작성 (DB 비밀번호 등)
    - [ ] Volume 데이터 백업
    - [ ] Health Check 엔드포인트 (/health, /ready)
  
  팀:
    - [ ] Kubernetes 기본 교육
    - [ ] kubectl 사용법
    - [ ] 트러블슈팅 가이드
    - [ ] 롤백 계획
  
  모니터링:
    - [ ] Prometheus 설치
    - [ ] Grafana 대시보드
    - [ ] 알림 설정 (Slack, PagerDuty)
    - [ ] 로그 수집 (ELK, Loki)
```

---

## 🔧 실습 2: Kompose로 자동 변환

### Step 1: Kompose 설치

```bash
# macOS
brew install kompose

# Linux
curl -L https://github.com/kubernetes/kompose/releases/download/v1.31.2/kompose-linux-amd64 -o kompose
chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose

# Windows
choco install kubernetes-kompose
```

### Step 2: 자동 변환

```bash
# docker-compose.yml이 있는 디렉토리에서
kompose convert

# 출력:
# INFO Kubernetes file "frontend-deployment.yaml" created
# INFO Kubernetes file "backend-deployment.yaml" created
# INFO Kubernetes file "postgres-deployment.yaml" created
# INFO Kubernetes file "frontend-service.yaml" created
# INFO Kubernetes file "backend-service.yaml" created
# INFO Kubernetes file "postgres-service.yaml" created

# 생성된 파일 확인
ls *.yaml
```

### Step 3: 변환 결과 검토 및 수정

```yaml
# Kompose가 생성한 YAML (수정 필요)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1  # ← 수정: 3으로 증가
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: myapp:latest  # ← 수정: Registry 경로 추가
        ports:
        - containerPort: 8080
        # ← 추가 필요:
        # - resources (limits/requests)
        # - livenessProbe
        # - readinessProbe
        # - env (ConfigMap/Secret으로 분리)
```

---

## 🔧 실습 3: 수동 변환 (Best Practice)

### Step 1: 원본 docker-compose.yml

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    image: myapp-frontend:latest
    ports:
      - "80:3000"
    environment:
      - REACT_APP_API_URL=http://backend:8080
    depends_on:
      - backend

  backend:
    image: myapp-backend:latest
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_USER=myuser
      - DB_PASSWORD=secret  # ← Secret으로!
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_USER=myuser
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

### Step 2: Kubernetes YAML (Best Practice)

```yaml
# 1. Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
# 2. ConfigMap (환경 변수)
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: myapp
data:
  DB_HOST: postgres
  REDIS_HOST: redis
  DB_USER: myuser
---
# 3. Secret (민감 정보)
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
  namespace: myapp
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0  # base64: secret
---
# 4. PVC (데이터 영속성)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: myapp
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: myapp
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
# 5. PostgreSQL StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: myapp
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: DB_PASSWORD
        - name: POSTGRES_DB
          value: mydb
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - myuser
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: myapp
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
---
# 6. Redis Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-storage
          mountPath: /data
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: myapp
spec:
  selector:
    app: redis
  ports:
  - port: 6379
---
# 7. Backend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: myregistry.io/myapp-backend:v1.0.0
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: backend-config
        - secretRef:
            name: backend-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: myapp
spec:
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
---
# 8. Frontend Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: myregistry.io/myapp-frontend:v1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: REACT_APP_API_URL
          value: "http://backend:8080"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: myapp
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
```

---

## 🔧 실습 4: 단계별 마이그레이션

### Step 1: 테스트 환경 배포

```bash
# 1. Namespace 생성
kubectl create namespace myapp-test

# 2. Secret 생성 (Base64 인코딩)
echo -n "secret" | base64
# c2VjcmV0

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
  namespace: myapp-test
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0
EOF

# 3. 모든 리소스 배포
kubectl apply -f kubernetes/ -n myapp-test

# 4. 상태 확인
kubectl get all -n myapp-test

# 5. Pod 로그
kubectl logs -f deployment/backend -n myapp-test

# 6. 서비스 접속 테스트
kubectl port-forward service/frontend 8080:80 -n myapp-test
curl http://localhost:8080
```

### Step 2: 데이터 마이그레이션

```bash
# PostgreSQL 데이터 백업 (Docker)
docker exec postgres pg_dump -U myuser mydb > backup.sql

# Kubernetes Pod에 복사
kubectl cp backup.sql myapp-test/postgres-0:/tmp/backup.sql

# 복구
kubectl exec -it postgres-0 -n myapp-test -- psql -U myuser -d mydb -f /tmp/backup.sql

# 검증
kubectl exec -it postgres-0 -n myapp-test -- psql -U myuser -d mydb -c "SELECT COUNT(*) FROM users;"
```

### Step 3: Canary 배포 (점진적 전환)

```yaml
# canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-canary
  namespace: myapp
spec:
  replicas: 1  # ← 10% 트래픽
  selector:
    matchLabels:
      app: backend
      version: v2
  template:
    metadata:
      labels:
        app: backend
        version: v2
    spec:
      containers:
      - name: backend
        image: myregistry.io/myapp-backend:v2.0.0
        # ... (나머지 동일)
---
# 기존 Deployment (90% 트래픽)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-stable
  namespace: myapp
spec:
  replicas: 9  # ← 90% 트래픽
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      containers:
      - name: backend
        image: myregistry.io/myapp-backend:v1.0.0
---
# Service (모든 버전)
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: myapp
spec:
  selector:
    app: backend  # version 제외 → 모든 버전
  ports:
  - port: 8080
```

```bash
# 배포
kubectl apply -f canary-deployment.yaml

# 모니터링
kubectl get pods -l app=backend -n myapp
# NAME                               READY   STATUS
# backend-stable-xxx                 1/1     Running
# backend-stable-yyy                 1/1     Running
# ...
# backend-canary-zzz                 1/1     Running

# 문제 없으면 Canary 확대
kubectl scale deployment backend-canary --replicas=5 -n myapp
kubectl scale deployment backend-stable --replicas=5 -n myapp

# 최종 전환
kubectl scale deployment backend-canary --replicas=10 -n myapp
kubectl scale deployment backend-stable --replicas=0 -n myapp
```

---

## 🔧 실습 5: Helm Chart로 패키징

### Step 1: Helm Chart 생성

```bash
# Helm 설치
brew install helm  # macOS

# Chart 생성
helm create myapp

# 구조
myapp/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── secret.yaml
└── charts/
```

### Step 2: values.yaml

```yaml
# values.yaml
replicaCount: 3

image:
  repository: myregistry.io/myapp-backend
  tag: v1.0.0
  pullPolicy: IfNotPresent

service:
  type: LoadBalancer
  port: 80
  targetPort: 8080

resources:
  requests:
    memory: "256Mi"
    cpu: "200m"
  limits:
    memory: "512Mi"
    cpu: "500m"

env:
  DB_HOST: postgres
  REDIS_HOST: redis

secret:
  DB_PASSWORD: secret
```

### Step 3: templates/deployment.yaml

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ .Chart.Name }}-secret
              key: DB_PASSWORD
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

### Step 4: 배포

```bash
# Dry-run (테스트)
helm install myapp ./myapp --dry-run --debug

# 설치
helm install myapp ./myapp

# 업그레이드
helm upgrade myapp ./myapp --set image.tag=v2.0.0

# 롤백
helm rollback myapp 1

# 삭제
helm uninstall myapp
```

---

## 🔧 실습 6: CI/CD 통합

### Step 1: GitHub Actions

```yaml
# .github/workflows/deploy.yaml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    # Docker 이미지 빌드
    - name: Build image
      run: |
        docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
        docker tag ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
                   ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
    
    # Registry에 Push
    - name: Push image
      run: |
        echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
        docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
    
    # Kubernetes 배포
    - name: Deploy to Kubernetes
      uses: azure/k8s-deploy@v1
      with:
        manifests: |
          kubernetes/deployment.yaml
          kubernetes/service.yaml
        images: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        kubectl-version: 'latest'
```

---

## 🔧 실습 7: 모니터링 및 검증

### Step 1: 헬스 체크

```bash
# Pod 상태
kubectl get pods -n myapp

# 로그 확인
kubectl logs -f deployment/backend -n myapp

# 이벤트
kubectl get events -n myapp --sort-by='.lastTimestamp'

# 리소스 사용
kubectl top pods -n myapp
kubectl top nodes
```

### Step 2: 성능 테스트

```bash
# Apache Bench
kubectl run -it ab --rm --image=httpd:alpine --restart=Never -- \
  ab -n 10000 -c 100 http://frontend/

# Siege
kubectl run -it siege --rm --image=yokogawa/siege --restart=Never -- \
  siege -c 50 -t 30s http://frontend/
```

### Step 3: 롤백 계획

```bash
# Deployment 히스토리
kubectl rollout history deployment/backend -n myapp

# 롤백 (이전 버전)
kubectl rollout undo deployment/backend -n myapp

# 특정 버전
kubectl rollout undo deployment/backend --to-revision=2 -n myapp

# Helm 롤백
helm rollback myapp 1
```

---

## 💡 마이그레이션 체크리스트

```
□ 준비 단계
  ✅ Kubernetes 클러스터 준비
  ✅ 팀 교육
  ✅ 백업 완료
  ✅ 롤백 계획

□ 변환 단계
  ✅ YAML 작성
  ✅ ConfigMap/Secret 분리
  ✅ Resource Limits 설정
  ✅ Health Probes 추가

□ 배포 단계
  ✅ 테스트 환경 배포
  ✅ 기능 테스트
  ✅ 성능 테스트
  ✅ Canary 배포

□ 검증 단계
  ✅ 모니터링 확인
  ✅ 로그 수집
  ✅ 알림 작동
  ✅ 문서화

□ 최적화 단계
  ✅ HPA 설정
  ✅ 리소스 튜닝
  ✅ 비용 최적화
  ✅ SLO 설정
```

---

## 📚 참고 자료

- [Kompose](https://kompose.io/)
- [Helm](https://helm.sh/)
- [Kubernetes Migration Best Practices](https://kubernetes.io/docs/tasks/configure-pod-container/)
- [Cloud Provider Migration Guides](https://cloud.google.com/kubernetes-engine/docs/how-to)

---

## 🤔 생각해볼 문제

1. 모든 애플리케이션을 Kubernetes로 옮겨야 하는가?
2. 마이그레이션 중 다운타임을 완전히 피할 수 있는가?
3. Kubernetes로 옮기면 비용이 줄어드는가?

> 💡 **답변**:
> 
> **1) 모든 애플리케이션을 K8s로?**
> 
> ```
> NO! 상황에 따라 다름
> 
> Kubernetes 적합:
> ✅ 마이크로서비스 (10개 이상 서비스)
> ✅ 높은 트래픽 (확장 필요)
> ✅ High Availability 필요
> ✅ 복잡한 배포 (Canary, Blue-Green)
> ✅ 멀티 클라우드
> 
> Kubernetes 부적합:
> ❌ 간단한 앱 (모놀리스)
> ❌ 낮은 트래픽 (<1000 req/day)
> ❌ 소규모 팀 (<3명)
> ❌ 레거시 앱 (변경 어려움)
> 
> 대안:
> - Docker Compose (개발)
> - Serverless (Lambda, Cloud Run)
> - PaaS (Heroku, Railway, Render)
> - Managed Services (RDS, Cloud SQL)
> 
> 결정 기준:
> 1. 복잡도 vs 이득
> 2. 팀 역량
> 3. 비용
> 4. 시간
> 
> "K8s는 도구일 뿐, 만능이 아님"
> ```
> 
> **2) 완전한 무중단 마이그레이션?**
> 
> ```
> 가능하지만 복잡함
> 
> 시나리오:
> 
> 1. DNS 기반 전환:
>    Docker (old.example.com)
>    K8s (new.example.com)
>    → DNS 전환 (TTL 짧게)
>    → 점진적 트래픽 이동
> 
> 2. Load Balancer 기반:
>    LB → Docker (90%)
>       → K8s (10%)
>    → 점진적 비율 조정
> 
> 3. Service Mesh:
>    Istio/Linkerd로 트래픽 분할
>    Docker ↔ K8s 투명하게 통신
> 
> 주의사항:
> ⚠️  Stateful 서비스 (DB)
>    - 복제 설정
>    - Read/Write 분리
>    - 최종 데이터 동기화 필요
> 
> ⚠️  세션 관리
>    - Sticky Session 끄기
>    - Redis 등 외부 세션 저장소
> 
> ⚠️  롤백 계획
>    - 언제든 Docker로 돌아갈 수 있어야
>    - 데이터 양방향 동기화
> 
> 현실:
> - 완벽한 무중단은 어려움
> - 5-10분 점검 시간 권장
> - 새벽 시간 배포
> ```
> 
> **3) K8s로 비용 절감?**
> 
> ```
> 단순 답변: NO, 초기 비용 증가
> 
> 비용 구조:
> 
> Docker Compose (단일 서버):
> - 서버: $50/month
> - 관리: 최소
> - 학습: 낮음
> = $50/month
> 
> Kubernetes (클러스터):
> - Control Plane: $70/month (GKE)
> - Worker Nodes: $100-500/month
> - Load Balancer: $20/month
> - 관리: 높음 (SRE 필요)
> - 학습: 높음
> = $200-600/month (초기)
> 
> 하지만...
> 
> 장기적 이득:
> ✅ 자동 스케일링 → 트래픽 따라 조정
> ✅ 리소스 효율 → Bin Packing
> ✅ 다운타임 감소 → 매출 손실 방지
> ✅ 개발 생산성 → 빠른 배포
> 
> Break-even:
> - 트래픽 증가 시
> - 서비스 개수 증가 시
> - 글로벌 배포 시
> 
> 비용 최적화:
> ✅ Spot/Preemptible Instances (50-80% 할인)
> ✅ HPA (필요할 때만 Pod)
> ✅ Cluster Autoscaler (필요할 때만 Node)
> ✅ Resource Requests 최적화
> ✅ Namespace별 LimitRange
> 
> 결론:
> - 초기: 비용 증가
> - 중장기: 규모 경제
> - 트래픽/서비스 많으면 저렴
> - 적으면 비쌈
> ```

---

## 📌 핵심 요약

```
마이그레이션 핵심:

전략:
- Big Bang: 빠르지만 위험
- Strangler Fig: 느리지만 안전 (권장)
- Hybrid: 점진적 전환

단계:
1. 준비 (클러스터, 교육)
2. 변환 (YAML 작성)
3. 검증 (테스트)
4. 배포 (Canary)
5. 최적화 (HPA, 비용)

도구:
- Kompose (자동 변환)
- Helm (패키징)
- CI/CD (자동화)

Best Practices:
✅ 점진적 전환
✅ 충분한 테스트
✅ 모니터링 필수
✅ 롤백 계획
✅ 팀 교육
```

---

<div align="center">

**[⬅️ 이전: Deployment Patterns](./03-Deployment-Patterns.md)** | **[🏠 Kubernetes Bridge 섹션 완료!](./README.md)**

</div>
