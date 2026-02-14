# 02. Sidecar Pattern - 사이드카 컨테이너 활용

## 🎯 이 챕터에서 배울 것

- **Sidecar 패턴 개념**: 보조 컨테이너의 역할
- **로깅 사이드카**: 중앙 집중식 로그 수집
- **프록시 사이드카**: 서비스 메시 기초
- **모니터링 사이드카**: 메트릭 수집 및 전송
- **설정 동기화**: 동적 설정 관리
- **실전 구현**: Kubernetes와 Docker Compose

## 📌 왜 중요한가?

**"Sidecar 패턴은 메인 애플리케이션의 코드 변경 없이 부가 기능을 추가합니다."**

```
Sidecar 패턴의 핵심:

Without Sidecar (전통적 방식):
┌─────────────────────────────────────────────────┐
│ Application Container                           │
│                                                 │
│  ┌──────────────┐                               │
│  │ Business     │                               │
│  │ Logic        │                               │
│  ├──────────────┤                               │
│  │ Logging      │ ← 애플리케이션에 포함              │
│  │ Monitoring   │ ← 모든 서비스에 중복 구현          │
│  │ Proxy        │ ← 코드 변경 필요                 │
│  │ Config Sync  │ ← 유지보수 어려움                 │
│  └──────────────┘                               │
└─────────────────────────────────────────────────┘

문제점:
❌ 코드 중복 (모든 서비스에 동일 기능)
❌ 기술 스택 제약 (Java 앱에 Python 로깅 불가)
❌ 배포 결합 (부가 기능 변경 시 앱 재배포)
❌ 팀 간 충돌 (인프라 팀 vs 개발 팀)

With Sidecar Pattern:
┌─────────────────────────────────────────────────┐
│ Pod (Kubernetes) / Service (Docker Compose)     │
│                                                 │
│  ┌──────────────┐      ┌──────────────┐         │
│  │ Main         │      │ Sidecar      │         │
│  │ Application  │◄────►│ Container    │         │
│  │              │      │              │         │
│  │ - Business   │      │ - Logging    │         │
│  │   Logic      │      │ - Monitoring │         │
│  │ - REST API   │      │ - Proxy      │         │
│  │              │      │ - Config     │         │
│  └──────────────┘      └──────────────┘         │
│         │                      │                │
│         │ Shared:              │                │
│         │ - Localhost          │                │
│         │ - Volumes            │                │
│         │ - Network            │                │
└─────────────────────────────────────────────────┘

장점:
✅ 관심사 분리 (앱 로직 vs 인프라)
✅ 재사용 가능 (모든 서비스에 동일 사이드카)
✅ 기술 독립성 (Python 앱 + Go 사이드카)
✅ 독립 배포 (사이드카만 업데이트)
✅ 표준화 (인프라 팀이 중앙 관리)

사이드카 vs 라이브러리:
┌──────────────┬──────────────┬──────────────┐
│ 기준          │ 라이브러리      │ 사이드카       │
├──────────────┼──────────────┼──────────────┤
│ 언어 의존성     │ ✅ 있음       │ ❌ 없음       │
├──────────────┼──────────────┼──────────────┤
│ 배포          │ 앱과 함께      │ 독립           │
├──────────────┼──────────────┼──────────────┤
│ 리소스         │ 공유          │ 격리          │
├──────────────┼──────────────┼──────────────┤
│ 버전 관리      │ 복잡          │ 단순          │
└──────────────┴──────────────┴──────────────┘
```

**실무 영향:**
- **서비스 메시**: Istio, Linkerd의 핵심 (Envoy 사이드카)
- **로깅 표준화**: 모든 서비스 동일한 로깅 방식
- **보안**: TLS 암호화를 사이드카가 처리
- **운영 간소화**: 인프라 로직을 앱에서 분리

---

## 🔬 Deep Dive

### 1. Sidecar 패턴 기본 원리

#### 공유 리소스

```
Sidecar와 Main Container의 공유:

1. Network Namespace (localhost 공유):
┌─────────────────────────────────────────────┐
│ Pod                                         │
│                                             │
│  ┌──────────────┐    ┌──────────────┐       │
│  │ App          │    │ Sidecar      │       │
│  │ :8080        │←─→ │ :8081        │       │
│  └──────────────┘    └──────────────┘       │
│         │                    │              │
│         └────────┬───────────┘              │
│              localhost                      │
└─────────────────────────────────────────────┘

# App → Sidecar
curl http://localhost:8081/metrics

# Sidecar → App
curl http://localhost:8080/health

2. Volume 공유 (파일 시스템):
┌─────────────────────────────────────────────┐
│ Pod                                         │
│                                             │
│  ┌──────────────┐    ┌──────────────┐       │
│  │ App          │    │ Log Sidecar  │       │
│  │              │    │              │       │
│  │ write logs   │    │ read logs    │       │
│  │     ↓        │    │     ↓        │       │
│  └─────┼────────┘    └─────┼────────┘       │
│        │                   │                │
│        └───────┬───────────┘                │
│                ▼                            │
│        ┌──────────────┐                     │
│        │ Shared Vol   │                     │
│        │ /var/log/app │                     │
│        └──────────────┘                     │
└─────────────────────────────────────────────┘

3. Process Namespace (선택적):
- 같은 PID namespace 공유
- 서로의 프로세스 볼 수 있음
- 디버깅/모니터링 용도

4. IPC Namespace:
- 공유 메모리
- Semaphore
- Message Queue
```

#### 생명주기 관리

```
Sidecar 생명주기:

시작 순서:
1. Init Container (선택)
   - 사전 준비 작업
   - 완료 후 종료
   
2. Main + Sidecar (동시)
   - 같이 시작
   - 순서 보장 없음
   
3. Readiness Probe
   - 둘 다 Ready 될 때까지 대기

종료 순서:
1. SIGTERM 전송
   - Main과 Sidecar 동시에
   
2. Grace Period (30초 기본)
   - 정리 작업 수행
   
3. SIGKILL (Grace Period 후)
   - 강제 종료

문제: Sidecar가 먼저 종료되면?
┌──────────────┐      ┌──────────────┐
│ App          │ ───→ │ Log Sidecar  │
│ (still       │      │ (terminated) │
│  writing)    │      │              │
└──────────────┘      └──────────────┘
로그 손실 가능!

해결: PreStop Hook
spec:
  containers:
  - name: log-sidecar
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]
```

---

### 2. 로깅 사이드카

#### 로그 수집 패턴

```
중앙 집중식 로깅:

Before (각 앱이 직접):
┌──────────┐      ┌──────────┐      ┌──────────┐
│ Service1 │─────→│ Service2 │─────→│ Service3 │
└────┬─────┘      └────┬─────┘      └────┬─────┘
     │                 │                 │
     │ (Log Library)   │ (Log Library)   │
     └─────────┬───────┴─────────┬───────┘
               ▼                 ▼
       ┌────────────────────────────┐
       │ Elasticsearch / Splunk     │
       └────────────────────────────┘

문제:
- 모든 서비스에 로깅 라이브러리 추가
- 언어마다 다른 라이브러리
- 설정 변경 시 모든 앱 재배포

After (Sidecar 패턴):
┌─────────────────────┐  ┌─────────────────────┐
│ Service1            │  │ Service2            │
│ ┌─────┐  ┌────────┐ │  │ ┌─────┐  ┌────────┐ │
│ │ App │  │FluentD │ │  │ │ App │  │FluentD │ │
│ │     │─→│Sidecar │─┼──┼─│     │─→│Sidecar │─┼─┐
│ └─────┘  └────────┘ │  │ └─────┘  └────────┘ │ │
└─────────────────────┘  └─────────────────────┘ │
                                                 │
                                                 ▼
                              ┌──────────────────────┐
                              │ Elasticsearch        │
                              └──────────────────────┘

장점:
✅ 앱 코드 변경 없음
✅ 언어 독립적
✅ 사이드카만 업데이트
✅ 표준화된 로그 포맷
```

---

### 3. 프록시 사이드카

#### Service Mesh 기초

```
Envoy Sidecar (Istio, Linkerd):

┌─────────────────────────────────────────────┐
│ Service A Pod                               │
│                                             │
│  ┌──────────────┐    ┌──────────────┐       │
│  │ App          │    │ Envoy        │       │
│  │ :8080        │◄──►│ Proxy        │       │
│  └──────────────┘    └──────┬───────┘       │
└───────────────────────────────┼─────────────┘
                                │
                                │ mTLS, Retry,
                                │ Circuit Breaker,
                                │ Load Balancing
                                ▼
┌─────────────────────────────────────────────┐
│ Service B Pod                               │
│                                             │
│  ┌──────────────┐    ┌──────────────┐       │
│  │ Envoy        │    │ App          │       │
│  │ Proxy        │◄──►│ :8080        │       │
│  └──────────────┘    └──────────────┘       │
└─────────────────────────────────────────────┘

Envoy 기능:
1. Traffic Management
   - Load Balancing (Round Robin, Least Request)
   - Retry (자동 재시도)
   - Circuit Breaker (장애 격리)
   - Timeout (요청 제한 시간)

2. Security
   - mTLS (자동 암호화)
   - Authentication (인증)
   - Authorization (인가)

3. Observability
   - Access Logs (모든 요청/응답)
   - Metrics (Prometheus)
   - Distributed Tracing (Jaeger, Zipkin)

앱 코드:
# Before (직접 구현)
response = requests.get(
    'https://service-b',
    cert=('client.crt', 'client.key'),
    verify='ca.crt',
    timeout=3,
    retries=3
)

# After (Envoy가 처리)
response = requests.get('http://service-b')
# mTLS, Retry, Timeout 모두 Envoy가 자동 처리!
```

---

## 🔧 실습 1: 로깅 사이드카 (Fluentd)

### Step 1: 애플리케이션 컨테이너

```python
# app/main.py
from flask import Flask
import logging
import json
import time

app = Flask(__name__)

# 파일 로깅 설정
logging.basicConfig(
    filename='/var/log/app/app.log',
    level=logging.INFO,
    format='%(message)s'
)

@app.route('/')
def index():
    log_entry = {
        'timestamp': time.time(),
        'level': 'INFO',
        'message': 'Index page accessed',
        'endpoint': '/'
    }
    logging.info(json.dumps(log_entry))
    return 'Hello, World!'

@app.route('/api/users')
def users():
    log_entry = {
        'timestamp': time.time(),
        'level': 'INFO',
        'message': 'Users API called',
        'endpoint': '/api/users'
    }
    logging.info(json.dumps(log_entry))
    return json.dumps([{'id': 1, 'name': 'Alice'}])

@app.route('/error')
def error():
    log_entry = {
        'timestamp': time.time(),
        'level': 'ERROR',
        'message': 'Error endpoint called',
        'endpoint': '/error'
    }
    logging.error(json.dumps(log_entry))
    return 'Error occurred', 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

```dockerfile
# app/Dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN pip install flask

COPY main.py .

# 로그 디렉토리 생성
RUN mkdir -p /var/log/app

CMD ["python", "main.py"]
```

### Step 2: Fluentd 사이드카

```ruby
# fluentd/fluent.conf
<source>
  @type tail
  path /var/log/app/app.log
  pos_file /var/log/app/app.log.pos
  tag app.logs
  <parse>
    @type json
  </parse>
</source>

# 파싱 및 필터링
<filter app.logs>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    service_name "my-app"
  </record>
</filter>

# Elasticsearch로 전송
<match app.logs>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix app-logs
  <buffer>
    flush_interval 5s
  </buffer>
</match>
```

```dockerfile
# fluentd/Dockerfile
FROM fluent/fluentd:v1.16-1

USER root

RUN gem install fluent-plugin-elasticsearch

USER fluent
```

### Step 3: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 메인 애플리케이션
  app:
    build: ./app
    ports:
      - "8080:8080"
    volumes:
      - logs:/var/log/app
    networks:
      - app-network

  # Fluentd 사이드카
  fluentd:
    build: ./fluentd
    volumes:
      - logs:/var/log/app:ro
      - ./fluentd/fluent.conf:/fluentd/etc/fluent.conf
    depends_on:
      - elasticsearch
    networks:
      - app-network

  # Elasticsearch (로그 저장소)
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    networks:
      - app-network

  # Kibana (로그 시각화)
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - app-network

volumes:
  logs:

networks:
  app-network:
    driver: bridge
```

### Step 4: 테스트

```bash
# 1. 서비스 시작
docker-compose up -d

# 2. 로그 생성
for i in {1..10}; do
  curl http://localhost:8080/
  curl http://localhost:8080/api/users
  curl http://localhost:8080/error
  sleep 1
done

# 3. Elasticsearch에 로그 확인
curl http://localhost:9200/app-logs-*/_search?pretty

# 4. Kibana에서 시각화
# http://localhost:5601
# Discover → Create index pattern: app-logs-*

# 5. 정리
docker-compose down -v
```

---

## 🔧 실습 2: 모니터링 사이드카 (Prometheus Exporter)

### Step 1: 애플리케이션 (메트릭 노출)

```python
# app/main.py
from flask import Flask, jsonify
import time
import random

app = Flask(__name__)

# In-memory 메트릭
metrics = {
    'request_count': 0,
    'error_count': 0,
    'request_duration_sum': 0.0,
    'request_duration_count': 0
}

@app.route('/')
def index():
    start = time.time()
    
    # 비즈니스 로직
    time.sleep(random.uniform(0.01, 0.1))
    
    # 메트릭 업데이트
    metrics['request_count'] += 1
    duration = time.time() - start
    metrics['request_duration_sum'] += duration
    metrics['request_duration_count'] += 1
    
    return 'Hello, World!'

@app.route('/error')
def error():
    metrics['request_count'] += 1
    metrics['error_count'] += 1
    return 'Error', 500

# 메트릭 엔드포인트 (사이드카가 스크래핑)
@app.route('/metrics')
def app_metrics():
    avg_duration = (
        metrics['request_duration_sum'] / metrics['request_duration_count']
        if metrics['request_duration_count'] > 0 else 0
    )
    
    return jsonify({
        'request_count': metrics['request_count'],
        'error_count': metrics['error_count'],
        'avg_duration': avg_duration
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Step 2: Prometheus Exporter 사이드카

```python
# exporter/exporter.py
from prometheus_client import start_http_server, Gauge, Counter
import requests
import time

# Prometheus 메트릭
request_total = Counter('app_requests_total', 'Total requests')
error_total = Counter('app_errors_total', 'Total errors')
avg_duration = Gauge('app_request_duration_avg', 'Average request duration')

def collect_metrics():
    """앱에서 메트릭 수집"""
    while True:
        try:
            # localhost로 앱 메트릭 조회
            response = requests.get('http://localhost:8080/metrics', timeout=5)
            data = response.json()
            
            # Prometheus 메트릭 업데이트
            request_total._value.set(data['request_count'])
            error_total._value.set(data['error_count'])
            avg_duration.set(data['avg_duration'])
            
        except Exception as e:
            print(f"Error collecting metrics: {e}")
        
        time.sleep(5)  # 5초마다 수집

if __name__ == '__main__':
    # Prometheus가 스크래핑할 포트
    start_http_server(9090)
    print("Exporter started on :9090")
    
    collect_metrics()
```

```dockerfile
# exporter/Dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN pip install prometheus-client requests

COPY exporter.py .

CMD ["python", "exporter.py"]
```

### Step 3: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 메인 애플리케이션
  app:
    build: ./app
    ports:
      - "8080:8080"
    networks:
      - monitoring

  # Prometheus Exporter 사이드카
  exporter:
    build: ./exporter
    ports:
      - "9090:9090"
    network_mode: "service:app"  # 네트워크 공유
    depends_on:
      - app

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9091:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    networks:
      - monitoring

  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

```yaml
# prometheus.yml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'app-exporter'
    static_configs:
      - targets: ['exporter:9090']
```

### Step 4: 테스트

```bash
# 1. 시작
docker-compose up -d

# 2. 트래픽 생성
for i in {1..100}; do
  curl http://localhost:8080/
  [ $((RANDOM % 10)) -eq 0 ] && curl http://localhost:8080/error
  sleep 0.5
done

# 3. Prometheus 확인
# http://localhost:9091
# Query: app_requests_total

# 4. Grafana 대시보드
# http://localhost:3000 (admin/admin)
# Add data source: Prometheus (http://prometheus:9090)
# Create dashboard with app_requests_total, app_errors_total
```

---

## 🔧 실습 3: 프록시 사이드카 (Nginx)

### Step 1: Nginx Sidecar 설정

```nginx
# nginx/nginx.conf
events {
    worker_connections 1024;
}

http {
    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=app_limit:10m rate=10r/s;

    # 액세스 로그
    log_format detailed '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $body_bytes_sent '
                       '"$http_referer" "$http_user_agent" '
                       'rt=$request_time';

    access_log /var/log/nginx/access.log detailed;

    upstream app {
        server localhost:8080;
    }

    server {
        listen 80;

        # Rate Limiting 적용
        location / {
            limit_req zone=app_limit burst=20;
            
            # 프록시 설정
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            # Timeout 설정
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
        }

        # Health Check (직접 처리)
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # Metrics (Nginx 자체 메트릭)
        location /nginx_status {
            stub_status on;
            access_log off;
        }
    }
}
```

### Step 2: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 메인 애플리케이션
  app:
    build: ./app
    networks:
      - sidecar

  # Nginx 프록시 사이드카
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - nginx-logs:/var/log/nginx
    network_mode: "service:app"  # 네트워크 공유
    depends_on:
      - app

  # 로그 수집 사이드카
  log-collector:
    image: fluent/fluentd:v1.16-1
    volumes:
      - nginx-logs:/var/log/nginx:ro
      - ./fluentd/fluent.conf:/fluentd/etc/fluent.conf
    networks:
      - sidecar

volumes:
  nginx-logs:

networks:
  sidecar:
    driver: bridge
```

### Step 3: 테스트

```bash
# 1. 시작
docker-compose up -d

# 2. Rate Limiting 테스트
for i in {1..30}; do
  curl http://localhost:8080/
done
# 429 Too Many Requests 발생

# 3. Nginx 메트릭
curl http://localhost:8080/nginx_status

# 4. 액세스 로그
docker exec -it <nginx-container> tail -f /var/log/nginx/access.log
```

---

## 🔧 실습 4: 설정 동기화 사이드카

### Step 1: 설정 동기화 스크립트

```python
# config-sync/sync.py
import requests
import time
import os
import json

CONFIG_URL = os.getenv('CONFIG_URL', 'http://config-server:8080/config')
CONFIG_FILE = '/etc/app/config.json'
SYNC_INTERVAL = int(os.getenv('SYNC_INTERVAL', '10'))

def fetch_config():
    """원격 설정 서버에서 설정 가져오기"""
    try:
        response = requests.get(CONFIG_URL, timeout=5)
        return response.json()
    except Exception as e:
        print(f"Failed to fetch config: {e}")
        return None

def write_config(config):
    """설정 파일 쓰기"""
    try:
        with open(CONFIG_FILE, 'w') as f:
            json.dump(config, f, indent=2)
        print(f"Config updated: {config}")
        return True
    except Exception as e:
        print(f"Failed to write config: {e}")
        return False

def sync_config():
    """설정 동기화 루프"""
    current_config = None
    
    while True:
        new_config = fetch_config()
        
        if new_config and new_config != current_config:
            if write_config(new_config):
                current_config = new_config
                # 메인 앱에 SIGHUP 전송 (재로드)
                os.system('pkill -HUP -f "python main.py"')
        
        time.sleep(SYNC_INTERVAL)

if __name__ == '__main__':
    os.makedirs(os.path.dirname(CONFIG_FILE), exist_ok=True)
    sync_config()
```

### Step 2: 메인 애플리케이션 (설정 읽기)

```python
# app/main.py
from flask import Flask, jsonify
import json
import signal
import os

app = Flask(__name__)

CONFIG_FILE = '/etc/app/config.json'
config = {}

def load_config():
    """설정 파일 로드"""
    global config
    try:
        with open(CONFIG_FILE, 'r') as f:
            config = json.load(f)
        print(f"Config loaded: {config}")
    except FileNotFoundError:
        print("Config file not found, using defaults")
        config = {'feature_flag': False, 'max_items': 10}

def reload_config(signum, frame):
    """SIGHUP 시그널 핸들러"""
    print("Received SIGHUP, reloading config...")
    load_config()

# 시그널 핸들러 등록
signal.signal(signal.SIGHUP, reload_config)

# 초기 로드
load_config()

@app.route('/config')
def get_config():
    return jsonify(config)

@app.route('/api/items')
def get_items():
    max_items = config.get('max_items', 10)
    return jsonify({
        'items': list(range(max_items)),
        'total': max_items
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Step 3: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 설정 서버 (시뮬레이션)
  config-server:
    image: nginx:alpine
    volumes:
      - ./config-server/config.json:/usr/share/nginx/html/config:ro
    networks:
      - app-network

  # 메인 애플리케이션
  app:
    build: ./app
    volumes:
      - app-config:/etc/app
    ports:
      - "8080:8080"
    networks:
      - app-network

  # 설정 동기화 사이드카
  config-sync:
    build: ./config-sync
    environment:
      - CONFIG_URL=http://config-server/config
      - SYNC_INTERVAL=5
    volumes:
      - app-config:/etc/app
    pid: "service:app"  # PID namespace 공유
    depends_on:
      - config-server
      - app
    networks:
      - app-network

volumes:
  app-config:

networks:
  app-network:
    driver: bridge
```

---

## 🔧 실습 5: Kubernetes Sidecar (실전)

### Step 1: Pod with Sidecar

```yaml
# pod-with-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecars
  labels:
    app: myapp
spec:
  # 메인 컨테이너
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
    - name: config
      mountPath: /etc/app
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"

  # 로깅 사이드카
  - name: log-collector
    image: fluent/fluentd:v1.16-1
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true
    - name: fluentd-config
      mountPath: /fluentd/etc
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"

  # 모니터링 사이드카
  - name: metrics-exporter
    image: prom/node-exporter:latest
    ports:
    - containerPort: 9100
    resources:
      requests:
        memory: "32Mi"
        cpu: "25m"
      limits:
        memory: "64Mi"
        cpu: "50m"

  # 프록시 사이드카
  - name: envoy
    image: envoyproxy/envoy:v1.28-latest
    ports:
    - containerPort: 15001
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"

  # 볼륨
  volumes:
  - name: logs
    emptyDir: {}
  - name: config
    configMap:
      name: app-config
  - name: fluentd-config
    configMap:
      name: fluentd-config
  - name: envoy-config
    configMap:
      name: envoy-config
```

### Step 2: Deployment with Sidecars

```yaml
# deployment-with-sidecars.yaml
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
      # 메인 앱
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 3

      # Envoy 사이드카
      - name: envoy
        image: envoyproxy/envoy:v1.28-latest
        ports:
        - containerPort: 15001
          name: envoy-admin
        volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]

      volumes:
      - name: envoy-config
        configMap:
          name: envoy-config
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 사이드카 타입           │ 용도                         │
├──────────────────────┼────────────────────────────┤
│ Logging              │ 로그 수집 및 전송              │
├──────────────────────┼────────────────────────────┤
│ Monitoring           │ 메트릭 수집 및 노출             │
├──────────────────────┼────────────────────────────┤
│ Proxy                │ 트래픽 제어, mTLS             │
├──────────────────────┼────────────────────────────┤
│ Config Sync          │ 동적 설정 동기화               │
├──────────────────────┼────────────────────────────┤
│ Secret Management    │ Vault 통합                  │
├──────────────────────┼────────────────────────────┤
│ Service Mesh         │ Envoy, Linkerd             │
└──────────────────────┴────────────────────────────┘

Best Practices:
✅ 리소스 제한 설정 (메모리, CPU)
✅ PreStop Hook으로 순서 제어
✅ Health Check 구현
✅ 로그 레벨 조정 가능
✅ 메트릭 노출 (Prometheus)
```

---

## 🎓 연습 문제

### 문제 1: Sidecar가 Main Container보다 먼저 종료되지 않게 하려면?

<details>
<summary>정답 보기</summary>

```yaml
# PreStop Hook 사용
spec:
  containers:
  - name: app
    image: myapp:latest
    # Main container는 즉시 종료

  - name: log-sidecar
    image: fluentd:latest
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 10"]
    # 10초 대기 후 종료
```

**동작:**
1. SIGTERM 전송 (Main + Sidecar 동시)
2. Main: 즉시 종료 시작
3. Sidecar: PreStop Hook 실행 (10초 sleep)
4. Main: 종료 완료 (2초)
5. Sidecar: sleep 완료 후 종료 (10초)

**결과:** Sidecar가 Main보다 나중에 종료되어 로그 손실 방지

</details>

### 문제 2: Sidecar의 리소스 사용량이 과도하면 어떻게 제한하는가?

<details>
<summary>정답 보기</summary>

```yaml
# Kubernetes
spec:
  containers:
  - name: log-sidecar
    image: fluentd:latest
    resources:
      requests:
        memory: "64Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "100m"
```

```yaml
# Docker Compose
services:
  log-sidecar:
    image: fluentd:latest
    deploy:
      resources:
        limits:
          cpus: '0.1'
          memory: 128M
        reservations:
          cpus: '0.05'
          memory: 64M
```

**Best Practices:**
- Sidecar는 Main보다 적은 리소스
- 일반적으로 Main의 10-20%
- Monitoring으로 실제 사용량 확인 후 조정

</details>

### 문제 3: 여러 Sidecar를 효율적으로 관리하려면?

<details>
<summary>정답 보기</summary>

**1. Helm Chart로 템플릿화:**
```yaml
# templates/deployment.yaml
{{- if .Values.sidecars.logging.enabled }}
- name: log-sidecar
  image: {{ .Values.sidecars.logging.image }}
  {{- with .Values.sidecars.logging.resources }}
  resources: {{ toYaml . | nindent 4 }}
  {{- end }}
{{- end }}

{{- if .Values.sidecars.monitoring.enabled }}
- name: metrics-sidecar
  image: {{ .Values.sidecars.monitoring.image }}
  {{- with .Values.sidecars.monitoring.resources }}
  resources: {{ toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

```yaml
# values.yaml
sidecars:
  logging:
    enabled: true
    image: fluentd:v1.16
    resources:
      limits:
        memory: 128Mi
  monitoring:
    enabled: true
    image: prom/node-exporter:latest
    resources:
      limits:
        memory: 64Mi
```

**2. Kustomize로 오버레이:**
```yaml
# base/deployment.yaml (기본)
spec:
  containers:
  - name: app
    image: myapp:latest

# overlays/production/sidecar-patch.yaml
spec:
  containers:
  - name: log-sidecar
    image: fluentd:v1.16
  - name: envoy
    image: envoyproxy/envoy:v1.28
```

**3. Service Mesh (Istio):**
```yaml
# 자동 Envoy Sidecar 주입
metadata:
  labels:
    sidecar.istio.io/inject: "true"
# Istio가 자동으로 Envoy 추가
```

</details>

---

## 📌 핵심 요약

```
Sidecar 패턴 원칙:
1. 관심사 분리: 앱 로직 vs 인프라 로직
2. 재사용성: 모든 서비스에 동일 사이드카
3. 기술 독립성: 언어 제약 없음
4. 독립 배포: 사이드카만 업데이트
5. 표준화: 중앙 관리

일반적인 사이드카:
- Logging: Fluentd, Filebeat
- Monitoring: Prometheus Exporter, Telegraf
- Proxy: Envoy, Nginx, HAProxy
- Service Mesh: Istio (Envoy), Linkerd
- Secret: Vault Agent
```

---

## 📚 참고 자료

- [Kubernetes Sidecar Containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- [Istio Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)
- [Fluentd Documentation](https://docs.fluentd.org/)
- [Envoy Proxy](https://www.envoyproxy.io/)

---

## 🤔 생각해볼 문제

1. Sidecar 패턴과 DaemonSet의 차이는? 언제 DaemonSet을 사용해야 하는가?
2. Sidecar가 많아지면 Pod가 무거워진다. 어떻게 최적화할 수 있는가?
3. Sidecar와 Init Container의 차이는? 각각 언제 사용하는가?

> 💡 **답변**:
> 
> **1) Sidecar vs DaemonSet:**
> 
> **Sidecar (Pod 내부):**
> - 각 Pod에 포함
> - 특정 애플리케이션 전용
> - Pod와 생명주기 공유
> 
> **DaemonSet (Node 레벨):**
> - 모든 Node에 1개
> - Node 전체 서비스
> - 모든 Pod 공유
> 
> **선택 기준:**
> ```
> Sidecar 사용:
> - 앱별 설정 필요 (앱 A의 로그 vs 앱 B의 로그)
> - 독립적 스케일링
> - 높은 격리
> 
> DaemonSet 사용:
> - Node 수준 작업 (로그 수집, 모니터링)
> - 리소스 절약 (Node당 1개)
> - 모든 Pod에 동일한 서비스
> 
> 예:
> Sidecar: Envoy (앱별 프록시 설정)
> DaemonSet: Node Exporter (Node 메트릭)
> ```
> 
> **2) Sidecar 최적화:**
> 
> ```yaml
> # 1. 경량 이미지 사용
> # ❌ Heavy
> image: fluentd:latest  # 300MB
> 
> # ✅ Light
> image: fluent/fluentd:v1.16-debian-1  # 50MB
> 
> # 2. 리소스 제한
> resources:
>   limits:
>     memory: 64Mi  # 최소한으로
>     cpu: 50m
> 
> # 3. 필요한 Sidecar만
> # ❌ 모든 Pod에 모든 Sidecar
> # ✅ 필요한 Pod에만 선택적 추가
> 
> # 4. DaemonSet 고려
> # 로그 수집: Sidecar → DaemonSet
> 
> # 5. Service Mesh 사용
> # 여러 Sidecar 대신 Envoy 하나로 통합
> ```
> 
> **3) Sidecar vs Init Container:**
> 
> ```
> Init Container:
> - 순차 실행
> - 완료 후 종료
> - Main Container 시작 전
> - 사전 준비 작업
> 
> Sidecar:
> - Main과 동시 실행
> - 계속 실행
> - Main과 함께 종료
> - 지속적 보조 작업
> 
> 사용 예:
> Init Container:
> - DB 마이그레이션
> - 설정 파일 다운로드
> - 권한 설정
> - Git clone
> 
> Sidecar:
> - 로그 수집 (지속)
> - 프록시 (지속)
> - 메트릭 수집 (지속)
> - 설정 동기화 (지속)
> ```

---

<div align="center">

**[⬅️ 이전: Microservices](01-Microservices.md)** | **[다음: Ambassador Pattern ➡️](./03-Ambassador-Pattern.md)**

</div>
