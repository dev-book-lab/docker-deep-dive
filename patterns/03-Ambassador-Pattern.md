# 03. Ambassador Pattern - 프록시 패턴 구현

## 🎯 이 챕터에서 배울 것

- **Ambassador 패턴 개념**: 외부 서비스 통신 프록시
- **로컬 프록시**: 복잡한 네트워크 로직 캡슐화
- **재시도 및 Circuit Breaker**: 안정성 향상
- **캐싱**: 성능 최적화
- **모니터링**: 통신 메트릭 수집
- **실전 구현**: Redis, Database, External API 프록시

## 📌 왜 중요한가?

**"Ambassador 패턴은 외부 서비스와의 통신을 단순화하고 안정성을 높입니다."**

```
Ambassador 패턴의 핵심:

Without Ambassador (직접 통신):
┌─────────────────────────────────────────────────┐
│ Application                                     │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │ Business Logic                           │   │
│  ├──────────────────────────────────────────┤   │
│  │ + Retry Logic (3번 재시도)                 │   │
│  │ + Circuit Breaker (장애 감지)              │   │
│  │ + Connection Pooling                     │   │
│  │ + TLS/SSL 설정                            │   │
│  │ + Timeout 관리                            │   │
│  │ + Metrics 수집                            │   │
│  │ + Logging                                │   │
│  └──────────────────────────────────────────┘   │
└─────────────────┬───────────────────────────────┘
                  │
                  │ 복잡한 네트워크 로직
                  │ 모든 서비스에 중복 구현
                  ▼
          ┌──────────────┐
          │ External     │
          │ Service      │
          │ (Database,   │
          │  API, Redis) │
          └──────────────┘

문제점:
❌ 코드 중복 (모든 앱에 동일한 로직)
❌ 복잡도 증가 (비즈니스 로직 + 네트워크 로직)
❌ 일관성 부족 (팀마다 다른 구현)
❌ 테스트 어려움
❌ 기술 스택 제약

With Ambassador Pattern:
┌─────────────────────────────────────────────────┐
│ Pod / Service                                   │
│                                                 │
│  ┌──────────────┐      ┌──────────────┐         │
│  │ Application  │      │ Ambassador   │         │
│  │              │      │ Proxy        │         │
│  │ - Business   │◄────►│              │         │
│  │   Logic      │      │ - Retry      │         │
│  │ - Simple     │      │ - Circuit    │         │
│  │   API Call   │      │ - Pool       │         │
│  │              │      │ - TLS        │         │
│  │              │      │ - Timeout    │         │
│  │              │      │ - Metrics    │         │
│  └──────────────┘      └──────┬───────┘         │
│         │                     │                 │
│         │ localhost:6379      │                 │
└─────────┴─────────────────────┼─────────────────┘
                                │
                                │ 복잡한 로직 캡슐화
                                ▼
                        ┌──────────────┐
                        │ External     │
                        │ Service      │
                        │ (Redis,      │
                        │  Database)   │
                        └──────────────┘

장점:
✅ 관심사 분리 (비즈니스 vs 네트워크)
✅ 재사용 가능 (모든 서비스 동일 프록시)
✅ 언어 독립적 (Python 앱 + Go 프록시)
✅ 표준화 (중앙 관리)
✅ 테스트 용이 (모킹 가능)
✅ 독립 배포 (프록시만 업데이트)

Ambassador vs Sidecar:
┌──────────────┬──────────────┬──────────────┐
│ 기준          │ Sidecar      │ Ambassador   │
├──────────────┼──────────────┼──────────────┤
│ 대상          │ 앱 자체        │ 외부 서비스     │
├──────────────┼──────────────┼──────────────┤
│ 방향          │ Inbound/Out  │ Outbound     │
├──────────────┼──────────────┼──────────────┤
│ 예시          │ 로깅, 모니터링   │ DB, API 프록시│
└──────────────┴──────────────┴──────────────┘

실제로 Ambassador는 Sidecar의 특수한 형태
- Sidecar: 범용 보조 컨테이너
- Ambassador: 외부 통신 전용 Sidecar
```

**실무 영향:**
- **안정성**: Retry, Circuit Breaker로 장애 대응
- **성능**: Connection Pool, 캐싱
- **보안**: TLS Termination, 인증 추가
- **관찰성**: 통신 메트릭 수집

---

## 🔬 Deep Dive

### 1. Ambassador 패턴 기본

#### 프록시 역할

```
Ambassador가 처리하는 것들:

1. 연결 관리:
┌──────────────┐                ┌──────────────┐
│ App          │                │ Database     │
└──────┬───────┘                └───────▲──────┘
       │                                │
       │ Simple:                        │ Complex:
       │ localhost:5432                 │ Connection Pool
       │                                │ Keep-alive
       ▼                                │ Reconnect
┌───────────────────────────────────────┴─────┐
│ Ambassador                                  │
│ - Connection Pool (10개 유지)                 │
│ - Health Check (주기적 확인)                   │
│ - Automatic Reconnect                       │
└─────────────────────────────────────────────┘

2. 안정성:
┌──────────────┐
│ App          │
└──────┬───────┘
       │ 1회 호출
       ▼
┌─────────────────────────────────────────────┐
│ Ambassador                                  │
│                                             │
│  Try 1: ──────────────→ ✅                  │
│  Try 2: (실패 시 재시도)                        │
│  Try 3: (최대 3회)                            │
│                                             │
│  Circuit Breaker:                           │
│  - 연속 5회 실패 → OPEN (즉시 에러)              │
│  - 30초 후 → HALF_OPEN (1개 시도)              │
│  - 성공 → CLOSED (정상)                       │
└─────────────────────────────────────────────┘

3. 성능:
┌──────────────┐
│ App          │
└──────┬───────┘
       │ GET /api/data
       ▼
┌─────────────────────────────────────────────┐
│ Ambassador                                  │
│                                             │
│  Cache Check:                               │
│  ┌────────────────┐                         │
│  │ Local Cache    │                         │
│  │ /api/data → {} │ ← Hit (즉시 반환)         │
│  └────────────────┘                         │
│                                             │
│  Miss → External Request → Cache Update     │
└─────────────────────────────────────────────┘

4. 보안:
┌──────────────┐                ┌──────────────┐
│ App          │                │ External API │
└──────┬───────┘                └───────▲──────┘
       │ HTTP (plain)                   │ HTTPS (TLS)
       ▼                                │
┌───────────────────────────────────────┴────┐
│ Ambassador                                 │
│ - TLS Termination                          │
│ - API Key Injection                        │
│ - Rate Limiting                            │
└────────────────────────────────────────────┘

5. 관찰성:
┌──────────────┐
│ App          │
└──────┬───────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│ Ambassador                                  │
│                                             │
│  Metrics:                                   │
│  - request_total                            │
│  - request_duration_seconds                 │
│  - error_count                              │
│  - circuit_breaker_state                    │
│                                             │
│  Logs:                                      │
│  - [INFO] Request: GET /api/data            │
│  - [WARN] Retry attempt 2/3                 │
│  - [ERROR] Circuit breaker OPEN             │
└─────────────────────────────────────────────┘
```

---

### 2. 주요 패턴

#### Retry Pattern

```python
# Ambassador에서 Retry 구현
import time
import requests
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type
)

class Ambassador:
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        retry=retry_if_exception_type(requests.exceptions.RequestException)
    )
    def call_external_api(self, url):
        """재시도 로직이 포함된 API 호출"""
        print(f"Calling {url}...")
        response = requests.get(url, timeout=5)
        response.raise_for_status()
        return response.json()

# 앱 코드 (단순)
ambassador = Ambassador()
data = ambassador.call_external_api("http://external-api/data")
# 실패 시 자동으로 1초, 2초, 4초 대기 후 재시도
```

#### Circuit Breaker Pattern

```python
from pybreaker import CircuitBreaker

# 설정
db_breaker = CircuitBreaker(
    fail_max=5,           # 5회 실패 시 OPEN
    timeout_duration=30,  # 30초 후 HALF_OPEN
    reset_timeout=60      # 성공 시 CLOSED로 복구
)

@db_breaker
def query_database(query):
    # DB 쿼리
    return execute_query(query)

# 상태 확인
if db_breaker.current_state == "open":
    print("Circuit breaker OPEN - 즉시 에러 반환")
elif db_breaker.current_state == "half-open":
    print("Circuit breaker HALF_OPEN - 테스트 중")
else:
    print("Circuit breaker CLOSED - 정상")
```

---

## 🔧 실습 1: Redis Ambassador

### Step 1: Redis Ambassador 구현

```python
# redis-ambassador/ambassador.py
import redis
import time
import json
from flask import Flask, request, jsonify

app = Flask(__name__)

# Redis 연결 풀
redis_pool = redis.ConnectionPool(
    host='redis-server',
    port=6379,
    max_connections=10,
    decode_responses=True
)

# 메트릭
metrics = {
    'total_requests': 0,
    'cache_hits': 0,
    'cache_misses': 0,
    'errors': 0
}

def get_redis_client():
    return redis.Redis(connection_pool=redis_pool)

@app.route('/health')
def health():
    """헬스 체크"""
    try:
        client = get_redis_client()
        client.ping()
        return jsonify({'status': 'healthy'})
    except:
        return jsonify({'status': 'unhealthy'}), 503

@app.route('/get/<key>')
def get_value(key):
    """캐시에서 값 조회"""
    metrics['total_requests'] += 1
    
    try:
        client = get_redis_client()
        value = client.get(key)
        
        if value:
            metrics['cache_hits'] += 1
            return jsonify({
                'key': key,
                'value': value,
                'cache': 'hit'
            })
        else:
            metrics['cache_misses'] += 1
            return jsonify({
                'key': key,
                'value': None,
                'cache': 'miss'
            }), 404
    
    except Exception as e:
        metrics['errors'] += 1
        return jsonify({'error': str(e)}), 500

@app.route('/set/<key>', methods=['POST'])
def set_value(key):
    """캐시에 값 저장"""
    data = request.get_json()
    value = data.get('value')
    ttl = data.get('ttl', 3600)  # 기본 1시간
    
    try:
        client = get_redis_client()
        client.setex(key, ttl, value)
        
        return jsonify({
            'key': key,
            'value': value,
            'ttl': ttl
        })
    
    except Exception as e:
        metrics['errors'] += 1
        return jsonify({'error': str(e)}), 500

@app.route('/delete/<key>', methods=['DELETE'])
def delete_value(key):
    """캐시에서 값 삭제"""
    try:
        client = get_redis_client()
        deleted = client.delete(key)
        
        return jsonify({
            'key': key,
            'deleted': bool(deleted)
        })
    
    except Exception as e:
        metrics['errors'] += 1
        return jsonify({'error': str(e)}), 500

@app.route('/metrics')
def get_metrics():
    """메트릭 조회"""
    hit_rate = (
        metrics['cache_hits'] / metrics['total_requests'] * 100
        if metrics['total_requests'] > 0 else 0
    )
    
    return jsonify({
        **metrics,
        'cache_hit_rate': f"{hit_rate:.2f}%"
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=6380)
```

```dockerfile
# redis-ambassador/Dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN pip install flask redis

COPY ambassador.py .

CMD ["python", "ambassador.py"]
```

### Step 2: 애플리케이션 (Ambassador 사용)

```python
# app/main.py
from flask import Flask, jsonify
import requests

app = Flask(__name__)

# Ambassador를 통해 Redis 접근
REDIS_AMBASSADOR = "http://localhost:6380"

@app.route('/api/users/<user_id>')
def get_user(user_id):
    """사용자 조회 (캐싱)"""
    
    # 1. 캐시 확인
    try:
        response = requests.get(f'{REDIS_AMBASSADOR}/get/user:{user_id}')
        if response.status_code == 200:
            data = response.json()
            return jsonify({
                'user_id': user_id,
                'data': data['value'],
                'source': 'cache'
            })
    except:
        pass
    
    # 2. DB에서 조회 (시뮬레이션)
    user_data = {
        'id': user_id,
        'name': f'User {user_id}',
        'email': f'user{user_id}@example.com'
    }
    
    # 3. 캐시에 저장
    try:
        requests.post(
            f'{REDIS_AMBASSADOR}/set/user:{user_id}',
            json={'value': str(user_data), 'ttl': 300}
        )
    except:
        pass
    
    return jsonify({
        'user_id': user_id,
        'data': user_data,
        'source': 'database'
    })

@app.route('/metrics')
def metrics():
    """캐시 메트릭"""
    try:
        response = requests.get(f'{REDIS_AMBASSADOR}/metrics')
        return response.json()
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Step 3: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Redis 서버
  redis-server:
    image: redis:7-alpine
    networks:
      - backend

  # Redis Ambassador
  redis-ambassador:
    build: ./redis-ambassador
    ports:
      - "6380:6380"
    depends_on:
      - redis-server
    networks:
      - backend
      - app

  # 애플리케이션
  app:
    build: ./app
    ports:
      - "8080:8080"
    network_mode: "service:redis-ambassador"  # 네트워크 공유
    depends_on:
      - redis-ambassador

networks:
  backend:
    driver: bridge
  app:
    driver: bridge
```

### Step 4: 테스트

```bash
# 1. 서비스 시작
docker-compose up -d

# 2. 사용자 조회 (캐시 미스)
curl http://localhost:8080/api/users/1
# {"user_id":"1","data":{...},"source":"database"}

# 3. 동일 사용자 조회 (캐시 히트)
curl http://localhost:8080/api/users/1
# {"user_id":"1","data":{...},"source":"cache"}

# 4. 메트릭 확인
curl http://localhost:8080/metrics
# {
#   "total_requests": 2,
#   "cache_hits": 1,
#   "cache_misses": 1,
#   "cache_hit_rate": "50.00%"
# }

# 5. 부하 테스트
for i in {1..100}; do
  curl -s http://localhost:8080/api/users/$((RANDOM % 10)) > /dev/null
done

curl http://localhost:8080/metrics

# 6. 정리
docker-compose down -v
```

---

## 🔧 실습 2: Database Ambassador (with Circuit Breaker)

### Step 1: Database Ambassador 구현

```python
# db-ambassador/ambassador.py
from flask import Flask, request, jsonify
import psycopg2
from psycopg2 import pool
from pybreaker import CircuitBreaker
import time

app = Flask(__name__)

# Connection Pool
db_pool = psycopg2.pool.SimpleConnectionPool(
    1, 10,  # min=1, max=10
    host='postgres',
    database='app_db',
    user='postgres',
    password='password'
)

# Circuit Breaker
db_breaker = CircuitBreaker(
    fail_max=3,
    timeout_duration=10,
    reset_timeout=30
)

# 메트릭
metrics = {
    'total_queries': 0,
    'successful_queries': 0,
    'failed_queries': 0,
    'circuit_breaker_state': 'closed'
}

@db_breaker
def execute_query(query, params=None):
    """Circuit Breaker가 적용된 쿼리 실행"""
    conn = db_pool.getconn()
    try:
        cur = conn.cursor()
        cur.execute(query, params)
        
        if query.strip().upper().startswith('SELECT'):
            result = cur.fetchall()
            columns = [desc[0] for desc in cur.description]
            return [dict(zip(columns, row)) for row in result]
        else:
            conn.commit()
            return {'affected_rows': cur.rowcount}
    finally:
        cur.close()
        db_pool.putconn(conn)

@app.route('/health')
def health():
    """헬스 체크"""
    try:
        conn = db_pool.getconn()
        cur = conn.cursor()
        cur.execute('SELECT 1')
        cur.close()
        db_pool.putconn(conn)
        
        return jsonify({
            'status': 'healthy',
            'circuit_breaker': db_breaker.current_state
        })
    except:
        return jsonify({
            'status': 'unhealthy',
            'circuit_breaker': db_breaker.current_state
        }), 503

@app.route('/query', methods=['POST'])
def query():
    """SQL 쿼리 실행"""
    data = request.get_json()
    sql = data.get('query')
    params = data.get('params')
    
    metrics['total_queries'] += 1
    metrics['circuit_breaker_state'] = db_breaker.current_state
    
    try:
        result = execute_query(sql, params)
        metrics['successful_queries'] += 1
        
        return jsonify({
            'success': True,
            'data': result,
            'circuit_breaker': db_breaker.current_state
        })
    
    except Exception as e:
        metrics['failed_queries'] += 1
        
        return jsonify({
            'success': False,
            'error': str(e),
            'circuit_breaker': db_breaker.current_state
        }), 500

@app.route('/metrics')
def get_metrics():
    """메트릭 조회"""
    success_rate = (
        metrics['successful_queries'] / metrics['total_queries'] * 100
        if metrics['total_queries'] > 0 else 0
    )
    
    return jsonify({
        **metrics,
        'success_rate': f"{success_rate:.2f}%"
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5433)
```

```txt
# db-ambassador/requirements.txt
flask==2.3.0
psycopg2-binary==2.9.6
pybreaker==1.0.1
```

### Step 2: 애플리케이션

```python
# app/main.py
from flask import Flask, jsonify, request
import requests

app = Flask(__name__)

DB_AMBASSADOR = "http://localhost:5433"

@app.route('/api/products')
def get_products():
    """상품 목록 조회"""
    try:
        response = requests.post(
            f'{DB_AMBASSADOR}/query',
            json={'query': 'SELECT * FROM products LIMIT 10'},
            timeout=5
        )
        data = response.json()
        
        if data['success']:
            return jsonify({
                'products': data['data'],
                'circuit_breaker': data['circuit_breaker']
            })
        else:
            return jsonify(data), 500
    
    except Exception as e:
        return jsonify({'error': str(e)}), 503

@app.route('/api/products', methods=['POST'])
def create_product():
    """상품 생성"""
    product_data = request.get_json()
    
    try:
        response = requests.post(
            f'{DB_AMBASSADOR}/query',
            json={
                'query': 'INSERT INTO products (name, price) VALUES (%s, %s)',
                'params': [product_data['name'], product_data['price']]
            },
            timeout=5
        )
        data = response.json()
        
        return jsonify(data)
    
    except Exception as e:
        return jsonify({'error': str(e)}), 503

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Step 3: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend

  # Database Ambassador
  db-ambassador:
    build: ./db-ambassador
    ports:
      - "5433:5433"
    depends_on:
      - postgres
    networks:
      - backend
      - app

  # 애플리케이션
  app:
    build: ./app
    ports:
      - "8080:8080"
    network_mode: "service:db-ambassador"
    depends_on:
      - db-ambassador

networks:
  backend:
  app:
```

```sql
-- init.sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products (name, price) VALUES
('Laptop', 999.99),
('Mouse', 29.99),
('Keyboard', 79.99);
```

---

## 🔧 실습 3: External API Ambassador (with Retry)

### Step 1: API Ambassador 구현

```python
# api-ambassador/ambassador.py
from flask import Flask, request, jsonify
import requests
from tenacity import retry, stop_after_attempt, wait_exponential
import time

app = Flask(__name__)

# 메트릭
metrics = {
    'total_requests': 0,
    'successful_requests': 0,
    'failed_requests': 0,
    'retry_count': 0
}

class APIAmbassador:
    def __init__(self, base_url):
        self.base_url = base_url
        self.session = requests.Session()
        # Connection Pool
        adapter = requests.adapters.HTTPAdapter(
            pool_connections=10,
            pool_maxsize=10
        )
        self.session.mount('http://', adapter)
        self.session.mount('https://', adapter)
    
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=1, max=10),
        reraise=True
    )
    def request(self, method, path, **kwargs):
        """재시도 로직이 포함된 HTTP 요청"""
        url = f"{self.base_url}{path}"
        
        # Timeout 기본값
        kwargs.setdefault('timeout', 10)
        
        print(f"[Ambassador] {method} {url}")
        response = self.session.request(method, url, **kwargs)
        response.raise_for_status()
        
        return response

ambassador = APIAmbassador('https://jsonplaceholder.typicode.com')

@app.route('/api/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def proxy_request(path):
    """API 프록시"""
    metrics['total_requests'] += 1
    
    try:
        # 요청 프록시
        response = ambassador.request(
            request.method,
            f'/{path}',
            json=request.get_json() if request.is_json else None,
            params=request.args
        )
        
        metrics['successful_requests'] += 1
        
        return jsonify({
            'success': True,
            'data': response.json(),
            'status_code': response.status_code
        })
    
    except requests.exceptions.RetryError as e:
        metrics['failed_requests'] += 1
        metrics['retry_count'] += 1
        
        return jsonify({
            'success': False,
            'error': 'Max retries exceeded',
            'details': str(e)
        }), 503
    
    except Exception as e:
        metrics['failed_requests'] += 1
        
        return jsonify({
            'success': False,
            'error': str(e)
        }), 500

@app.route('/metrics')
def get_metrics():
    """메트릭 조회"""
    success_rate = (
        metrics['successful_requests'] / metrics['total_requests'] * 100
        if metrics['total_requests'] > 0 else 0
    )
    
    return jsonify({
        **metrics,
        'success_rate': f"{success_rate:.2f}%"
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8081)
```

---

## 🔧 실습 4: Kubernetes Ambassador

### Step 1: Kubernetes Deployment

```yaml
# deployment-with-ambassador.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-ambassador-config
data:
  ambassador.py: |
    # (실습 1의 ambassador.py 내용)

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      # 메인 애플리케이션
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: REDIS_HOST
          value: "localhost"
        - name: REDIS_PORT
          value: "6380"

      # Redis Ambassador
      - name: redis-ambassador
        image: python:3.9-slim
        command: ["python", "/app/ambassador.py"]
        ports:
        - containerPort: 6380
        volumeMounts:
        - name: config
          mountPath: /app
        env:
        - name: REDIS_SERVER
          value: "redis-service"

      volumes:
      - name: config
        configMap:
          name: redis-ambassador-config

---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 패턴                  │ 설명                        │
├──────────────────────┼────────────────────────────┤
│ Retry                │ 일시적 장애 재시도             │
├──────────────────────┼────────────────────────────┤
│ Circuit Breaker      │ 장애 격리 및 빠른 실패          │
├──────────────────────┼────────────────────────────┤
│ Connection Pool      │ 연결 재사용 (성능)             │
├──────────────────────┼────────────────────────────┤
│ Caching              │ 응답 캐싱 (성능)              │
├──────────────────────┼────────────────────────────┤
│ TLS Termination      │ 암호화 처리 위임               │
├──────────────────────┼────────────────────────────┤
│ Monitoring           │ 통신 메트릭 수집               │
└──────────────────────┴────────────────────────────┘
```

---

## 🎓 연습 문제

### 문제 1: Circuit Breaker의 OPEN 상태에서 어떻게 복구하는가?

<details>
<summary>정답 보기</summary>

**Circuit Breaker 상태 전이:**

```
CLOSED (정상)
  │
  │ 연속 5회 실패
  ▼
OPEN (차단)
  │ 모든 요청 즉시 실패
  │ 30초 대기
  ▼
HALF_OPEN (테스트)
  │
  ├─ 성공 → CLOSED (복구)
  └─ 실패 → OPEN (재차단)
```

**구현:**
```python
from pybreaker import CircuitBreaker

breaker = CircuitBreaker(
    fail_max=5,           # 5회 실패 → OPEN
    timeout_duration=30,  # 30초 후 HALF_OPEN
    reset_timeout=60      # HALF_OPEN에서 성공 유지 시간
)

@breaker
def call_external_service():
    response = requests.get('http://external-api', timeout=5)
    return response.json()

# 복구 과정:
# 1. OPEN: 30초 대기
# 2. HALF_OPEN: 1개 요청 허용
# 3. 성공 → CLOSED
# 4. 실패 → OPEN (30초 재대기)
```

**모니터링:**
```python
if breaker.current_state == 'open':
    # 대체 데이터 반환
    return fallback_data
```

</details>

### 문제 2: Ambassador에서 캐싱 TTL을 어떻게 결정하는가?

<details>
<summary>정답 보기</summary>

**TTL 결정 기준:**

```python
# 1. 데이터 변경 빈도
@app.route('/cache/<key>')
def cache_value(key):
    if key.startswith('user:'):
        ttl = 3600  # 사용자 정보: 1시간
    elif key.startswith('product:'):
        ttl = 300   # 상품 정보: 5분 (재고 변동)
    elif key.startswith('config:'):
        ttl = 86400 # 설정: 24시간
    else:
        ttl = 600   # 기본: 10분

# 2. 데이터 중요도
if is_critical_data(key):
    ttl = 60  # 짧게 (최신성 중요)
else:
    ttl = 3600  # 길게 (성능 우선)

# 3. Cache-Control 헤더 존중
response = requests.get(url)
cache_control = response.headers.get('Cache-Control')
if 'max-age' in cache_control:
    ttl = int(cache_control.split('max-age=')[1])
```

**동적 TTL:**
```python
# 사용 빈도 기반
cache_stats = get_cache_stats(key)
if cache_stats['hit_rate'] > 0.8:
    ttl *= 2  # 자주 사용되면 TTL 증가
```

</details>

### 문제 3: Ambassador와 Service Mesh(Istio)의 차이는?

<details>
<summary>정답 보기</summary>

**Ambassador (Pod-level):**
```yaml
spec:
  containers:
  - name: app
  - name: ambassador  # 직접 구현
    image: custom-ambassador
```

**Service Mesh (Infrastructure-level):**
```yaml
metadata:
  labels:
    sidecar.istio.io/inject: "true"  # 자동 주입
# Istio가 Envoy 프록시 자동 추가
```

**비교:**
```
┌──────────────────┬──────────────┬──────────────┐
│ 기준              │ Ambassador   │ Service Mesh │
├──────────────────┼──────────────┼──────────────┤
│ 범위              │ 단일 외부 서비스 │ 모든 통신      │
├──────────────────┼──────────────┼──────────────┤
│ 복잡도             │ 낮음          │ 높음          │
├──────────────────┼──────────────┼──────────────┤
│ 설정              │ 수동          │ 자동          │
├──────────────────┼──────────────┼──────────────┤
│ 커스터마이징        │ 완전 제어       │ 제한적        │
├──────────────────┼──────────────┼──────────────┤
│ 학습 곡선          │ 낮음          │ 높음          │
└──────────────────┴──────────────┴──────────────┘

선택 기준:
Ambassador:
- 특정 외부 서비스만 프록시
- 간단한 로직 (Retry, Cache)
- 빠른 구현

Service Mesh:
- 모든 서비스 간 통신
- mTLS, Routing, Canary 등
- 엔터프라이즈 환경
```

</details>

---

## 📌 핵심 요약

```
Ambassador 패턴 핵심:
1. 외부 서비스 통신 프록시
2. 복잡한 네트워크 로직 캡슐화
3. Retry, Circuit Breaker, Cache
4. 언어 독립적 구현
5. 독립적 배포 및 업데이트

Best Practices:
✅ Connection Pool 사용
✅ Circuit Breaker 구현
✅ Retry with Exponential Backoff
✅ 메트릭 수집
✅ Health Check
✅ Timeout 설정
```

---

## 📚 참고 자료

- [Ambassador Pattern - Microsoft](https://docs.microsoft.com/en-us/azure/architecture/patterns/ambassador)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Connection Pooling Best Practices](https://stackoverflow.com/questions/tagged/connection-pooling)

---

## 🤔 생각해볼 문제

1. Ambassador와 Reverse Proxy(Nginx, HAProxy)의 차이는 무엇인가?
2. Connection Pool의 크기를 어떻게 결정해야 하는가?
3. Ambassador가 Single Point of Failure가 될 수 있는가? 어떻게 해결하는가?

> 💡 **답변**:
> 
> **1) Ambassador vs Reverse Proxy:**
> 
> **Reverse Proxy (외부 → 내부):**
> ```
> Internet → Nginx → Backend Servers
> - Load Balancing
> - SSL Termination
> - Request Routing
> ```
> 
> **Ambassador (내부 → 외부):**
> ```
> App → Ambassador → External Service
> - Outbound Traffic
> - Retry, Circuit Breaker
> - Connection Pool
> ```
> 
> **2) Connection Pool 크기:**
> ```
> Pool Size = (예상 동시 요청) × (평균 응답 시간 / 요청 간격)
> 
> 예:
> - 초당 100 요청
> - 평균 응답: 100ms
> - Pool = 100 × 0.1 = 10
> 
> 모니터링:
> - Pool 사용률 > 80% → 증가
> - Pool 사용률 < 20% → 감소
> ```
> 
> **3) SPOF 해결:**
> ```
> ❌ SPOF:
> App → Ambassador (1개) → DB
>       (장애 시 전체 중단)
> 
> ✅ 해결:
> 1. Ambassador를 Sidecar로 (Pod당 1개)
> 2. Health Check
> 3. Graceful Shutdown
> ```

---

<div align="center">

**[⬅️ 이전: Sidecar Pattern](./02-Sidecar-Pattern.md)** | **[다음: Adapter Pattern ➡️](./04-Adapter-Pattern.md)**

</div>
