# 06. Health Checks - 헬스 체크 전략

## 🎯 이 챕터에서 배울 것

- **Health Check 개념**: Liveness vs Readiness vs Startup Probe
- **헬스 체크 전략**: HTTP, TCP, Command 기반
- **실패 처리**: 재시작, 트래픽 차단
- **고급 패턴**: Deep Health Check, Dependency Check
- **성능 고려사항**: Timeout, Threshold 설정
- **실전 구현**: 다양한 헬스 체크 시나리오

## 📌 왜 중요한가?

**"Health Check는 애플리케이션의 자동 복구와 안정적인 트래픽 라우팅의 핵심입니다."**

```
Health Check의 핵심:

Without Health Check (문제):
┌────────────────────────────────────────────────┐
│ Service (Load Balancer)                        │
│                                                │
│  트래픽 라우팅:                                   │
│  Round Robin (무조건 순환)                        │
└────┬────────┬────────┬────────┬────────────────┘
     │        │        │        │
     ▼        ▼        ▼        ▼
  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
  │Pod 1│ │Pod 2│ │Pod 3│ │Pod 4│
  │ ✅  │ │ ❌  │ │ ✅ │ │ ❌  │
  │Ready│ │Crash│ │Ready│ │Slow │
  └─────┘ └─────┘ └─────┘ └─────┘
     │        │        │        │
     │        │        │        │
  요청 성공  502 에러  요청 성공  타임아웃

문제점:
❌ 50% 요청 실패 (Pod 2, 4로 가는 트래픽)
❌ 사용자 경험 저하
❌ 수동 개입 필요
❌ 장애 지속

With Health Check:
┌─────────────────────────────────────────────────┐
│ Service (Smart Load Balancer)                   │
│                                                 │
│  트래픽 라우팅:                                    │
│  Only to Healthy Pods                           │
└────┬────────────────┬───────────────────────────┘
     │                │
     │ Health Check   │ Health Check
     ▼                ▼
  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
  │Pod 1│ │Pod 2│ │Pod 3│ │Pod 4│
  │ ✅  │ │ ❌  │ │ ✅  │ │ ❌  │
  │Ready│ │Crash│ │Ready│ │Slow │
  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘
     │       │       │       │
  트래픽 O  제외 X  트래픽 O  제외 X
     │               │
     └───────┬───────┘
             │
         100% 성공!

장점:
✅ 자동 장애 감지
✅ 트래픽 자동 우회
✅ 자동 재시작 (Liveness)
✅ 사용자 영향 최소화

3가지 Probe:
┌─────────────────┬──────────────┬──────────────┐
│ Probe           │ 용도          │ 실패 시        │
├─────────────────┼──────────────┼──────────────┤
│ Liveness        │ 살아있는가?     │ 재시작        │
├─────────────────┼──────────────┼──────────────┤
│ Readiness       │ 준비됐는가?     │ 트래픽 차단     │
├─────────────────┼──────────────┼──────────────┤
│ Startup         │ 시작됐는가?     │ 대기 또는      │
│                 │              │ 재시작         │
└─────────────────┴──────────────┴──────────────┘

Liveness Probe:
┌──────────────────────────────────────────────┐
│ Container가 살아있는지 확인                      │
│                                              │
│ 실패 시: Container 재시작                       │
│                                              │
│ 예:                                          │
│ - Deadlock 감지                               │
│ - Out of Memory                              │
│ - Infinite Loop                              │
│                                              │
│ GET /health → 200 OK                         │
│             → 500 Error → 재시작!              │
└──────────────────────────────────────────────┘

Readiness Probe:
┌──────────────────────────────────────────────┐
│ Container가 트래픽을 받을 준비가 됐는지             │
│                                              │
│ 실패 시: Service에서 제외 (재시작 X)               │
│                                              │
│ 예:                                          │
│ - DB 연결 끊김 (일시적)                          │
│ - 의존 서비스 장애                               │
│ - 초기화 진행 중                                │
│                                              │
│ GET /ready → 200 OK → 트래픽 O                 │
│            → 503 Error → 트래픽 X              │
│                                              │
│ (Container는 살아있지만 요청 처리 불가)             │
└──────────────────────────────────────────────┘

Startup Probe:
┌──────────────────────────────────────────────┐
│ Container가 시작을 완료했는지 확인                 │
│                                              │
│ 실패 시: Liveness/Readiness 시작 안 함           │
│         (Startup 성공까지 대기)                 │
│                                              │
│ 예:                                          │
│ - 느린 초기화 (대용량 데이터 로드)                  │
│ - JVM Warm-up                                │
│ - ML Model 로딩                               │
│                                              │
│ Startup 성공 후 → Liveness/Readiness 시작       │
└──────────────────────────────────────────────┘

실행 흐름:
1. Container 시작
2. Startup Probe 확인 (성공할 때까지)
3. Liveness Probe 시작 (주기적)
4. Readiness Probe 시작 (주기적)
```

**실무 영향:**
- **가용성**: 자동 장애 복구로 다운타임 최소화
- **신뢰성**: 문제 있는 Pod 자동 제외
- **배포 안정성**: 새 버전이 준비될 때까지 대기
- **사용자 경험**: 에러 없는 안정적 서비스

---

## 🔬 Deep Dive

### 1. Probe 타입

#### HTTP GET

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Health-Check
  initialDelaySeconds: 30  # 시작 후 30초 대기
  periodSeconds: 10        # 10초마다 체크
  timeoutSeconds: 5        # 5초 Timeout
  successThreshold: 1      # 1번 성공 시 OK
  failureThreshold: 3      # 3번 연속 실패 시 재시작

# 동작:
# GET http://container-ip:8080/health
# Response 200-399: 성공
# Response 400+: 실패
# Timeout: 실패
```

#### TCP Socket

```yaml
livenessProbe:
  tcpSocket:
    port: 3306  # MySQL
  initialDelaySeconds: 15
  periodSeconds: 20

# 동작:
# TCP 연결 시도
# 연결 성공: OK
# 연결 실패/Timeout: 실패
```

#### Exec Command

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5

# 동작:
# 컨테이너 내부에서 명령 실행
# Exit Code 0: 성공
# Exit Code 1+: 실패
```

---

### 2. 설정 파라미터

```yaml
probe:
  initialDelaySeconds: 30
  # Container 시작 후 첫 체크까지 대기 시간
  # 너무 짧으면: 아직 준비 안 됐는데 체크 → 실패
  # 너무 길면: 실제 장애 감지 늦음
  
  periodSeconds: 10
  # 체크 간격
  # 짧으면: 리소스 사용 증가
  # 길면: 장애 감지 늦음
  
  timeoutSeconds: 5
  # 체크 Timeout
  # 응답이 이 시간 내 없으면 실패
  
  successThreshold: 1
  # 연속 성공 횟수 (Ready로 전환)
  # Liveness는 항상 1
  
  failureThreshold: 3
  # 연속 실패 횟수 (재시작 또는 Not Ready)
  # 일시적 장애 허용
```

---

## 🔧 실습 1: 기본 Health Check

### Step 1: Simple Health Endpoint

```python
# app/main.py
from flask import Flask, jsonify
import time
import random

app = Flask(__name__)

# 앱 상태
app_state = {
    'started_at': time.time(),
    'ready': False,
    'healthy': True,
    'request_count': 0
}

@app.route('/')
def index():
    """메인 엔드포인트"""
    app_state['request_count'] += 1
    
    # 10% 확률로 에러 (테스트용)
    if random.random() < 0.1:
        return jsonify({'error': 'Random error'}), 500
    
    return jsonify({
        'message': 'Hello, World!',
        'request_count': app_state['request_count']
    })

@app.route('/health')
def health():
    """Liveness Probe: 프로세스가 살아있는가?"""
    if not app_state['healthy']:
        return jsonify({
            'status': 'unhealthy',
            'reason': 'Application marked as unhealthy'
        }), 503
    
    return jsonify({
        'status': 'healthy',
        'uptime': time.time() - app_state['started_at']
    })

@app.route('/ready')
def ready():
    """Readiness Probe: 트래픽을 받을 준비가 됐는가?"""
    # 시작 후 10초 대기 (초기화 시뮬레이션)
    if time.time() - app_state['started_at'] < 10:
        return jsonify({
            'status': 'not ready',
            'reason': 'Still initializing'
        }), 503
    
    if not app_state['ready']:
        return jsonify({
            'status': 'not ready',
            'reason': 'Dependencies not available'
        }), 503
    
    return jsonify({
        'status': 'ready',
        'uptime': time.time() - app_state['started_at']
    })

@app.route('/startup')
def startup():
    """Startup Probe: 시작이 완료됐는가?"""
    # 5초 후 시작 완료
    if time.time() - app_state['started_at'] < 5:
        return jsonify({
            'status': 'starting',
            'progress': (time.time() - app_state['started_at']) / 5 * 100
        }), 503
    
    app_state['ready'] = True
    return jsonify({
        'status': 'started',
        'uptime': time.time() - app_state['started_at']
    })

# 관리 엔드포인트 (테스트용)
@app.route('/admin/set-healthy/<value>')
def set_healthy(value):
    """헬스 상태 변경"""
    app_state['healthy'] = value.lower() == 'true'
    return jsonify({'healthy': app_state['healthy']})

@app.route('/admin/set-ready/<value>')
def set_ready(value):
    """준비 상태 변경"""
    app_state['ready'] = value.lower() == 'true'
    return jsonify({'ready': app_state['ready']})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Step 2: Kubernetes Deployment

```yaml
# deployment-with-probes.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        
        # Startup Probe (시작 확인)
        startupProbe:
          httpGet:
            path: /startup
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 2
          failureThreshold: 30  # 최대 60초 대기
        
        # Liveness Probe (생존 확인)
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness Probe (준비 확인)
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2
        
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
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

### Step 3: 테스트

```bash
# 1. 배포
kubectl apply -f deployment-with-probes.yaml

# 2. Pod 상태 확인
kubectl get pods -w
# NAME                    READY   STATUS    RESTARTS   AGE
# myapp-xxx-yyy           0/1     Running   0          2s   ← Startup 중
# myapp-xxx-yyy           0/1     Running   0          7s   ← Startup 완료, Readiness 대기
# myapp-xxx-yyy           1/1     Running   0          12s  ← Ready!

# 3. Probe 이벤트 확인
kubectl describe pod myapp-xxx-yyy
# Events:
#   Startup probe failed: HTTP probe failed
#   Readiness probe failed: HTTP probe failed
#   Readiness probe succeeded

# 4. 장애 시뮬레이션 - Unhealthy
kubectl exec -it myapp-xxx-yyy -- curl localhost:8080/admin/set-healthy/false

# 5. Liveness 실패 확인
kubectl get pods -w
# myapp-xxx-yyy   1/1   Running   0     30s
# myapp-xxx-yyy   1/1   Running   1     40s  ← 재시작됨!

# 6. Readiness 테스트
kubectl exec -it myapp-xxx-yyy -- curl localhost:8080/admin/set-ready/false

# 7. Service에서 제외 확인
kubectl get endpoints myapp
# NAME    ENDPOINTS
# myapp   10.1.1.2:8080,10.1.1.3:8080  ← 1개 제외됨 (3개 → 2개)

# 8. 복구
kubectl exec -it myapp-xxx-yyy -- curl localhost:8080/admin/set-ready/true

# 9. Service에 재포함
kubectl get endpoints myapp
# NAME    ENDPOINTS
# myapp   10.1.1.2:8080,10.1.1.3:8080,10.1.1.4:8080  ← 복구됨
```

---

## 🔧 실습 2: Deep Health Check (의존성 체크)

### Step 1: 의존성 포함 Health Check

```python
# app/health.py
from flask import Flask, jsonify
import requests
import psycopg2

app = Flask(__name__)

def check_database():
    """데이터베이스 연결 확인"""
    try:
        conn = psycopg2.connect(
            host='postgres',
            database='mydb',
            user='user',
            password='pass',
            connect_timeout=3
        )
        conn.close()
        return True, "Database OK"
    except Exception as e:
        return False, f"Database error: {str(e)}"

def check_redis():
    """Redis 연결 확인"""
    try:
        import redis
        r = redis.Redis(host='redis', port=6379, socket_timeout=3)
        r.ping()
        return True, "Redis OK"
    except Exception as e:
        return False, f"Redis error: {str(e)}"

def check_external_api():
    """외부 API 확인"""
    try:
        response = requests.get(
            'https://api.example.com/health',
            timeout=5
        )
        if response.status_code == 200:
            return True, "External API OK"
        else:
            return False, f"External API returned {response.status_code}"
    except Exception as e:
        return False, f"External API error: {str(e)}"

@app.route('/health')
def health():
    """Liveness: 자체 프로세스만 확인"""
    # 의존성 실패해도 재시작하면 안 됨
    # (의존성은 일시적 장애일 수 있음)
    return jsonify({
        'status': 'healthy',
        'checks': {
            'process': 'ok'
        }
    })

@app.route('/ready')
def ready():
    """Readiness: 모든 의존성 확인"""
    checks = {}
    all_ready = True
    
    # 데이터베이스
    db_ok, db_msg = check_database()
    checks['database'] = {'status': 'ok' if db_ok else 'fail', 'message': db_msg}
    all_ready = all_ready and db_ok
    
    # Redis
    redis_ok, redis_msg = check_redis()
    checks['redis'] = {'status': 'ok' if redis_ok else 'fail', 'message': redis_msg}
    all_ready = all_ready and redis_ok
    
    # 외부 API
    api_ok, api_msg = check_external_api()
    checks['external_api'] = {'status': 'ok' if api_ok else 'fail', 'message': api_msg}
    all_ready = all_ready and api_ok
    
    status_code = 200 if all_ready else 503
    
    return jsonify({
        'status': 'ready' if all_ready else 'not ready',
        'checks': checks
    }), status_code

@app.route('/health/live')
def health_live():
    """간단한 Liveness (빠른 응답)"""
    return jsonify({'status': 'alive'}), 200

@app.route('/health/deep')
def health_deep():
    """Deep Health Check (모든 의존성)"""
    checks = {}
    
    # 모든 체크 수행
    db_ok, db_msg = check_database()
    checks['database'] = {'status': 'ok' if db_ok else 'fail', 'message': db_msg}
    
    redis_ok, redis_msg = check_redis()
    checks['redis'] = {'status': 'ok' if redis_ok else 'fail', 'message': redis_msg}
    
    api_ok, api_msg = check_external_api()
    checks['external_api'] = {'status': 'ok' if api_ok else 'fail', 'message': api_msg}
    
    all_ok = db_ok and redis_ok and api_ok
    
    return jsonify({
        'status': 'healthy' if all_ok else 'degraded',
        'checks': checks
    }), 200  # 항상 200 (모니터링용)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

---

## 🔧 실습 3: TCP와 Exec Probe

### Step 1: Database Health Check

```yaml
# postgres-deployment.yaml
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
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: mypassword
        ports:
        - containerPort: 5432
        
        # TCP Probe
        livenessProbe:
          tcpSocket:
            port: 5432
          initialDelaySeconds: 30
          periodSeconds: 10
        
        # Exec Probe
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - pg_isready -U postgres
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Step 2: Redis Health Check

```yaml
# redis-deployment.yaml
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
        
        # Liveness: TCP
        livenessProbe:
          tcpSocket:
            port: 6379
          initialDelaySeconds: 10
          periodSeconds: 5
        
        # Readiness: Redis PING 명령
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 3
          timeoutSeconds: 1
```

---

## 🔧 실습 4: Custom Health Check Script

### Step 1: 복잡한 Health Check

```python
# health_check.py
#!/usr/bin/env python3
import sys
import psycopg2
import redis
import requests
import time

def check_all():
    """모든 의존성 체크"""
    checks = []
    
    # 1. Database
    try:
        conn = psycopg2.connect(
            host='localhost',
            port=5432,
            user='user',
            password='pass',
            connect_timeout=3
        )
        # 실제 쿼리 실행
        cur = conn.cursor()
        cur.execute('SELECT 1')
        cur.close()
        conn.close()
        checks.append(('database', True, 'OK'))
    except Exception as e:
        checks.append(('database', False, str(e)))
    
    # 2. Redis
    try:
        r = redis.Redis(host='localhost', port=6379, socket_timeout=3)
        r.ping()
        # 읽기/쓰기 테스트
        r.set('healthcheck', 'ok', ex=10)
        value = r.get('healthcheck')
        if value != b'ok':
            raise Exception('Redis read/write failed')
        checks.append(('redis', True, 'OK'))
    except Exception as e:
        checks.append(('redis', False, str(e)))
    
    # 3. Disk Space
    try:
        import shutil
        stat = shutil.disk_usage('/')
        free_percent = stat.free / stat.total * 100
        if free_percent < 10:
            checks.append(('disk', False, f'Low disk space: {free_percent:.1f}%'))
        else:
            checks.append(('disk', True, f'Disk OK: {free_percent:.1f}% free'))
    except Exception as e:
        checks.append(('disk', False, str(e)))
    
    # 4. Memory
    try:
        with open('/proc/meminfo') as f:
            meminfo = {}
            for line in f:
                parts = line.split(':')
                if len(parts) == 2:
                    meminfo[parts[0].strip()] = int(parts[1].strip().split()[0])
        
        mem_available = meminfo.get('MemAvailable', 0)
        mem_total = meminfo.get('MemTotal', 1)
        mem_percent = mem_available / mem_total * 100
        
        if mem_percent < 10:
            checks.append(('memory', False, f'Low memory: {mem_percent:.1f}% available'))
        else:
            checks.append(('memory', True, f'Memory OK: {mem_percent:.1f}% available'))
    except Exception as e:
        checks.append(('memory', False, str(e)))
    
    # 결과 출력 및 Exit Code
    all_ok = True
    for name, ok, msg in checks:
        status = '✅' if ok else '❌'
        print(f"{status} {name}: {msg}")
        all_ok = all_ok and ok
    
    sys.exit(0 if all_ok else 1)

if __name__ == '__main__':
    check_all()
```

```yaml
# deployment-custom-health.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        
        # Custom Health Check Script
        livenessProbe:
          exec:
            command:
            - python3
            - /app/health_check.py
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
```

---

## 🔧 실습 5: Graceful Shutdown with Health Check

### Step 1: PreStop Hook 통합

```yaml
# deployment-graceful.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 5
        
        # Lifecycle Hooks
        lifecycle:
          preStop:
            exec:
              command:
              - sh
              - -c
              - |
                # Readiness를 false로 설정
                curl -X POST localhost:8080/admin/set-ready/false
                
                # 15초 대기 (진행 중인 요청 완료)
                sleep 15
                
                # Graceful Shutdown
                kill -TERM 1
        
        # Termination Grace Period
        terminationGracePeriodSeconds: 30
```

```python
# app/graceful.py
from flask import Flask
import signal
import sys
import time

app = Flask(__name__)

app_state = {
    'ready': True,
    'shutting_down': False
}

def signal_handler(sig, frame):
    """SIGTERM 핸들러"""
    print("Received SIGTERM, starting graceful shutdown...")
    app_state['shutting_down'] = True
    app_state['ready'] = False
    
    # 진행 중인 요청 대기
    time.sleep(5)
    
    print("Shutdown complete")
    sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)

@app.route('/ready')
def ready():
    if app_state['shutting_down'] or not app_state['ready']:
        return jsonify({'status': 'not ready'}), 503
    return jsonify({'status': 'ready'})

@app.route('/admin/set-ready/<value>')
def set_ready(value):
    app_state['ready'] = value.lower() == 'true'
    return jsonify({'ready': app_state['ready']})
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ Probe 타입            │ 용도                        │
├──────────────────────┼────────────────────────────┤
│ HTTP GET             │ 웹 애플리케이션                │
├──────────────────────┼────────────────────────────┤
│ TCP Socket           │ DB, Cache, Message Queue   │
├──────────────────────┼────────────────────────────┤
│ Exec Command         │ 커스텀 체크 스크립트            │
└──────────────────────┴────────────────────────────┘

Health Check 전략:
1. Liveness: 가볍게 (프로세스만)
2. Readiness: 정확하게 (의존성 포함)
3. Startup: 충분한 시간
```

---

## 🎓 연습 문제

### 문제 1: Liveness와 Readiness를 같은 엔드포인트로 사용하면 안 되는 이유는?

<details>
<summary>정답 보기</summary>

**잘못된 설정:**
```yaml
livenessProbe:
  httpGet:
    path: /health  # ← 의존성 체크 포함
readinessProbe:
  httpGet:
    path: /health  # ← 동일한 엔드포인트
```

**문제:**
```
시나리오: DB 일시적 장애

1. /health에서 DB 체크 실패
2. Liveness 실패 → Container 재시작
3. 재시작해도 DB는 여전히 장애
4. 다시 Liveness 실패 → 재시작
5. CrashLoopBackOff 발생!

올바른 해결:
- Liveness: 프로세스 자체만 체크
- Readiness: 의존성 포함 체크
```

**올바른 설정:**
```yaml
livenessProbe:
  httpGet:
    path: /health/live  # 프로세스만
readinessProbe:
  httpGet:
    path: /health/ready  # 의존성 포함
```

```python
@app.route('/health/live')
def health_live():
    # 빠르고 간단
    return jsonify({'status': 'alive'})

@app.route('/health/ready')
def health_ready():
    # DB, Redis 등 체크
    if not check_database():
        return jsonify({'status': 'not ready'}), 503
    return jsonify({'status': 'ready'})
```

</details>

### 문제 2: initialDelaySeconds를 어떻게 설정해야 하는가?

<details>
<summary>정답 보기</summary>

**고려사항:**

**1. 애플리케이션 시작 시간 측정:**
```bash
# 로컬에서 측정
time docker run myapp

# 실제 시간: 8초
# initialDelaySeconds: 10초 (여유)
```

**2. Startup Probe 사용 (권장):**
```yaml
# 느린 시작 앱
startupProbe:
  httpGet:
    path: /startup
  periodSeconds: 5
  failureThreshold: 30  # 최대 150초 (5 × 30)

livenessProbe:
  httpGet:
    path: /health
  initialDelaySeconds: 0  # Startup 완료 후 시작
  periodSeconds: 10
```

**3. 환경별 다르게 설정:**
```yaml
# Development (빠른 이미지)
initialDelaySeconds: 5

# Production (큰 이미지, 데이터 로드)
initialDelaySeconds: 30
```

**안티패턴:**
```yaml
# ❌ 너무 짧음
initialDelaySeconds: 0
# → 앱이 준비 안 됐는데 체크 → 실패 → 재시작

# ❌ 너무 김
initialDelaySeconds: 300
# → 실제 장애 감지 늦음
```

</details>

### 문제 3: Health Check가 리소스를 많이 사용한다면?

<details>
<summary>정답 보기</summary>

**문제:**
```python
# ❌ 무거운 Health Check
@app.route('/health')
def health():
    # DB 풀 스캔
    conn.execute('SELECT COUNT(*) FROM users')
    
    # 외부 API 10개 호출
    for api in apis:
        requests.get(api)
    
    # 파일 시스템 체크
    check_all_files()
    
    return jsonify({'status': 'ok'})

# periodSeconds: 5
# → 5초마다 무거운 작업!
```

**해결:**

**1. 경량화:**
```python
# ✅ 가벼운 Liveness
@app.route('/health')
def health():
    return jsonify({'status': 'ok'})  # 즉시 반환

# ✅ Readiness는 적당히
@app.route('/ready')
def ready():
    # DB Ping만 (쿼리 X)
    conn.ping()
    return jsonify({'status': 'ready'})
```

**2. 주기 조정:**
```yaml
# 가벼운 체크: 짧은 주기
livenessProbe:
  periodSeconds: 5

# 무거운 체크: 긴 주기
readinessProbe:
  periodSeconds: 30
```

**3. 캐싱:**
```python
from functools import lru_cache
import time

@lru_cache(maxsize=1)
def check_dependencies_cached():
    # 결과를 10초 캐싱
    return check_all_dependencies()

@app.route('/ready')
def ready():
    # 캐시된 결과 사용
    if check_dependencies_cached():
        return jsonify({'status': 'ready'})
    return jsonify({'status': 'not ready'}), 503
```

**4. 분리:**
```python
# Liveness: 초경량
@app.route('/health/live')
def health_live():
    return 'ok'

# Deep Check: 별도 엔드포인트 (모니터링용)
@app.route('/health/deep')
def health_deep():
    # 모든 체크 수행 (periodSeconds 길게)
    return jsonify(check_everything())
```

</details>

---

## 📌 핵심 요약

```
Health Check 핵심:
1. Liveness: 프로세스 살아있는가? (재시작)
2. Readiness: 트래픽 받을 준비? (트래픽 차단)
3. Startup: 시작 완료? (대기)

Best Practices:
✅ Liveness는 가볍게
✅ Readiness는 정확하게
✅ 적절한 Timeout 설정
✅ failureThreshold로 일시적 장애 허용
✅ Graceful Shutdown 통합
```

---

## 📚 참고 자료

- [Kubernetes Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Health Check Patterns](https://microservices.io/patterns/observability/health-check-api.html)

---

## 🤔 생각해볼 문제

1. Health Check 엔드포인트를 Public으로 노출해도 되는가?
2. 모든 Pod가 Not Ready 상태가 되면 어떻게 되는가?
3. Health Check가 실패하는 동안의 로그는 어떻게 수집하는가?

> 💡 **답변**:
> 
> **1) Health Check 공개 여부:**
> 
> **보안 고려사항:**
> - 시스템 정보 노출 (버전, 의존성)
> - DoS 공격 표적
> - 내부 구조 파악
> 
> **해결:**
> ```
> Public Health: 최소 정보
> GET /health → 200 OK
> 
> Internal Health: 상세 정보
> GET /health/detailed → {
>   "db": "connected",
>   "redis": "ok",
>   "version": "1.2.3"
> }
> 
> 인증 추가:
> GET /health/admin
> Authorization: Bearer <token>
> ```
> 
> **2) 모든 Pod Not Ready:**
> ```
> Service가 Endpoint 0개 → 503 Error
> 
> 해결:
> 1. minAvailable 설정
>    spec:
>      minAvailable: 1
> 
> 2. PodDisruptionBudget
>    apiVersion: policy/v1
>    kind: PodDisruptionBudget
>    spec:
>      minAvailable: 50%
> ```
> 
> **3) 실패 중 로그 수집:**
> ```
> kubectl logs <pod> --previous
> # 재시작 전 로그
> 
> Fluentd/ELK로 중앙 집중:
> - 실시간 수집
> - 재시작 후에도 보존
> ```

---

<div align="center">

**[⬅️ 이전: Init Containers](./05-Init-Containers.md)** | **[다음: Graceful Shutdown ➡️](./07-Graceful-Shutdown.md)**

</div>
