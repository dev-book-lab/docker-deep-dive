# 08. Load Balancing - 로드 밸런싱

## 🎯 이 챕터에서 배울 것

- Docker **내장 로드 밸런서** 동작 원리
- **Swarm 모드** 로드 밸런싱
- **외부 로드 밸런서** 연동 (Nginx, HAProxy, Traefik)
- **헬스 체크**와 장애 복구

## 📌 왜 중요한가?

**"로드 밸런싱은 고가용성과 확장성의 핵심입니다."**

```
로드 밸런싱 없음:
Client → Single Server
- 단일 장애점 (SPOF)
- 확장 불가
- 과부하 위험

로드 밸런싱:
              ┌─→ Server 1
Client → LB ──┼─→ Server 2
              └─→ Server 3
- 고가용성
- 수평 확장
- 트래픽 분산

효과:
┌────────────────┬──────────┬──────────┐
│ 지표            │ 없음      │ 있음      │
├────────────────┼──────────┼──────────┤
│ 가용성           │ 95%      │ 99.9%    │
├────────────────┼──────────┼──────────┤
│ 처리량           │ 1,000/s  │ 10,000/s │
├────────────────┼──────────┼──────────┤
│ 응답시간         │ 500ms    │ 50ms     │
├────────────────┼──────────┼──────────┤
│ 확장            │ 불가      │ 자동      │
└────────────────┴──────────┴──────────┘
```

**실무 영향:**
- 가용성: 무중단 서비스 제공
- 성능: 트래픽 분산으로 응답 속도 개선
- 확장: 서버 추가로 용량 증대
- 배포: 무중단 업데이트 가능

---

## 🔬 Deep Dive

### 1. Docker 내장 로드 밸런서

#### DNS 라운드 로빈

```
기본 메커니즘:
- 같은 네트워크 별칭의 여러 컨테이너
- DNS가 모든 IP 반환
- 클라이언트가 랜덤 선택

예시:
┌─────────────────────────────────────┐
│ DNS Query: "web"                    │
├─────────────────────────────────────┤
│ Answer:                             │
│ - 10.0.1.2 (web-1)                  │
│ - 10.0.1.3 (web-2)                  │
│ - 10.0.1.4 (web-3)                  │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ Client picks one randomly           │
└─────────────────────────────────────┘

장점:
+ 설정 간단
+ 자동 동작

단점:
- 클라이언트 캐싱
- 헬스 체크 없음
- 세션 유지 불가
```

#### 실습

```bash
# 네트워크 생성
docker network create lb-net

# 백엔드 서버 3대
docker run -d --name web-1 \
  --network lb-net \
  --network-alias web \
  nginx:alpine

docker run -d --name web-2 \
  --network lb-net \
  --network-alias web \
  nginx:alpine

docker run -d --name web-3 \
  --network lb-net \
  --network-alias web \
  nginx:alpine

# DNS 확인
docker run --rm --network lb-net alpine nslookup web
# Name:	web
# Address: 10.0.1.2
# Address: 10.0.1.3
# Address: 10.0.1.4

# 부하 테스트
docker run --rm --network lb-net curlimages/curl sh -c '
  for i in $(seq 1 12); do
    curl -s http://web | grep -o "web-[0-9]"
  done
'
# web-2
# web-1
# web-3
# web-1
# web-2
# ...

# 정리
docker rm -f web-1 web-2 web-3
docker network rm lb-net
```

---

### 2. Docker Swarm 로드 밸런싱

#### Ingress 네트워크

```
Swarm Ingress:
- 클러스터 전체 로드 밸런서
- 어느 노드로든 접근 가능
- 자동 라우팅
- 내장 헬스 체크

구조:
┌─────────────────────────────────────┐
│ Client                              │
└───────────┬─────────────────────────┘
            │ :8080
    ┌───────▼────────┬────────────────┐─────────────────┐
    │ Node 1         │ Node 2         │ Node 3          │
    │ Ingress: 8080  │ Ingress: 8080  │ Ingress: 8080   │
    └───────┬────────┴────────┬───────┴────────┬────────┘
            │                 │                │
            │    IPVS Load Balancer (Kernel)   │
            │                 │                │
    ┌───────▼────────┬────────▼───────┬────────▼───────┐
    │ Task 1         │ Task 2         │ Task 3         │
    │ Container      │ Container      │ Container      │
    │ 10.0.0.2:80    │ 10.0.0.3:80    │ 10.0.0.4:80    │
    └────────────────┴────────────────┴────────────────┘

특징:
- 모든 노드가 로드 밸런서
- IPVS (IP Virtual Server) 사용
- Layer 4 로드 밸런싱
- 연결 추적 (Connection tracking)
```

#### Swarm 서비스 배포

```bash
# Swarm 초기화
docker swarm init

# 서비스 생성 (3개 레플리카)
docker service create \
  --name web \
  --replicas 3 \
  --publish published=8080,target=80 \
  nginx:alpine

# 서비스 확인
docker service ls
# ID        NAME  MODE        REPLICAS  IMAGE
# abc123    web   replicated  3/3       nginx:alpine

# 레플리카 위치
docker service ps web
# ID       NAME   IMAGE         NODE    DESIRED STATE  CURRENT STATE
# xyz1     web.1  nginx:alpine  node1   Running        Running
# xyz2     web.2  nginx:alpine  node1   Running        Running
# xyz3     web.3  nginx:alpine  node1   Running        Running

# 어느 노드로든 접근 가능
curl http://localhost:8080
# Welcome to nginx!

# 부하 분산 확인
for i in {1..12}; do
  curl -s http://localhost:8080 | grep -o "web\.[0-9]"
done
# web.1
# web.3
# web.2
# web.1
# web.2
# ...

# 서비스 스케일링
docker service scale web=5

# 즉시 반영
docker service ps web
# 5개 레플리카로 증가

# 정리
docker service rm web
docker swarm leave --force
```

---

### 3. Nginx 로드 밸런서

#### 기본 설정

```nginx
# nginx.conf
upstream backend {
    # 기본: 라운드 로빈
    server web-1:80;
    server web-2:80;
    server web-3:80;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

#### 고급 로드 밸런싱 알고리즘

```nginx
# 1. 라운드 로빈 (기본)
upstream backend {
    server web-1:80;
    server web-2:80;
    server web-3:80;
}

# 2. 가중치 (Weighted)
upstream backend {
    server web-1:80 weight=3;  # 3배 많은 트래픽
    server web-2:80 weight=2;
    server web-3:80 weight=1;
}

# 3. 최소 연결 (Least Connections)
upstream backend {
    least_conn;
    server web-1:80;
    server web-2:80;
    server web-3:80;
}

# 4. IP 해시 (세션 유지)
upstream backend {
    ip_hash;
    server web-1:80;
    server web-2:80;
    server web-3:80;
}

# 5. 일반 해시 (커스텀 키)
upstream backend {
    hash $request_uri consistent;
    server web-1:80;
    server web-2:80;
    server web-3:80;
}
```

#### 헬스 체크

```nginx
upstream backend {
    server web-1:80 max_fails=3 fail_timeout=30s;
    server web-2:80 max_fails=3 fail_timeout=30s;
    server web-3:80 max_fails=3 fail_timeout=30s;
}

# max_fails: 실패 횟수 임계값
# fail_timeout: 재시도 대기 시간
```

#### 실습

```bash
# Docker Compose 구성
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web-1
      - web-2
      - web-3
    networks:
      - app-net

  web-1:
    image: nginx:alpine
    volumes:
      - ./html/web-1:/usr/share/nginx/html:ro
    networks:
      - app-net

  web-2:
    image: nginx:alpine
    volumes:
      - ./html/web-2:/usr/share/nginx/html:ro
    networks:
      - app-net

  web-3:
    image: nginx:alpine
    volumes:
      - ./html/web-3:/usr/share/nginx/html:ro
    networks:
      - app-net

networks:
  app-net:
EOF

# HTML 파일 생성
mkdir -p html/{web-1,web-2,web-3}
echo "<h1>Web Server 1</h1>" > html/web-1/index.html
echo "<h1>Web Server 2</h1>" > html/web-2/index.html
echo "<h1>Web Server 3</h1>" > html/web-3/index.html

# Nginx 설정
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server web-1:80;
        server web-2:80;
        server web-3:80;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
EOF

# 시작
docker-compose up -d

# 테스트
for i in {1..12}; do
  curl -s http://localhost | grep -o "Server [0-9]"
done
# Server 1
# Server 2
# Server 3
# Server 1
# ...

# 정리
docker-compose down
```

---

### 4. HAProxy

#### 기본 설정

```haproxy
# haproxy.cfg
global
    log stdout format raw local0
    maxconn 4096

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    server web-1 web-1:80 check
    server web-2 web-2:80 check
    server web-3 web-3:80 check

listen stats
    bind *:8404
    stats enable
    stats uri /
    stats refresh 30s
```

#### 로드 밸런싱 알고리즘

```haproxy
# 1. 라운드 로빈
backend http_back
    balance roundrobin
    server web-1 web-1:80 check
    server web-2 web-2:80 check
    server web-3 web-3:80 check

# 2. 최소 연결
backend http_back
    balance leastconn
    server web-1 web-1:80 check
    server web-2 web-2:80 check
    server web-3 web-3:80 check

# 3. 소스 IP 해시
backend http_back
    balance source
    server web-1 web-1:80 check
    server web-2 web-2:80 check
    server web-3 web-3:80 check

# 4. URI 해시
backend http_back
    balance uri
    server web-1 web-1:80 check
    server web-2 web-2:80 check
    server web-3 web-3:80 check

# 5. 가중치
backend http_back
    balance roundrobin
    server web-1 web-1:80 check weight 3
    server web-2 web-2:80 check weight 2
    server web-3 web-3:80 check weight 1
```

#### 고급 헬스 체크

```haproxy
backend http_back
    balance roundrobin
    
    # HTTP 헬스 체크
    option httpchk GET /health
    http-check expect status 200
    
    # 서버 설정
    server web-1 web-1:80 check inter 2000 rise 2 fall 3
    server web-2 web-2:80 check inter 2000 rise 2 fall 3
    server web-3 web-3:80 check inter 2000 rise 2 fall 3

# inter: 체크 간격 (2초)
# rise: 정상 판정 횟수 (2회 연속 성공)
# fall: 비정상 판정 횟수 (3회 연속 실패)
```

#### 실습

```yaml
# docker-compose.yml
version: '3.8'

services:
  haproxy:
    image: haproxy:2.6-alpine
    ports:
      - "80:80"
      - "8404:8404"  # Stats
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    depends_on:
      - web-1
      - web-2
      - web-3
    networks:
      - app-net

  web-1:
    image: nginx:alpine
    volumes:
      - ./html/web-1:/usr/share/nginx/html:ro
    networks:
      - app-net

  web-2:
    image: nginx:alpine
    volumes:
      - ./html/web-2:/usr/share/nginx/html:ro
    networks:
      - app-net

  web-3:
    image: nginx:alpine
    volumes:
      - ./html/web-3:/usr/share/nginx/html:ro
    networks:
      - app-net

networks:
  app-net:
```

```bash
# 시작
docker-compose up -d

# Stats UI 확인
# http://localhost:8404

# 부하 테스트
ab -n 10000 -c 100 http://localhost/

# 정리
docker-compose down
```

---

### 5. Traefik (동적 로드 밸런서)

#### Docker 라벨 기반 설정

```yaml
# docker-compose.yml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"  # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-net

  web:
    image: nginx:alpine
    deploy:
      replicas: 3
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`localhost`)"
      - "traefik.http.services.web.loadbalancer.server.port=80"
    networks:
      - traefik-net

networks:
  traefik-net:
```

#### 고급 설정

```yaml
services:
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--metrics.prometheus=true"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-net

  web:
    image: nginx:alpine
    deploy:
      replicas: 3
    labels:
      # 기본 라우팅
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`example.com`)"
      - "traefik.http.routers.web.entrypoints=web"
      
      # 로드 밸런싱
      - "traefik.http.services.web.loadbalancer.server.port=80"
      - "traefik.http.services.web.loadbalancer.sticky.cookie=true"
      - "traefik.http.services.web.loadbalancer.sticky.cookie.name=lb"
      
      # 헬스 체크
      - "traefik.http.services.web.loadbalancer.healthcheck.path=/health"
      - "traefik.http.services.web.loadbalancer.healthcheck.interval=10s"
      - "traefik.http.services.web.loadbalancer.healthcheck.timeout=3s"
      
      # 미들웨어
      - "traefik.http.routers.web.middlewares=rate-limit"
      - "traefik.http.middlewares.rate-limit.ratelimit.average=100"
    networks:
      - traefik-net

networks:
  traefik-net:
```

---

## 💻 실습

### 실습 1: Swarm 로드 밸런싱

#### 클러스터 구성

```bash
# Swarm 초기화
docker swarm init

# 서비스 생성
docker service create \
  --name web \
  --replicas 5 \
  --publish published=8080,target=80 \
  --update-delay 10s \
  --update-parallelism 2 \
  nginx:alpine

# 서비스 확인
docker service ps web
```

#### 부하 분산 확인

```bash
# 부하 테스트
ab -n 100000 -c 1000 http://localhost:8080/

# 결과:
# Requests per second:    15234.56 [#/sec]
# Time per request:       65.637 [ms]

# 분산 확인
docker service logs web | grep "GET /" | \
  awk '{print $4}' | sort | uniq -c
# 20000 web.1
# 20000 web.2
# 20000 web.3
# 20000 web.4
# 20000 web.5
# 균등 분산!
```

#### 무중단 업데이트

```bash
# 새 버전 배포
docker service update \
  --image nginx:1.21-alpine \
  web

# 업데이트 진행 확인
docker service ps web
# ID       NAME      IMAGE              DESIRED STATE
# abc123   web.1     nginx:1.21-alpine  Running
# def456   web.1     nginx:alpine       Shutdown
# ...

# 트래픽은 계속 처리됨!
curl http://localhost:8080
# ✅ 무중단

# 정리
docker service rm web
docker swarm leave --force
```

---

### 실습 2: Nginx 고급 로드 밸런싱

#### Sticky Session

```nginx
# nginx.conf
upstream backend {
    ip_hash;  # 동일 IP는 동일 서버로
    server web-1:80;
    server web-2:80;
    server web-3:80;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # 세션 쿠키
        proxy_cookie_path / "/; HttpOnly; Secure";
    }
}
```

#### 동적 가중치 조정

```bash
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - app-net

  # 고성능 서버
  web-1:
    image: nginx:alpine
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
    networks:
      - app-net

  # 중성능 서버
  web-2:
    image: nginx:alpine
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
    networks:
      - app-net

  # 저성능 서버
  web-3:
    image: nginx:alpine
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    networks:
      - app-net

networks:
  app-net:
```

```nginx
# nginx.conf - 가중치 설정
upstream backend {
    server web-1:80 weight=4;  # 고성능 → 많은 트래픽
    server web-2:80 weight=2;  # 중성능
    server web-3:80 weight=1;  # 저성능 → 적은 트래픽
}
```

---

### 실습 3: HAProxy + 헬스 체크

#### 애플리케이션

```python
# app.py
from flask import Flask, jsonify
import random
import time

app = Flask(__name__)

# 서버 상태 (시뮬레이션)
healthy = True

@app.route('/')
def index():
    return f"Hello from {app.config['SERVER_ID']}"

@app.route('/health')
def health():
    global healthy
    if healthy:
        return jsonify({"status": "healthy"}), 200
    else:
        return jsonify({"status": "unhealthy"}), 503

@app.route('/fail')
def fail():
    global healthy
    healthy = False
    return "Server marked as unhealthy"

@app.route('/recover')
def recover():
    global healthy
    healthy = True
    return "Server marked as healthy"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```dockerfile
# Dockerfile
FROM python:3.9-alpine
WORKDIR /app
RUN pip install flask
COPY app.py .
CMD ["python", "app.py"]
```

#### HAProxy 설정

```haproxy
# haproxy.cfg
global
    log stdout format raw local0

defaults
    log global
    mode http
    option httplog
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200
    
    server web-1 web-1:5000 check inter 2s rise 2 fall 3
    server web-2 web-2:5000 check inter 2s rise 2 fall 3
    server web-3 web-3:5000 check inter 2s rise 2 fall 3

listen stats
    bind *:8404
    stats enable
    stats uri /
```

#### 장애 시뮬레이션

```bash
# 서비스 시작
docker-compose up -d

# Stats 확인: http://localhost:8404

# 정상 상태 확인
curl http://localhost/
# Hello from web-1

# 서버 1 장애 발생
curl http://web-1:5000/fail

# HAProxy가 자동 감지 (2초 × 3회 = 6초 후)
# web-1이 제외됨

# 트래픽 확인
for i in {1..20}; do
  curl -s http://localhost/ | grep -o "web-[0-9]"
done
# web-2
# web-3
# web-2
# web-3
# ... (web-1 없음!)

# 서버 1 복구
curl http://web-1:5000/recover

# HAProxy가 자동 감지 (2초 × 2회 = 4초 후)
# web-1이 다시 추가됨

# 트래픽 확인
for i in {1..20}; do
  curl -s http://localhost/ | grep -o "web-[0-9]"
done
# web-1  ← 복귀!
# web-2
# web-3
# ...
```

---

### 실습 4: Traefik 동적 라우팅

#### 설정

```yaml
# docker-compose.yml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-net

  # Service A
  api-v1:
    image: nginx:alpine
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api-v1.rule=Host(`api.localhost`) && PathPrefix(`/v1`)"
      - "traefik.http.services.api-v1.loadbalancer.server.port=80"
    networks:
      - traefik-net

  # Service B
  api-v2:
    image: nginx:alpine
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api-v2.rule=Host(`api.localhost`) && PathPrefix(`/v2`)"
      - "traefik.http.services.api-v2.loadbalancer.server.port=80"
    networks:
      - traefik-net

  # Web
  web:
    image: nginx:alpine
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.web.rule=Host(`localhost`)"
      - "traefik.http.services.web.loadbalancer.server.port=80"
    networks:
      - traefik-net

networks:
  traefik-net:
```

#### 동적 스케일링

```bash
# 서비스 시작
docker-compose up -d

# Dashboard: http://localhost:8080

# web 서비스 스케일 업
docker-compose up -d --scale web=5

# Traefik이 자동 감지!
# Dashboard에서 5개 인스턴스 확인

# 부하 분산 확인
for i in {1..50}; do
  curl -s http://localhost/ | grep -o "web-[0-9]"
done

# 스케일 다운
docker-compose up -d --scale web=2

# 자동 반영!
```

---

## 🔥 실전 적용

### 시나리오 1: 다층 로드 밸런싱

**아키텍처:**

```
Internet
  ↓
External LB (Nginx/HAProxy)
  ↓
┌─────────┬─────────┬─────────┐
│ Node 1  │ Node 2  │ Node 3  │
│ Traefik │ Traefik │ Traefik │ ← Service Discovery
└────┬────┴────┬────┴────┬────┘
     ↓         ↓         ↓
  ┌─────────────────────────┐
  │   Docker Swarm          │
  │   (Ingress LB)          │ ← Container LB
  └─────────────────────────┘
     ↓         ↓         ↓
 Container Container Container
```

**구성:**

```yaml
# docker-stack.yml
version: '3.8'

services:
  # External LB (Nginx)
  nginx:
    image: nginx:alpine
    ports:
      - target: 80
        published: 80
        mode: host
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
    networks:
      - frontend

  # Service Discovery (Traefik)
  traefik:
    image: traefik:v2.10
    command:
      - "--providers.docker.swarmmode=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - target: 80
        published: 8080
        mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
    networks:
      - frontend
      - backend

  # Application
  app:
    image: myapp:latest
    deploy:
      replicas: 10
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=PathPrefix(`/api`)"
      - "traefik.http.services.app.loadbalancer.server.port=8080"
    networks:
      - backend

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
```

---

### 시나리오 2: 카나리 배포

**전략:**

```
Production (90%) → v1.0
Canary (10%)     → v2.0

점진적 전환:
90/10 → 70/30 → 50/50 → 30/70 → 10/90 → 0/100
```

**HAProxy 설정:**

```haproxy
# haproxy.cfg
frontend http_front
    bind *:80
    
    # 10% canary
    acl canary_selected rand(100) lt 10
    
    use_backend canary_back if canary_selected
    default_backend production_back

backend production_back
    balance roundrobin
    server prod-1 prod-1:80 check
    server prod-2 prod-2:80 check
    server prod-3 prod-3:80 check

backend canary_back
    balance roundrobin
    server canary-1 canary-1:80 check
    server canary-2 canary-2:80 check
```

**Nginx 설정:**

```nginx
upstream production {
    server prod-1:80;
    server prod-2:80;
    server prod-3:80;
}

upstream canary {
    server canary-1:80;
    server canary-2:80;
}

split_clients "${remote_addr}${http_user_agent}${date_gmt}" $backend {
    10%     canary;  # 10% canary
    *       production;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://$backend;
    }
}
```

---

## ⚡ 로드 밸런싱 체크리스트

### 알고리즘 선택

```
□ 라운드 로빈 (균등 분산)
□ 가중치 (서버 성능 차이)
□ 최소 연결 (동적 부하 분산)
□ IP 해시 (세션 유지)
□ URI 해시 (캐시 최적화)
```

### 헬스 체크

```
□ HTTP 헬스 체크 (/health)
□ 체크 간격 설정
□ 실패 임계값 (fall)
□ 복구 임계값 (rise)
□ 타임아웃 설정
```

### 성능 최적화

```
□ Keepalive 활성화
□ 연결 풀 크기
□ 타임아웃 조정
□ 버퍼 크기
□ 워커 프로세스 수
```

### 모니터링

```
□ 요청 분산 비율
□ 응답 시간 (p50, p95, p99)
□ 에러율
□ 서버 상태 (up/down)
□ 연결 수
```

### 고가용성

```
□ LB 이중화 (Active-Passive)
□ 자동 장애 복구
□ 무중단 배포
□ 헬스 체크 자동화
□ 백업 서버 준비
```

---

## 🚫 안티패턴

### 1. 헬스 체크 없음

```haproxy
# ❌ 헬스 체크 없음
backend http_back
    server web-1 web-1:80
    server web-2 web-2:80
# 죽은 서버로도 트래픽 전송

# ✅ 헬스 체크
backend http_back
    option httpchk GET /health
    server web-1 web-1:80 check
    server web-2 web-2:80 check
```

### 2. 단일 로드 밸런서

```yaml
# ❌ SPOF (Single Point of Failure)
services:
  lb:
    image: nginx
    ports:
      - "80:80"
    # LB가 죽으면 전체 서비스 중단

# ✅ 이중화
services:
  lb-1:
    image: nginx
  lb-2:
    image: nginx
  # Keepalived/VRRP로 VIP 관리
```

### 3. 부적절한 알고리즘

```nginx
# ❌ 세션 서비스에서 라운드 로빈
upstream backend {
    # 세션 유실!
    server web-1:80;
    server web-2:80;
}

# ✅ IP 해시 (세션 유지)
upstream backend {
    ip_hash;
    server web-1:80;
    server web-2:80;
}
```

### 4. 타임아웃 미설정

```nginx
# ❌ 기본 타임아웃 (너무 김)
upstream backend {
    server web-1:80;
}

# ✅ 적절한 타임아웃
upstream backend {
    server web-1:80;
}

location / {
    proxy_pass http://backend;
    proxy_connect_timeout 5s;
    proxy_send_timeout 10s;
    proxy_read_timeout 30s;
}
```

---

## 🎓 핵심 정리

### 1. 로드 밸런싱 레벨

```
L4 (Transport Layer):
- IP + Port 기반
- 빠름
- 프로토콜 무관
- 예: IPVS, HAProxy TCP

L7 (Application Layer):
- HTTP 헤더/경로 기반
- 느림 (상대적)
- 고급 라우팅
- 예: Nginx, HAProxy HTTP, Traefik
```

### 2. 알고리즘 비교

```
┌─────────────┬────────┬──────────┬──────────┐
│ 알고리즘      │사용 케이스│ 세션 유지  │ 복잡도     │
├─────────────┼────────┼──────────┼──────────┤
│ Round Robin │ 일반    │ ❌       │ O(1)     │
├─────────────┼────────┼──────────┼──────────┤
│ Weighted    │ 성능 차이│ ❌       │ O(1)     │
├─────────────┼────────┼──────────┼──────────┤
│ Least Conn  │ 긴 연결 │ ❌        │ O(n)     │
├─────────────┼────────┼──────────┼──────────┤
│ IP Hash     │ 세션    │ ✅       │ O(1)     │
├─────────────┼────────┼──────────┼──────────┤
│ URI Hash    │ 캐시    │ ✅       │ O(1)     │
└─────────────┴────────┴──────────┴──────────┘
```

### 3. 헬스 체크

```
Active:
- LB가 주기적으로 확인
- HTTP GET /health
- TCP 연결

Passive:
- 실제 트래픽 모니터링
- 에러율 기반
- 빠른 감지
```

### 4. 핵심 명령어

```bash
# Swarm
docker service create --replicas N
docker service scale <service>=N
docker service ps <service>

# Stats
curl http://lb:8404/stats  # HAProxy
curl http://lb:8080/api    # Traefik

# 테스트
ab -n 10000 -c 100 http://lb/
```

---

## 📚 참고 자료

- [Docker Swarm Load Balancing](https://docs.docker.com/engine/swarm/ingress/)
- [Nginx Load Balancing](http://nginx.org/en/docs/http/load_balancing.html)
- [HAProxy Documentation](https://www.haproxy.org/documentation.html)
- [Traefik Documentation](https://doc.traefik.io/traefik/)

---

## 🤔 생각해볼 문제

1. Swarm Ingress 로드 밸런서가 IPVS를 사용하는 이유는?
2. Sticky Session이 수평 확장을 어렵게 만드는 이유는?
3. L4와 L7 로드 밸런서의 성능 차이는 얼마나 될까?

> 💡 **답변**: 1) IPVS는 리눅스 커널의 네트워크 스택에서 동작하는 L4 로드 밸런서로, iptables 기반보다 10-100배 빠름 (해시 테이블 vs 선형 탐색), 수만 개의 동시 연결 처리 가능, 메모리 효율적, 다양한 스케줄링 알고리즘 지원 (rr, lc, wrr 등), 커널 레벨이라 컨텍스트 스위칭 없음, 2) 특정 세션이 특정 서버에 고정되면 해당 서버 장애 시 세션 유실, 서버별 부하 불균형 발생 가능 (특정 사용자의 트래픽이 많으면), 새 서버 추가 시 기존 세션 재분배 어려움, 서버 제거 시 해당 세션 모두 끊김, Redis 같은 세션 스토어 사용이 더 나은 대안, 3) L4는 5-10μs (마이크로초), L7은 50-500μs, 약 10-50배 차이, L7은 HTTP 파싱/헤더 검사 오버헤드, TLS 종료 시 더 큰 차이, 하지만 L7은 경로 기반 라우팅/캐싱/압축 등 고급 기능 제공

---

<div align="center">

**[⬅️ 이전: DNS Resolution](./07-DNS-Resolution.md)** | **[다음: Network Security ➡️](./09-Network-Security.md)**

</div>
