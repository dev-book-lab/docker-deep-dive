# 01. Docker to K8s - Docker에서 Kubernetes로 전환

## 🎯 이 챕터에서 배울 것

- **개념 매핑**: Docker ↔ Kubernetes 개념 비교
- **아키텍처 차이**: 단일 호스트 vs 클러스터
- **YAML 변환**: docker-compose.yml → K8s YAML
- **주요 차이점**: 오케스트레이션의 본질적 차이
- **전환 전략**: 점진적 마이그레이션
- **실전 변환**: 실제 애플리케이션 변환

## 📌 왜 중요한가?

**"Docker를 알면 Kubernetes의 70%를 이미 알고 있습니다. 나머지 30%가 핵심입니다."**

```
Docker vs Kubernetes:

Docker (Single Host):
┌─────────────────────────────────────────────────┐
│ Docker Host                                     │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │Container1│  │Container2│  │Container3│       │
│  └──────────┘  └──────────┘  └──────────┘       │
│                                                 │
│  Docker Engine                                  │
│  - 컨테이너 실행                                   │
│  - 로컬 네트워킹                                   │
│  - 볼륨 관리                                      │
│                                                 │
│  한계:                                           │
│  ❌ 단일 호스트 (스케일링 제한)                       │
│  ❌ 수동 복구 (장애 시)                             │
│  ❌ 수동 로드 밸런싱                                │
│  ❌ 복잡한 배포                                    │
└─────────────────────────────────────────────────┘

Kubernetes (Cluster):
┌─────────────────────────────────────────────────┐
│ Kubernetes Cluster                              │
│                                                 │
│  Master Node (Control Plane)                    │
│  ┌──────────────────────────────────────────┐   │
│  │ API Server, Scheduler, Controller        │   │
│  └──────────────────────────────────────────┘   │
│                ↓                                │
│  ┌─────────────┬─────────────┬─────────────┐    │
│  │  Worker 1   │  Worker 2   │  Worker 3   │    │
│  │             │             │             │    │
│  │ ┌───┐ ┌───┐ │ ┌───┐ ┌───┐ │ ┌───┐ ┌───┐ │    │
│  │ │Pod│ │Pod│ │ │Pod│ │Pod│ │ │Pod│ │Pod│ │    │
│  │ └───┘ └───┘ │ └───┘ └───┘ │ └───┘ └───┘ │    │
│  └─────────────┴─────────────┴─────────────┘    │
│                                                 │
│  자동화:                                          │
│  ✅ 자동 스케일링 (HPA, VPA)                       │
│  ✅ 자동 복구 (Self-healing)                      │
│  ✅ 로드 밸런싱 (Service)                          │
│  ✅ 롤링 업데이트 (Deployment)                     │
└─────────────────────────────────────────────────┘

개념 매핑:
┌──────────────────┬─────────────────────────────┐
│ Docker           │ Kubernetes                  │
├──────────────────┼─────────────────────────────┤
│ Container        │ Container (in Pod)          │
├──────────────────┼─────────────────────────────┤
│ docker run       │ Pod / Deployment            │
├──────────────────┼─────────────────────────────┤
│ docker-compose   │ Deployment + Service        │
├──────────────────┼─────────────────────────────┤
│ Volumes          │ PersistentVolumeClaim (PVC) │
├──────────────────┼─────────────────────────────┤
│ Networks         │ Service, Ingress            │
├──────────────────┼─────────────────────────────┤
│ Environment Vars │ ConfigMap, Secret           │
├──────────────────┼─────────────────────────────┤
│ docker logs      │ kubectl logs                │
├──────────────────┼─────────────────────────────┤
│ docker exec      │ kubectl exec                │
└──────────────────┴─────────────────────────────┘

핵심 차이점:
┌─────────────────────────────────────────────────┐
│ 1. 선언적 vs 명령적                                │
│    Docker: docker run ... (명령)                 │
│    K8s: YAML 선언 → K8s가 실행                     │
│                                                 │
│ 2. 단일 vs 다중 호스트                              │
│    Docker: 1대 서버                               │
│    K8s: 여러 노드 클러스터                           │
│                                                 │
│ 3. 수동 vs 자동 관리                               │
│    Docker: 수동 재시작, 스케일링                     │
│    K8s: 자동 복구, 스케일링                         │
│                                                 │
│ 4. 추상화 수준                                     │
│    Docker: 컨테이너 중심                           │
│    K8s: 워크로드 중심 (Pod, Deployment)            │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **확장성**: 클러스터로 무한 확장
- **안정성**: 자동 복구, Self-healing
- **효율성**: 리소스 최적 활용
- **표준화**: 업계 표준 오케스트레이션

---

## 🔬 Deep Dive

### 1. 기본 개념 비교

#### Docker Container → Kubernetes Pod

```bash
# Docker
docker run -d --name nginx -p 8080:80 nginx

# Kubernetes (명령적)
kubectl run nginx --image=nginx --port=80

# Kubernetes (선언적)
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

#### Docker Compose → Kubernetes Deployment + Service

```yaml
# docker-compose.yml
version: '3'
services:
  web:
    image: nginx
    ports:
      - "8080:80"
    replicas: 3
```

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

---

## 🔧 실습 1: 간단한 애플리케이션 변환

### Step 1: Docker 버전

```yaml
# docker-compose.yml
version: '3.8'
services:
  backend:
    image: myapp:latest
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
    depends_on:
      - postgres
  
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

### Step 2: Kubernetes 변환

```yaml
# kubernetes/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
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
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: postgres
        - name: DB_PORT
          value: "5432"
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
  type: LoadBalancer
```

```yaml
# kubernetes/postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
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
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  password: c2VjcmV0  # base64 encoded "secret"
```

### Step 3: 배포 및 확인

```bash
# Docker
docker-compose up -d

# Kubernetes
kubectl apply -f kubernetes/

# 상태 확인
kubectl get pods
kubectl get services
kubectl get pvc

# 로그 확인
kubectl logs -f deployment/backend

# 접속 테스트
kubectl port-forward service/backend 8080:8080
curl http://localhost:8080
```

---

## 🔧 실습 2: 환경 변수와 Secret

### Step 1: Docker 환경 변수

```yaml
# docker-compose.yml
services:
  backend:
    environment:
      - API_KEY=my-api-key
      - DB_PASSWORD=secret
```

### Step 2: Kubernetes ConfigMap + Secret

```yaml
# kubernetes/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  API_URL: "https://api.example.com"
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
type: Opaque
data:
  API_KEY: bXktYXBpLWtleQ==      # base64: my-api-key
  DB_PASSWORD: c2VjcmV0           # base64: secret
```

```yaml
# kubernetes/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    spec:
      containers:
      - name: backend
        image: myapp:latest
        env:
        # ConfigMap에서
        - name: API_URL
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: API_URL
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: LOG_LEVEL
        # Secret에서
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: API_KEY
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: DB_PASSWORD
```

```bash
# ConfigMap/Secret 생성
kubectl apply -f kubernetes/configmap.yaml

# 확인
kubectl get configmap backend-config -o yaml
kubectl get secret backend-secret -o yaml

# Secret 생성 (명령어)
kubectl create secret generic backend-secret \
  --from-literal=API_KEY=my-api-key \
  --from-literal=DB_PASSWORD=secret
```

---

## 🔧 실습 3: 네트워크 매핑

### Step 1: Docker Networks

```yaml
# docker-compose.yml
services:
  frontend:
    networks:
      - frontend
  
  backend:
    networks:
      - frontend
      - backend
  
  database:
    networks:
      - backend

networks:
  frontend:
  backend:
```

### Step 2: Kubernetes Services

```yaml
# Kubernetes에서는 Service로 네트워크 관리

# Frontend (외부 접근 가능)
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  type: LoadBalancer  # 외부 노출
  ports:
  - port: 80
    targetPort: 3000

---
# Backend (내부 접근만)
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  type: ClusterIP  # 내부만
  ports:
  - port: 8080
    targetPort: 8080

---
# Database (내부 접근만)
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  selector:
    app: database
  type: ClusterIP
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
# 접속 방법
# Frontend → Backend: http://backend:8080
# Backend → Database: postgresql://database:5432
```

---

## 🔧 실습 4: Volume 매핑

### Step 1: Docker Volumes

```yaml
# docker-compose.yml
services:
  database:
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./config:/etc/config:ro

volumes:
  db-data:
```

### Step 2: Kubernetes PVC

```yaml
# kubernetes/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
data:
  postgresql.conf: |
    max_connections = 100
    shared_buffers = 128MB
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
spec:
  template:
    spec:
      containers:
      - name: postgres
        volumeMounts:
        # PVC 마운트
        - name: db-storage
          mountPath: /var/lib/postgresql/data
        # ConfigMap 마운트 (읽기 전용)
        - name: config
          mountPath: /etc/config
          readOnly: true
      volumes:
      - name: db-storage
        persistentVolumeClaim:
          claimName: db-data
      - name: config
        configMap:
          name: database-config
```

---

## 🔧 실습 5: Health Checks

### Step 1: Docker Health Check

```dockerfile
# Dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

```yaml
# docker-compose.yml
services:
  backend:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 3s
      retries: 3
```

### Step 2: Kubernetes Probes

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    spec:
      containers:
      - name: backend
        image: myapp:latest
        ports:
        - containerPort: 8080
        
        # Liveness Probe (재시작 여부)
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Readiness Probe (트래픽 수신 여부)
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Startup Probe (시작 완료 여부)
        startupProbe:
          httpGet:
            path: /startup
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 5
          failureThreshold: 30  # 최대 150초 대기
```

---

## 🔧 실습 6: 명령어 비교

### Docker vs Kubernetes 명령어

```bash
# ============ 실행 ============
# Docker
docker run -d --name nginx nginx
docker-compose up -d

# Kubernetes
kubectl run nginx --image=nginx
kubectl apply -f deployment.yaml

# ============ 목록 ============
# Docker
docker ps
docker-compose ps

# Kubernetes
kubectl get pods
kubectl get deployments
kubectl get services

# ============ 로그 ============
# Docker
docker logs nginx
docker logs -f nginx

# Kubernetes
kubectl logs nginx
kubectl logs -f nginx
kubectl logs -f deployment/nginx

# ============ 쉘 접속 ============
# Docker
docker exec -it nginx /bin/bash

# Kubernetes
kubectl exec -it nginx -- /bin/bash

# ============ 중지/삭제 ============
# Docker
docker stop nginx
docker rm nginx
docker-compose down

# Kubernetes
kubectl delete pod nginx
kubectl delete deployment nginx
kubectl delete -f deployment.yaml

# ============ 상태 확인 ============
# Docker
docker inspect nginx
docker stats nginx

# Kubernetes
kubectl describe pod nginx
kubectl top pod nginx

# ============ 네트워크 ============
# Docker
docker network ls
docker network inspect bridge

# Kubernetes
kubectl get services
kubectl get ingress

# ============ 볼륨 ============
# Docker
docker volume ls
docker volume inspect myvolume

# Kubernetes
kubectl get pv
kubectl get pvc
kubectl describe pvc mypvc
```

---

## 🔧 실습 7: 전체 애플리케이션 변환

### Step 1: Docker Compose (Full Stack)

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    image: frontend:latest
    ports:
      - "80:3000"
    environment:
      - REACT_APP_API_URL=http://backend:8080
    depends_on:
      - backend

  backend:
    image: backend:latest
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
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

### Step 2: Kubernetes (Full Stack)

```yaml
# kubernetes/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
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
        image: frontend:latest
        ports:
        - containerPort: 3000
        env:
        - name: REACT_APP_API_URL
          value: "http://backend:8080"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
```

```yaml
# kubernetes/backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
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
        image: backend:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: postgres
        - name: REDIS_HOST
          value: redis
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

```yaml
# kubernetes/postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
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
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  type: ClusterIP
  ports:
  - port: 5432
```

```yaml
# kubernetes/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
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
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  type: ClusterIP
  ports:
  - port: 6379
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

```bash
# 배포
kubectl apply -f kubernetes/

# 확인
kubectl get all
kubectl get pvc

# 접속
kubectl port-forward service/frontend 8080:80
```

---

## 💡 변환 체크리스트

```
Docker Compose → Kubernetes:

□ Services
  ✅ 각 service → Deployment
  ✅ Port 매핑 → Service
  ✅ depends_on → readinessProbe

□ Environment Variables
  ✅ 일반 환경변수 → ConfigMap
  ✅ 민감 정보 → Secret

□ Volumes
  ✅ Named volumes → PVC
  ✅ Bind mounts → ConfigMap/Secret

□ Networks
  ✅ Custom networks → Service (ClusterIP)
  ✅ External access → Service (LoadBalancer/NodePort)

□ Health Checks
  ✅ healthcheck → livenessProbe + readinessProbe

□ Replicas
  ✅ deploy.replicas → Deployment.spec.replicas

□ Resource Limits
  ✅ mem_limit, cpus → resources.limits
```

---

## 📌 핵심 요약

```
Docker → Kubernetes 전환:

개념 매핑:
- Container → Pod
- docker-compose → Deployment + Service
- Volume → PVC
- Network → Service
- Environment → ConfigMap + Secret

주요 차이:
1. 단일 호스트 → 클러스터
2. 명령적 → 선언적
3. 수동 관리 → 자동 관리
4. 제한적 스케일링 → 무한 확장

Best Practices:
✅ 선언적 YAML 사용
✅ ConfigMap/Secret 분리
✅ Health Probes 설정
✅ Resource Limits 지정
✅ Labels로 조직화
```

---

## 📚 참고 자료

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Docker to Kubernetes Migration Guide](https://kubernetes.io/docs/tasks/configure-pod-container/)
- [Kompose - Compose to Kubernetes](https://kompose.io/)
- [Docker Compose vs Kubernetes](https://www.docker.com/blog/kubernetes-vs-docker-compose/)

---

## 🤔 생각해볼 문제

1. Docker Compose를 계속 사용해야 하는 상황은 언제인가?
2. Kubernetes는 모든 문제를 해결하는가?
3. 소규모 팀에서 Kubernetes를 도입해야 하는가?

> 💡 **답변**:
> 
> **1) Docker Compose 사용 상황:**
> 
> ```
> ✅ 적합한 경우:
> - 개발 환경 (로컬)
> - 간단한 애플리케이션 (5개 이하 서비스)
> - 단일 서버 배포
> - CI/CD 테스트 환경
> - 학습 및 프로토타입
> 
> 예:
> - 개인 프로젝트
> - 스타트업 초기 (MVP)
> - 사내 도구
> ```
> 
> **2) Kubernetes의 한계:**
> 
> ```
> Kubernetes가 해결 못 하는 것:
> ❌ 애플리케이션 버그
> ❌ 잘못된 아키텍처 설계
> ❌ 데이터베이스 성능
> ❌ 네트워크 물리적 제한
> 
> Kubernetes는:
> ✅ 오케스트레이션 도구
> ✅ 배포 자동화
> ✅ 스케일링 관리
> ✅ Self-healing
> 
> 여전히 필요한 것:
> - 좋은 코드
> - 적절한 아키텍처
> - 모니터링
> - 보안
> ```
> 
> **3) 소규모 팀의 K8s 도입:**
> 
> ```
> 고려사항:
> 
> 도입하지 말아야:
> ❌ 팀원 < 3명
> ❌ 트래픽 < 1000 req/day
> ❌ 학습 시간 없음
> ❌ 단순한 애플리케이션
> 
> 도입 고려:
> ✅ 성장 계획 있음
> ✅ 마이크로서비스 아키텍처
> ✅ 자동 스케일링 필요
> ✅ High Availability 필요
> 
> 대안:
> - Managed Kubernetes (GKE, EKS, AKS)
> - Serverless (Lambda, Cloud Run)
> - PaaS (Heroku, Railway)
> 
> 결론:
> "필요에 따라 점진적으로"
> Docker Compose → Managed K8s → Self-hosted K8s
> ```

---

<div align="center">

**[다음: Pod Concepts ➡️](./02-Pod-Concepts.md)**

</div>
