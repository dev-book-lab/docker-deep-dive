# 02. Pod Concepts - Pod 개념 완전 이해

## 🎯 이 챕터에서 배울 것

- **Pod란?**: Kubernetes의 가장 작은 배포 단위
- **Pod vs Container**: 차이점과 관계
- **Multi-container Pod**: 하나의 Pod에 여러 컨테이너
- **Pod Lifecycle**: 생성부터 종료까지
- **Pod Patterns**: Sidecar, Init Container
- **실전 사용**: 언제 Single/Multi Container를 사용하는가

## 📌 왜 중요한가?

**"Pod를 이해하지 못하면 Kubernetes를 이해할 수 없습니다."**

```
Pod의 본질:

Container (Docker):
┌─────────────────────────────────────────────────┐
│ 1 Container = 1 프로세스                          │
│                                                 │
│  ┌──────────────┐                               │
│  │  Container   │                               │
│  │  (nginx)     │                               │
│  └──────────────┘                               │
│                                                 │
│  - 격리된 파일 시스템                               │
│  - 독립적 네트워크                                  │
│  - 독립적 프로세스 공간                              │
└─────────────────────────────────────────────────┘

Pod (Kubernetes):
┌─────────────────────────────────────────────────┐
│ 1 Pod = 1개 이상의 Container + 공유 리소스           │
│                                                 │
│  ┌────────────────────────────────────────┐     │
│  │ Pod                                    │     │
│  │                                        │     │
│  │  ┌──────────┐    ┌──────────┐          │     │
│  │  │Container1│    │Container2│          │     │
│  │  │ (nginx)  │    │ (sidecar)│          │     │
│  │  └────┬─────┘    └────┬─────┘          │     │
│  │       │               │                │     │
│  │       └───────┬───────┘                │     │
│  │               ▼                        │     │
│  │     공유 리소스:                          │     │
│  │     - Network (같은 IP)                 │     │
│  │     - Volumes (공유 스토리지)             │     │
│  │     - IPC (프로세스 간 통신)               │     │
│  └────────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘

왜 Pod인가?:
┌─────────────────────────────────────────────────┐
│ 1. 밀접하게 결합된 컨테이너를 함께 배치                  │
│    예: Web + Log collector                      │
│                                                 │
│ 2. 리소스 공유                                     │
│    - 같은 IP (localhost 통신)                     │
│    - 같은 Volume                                 │
│                                                 │
│ 3. 원자적 배포 단위                                 │
│    - 함께 스케줄링                                 │
│    - 함께 시작/종료                                │
│                                                 │
│ 4. 패턴 구현                                      │
│    - Sidecar (로깅, 모니터링)                      │
│    - Ambassador (프록시)                          │
│    - Adapter (데이터 변환)                         │
└─────────────────────────────────────────────────┘

Pod Lifecycle:
┌─────────────────────────────────────────────────┐
│                                                 │
│  Pending → Running → Succeeded/Failed           │
│     ↓         ↓           ↓                     │
│   Init    Running      Terminated               │
│  Containers Containers                          │
│                                                 │
│  Phase 설명:                                     │
│  - Pending: 스케줄링 대기 또는 이미지 Pull            │
│  - Running: 최소 1개 컨테이너 실행 중                │
│  - Succeeded: 모든 컨테이너 성공 종료                │
│  - Failed: 최소 1개 컨테이너 실패 종료                │
│  - Unknown: Pod 상태를 알 수 없음                   │
└─────────────────────────────────────────────────┘

Container States:
┌─────────────────────────────────────────────────┐
│  Waiting → Running → Terminated                 │
│                                                 │
│  Waiting:                                       │
│  - ContainerCreating                            │
│  - ImagePullBackOff                             │
│  - CrashLoopBackOff                             │
│                                                 │
│  Running:                                       │
│  - 정상 실행 중                                    │
│                                                 │
│  Terminated:                                    │
│  - Completed (exit 0)                           │
│  - Error (exit != 0)                            │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **설계**: 컨테이너 분리 vs 결합
- **배포**: 원자적 배포 단위
- **리소스**: 공유 네트워크/볼륨
- **패턴**: Sidecar, Init Container

---

## 🔧 실습 1: 기본 Pod 생성

### Step 1: 가장 간단한 Pod

```yaml
# simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.24
    ports:
    - containerPort: 80
```

```bash
# Pod 생성
kubectl apply -f simple-pod.yaml

# 상태 확인
kubectl get pods
kubectl get pod nginx-pod -o wide

# 상세 정보
kubectl describe pod nginx-pod

# 로그
kubectl logs nginx-pod

# 쉘 접속
kubectl exec -it nginx-pod -- /bin/bash

# 삭제
kubectl delete pod nginx-pod
```

### Step 2: 환경 변수와 리소스

```yaml
# pod-with-resources.yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
spec:
  containers:
  - name: backend
    image: node:18-alpine
    command: ["node", "server.js"]
    
    # 환경 변수
    env:
    - name: PORT
      value: "8080"
    - name: NODE_ENV
      value: "production"
    
    # 포트
    ports:
    - containerPort: 8080
      name: http
      protocol: TCP
    
    # 리소스 제한
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
```

```bash
# 배포
kubectl apply -f pod-with-resources.yaml

# 리소스 사용 확인
kubectl top pod backend-pod

# 리소스 제한 확인
kubectl describe pod backend-pod | grep -A 5 "Limits:"
```

---

## 🔧 실습 2: Multi-container Pod (Sidecar Pattern)

### Step 1: Web + Logging Sidecar

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  # 공유 Volume
  volumes:
  - name: shared-logs
    emptyDir: {}
  
  containers:
  # Main Container: Web Server
  - name: web
    image: nginx:1.24
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  
  # Sidecar Container: Log Collector
  - name: log-collector
    image: busybox
    command:
    - sh
    - -c
    - 'tail -f /logs/access.log'
    volumeMounts:
    - name: shared-logs
      mountPath: /logs
```

```bash
# 배포
kubectl apply -f multi-container-pod.yaml

# 두 컨테이너 확인
kubectl get pod web-with-logging -o jsonpath='{.spec.containers[*].name}'

# 각 컨테이너 로그
kubectl logs web-with-logging -c web
kubectl logs web-with-logging -c log-collector

# 특정 컨테이너에 접속
kubectl exec -it web-with-logging -c web -- /bin/bash
kubectl exec -it web-with-logging -c log-collector -- /bin/sh
```

### Step 2: 실전 Sidecar 예제 (App + Metrics Exporter)

```yaml
# app-with-metrics.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-metrics
  labels:
    app: myapp
spec:
  volumes:
  - name: metrics-data
    emptyDir: {}
  
  containers:
  # Main Application
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
      name: http
    volumeMounts:
    - name: metrics-data
      mountPath: /metrics
    env:
    - name: METRICS_PATH
      value: "/metrics/app.metrics"
  
  # Metrics Exporter (Prometheus)
  - name: metrics-exporter
    image: prom/node-exporter
    ports:
    - containerPort: 9100
      name: metrics
    volumeMounts:
    - name: metrics-data
      mountPath: /metrics
      readOnly: true
```

---

## 🔧 실습 3: Init Containers

### Step 1: Init Container 기본

```yaml
# pod-with-init.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  # Init Containers (순차 실행)
  initContainers:
  # 1. Database 준비 대기
  - name: wait-for-db
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Waiting for database..."
      until nc -z postgres 5432; do
        echo "Database not ready, waiting..."
        sleep 2
      done
      echo "Database is ready!"
  
  # 2. 설정 파일 다운로드
  - name: download-config
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Downloading config..."
      wget -O /config/app.config http://config-server/app.config
    volumeMounts:
    - name: config
      mountPath: /config
  
  # Main Container
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config
      mountPath: /etc/app
  
  volumes:
  - name: config
    emptyDir: {}
```

```bash
# 배포
kubectl apply -f pod-with-init.yaml

# Init Container 진행 상황 확인
kubectl get pod app-with-init
# NAME            READY   STATUS     RESTARTS   AGE
# app-with-init   0/1     Init:0/2   0          10s

# Init Container 로그
kubectl logs app-with-init -c wait-for-db
kubectl logs app-with-init -c download-config

# 완료 후 Main Container 실행
kubectl logs app-with-init -c app
```

### Step 2: 실전 Init Container (Database Migration)

```yaml
# app-with-migration.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-migration
spec:
  initContainers:
  # Database Migration
  - name: db-migration
    image: myapp:latest
    command: ["npm", "run", "migrate"]
    env:
    - name: DB_HOST
      value: postgres
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
  
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_HOST
      value: postgres
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

---

## 🔧 실습 4: Pod Lifecycle과 Probes

### Step 1: 완전한 Probes 설정

```yaml
# pod-with-probes.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-probes
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
    
    # Startup Probe (시작 확인)
    # 실패하면 다른 Probe 실행 안 함
    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 30  # 최대 150초 대기
    
    # Liveness Probe (살아있는지)
    # 실패하면 컨테이너 재시작
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 3
    
    # Readiness Probe (트래픽 수신 가능)
    # 실패하면 Service에서 제외
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      successThreshold: 1
      failureThreshold: 3
```

### Step 2: Probe 종류별 테스트

```yaml
# HTTP GET Probe
livenessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: health-check

# TCP Socket Probe
livenessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 15
  periodSeconds: 10

# Exec Command Probe
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## 🔧 실습 5: Pod Security Context

### Step 1: 보안 설정

```yaml
# secure-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod 레벨 보안 설정
  securityContext:
    runAsUser: 1000      # UID
    runAsGroup: 3000     # GID
    fsGroup: 2000        # 파일 시스템 그룹
    
  containers:
  - name: app
    image: myapp:latest
    
    # Container 레벨 보안 설정
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache
  
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

---

## 🔧 실습 6: Pod Affinity와 Anti-Affinity

### Step 1: Node Affinity (노드 선택)

```yaml
# pod-with-node-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-node-affinity
spec:
  affinity:
    # Node Affinity
    nodeAffinity:
      # 필수 조건
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - node1
            - node2
      
      # 선호 조건 (가능하면)
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-memory
  
  containers:
  - name: app
    image: myapp:latest
```

### Step 2: Pod Affinity (Pod 함께 배치)

```yaml
# pod-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  affinity:
    # Pod Affinity (같은 노드에 배치)
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - backend
        topologyKey: kubernetes.io/hostname
  
  containers:
  - name: frontend
    image: frontend:latest
```

### Step 3: Pod Anti-Affinity (Pod 분산 배치)

```yaml
# pod-anti-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-replica
  labels:
    app: myapp
spec:
  affinity:
    # Pod Anti-Affinity (다른 노드에 배치)
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - myapp
        topologyKey: kubernetes.io/hostname
  
  containers:
  - name: app
    image: myapp:latest
```

---

## 🔧 실습 7: Pod 디버깅 패턴

### Step 1: Ephemeral Debug Container

```bash
# 실행 중인 Pod에 디버그 컨테이너 추가
kubectl debug nginx-pod -it --image=busybox --target=nginx

# 새 Pod 생성 (디버그용)
kubectl debug nginx-pod -it --copy-to=nginx-debug --container=debug --image=busybox

# Node 디버깅
kubectl debug node/node1 -it --image=busybox
```

### Step 2: 문제 해결 패턴

```yaml
# debug-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
spec:
  containers:
  - name: debug
    image: nicolaka/netshoot  # 네트워크 디버깅 도구
    command: ["sleep", "infinity"]
    
    # 또는
    # image: busybox
    # image: alpine
```

```bash
# 네트워크 디버깅
kubectl exec -it debug-pod -- ping backend
kubectl exec -it debug-pod -- nslookup backend
kubectl exec -it debug-pod -- curl http://backend:8080

# DNS 확인
kubectl exec -it debug-pod -- cat /etc/resolv.conf

# 포트 확인
kubectl exec -it debug-pod -- nc -zv backend 8080
```

---

## 💡 Pod 패턴 정리

```
1. Single Container Pod:
   - 가장 일반적
   - 1 Pod = 1 책임

2. Sidecar Pattern:
   - Main + Helper 컨테이너
   - 로깅, 모니터링, 프록시

3. Ambassador Pattern:
   - Main + Proxy 컨테이너
   - 외부 서비스 추상화

4. Adapter Pattern:
   - Main + Adapter 컨테이너
   - 데이터 형식 변환

5. Init Container Pattern:
   - 초기화 작업
   - 순차 실행

언제 Multi-container를 사용하는가?
┌────────────────────────────────────────────────┐
│ ✅ 사용:                                        │
│  - 밀접하게 결합됨 (localhost 통신)                 │
│  - 같이 스케일링                                  │
│  - 리소스 공유 필요                               │
│  - 라이프사이클 공유                               │
│                                                │
│ ❌ 사용 안 함:                                   │
│  - 독립적으로 스케일링                              │
│  - 느슨한 결합                                    │
│  - 다른 배포 주기                                 │
└────────────────────────────────────────────────┘
```

---

## 📌 핵심 요약

```
Pod 핵심:

개념:
- Kubernetes의 최소 배포 단위
- 1개 이상의 컨테이너
- 공유 네트워크 + 볼륨

Lifecycle:
Pending → Running → Succeeded/Failed

Probes:
- startupProbe: 시작 확인
- livenessProbe: 살아있는지 (재시작)
- readinessProbe: 트래픽 수신 가능 (Service)

Patterns:
- Single Container (일반)
- Sidecar (로깅, 모니터링)
- Init Container (초기화)

Best Practices:
✅ Probes 항상 설정
✅ Resource Limits 지정
✅ Security Context 적용
✅ 적절한 패턴 선택
✅ Labels로 조직화
```

---

## 📚 참고 자료

- [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Multi-container Pod Patterns](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/)

---

## 🤔 생각해볼 문제

1. 하나의 Pod에 몇 개의 컨테이너를 넣어야 하는가?
2. Init Container와 Main Container의 차이는 무엇인가?
3. Liveness Probe와 Readiness Probe는 둘 다 필요한가?

> 💡 **답변**:
> 
> **1) Pod 내 컨테이너 수:**
> 
> ```
> 원칙: "밀접하게 결합된 것만 함께"
> 
> ✅ Single Container (대부분의 경우):
> - 1 Pod = 1 책임
> - 독립적 스케일링
> - 간단한 관리
> 
> ✅ Multi Container (특별한 경우):
> - Sidecar: 로깅, 모니터링
> - Ambassador: 프록시
> - Adapter: 데이터 변환
> 
> 판단 기준:
> 1. localhost로 통신해야 하는가? → Yes: 같은 Pod
> 2. 항상 함께 스케일링? → Yes: 같은 Pod
> 3. 같은 라이프사이클? → Yes: 같은 Pod
> 4. 리소스 공유 필요? → Yes: 같은 Pod
> 
> 예:
> ❌ Web + Database → 별도 Pod (다른 스케일링)
> ✅ Web + Log Collector → 같은 Pod (Sidecar)
> ❌ Frontend + Backend → 별도 Pod (느슨한 결합)
> ✅ App + Metrics Exporter → 같은 Pod (밀접한 결합)
> ```
> 
> **2) Init Container vs Main Container:**
> 
> ```
> Init Container:
> - 순차적 실행 (순서 보장)
> - Main Container 전에 실행
> - 완료 후 종료 (계속 실행 안 함)
> - 실패하면 Pod 시작 안 함
> 
> 용도:
> ✅ 전제 조건 확인 (DB 준비 대기)
> ✅ 초기 설정 (Config 다운로드)
> ✅ Migration (DB 스키마 업데이트)
> ✅ 보안 (Secret 준비)
> 
> Main Container:
> - 병렬 실행
> - 계속 실행
> - 재시작 정책 적용
> 
> 비유:
> Init Container = 공사 전 준비 작업
> Main Container = 실제 건물
> 
> 예:
> initContainers:
>   - wait-for-db    # 1. DB 대기
>   - run-migration  # 2. Migration
>   - setup-config   # 3. 설정
> containers:
>   - app            # 4. 앱 실행
> ```
> 
> **3) Liveness vs Readiness Probe:**
> 
> ```
> 둘 다 필요합니다!
> 
> Liveness Probe (살아있는가?):
> - 실패 → 컨테이너 재시작
> - 용도: 데드락, 무한 루프 감지
> - 예: /health (항상 200 OK)
> 
> Readiness Probe (트래픽 받을 준비?):
> - 실패 → Service에서 제외
> - 용도: 워밍업, DB 연결 대기
> - 예: /ready (DB 연결되면 200 OK)
> 
> 시나리오:
> 1. 앱 시작
>    Readiness: Fail → 트래픽 안 받음
>    Liveness: Pass → 재시작 안 함
> 
> 2. 워밍업 중
>    Readiness: Fail → 트래픽 안 받음
>    Liveness: Pass → 재시작 안 함
> 
> 3. 준비 완료
>    Readiness: Pass → 트래픽 받음
>    Liveness: Pass → 정상
> 
> 4. DB 연결 끊김
>    Readiness: Fail → 트래픽 안 받음
>    Liveness: Pass → 재시작 안 함 (일시적)
> 
> 5. 데드락 발생
>    Readiness: Fail → 트래픽 안 받음
>    Liveness: Fail → 재시작 (복구 시도)
> 
> Best Practice:
> ✅ 항상 둘 다 설정
> ✅ Liveness는 보수적으로 (재시작 비용 높음)
> ✅ Readiness는 적극적으로 (트래픽 제외 비용 낮음)
> ```

---

<div align="center">

**[⬅️ 이전: Docker to K8s](./01-Docker-to-K8s.md)** | **[다음: Deployment Patterns ➡️](./03-Deployment-Patterns.md)**

</div>
