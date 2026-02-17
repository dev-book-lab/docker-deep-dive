# 03. Deployment Patterns - Deployment와 StatefulSet

## 🎯 이 챕터에서 배울 것

- **Deployment**: Stateless 애플리케이션 배포
- **ReplicaSet**: Pod 복제 관리
- **StatefulSet**: Stateful 애플리케이션 배포
- **DaemonSet**: 모든 노드에 배포
- **Job & CronJob**: 일회성/주기적 작업
- **롤링 업데이트**: 무중단 배포
- **실전 패턴**: 배포 전략 선택

## 📌 왜 중요한가?

**"Pod는 직접 생성하지 않습니다. Deployment, StatefulSet 등을 사용합니다."**

```
워크로드 컨트롤러의 필요성:

Pod 직접 생성 (❌ 권장 안 함):
┌─────────────────────────────────────────────────┐
│ kubectl run nginx --image=nginx                 │
│                                                 │
│ 문제:                                            │
│ ❌ Pod 죽으면 자동 재시작 안 됨                      │
│ ❌ 스케일링 수동                                   │
│ ❌ 롤링 업데이트 불가                               │
│ ❌ 히스토리 관리 안 됨                              │
└─────────────────────────────────────────────────┘

Deployment 사용 (✅ 권장):
┌─────────────────────────────────────────────────┐
│ Deployment                                      │
│   ↓ 관리                                         │
│ ReplicaSet                                      │
│   ↓ 관리                                         │
│ Pods (replica: 3)                               │
│                                                 │
│ 장점:                                            │
│ ✅ 자동 복구 (Self-healing)                       │
│ ✅ 자동 스케일링 (HPA)                             │
│ ✅ 롤링 업데이트                                   │
│ ✅ 롤백 가능                                      │
│ ✅ 버전 히스토리                                   │
└─────────────────────────────────────────────────┘

워크로드 컨트롤러 종류:
┌──────────────┬────────────────────────────────┐
│ 컨트롤러       │ 용도                            │
├──────────────┼────────────────────────────────┤
│ Deployment   │ Stateless 앱 (Web, API)        │
├──────────────┼────────────────────────────────┤
│ StatefulSet  │ Stateful 앱 (DB, Queue)        │
├──────────────┼────────────────────────────────┤
│ DaemonSet    │ 모든 노드 (로깅, 모니터링)           │
├──────────────┼────────────────────────────────┤
│ Job          │ 일회성 작업 (Batch)               │
├──────────────┼────────────────────────────────┤
│ CronJob      │ 주기적 작업 (Backup)              │
└──────────────┴────────────────────────────────┘

Deployment vs StatefulSet:
┌─────────────────────────────────────────────────┐
│ Deployment (Stateless):                         │
│  ┌─────┐  ┌─────┐  ┌─────┐                      │
│  │Pod-1│  │Pod-2│  │Pod-3│                      │
│  └─────┘  └─────┘  └─────┘                      │
│  - 랜덤 이름 (hash)                               │
│  - 순서 없음                                      │
│  - 어느 Pod든 동일                                 │
│  - 공유 스토리지 (PVC)                             │
│                                                 │
│ StatefulSet (Stateful):                         │
│  ┌─────┐  ┌─────┐  ┌─────┐                      │
│  │ -0  │  │ -1  │  │ -2  │                      │
│  └──┬──┘  └──┬──┘  └──┬──┘                      │
│     │        │        │                         │
│  ┌──▼──┐  ┌─▼───┐  ┌─▼───┐                      │
│  │PVC-0│  │PVC-1│  │PVC-2│                      │
│  └─────┘  └─────┘  └─────┘                      │
│  - 순서 이름 (0, 1, 2)                            │
│  - 순서 보장 (0 → 1 → 2)                          │
│  - 고유 Identity                                 │
│  - 개별 스토리지                                   │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **안정성**: 자동 복구, Self-healing
- **확장성**: 손쉬운 스케일링
- **배포**: 무중단 업데이트
- **관리**: 선언적 관리

---

## 🔧 실습 1: Deployment 기본

### Step 1: 기본 Deployment

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  # 레플리카 수
  replicas: 3
  
  # Pod 선택자
  selector:
    matchLabels:
      app: nginx
  
  # Pod 템플릿
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

```bash
# 배포
kubectl apply -f nginx-deployment.yaml

# 확인
kubectl get deployments
kubectl get replicasets
kubectl get pods

# 상세 정보
kubectl describe deployment nginx-deployment

# 스케일링
kubectl scale deployment nginx-deployment --replicas=5

# 또는 YAML 수정
kubectl edit deployment nginx-deployment

# 롤아웃 상태
kubectl rollout status deployment nginx-deployment
```

### Step 2: Service와 함께

```yaml
# nginx-deployment-with-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
```

---

## 🔧 실습 2: 롤링 업데이트

### Step 1: 업데이트 전략

```yaml
# deployment-with-strategy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  
  # 업데이트 전략
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # 최대 2개 추가 Pod
      maxUnavailable: 1  # 최대 1개 불가 Pod
  
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
        image: myapp:v1
        ports:
        - containerPort: 8080
        
        # Readiness Probe (트래픽 받기 전 확인)
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
```

### Step 2: 롤링 업데이트 실행

```bash
# 이미지 업데이트 (v1 → v2)
kubectl set image deployment/myapp myapp=myapp:v2

# 또는 YAML 수정
kubectl edit deployment myapp

# 롤아웃 진행 상황
kubectl rollout status deployment myapp

# 롤아웃 히스토리
kubectl rollout history deployment myapp

# 롤백 (이전 버전)
kubectl rollout undo deployment myapp

# 특정 버전으로 롤백
kubectl rollout undo deployment myapp --to-revision=2

# 롤아웃 일시 정지
kubectl rollout pause deployment myapp

# 롤아웃 재개
kubectl rollout resume deployment myapp
```

### Step 3: 업데이트 과정 관찰

```bash
# 실시간 Pod 변화 관찰
watch kubectl get pods

# 출력 예:
# NAME                    READY   STATUS              RESTARTS   AGE
# myapp-v1-abc123         1/1     Running             0          5m
# myapp-v1-def456         1/1     Running             0          5m
# myapp-v1-ghi789         1/1     Terminating         0          5m
# myapp-v2-jkl012         1/1     Running             0          10s
# myapp-v2-mno345         0/1     ContainerCreating   0          2s

# 롤아웃 이벤트
kubectl describe deployment myapp

# ReplicaSet 변화
kubectl get replicasets
```

---

## 🔧 실습 3: StatefulSet (Database)

### Step 1: 기본 StatefulSet

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  
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
          name: postgres
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  
  # Volume Claim Templates (각 Pod마다 개별 PVC)
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
# Headless Service (StatefulSet 필수)
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None  # Headless
  selector:
    app: postgres
  ports:
  - port: 5432
    name: postgres
```

```bash
# 배포
kubectl apply -f postgres-statefulset.yaml

# 확인
kubectl get statefulsets
kubectl get pods
# postgres-0
# postgres-1
# postgres-2

# PVC 확인 (각 Pod마다)
kubectl get pvc
# postgres-storage-postgres-0
# postgres-storage-postgres-1
# postgres-storage-postgres-2

# 순차적 생성 확인
# 0 → Running 후 → 1 → Running 후 → 2

# 특정 Pod 접속
kubectl exec -it postgres-0 -- psql -U postgres

# DNS 확인
# postgres-0.postgres
# postgres-1.postgres
# postgres-2.postgres
```

### Step 2: StatefulSet 스케일링

```bash
# 스케일 업 (순차적)
kubectl scale statefulset postgres --replicas=5
# postgres-3 생성 → postgres-4 생성

# 스케일 다운 (역순)
kubectl scale statefulset postgres --replicas=2
# postgres-4 삭제 → postgres-3 삭제

# 주의: PVC는 자동 삭제 안 됨 (데이터 보존)
kubectl get pvc
```

---

## 🔧 실습 4: DaemonSet (로깅 에이전트)

### Step 1: DaemonSet

```yaml
# fluentd-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluentd
  
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      # 모든 노드에 배포 (tolerations)
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
        - name: FLUENT_ELASTICSEARCH_PORT
          value: "9200"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
      
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

```bash
# 배포
kubectl apply -f fluentd-daemonset.yaml

# 확인 (노드 수만큼 Pod)
kubectl get daemonset -n kube-system
kubectl get pods -n kube-system -l app=fluentd

# 새 노드 추가하면 자동으로 Pod 생성
```

---

## 🔧 실습 5: Job과 CronJob

### Step 1: Job (일회성)

```yaml
# backup-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: database-backup
spec:
  # 재시도
  backoffLimit: 3
  
  # 완료 조건
  completions: 1
  parallelism: 1
  
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: backup
        image: postgres:15
        command:
        - sh
        - -c
        - |
          pg_dump -h postgres -U postgres mydb > /backup/backup.sql
          echo "Backup completed at $(date)"
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: backup-storage
          mountPath: /backup
      volumes:
      - name: backup-storage
        persistentVolumeClaim:
          claimName: backup-pvc
```

```bash
# 실행
kubectl apply -f backup-job.yaml

# 상태 확인
kubectl get jobs
kubectl get pods

# 로그 확인
kubectl logs job/database-backup

# 완료 후 삭제
kubectl delete job database-backup
```

### Step 2: CronJob (주기적)

```yaml
# cronjob-backup.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  # 스케줄 (Cron 표현식)
  schedule: "0 2 * * *"  # 매일 새벽 2시
  
  # 동시 실행 정책
  concurrencyPolicy: Forbid  # 동시 실행 금지
  
  # 성공/실패 히스토리
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:15
            command:
            - sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              pg_dump -h postgres -U postgres mydb > /backup/backup_${TIMESTAMP}.sql
              echo "Backup completed: backup_${TIMESTAMP}.sql"
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

```bash
# 배포
kubectl apply -f cronjob-backup.yaml

# 확인
kubectl get cronjobs
kubectl get jobs

# 수동 실행 (테스트)
kubectl create job --from=cronjob/nightly-backup test-backup

# 스케줄 수정
kubectl edit cronjob nightly-backup

# 일시 정지
kubectl patch cronjob nightly-backup -p '{"spec":{"suspend":true}}'

# 재개
kubectl patch cronjob nightly-backup -p '{"spec":{"suspend":false}}'
```

---

## 🔧 실습 6: HorizontalPodAutoscaler (자동 스케일링)

### Step 1: HPA 설정

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  
  minReplicas: 2
  maxReplicas: 10
  
  metrics:
  # CPU 기반
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory 기반
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom Metric (예: Request per second)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

```bash
# Metrics Server 설치 (필수)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# HPA 배포
kubectl apply -f hpa.yaml

# 확인
kubectl get hpa
kubectl describe hpa myapp-hpa

# 실시간 모니터링
watch kubectl get hpa

# 부하 테스트
kubectl run -it load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://myapp-service; done"

# HPA 동작 확인
kubectl get hpa -w
```

---

## 🔧 실습 7: 배포 전략 비교

### Step 1: Recreate (전체 중단)

```yaml
# recreate-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-recreate
spec:
  replicas: 3
  
  strategy:
    type: Recreate  # 모두 종료 후 새로 생성
  
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
        image: myapp:v2
```

```bash
# 업데이트 과정
# 1. 모든 v1 Pod 종료
# 2. 모든 v2 Pod 생성
# → 다운타임 발생 (수 초)
```

### Step 2: RollingUpdate (무중단)

```yaml
# rolling-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-rolling
spec:
  replicas: 10
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3        # 최대 13개 Pod (10 + 3)
      maxUnavailable: 2  # 최소 8개 Pod (10 - 2)
  
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
        image: myapp:v2
```

### Step 3: Blue-Green (즉시 전환)

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:v1
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:v2
---
# service.yaml (전환용)
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # blue → green으로 전환
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Blue 배포
kubectl apply -f blue-deployment.yaml

# Green 배포 (대기)
kubectl apply -f green-deployment.yaml

# 서비스는 Blue 가리킴
kubectl apply -f service.yaml

# Green으로 전환 (즉시)
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"green"}}}'

# 문제 있으면 Blue로 롤백
kubectl patch service myapp-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

---

## 💡 배포 전략 선택 가이드

```
┌────────────────┬────────────┬────────────┬────────────┐
│ 전략            │ 다운타임     │ 리소스       │ 롤백 속도    │
├────────────────┼────────────┼────────────┼────────────┤
│ Recreate       │ 있음        │ 낮음        │ 빠름        │
├────────────────┼────────────┼────────────┼────────────┤
│ RollingUpdate  │ 없음        │ 중간        │ 느림        │
├────────────────┼────────────┼────────────┼────────────┤
│ Blue-Green     │ 없음        │ 높음 (2배)  │ 즉시         │
├────────────────┼────────────┼────────────┼────────────┤
│ Canary         │ 없음        │ 중간        │ 중간        │
└────────────────┴────────────┴────────────┴────────────┘

사용 시나리오:

Recreate:
✅ 개발/테스트 환경
✅ 다운타임 허용
✅ 리소스 제한

RollingUpdate:
✅ 프로덕션 (대부분)
✅ 무중단 배포 필요
✅ 점진적 전환

Blue-Green:
✅ 즉시 롤백 필요
✅ 리소스 여유
✅ 테스트 시간 충분

Canary:
✅ 위험한 변경
✅ A/B 테스팅
✅ 점진적 확대
```

---

## 📚 참고 자료

- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [DaemonSets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

---

## 🤔 생각해볼 문제

1. Deployment와 StatefulSet 중 어느 것을 사용해야 하는가?
2. 롤링 업데이트 중 문제가 발생하면 어떻게 되는가?
3. HPA는 만능인가?

> 💡 **답변**:
> 
> **1) Deployment vs StatefulSet:**
> 
> ```
> Deployment 사용 (Stateless):
> ✅ Web Server (Nginx, Apache)
> ✅ API Server (Node.js, Python, Go)
> ✅ Frontend (React, Vue)
> ✅ Stateless Service
> 
> 특징:
> - Pod 교체 가능 (interchangeable)
> - 공유 스토리지
> - 랜덤 이름
> - 빠른 스케일링
> 
> StatefulSet 사용 (Stateful):
> ✅ Database (MySQL, PostgreSQL, MongoDB)
> ✅ Message Queue (Kafka, RabbitMQ)
> ✅ Cache (Redis with persistence)
> ✅ Distributed System (Elasticsearch, ZooKeeper)
> 
> 특징:
> - Pod 고유 Identity 필요
> - 개별 스토리지 (각 Pod마다 PVC)
> - 순서 보장 (0 → 1 → 2)
> - 느린 스케일링 (순차적)
> 
> 판단 기준:
> 1. 데이터 지속성? → StatefulSet
> 2. Pod 간 순서? → StatefulSet
> 3. 고유 Identity? → StatefulSet
> 4. 위 모두 No? → Deployment
> 
> 실무 팁:
> - Database는 K8s 외부에서 관리 (RDS, Cloud SQL)
> - StatefulSet은 복잡함, 가능하면 피함
> ```
> 
> **2) 롤링 업데이트 중 문제 발생:**
> 
> ```
> Readiness Probe가 중요!
> 
> 시나리오:
> 1. v2 Pod 생성
> 2. Readiness Probe 실패
> 3. Service에 추가 안 됨 (트래픽 안 받음)
> 4. 계속 실패
> 5. progressDeadlineSeconds 초과
> 6. 롤아웃 중단
> 
> spec:
>   progressDeadlineSeconds: 600  # 10분
> 
> 결과:
> - v1 Pod 유지 (트래픽 계속 처리)
> - v2 Pod 생성 중단
> - 자동 롤백 안 함 (수동 필요)
> 
> kubectl rollout status deployment myapp
> # error: deployment "myapp" exceeded its progress deadline
> 
> kubectl rollout undo deployment myapp
> 
> Best Practice:
> ✅ Readiness Probe 필수
> ✅ progressDeadlineSeconds 설정
> ✅ maxUnavailable 보수적으로 (1-2)
> ✅ 모니터링 (rollout status)
> ```
> 
> **3) HPA의 한계:**
> 
> ```
> HPA가 못 하는 것:
> 
> ❌ CPU/Memory만으로 충분 안 함
>    - Request Queue Length
>    - Response Time
>    - Business Metric
>    → Custom Metrics 필요
> 
> ❌ 즉시 스케일링 안 됨
>    - Metric 수집 (15초)
>    - 의사 결정 (15초)
>    - Pod 시작 (30초+)
>    → 최소 1분
>    → Burst 트래픽 대응 못 함
> 
> ❌ 무한정 스케일 안 됨
>    - maxReplicas 제한
>    - 클러스터 리소스 제한
>    - 비용
> 
> ❌ 스케일 다운 너무 빠름
>    - 트래픽 급감 시 Pod 삭제
>    - 다시 증가하면 또 생성
>    → Flapping
> 
> 해결:
> ✅ KEDA (Event-driven autoscaling)
> ✅ VPA (Vertical Pod Autoscaler)
> ✅ Cluster Autoscaler (노드 추가)
> ✅ 적절한 behavior 설정
> 
> behavior:
>   scaleDown:
>     stabilizationWindowSeconds: 300  # 5분 대기
>     policies:
>     - type: Percent
>       value: 50  # 최대 50%씩만 감소
>       periodSeconds: 60
> 
> 결론: HPA는 기본, 추가 도구 필요
> ```

---

## 📌 핵심 요약

```
Deployment Patterns:

컨트롤러 선택:
- Stateless → Deployment
- Stateful → StatefulSet
- 모든 노드 → DaemonSet
- 일회성 → Job
- 주기적 → CronJob

배포 전략:
- Recreate: 다운타임 OK
- RollingUpdate: 무중단 (기본)
- Blue-Green: 즉시 롤백
- Canary: 점진적 확대

Best Practices:
✅ Pod 직접 생성 금지
✅ Readiness Probe 필수
✅ Resource Limits 설정
✅ HPA로 자동 스케일링
✅ 롤백 계획 수립
```

---

<div align="center">

**[⬅️ 이전: Pod Concepts](./02-Pod-Concepts.md)** | **[다음: Migration Guide ➡️](./04-Migration-Guide.md)**

</div>
