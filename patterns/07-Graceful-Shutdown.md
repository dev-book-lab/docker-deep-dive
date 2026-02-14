# 07. Graceful Shutdown - 우아한 종료 처리

## 🎯 이 챕터에서 배울 것

- **Graceful Shutdown 개념**: 안전한 종료 프로세스
- **시그널 처리**: SIGTERM, SIGKILL 이해
- **진행 중인 요청 처리**: 연결 드레이닝
- **리소스 정리**: DB 연결, 파일 핸들 종료
- **Zero-Downtime 배포**: 롤링 업데이트 전략
- **실전 구현**: Python, Go, Node.js 예제

## 📌 왜 중요한가?

**"Graceful Shutdown은 서비스 중단 없이 안전하게 컨테이너를 종료합니다."**

```
Graceful Shutdown의 핵심:

Without Graceful Shutdown (강제 종료):
┌─────────────────────────────────────────────────┐
│ Container Termination                           │
│                                                 │
│  kubectl delete pod myapp-xxx                   │
│         │                                       │
│         ▼                                       │
│  ┌────────────────────────────────────────┐     │
│  │ Container                              │     │
│  │                                        │     │
│  │ - Active Request 1 ────────────→ ❌    │     │
│  │ - Active Request 2 ────────────→ ❌    │     │
│  │ - Active Request 3 ────────────→ ❌    │     │
│  │ - DB Connection ──→ 끊김                │     │
│  │ - File Handle ──→ 손실                  │     │
│  │                                        │     │
│  │ SIGKILL (즉시 종료)                      │     │
│  └────────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘

결과:
❌ 진행 중인 요청 실패 (502, 503 에러)
❌ 데이터 손실 (트랜잭션 롤백)
❌ 리소스 누수 (DB 연결 미정리)
❌ 사용자 경험 저하

With Graceful Shutdown:
┌─────────────────────────────────────────────────┐
│ Graceful Termination                            │
│                                                 │
│  kubectl delete pod myapp-xxx                   │
│         │                                       │
│         ▼                                       │
│  1. SIGTERM 전송                                 │
│         │                                       │
│         ▼                                       │
│  ┌────────────────────────────────────────┐     │
│  │ Container                              │     │
│  │                                        │     │
│  │ Phase 1: Stop accepting new requests   │     │
│  │ ┌────────────────────────────────────┐ │     │
│  │ │ Readiness Probe → Not Ready        │ │     │
│  │ │ Service removes from Endpoints     │ │     │
│  │ └────────────────────────────────────┘ │     │
│  │                                        │     │
│  │ Phase 2: Wait for active requests      │     │
│  │ - Active Request 1 ────────────→ ✅    │     │
│  │ - Active Request 2 ────────────→ ✅    │     │
│  │ - Active Request 3 ────────────→ ✅    │     │
│  │                                        │     │
│  │ Phase 3: Cleanup                       │     │
│  │ - Close DB connections                 │     │
│  │ - Flush buffers                        │     │
│  │ - Save state                           │     │
│  │                                        │     │
│  │ Phase 4: Exit cleanly                  │     │
│  └────────────────────────────────────────┘     │
│         │                                       │
│         │ (Grace Period: 30초 기본)               │
│         ▼                                       │
│  2. SIGKILL (30초 후, 강제 종료)                   │
└─────────────────────────────────────────────────┘

결과:
✅ 모든 요청 완료 (에러 없음)
✅ 데이터 무결성 유지
✅ 리소스 정리 완료
✅ Zero-Downtime

시그널 흐름:
┌─────────────────────────────────────────┐
│ Kubernetes Pod Termination              │
│                                         │
│ 1. kubectl delete pod                   │
│    ↓                                    │
│ 2. Pod 상태: Terminating                 │
│    ↓                                    │
│ 3. PreStop Hook 실행 (있으면)              │
│    - 설정한 명령 실행                       │
│    - 완료 대기                            │
│    ↓                                    │
│ 4. SIGTERM 전송                          │
│    - Container의 PID 1 프로세스로          │
│    - 애플리케이션이 처리                     │
│    ↓                                    │
│ 5. Grace Period 대기 (기본 30초)           │
│    - 애플리케이션이 종료할 시간 제공            │
│    ↓                                    │
│ 6. SIGKILL 전송 (타임아웃 시)               │
│    - 강제 종료                            │
└─────────────────────────────────────────┘

Grace Period:
┌──────────────────────────────────────────┐
│ 0초      SIGTERM                          │
│ │                                        │
│ │   애플리케이션이 종료 작업 수행               │
│ │   - 새 요청 거부                          │
│ │   - 진행 중인 요청 완료                     │
│ │   - 리소스 정리                           │
│ │                                        │
│ 30초     SIGKILL (강제 종료)                │
└──────────────────────────────────────────┘

terminationGracePeriodSeconds:
- 기본값: 30초
- 조정 가능: 5초 ~ 수분
- 너무 짧으면: 요청 중단
- 너무 길면: 배포 느림
```

**실무 영향:**
- **가용성**: 배포 중 다운타임 0초
- **데이터 무결성**: 트랜잭션 안전하게 완료
- **사용자 경험**: 에러 없는 부드러운 전환
- **운영 효율성**: 자동화된 안전한 배포

---

## 🔬 Deep Dive

### 1. 시그널 처리

#### SIGTERM vs SIGKILL

```bash
# SIGTERM (15)
# - "정리하고 종료하세요"
# - 애플리케이션이 받아서 처리 가능
# - Graceful shutdown 수행
kill -TERM <pid>

# SIGKILL (9)
# - "즉시 종료"
# - 애플리케이션이 처리 불가
# - OS가 강제 종료
kill -KILL <pid>

# 실제 흐름
1. SIGTERM 전송 → 애플리케이션이 처리
2. 30초 대기
3. 아직 안 끝났으면 SIGKILL → 강제 종료
```

#### Python에서 시그널 처리

```python
import signal
import sys
import time

def signal_handler(sig, frame):
    """SIGTERM 핸들러"""
    print('Received SIGTERM, shutting down gracefully...')
    
    # 1. 새 요청 거부
    app.config['ACCEPTING_REQUESTS'] = False
    
    # 2. 진행 중인 요청 대기
    while app.active_requests > 0:
        print(f'Waiting for {app.active_requests} active requests...')
        time.sleep(1)
    
    # 3. 리소스 정리
    cleanup_resources()
    
    # 4. 종료
    print('Shutdown complete')
    sys.exit(0)

# 시그널 핸들러 등록
signal.signal(signal.SIGTERM, signal_handler)
signal.signal(signal.SIGINT, signal_handler)  # Ctrl+C
```

---

### 2. PreStop Hook

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    lifecycle:
      preStop:
        exec:
          command:
          - sh
          - -c
          - |
            # 1. Readiness를 false로
            curl -X POST localhost:8080/admin/ready/false
            
            # 2. 15초 대기 (진행 중인 요청 완료)
            sleep 15
            
            # 3. Graceful shutdown
            kill -TERM 1
    
    # Grace Period
    terminationGracePeriodSeconds: 30

# 실행 순서:
# 1. PreStop Hook 실행
# 2. SIGTERM 전송
# 3. 30초 대기
# 4. SIGKILL (필요 시)
```

---

## 🔧 실습 1: Python Flask Graceful Shutdown

### Step 1: Graceful Shutdown 구현

```python
# app/main.py
from flask import Flask, jsonify, request
import signal
import sys
import time
import threading

app = Flask(__name__)

# 앱 상태
state = {
    'accepting_requests': True,
    'active_requests': 0,
    'shutdown_initiated': False
}

# Request 카운터 데코레이터
def track_request(f):
    def wrapper(*args, **kwargs):
        if not state['accepting_requests']:
            return jsonify({'error': 'Service shutting down'}), 503
        
        state['active_requests'] += 1
        try:
            return f(*args, **kwargs)
        finally:
            state['active_requests'] -= 1
    
    wrapper.__name__ = f.__name__
    return wrapper

@app.route('/')
@track_request
def index():
    """메인 엔드포인트"""
    # 요청 처리 시뮬레이션
    time.sleep(2)
    return jsonify({
        'message': 'Hello, World!',
        'active_requests': state['active_requests']
    })

@app.route('/slow')
@track_request
def slow():
    """느린 요청 (10초)"""
    time.sleep(10)
    return jsonify({'message': 'Completed slow request'})

@app.route('/health')
def health():
    """Liveness Probe"""
    return jsonify({'status': 'healthy'})

@app.route('/ready')
def ready():
    """Readiness Probe"""
    if not state['accepting_requests']:
        return jsonify({
            'status': 'not ready',
            'reason': 'Shutting down'
        }), 503
    
    return jsonify({'status': 'ready'})

@app.route('/admin/ready/<value>')
def set_ready(value):
    """Readiness 상태 변경"""
    state['accepting_requests'] = value.lower() == 'true'
    return jsonify({'accepting_requests': state['accepting_requests']})

@app.route('/metrics')
def metrics():
    """메트릭"""
    return jsonify({
        'accepting_requests': state['accepting_requests'],
        'active_requests': state['active_requests'],
        'shutdown_initiated': state['shutdown_initiated']
    })

def graceful_shutdown(sig, frame):
    """Graceful Shutdown 핸들러"""
    if state['shutdown_initiated']:
        print('Shutdown already initiated')
        return
    
    state['shutdown_initiated'] = True
    print('\n' + '='*50)
    print('Received SIGTERM, starting graceful shutdown...')
    print('='*50)
    
    # 1. 새 요청 거부
    state['accepting_requests'] = False
    print('✅ Step 1: Stopped accepting new requests')
    
    # 2. 진행 중인 요청 완료 대기
    max_wait = 20  # 최대 20초
    waited = 0
    
    while state['active_requests'] > 0 and waited < max_wait:
        print(f'⏳ Step 2: Waiting for {state["active_requests"]} active request(s)... ({waited}s)')
        time.sleep(1)
        waited += 1
    
    if state['active_requests'] > 0:
        print(f'⚠️  Warning: {state["active_requests"]} request(s) still active after {max_wait}s')
    else:
        print('✅ Step 2: All requests completed')
    
    # 3. 리소스 정리
    print('🧹 Step 3: Cleaning up resources...')
    # DB 연결 종료, 파일 닫기 등
    time.sleep(1)
    print('✅ Step 3: Cleanup complete')
    
    # 4. 종료
    print('='*50)
    print('Shutdown complete - Exiting')
    print('='*50)
    sys.exit(0)

# 시그널 핸들러 등록
signal.signal(signal.SIGTERM, graceful_shutdown)
signal.signal(signal.SIGINT, graceful_shutdown)

if __name__ == '__main__':
    print('Starting application...')
    print('Press Ctrl+C or send SIGTERM for graceful shutdown')
    app.run(host='0.0.0.0', port=8080)
```

### Step 2: Kubernetes Deployment

```yaml
# deployment-graceful.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
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
        
        # Readiness Probe
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3
          failureThreshold: 2
        
        # Liveness Probe
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        
        # PreStop Hook
        lifecycle:
          preStop:
            exec:
              command:
              - sh
              - -c
              - |
                # Readiness를 false로
                curl -X POST http://localhost:8080/admin/ready/false
                
                # 5초 대기 (Service에서 제외되도록)
                sleep 5
        
        # Grace Period (SIGTERM → SIGKILL)
        terminationGracePeriodSeconds: 30
        
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
kubectl apply -f deployment-graceful.yaml

# 2. Pod 이름 확인
POD=$(kubectl get pod -l app=myapp -o jsonpath='{.items[0].metadata.name}')

# 3. 느린 요청 시작 (백그라운드)
kubectl exec -it $POD -- curl localhost:8080/slow &

# 4. 메트릭 확인
kubectl exec -it $POD -- curl localhost:8080/metrics
# {
#   "accepting_requests": true,
#   "active_requests": 1,
#   "shutdown_initiated": false
# }

# 5. Pod 삭제 (Graceful Shutdown 트리거)
kubectl delete pod $POD

# 6. 로그 확인 (실시간)
kubectl logs -f $POD

# 출력:
# ==================================================
# Received SIGTERM, starting graceful shutdown...
# ==================================================
# ✅ Step 1: Stopped accepting new requests
# ⏳ Step 2: Waiting for 1 active request(s)... (0s)
# ⏳ Step 2: Waiting for 1 active request(s)... (1s)
# ...
# ✅ Step 2: All requests completed
# 🧹 Step 3: Cleaning up resources...
# ✅ Step 3: Cleanup complete
# ==================================================
# Shutdown complete - Exiting
# ==================================================
```

---

## 🔧 실습 2: Go HTTP Server Graceful Shutdown

### Step 1: Go 구현

```go
// main.go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "sync/atomic"
    "syscall"
    "time"
)

var (
    activeRequests int64
    acceptingRequests atomic.Bool
)

func main() {
    acceptingRequests.Store(true)
    
    // HTTP Server 설정
    mux := http.NewServeMux()
    
    // Middleware: Request 추적
    handler := trackRequests(mux)
    
    // Routes
    mux.HandleFunc("/", indexHandler)
    mux.HandleFunc("/slow", slowHandler)
    mux.HandleFunc("/health", healthHandler)
    mux.HandleFunc("/ready", readyHandler)
    mux.HandleFunc("/metrics", metricsHandler)
    
    server := &http.Server{
        Addr:    ":8080",
        Handler: handler,
    }
    
    // Graceful Shutdown 설정
    go func() {
        sigChan := make(chan os.Signal, 1)
        signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)
        
        <-sigChan
        fmt.Println("\n" + strings.Repeat("=", 50))
        fmt.Println("Received SIGTERM, starting graceful shutdown...")
        fmt.Println(strings.Repeat("=", 50))
        
        // 1. 새 요청 거부
        acceptingRequests.Store(false)
        fmt.Println("✅ Step 1: Stopped accepting new requests")
        
        // 2. HTTP Server Graceful Shutdown
        ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
        defer cancel()
        
        if err := server.Shutdown(ctx); err != nil {
            log.Printf("Server shutdown error: %v", err)
        }
        
        // 3. 진행 중인 요청 대기
        for i := 0; i < 20; i++ {
            active := atomic.LoadInt64(&activeRequests)
            if active == 0 {
                break
            }
            fmt.Printf("⏳ Step 2: Waiting for %d active request(s)... (%ds)\n", active, i)
            time.Sleep(time.Second)
        }
        
        fmt.Println("✅ Step 2: All requests completed")
        
        // 4. 리소스 정리
        fmt.Println("🧹 Step 3: Cleaning up resources...")
        time.Sleep(time.Second)
        fmt.Println("✅ Step 3: Cleanup complete")
        
        fmt.Println(strings.Repeat("=", 50))
        fmt.Println("Shutdown complete - Exiting")
        fmt.Println(strings.Repeat("=", 50))
        
        os.Exit(0)
    }()
    
    // 서버 시작
    fmt.Println("Starting server on :8080")
    if err := server.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatalf("Server error: %v", err)
    }
}

func trackRequests(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !acceptingRequests.Load() {
            http.Error(w, "Service shutting down", http.StatusServiceUnavailable)
            return
        }
        
        atomic.AddInt64(&activeRequests, 1)
        defer atomic.AddInt64(&activeRequests, -1)
        
        next.ServeHTTP(w, r)
    })
}

func indexHandler(w http.ResponseWriter, r *http.Request) {
    time.Sleep(2 * time.Second)
    fmt.Fprintf(w, "Hello, World! Active requests: %d\n", atomic.LoadInt64(&activeRequests))
}

func slowHandler(w http.ResponseWriter, r *http.Request) {
    time.Sleep(10 * time.Second)
    fmt.Fprintln(w, "Completed slow request")
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, `{"status":"healthy"}`)
}

func readyHandler(w http.ResponseWriter, r *http.Request) {
    if !acceptingRequests.Load() {
        w.WriteHeader(http.StatusServiceUnavailable)
        fmt.Fprintln(w, `{"status":"not ready","reason":"Shutting down"}`)
        return
    }
    fmt.Fprintln(w, `{"status":"ready"}`)
}

func metricsHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, `{
        "accepting_requests": %v,
        "active_requests": %d
    }`, acceptingRequests.Load(), atomic.LoadInt64(&activeRequests))
}
```

---

## 🔧 실습 3: Node.js Express Graceful Shutdown

### Step 1: Node.js 구현

```javascript
// server.js
const express = require('express');
const http = require('http');

const app = express();
const server = http.createServer(app);

let acceptingRequests = true;
let activeRequests = 0;

// Middleware: Request 추적
app.use((req, res, next) => {
    if (!acceptingRequests) {
        return res.status(503).json({ error: 'Service shutting down' });
    }
    
    activeRequests++;
    
    res.on('finish', () => {
        activeRequests--;
    });
    
    next();
});

// Routes
app.get('/', (req, res) => {
    setTimeout(() => {
        res.json({
            message: 'Hello, World!',
            active_requests: activeRequests
        });
    }, 2000);
});

app.get('/slow', (req, res) => {
    setTimeout(() => {
        res.json({ message: 'Completed slow request' });
    }, 10000);
});

app.get('/health', (req, res) => {
    res.json({ status: 'healthy' });
});

app.get('/ready', (req, res) => {
    if (!acceptingRequests) {
        return res.status(503).json({
            status: 'not ready',
            reason: 'Shutting down'
        });
    }
    res.json({ status: 'ready' });
});

app.get('/metrics', (req, res) => {
    res.json({
        accepting_requests: acceptingRequests,
        active_requests: activeRequests
    });
});

// Graceful Shutdown
function gracefulShutdown(signal) {
    console.log('\n' + '='.repeat(50));
    console.log(`Received ${signal}, starting graceful shutdown...`);
    console.log('='.repeat(50));
    
    // 1. 새 요청 거부
    acceptingRequests = false;
    console.log('✅ Step 1: Stopped accepting new requests');
    
    // 2. HTTP Server 종료
    server.close(() => {
        console.log('✅ HTTP server closed');
    });
    
    // 3. 진행 중인 요청 대기
    const maxWait = 20000; // 20초
    const startTime = Date.now();
    
    const interval = setInterval(() => {
        const elapsed = Date.now() - startTime;
        
        if (activeRequests === 0) {
            clearInterval(interval);
            console.log('✅ Step 2: All requests completed');
            cleanup();
        } else if (elapsed > maxWait) {
            clearInterval(interval);
            console.log(`⚠️  Warning: ${activeRequests} request(s) still active after ${maxWait/1000}s`);
            cleanup();
        } else {
            console.log(`⏳ Step 2: Waiting for ${activeRequests} active request(s)... (${Math.floor(elapsed/1000)}s)`);
        }
    }, 1000);
}

function cleanup() {
    console.log('🧹 Step 3: Cleaning up resources...');
    
    // DB 연결 종료, 파일 닫기 등
    setTimeout(() => {
        console.log('✅ Step 3: Cleanup complete');
        console.log('='.repeat(50));
        console.log('Shutdown complete - Exiting');
        console.log('='.repeat(50));
        process.exit(0);
    }, 1000);
}

// 시그널 핸들러
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// 서버 시작
const PORT = process.env.PORT || 8080;
server.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
    console.log('Press Ctrl+C or send SIGTERM for graceful shutdown');
});
```

---

## 🔧 실습 4: Database Connection Cleanup

### Step 1: DB 연결 관리

```python
# app/db.py
import psycopg2
from psycopg2 import pool
import signal
import sys

class DatabaseManager:
    def __init__(self):
        self.connection_pool = psycopg2.pool.SimpleConnectionPool(
            1, 10,  # min, max connections
            host='postgres',
            database='mydb',
            user='user',
            password='pass'
        )
        self.active_connections = []
    
    def get_connection(self):
        """연결 풀에서 연결 가져오기"""
        conn = self.connection_pool.getconn()
        self.active_connections.append(conn)
        return conn
    
    def return_connection(self, conn):
        """연결 반환"""
        if conn in self.active_connections:
            self.active_connections.remove(conn)
        self.connection_pool.putconn(conn)
    
    def cleanup(self):
        """모든 연결 정리"""
        print(f"Closing {len(self.active_connections)} active DB connections...")
        
        # 진행 중인 트랜잭션 롤백
        for conn in self.active_connections:
            try:
                if not conn.closed:
                    conn.rollback()
                    conn.close()
            except Exception as e:
                print(f"Error closing connection: {e}")
        
        # 연결 풀 종료
        self.connection_pool.closeall()
        print("All DB connections closed")

db_manager = DatabaseManager()

def graceful_shutdown(sig, frame):
    print("Shutting down...")
    
    # DB 연결 정리
    db_manager.cleanup()
    
    # 기타 리소스 정리
    # ...
    
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)

# 사용 예
@app.route('/users')
def get_users():
    conn = db_manager.get_connection()
    try:
        cur = conn.cursor()
        cur.execute('SELECT * FROM users')
        users = cur.fetchall()
        cur.close()
        return jsonify(users)
    finally:
        db_manager.return_connection(conn)
```

---

## 🔧 실습 5: Zero-Downtime Deployment

### Step 1: Rolling Update 전략

```yaml
# deployment-zero-downtime.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  
  # Rolling Update 전략
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # 동시에 1개만 종료
      maxSurge: 1            # 동시에 1개만 추가 생성
  
  selector:
    matchLabels:
      app: myapp
  
  template:
    metadata:
      labels:
        app: myapp
        version: v2  # 새 버전
    spec:
      containers:
      - name: app
        image: myapp:v2
        ports:
        - containerPort: 8080
        
        # Readiness Probe (중요!)
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 2
          failureThreshold: 3
          successThreshold: 1  # 1번 성공 시 Ready
        
        # Liveness Probe
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        
        # PreStop Hook
        lifecycle:
          preStop:
            exec:
              command:
              - sh
              - -c
              - |
                # 1. Readiness false
                curl -X POST localhost:8080/admin/ready/false
                
                # 2. 대기 (새 연결 방지)
                sleep 10
        
        # Grace Period
        terminationGracePeriodSeconds: 30

# Rolling Update 흐름:
# 1. 새 Pod 1개 생성 (v2)
# 2. v2 Pod Ready 대기
# 3. v2 Pod Ready → Service에 추가
# 4. 기존 Pod 1개 종료 (v1)
# 5. v1 Pod Graceful Shutdown
# 6. 반복 (5개 모두 교체)
```

### Step 2: PodDisruptionBudget

```yaml
# pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 3  # 최소 3개는 항상 실행
  selector:
    matchLabels:
      app: myapp

# 효과:
# - 배포 중에도 최소 3개 Pod 유지
# - maxUnavailable: 2 (5 - 3)
# - 서비스 가용성 보장
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 단계                  │ 작업                        │
├──────────────────────┼────────────────────────────┤
│ 1. PreStop Hook      │ Readiness → false          │
├──────────────────────┼────────────────────────────┤
│ 2. Stop New Requests │ 새 요청 거부                  │
├──────────────────────┼────────────────────────────┤
│ 3. Drain Connections │ 진행 중인 요청 완료            │
├──────────────────────┼────────────────────────────┤
│ 4. Cleanup           │ DB, 파일 등 정리              │
├──────────────────────┼────────────────────────────┤
│ 5. Exit              │ 안전하게 종료                 │
└──────────────────────┴────────────────────────────┘
```

---

## 🎓 연습 문제

### 문제 1: Grace Period를 얼마로 설정해야 하는가?

<details>
<summary>정답 보기</summary>

**계산 공식:**
```
Grace Period = PreStop Hook 시간 + 최대 요청 시간 + 여유 시간

예:
- PreStop Hook: 5초
- 최대 요청 시간: 10초 (느린 엔드포인트)
- 여유: 5초
- 총: 20초

terminationGracePeriodSeconds: 20
```

**너무 짧으면:**
```
Grace Period: 5초
실제 요청: 10초

→ SIGKILL로 강제 종료
→ 요청 실패
```

**너무 길면:**
```
Grace Period: 300초
실제 종료: 15초

→ 285초 낭비
→ 배포 느려짐
```

**권장:**
```
일반 웹 앱: 30초 (기본값)
API 서버: 20-30초
배치 작업: 60-120초
ML 모델: 120-300초
```

</details>

### 문제 2: Rolling Update 중 502 에러가 발생한다면?

<details>
<summary>정답 보기</summary>

**원인:**
```
1. Readiness Probe 없음
   - 새 Pod가 아직 준비 안 됐는데 트래픽 받음
   
2. initialDelaySeconds 너무 짧음
   - 앱 시작 전에 Ready로 판단
   
3. PreStop Hook 없음
   - 종료 중인 Pod가 트래픽 받음
```

**해결:**

**1. Readiness Probe 추가:**
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10  # 앱 시작 시간
  periodSeconds: 2
  failureThreshold: 3
  successThreshold: 1
```

**2. PreStop Hook:**
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 10"]
```

**3. Rolling Update 속도 조절:**
```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 1  # 천천히
    maxSurge: 1
```

</details>

### 문제 3: SIGTERM을 받지 못하는 경우는?

<details>
<summary>정답 보기</summary>

**PID 1 문제:**
```dockerfile
# ❌ 잘못된 Dockerfile
FROM python:3.9
COPY app.py .
CMD python app.py

# 실제 프로세스:
# PID 1: /bin/sh -c "python app.py"
# PID 7: python app.py
# → SIGTERM이 PID 1로 가지만 처리 안 함!
```

**해결 1: exec 사용:**
```dockerfile
# ✅ exec로 PID 1 교체
CMD exec python app.py

# 또는
CMD ["python", "app.py"]  # JSON 형식 (권장)
```

**해결 2: tini 사용:**
```dockerfile
# tini: PID 1 init 시스템
RUN apt-get update && apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["python", "app.py"]
```

**검증:**
```bash
# 컨테이너 내부에서
ps aux
# PID 1이 애플리케이션인지 확인
```

</details>

---

## 📌 핵심 요약

```
Graceful Shutdown 핵심:
1. SIGTERM 처리
2. 새 요청 거부
3. 진행 중인 요청 완료
4. 리소스 정리
5. 안전하게 종료

Best Practices:
✅ 시그널 핸들러 구현
✅ PreStop Hook 사용
✅ Readiness Probe 필수
✅ 적절한 Grace Period
✅ Zero-Downtime 배포
```

---

## 📚 참고 자료

- [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Graceful Shutdown Patterns](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace)

---

## 🤔 생각해볼 문제

1. WebSocket 연결이 있을 때 Graceful Shutdown은?
2. 배치 작업이 진행 중일 때 종료하면?
3. Stateful Application의 Graceful Shutdown 전략은?

> 💡 **답변**:
> 
> **1) WebSocket Graceful Shutdown:**
> ```python
> connections = set()
> 
> @app.websocket('/ws')
> async def websocket_endpoint(websocket):
>     connections.add(websocket)
>     try:
>         # ...
>     finally:
>         connections.remove(websocket)
> 
> def shutdown():
>     # 모든 WebSocket에 종료 메시지
>     for ws in connections:
>         await ws.send_json({'type': 'shutdown'})
>     
>     # 연결 종료 대기
>     await asyncio.sleep(5)
> ```
> 
> **2) 배치 작업 중 종료:**
> ```python
> job_running = False
> 
> def long_running_job():
>     global job_running
>     job_running = True
>     try:
>         for i in range(1000):
>             if shutdown_initiated:
>                 save_checkpoint(i)
>                 break
>             process_item(i)
>     finally:
>         job_running = False
> 
> # Grace Period 길게
> terminationGracePeriodSeconds: 120
> ```
> 
> **3) Stateful App:**
> ```
> StatefulSet 사용:
> - Pod 순차 종료
> - Data Replication
> - Checkpoint 저장
> ```

---

<div align="center">

**[⬅️ 이전: Health Checks](./06-Health-Checks.md)** | **[다음: Configuration Management ➡️](./08-Configuration-Management.md)**

</div>
