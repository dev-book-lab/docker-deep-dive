# 03. Host Network - 호스트 네트워크

## 🎯 이 챕터에서 배울 것

- **Host 네트워크 모드**의 동작 원리
- Bridge vs Host 네트워크 **성능 비교**
- Host 모드의 **장단점**과 트레이드오프
- 실무 **사용 시나리오**와 베스트 프랙티스

## 📌 왜 중요한가?

**"Host 네트워크는 최고 성능이 필요한 특수한 경우에만 사용합니다."**

```
Bridge 네트워크:
- 격리된 네트워크 스택
- NAT 오버헤드
- 성능: 95% (약간 느림)
- 보안: 높음
- 사용: 일반적인 경우

Host 네트워크:
- 호스트와 동일한 네트워크
- NAT 없음
- 성능: 100% (최고)
- 보안: 낮음
- 사용: 특수한 경우만
```

**실무 영향:**
- 성능: 네트워크 집약적 워크로드 최적화
- 운영: 포트 관리 간소화
- 제약: 포트 충돌, 보안 약화
- 선택: 성능 vs 격리 트레이드오프

---

## 🔬 Deep Dive

### 1. Host 네트워크 모드란?

#### 기본 개념

```
Host 네트워크 모드:
- 컨테이너가 호스트의 네트워크 스택을 직접 사용
- 별도의 네트워크 네임스페이스 없음
- veth pair, bridge 없음
- NAT 없음

구조 비교:

Bridge 모드:
┌─────────────────────────────────────┐
│ Host                                │
│  ┌────────────┐                     │
│  │ Container  │                     │
│  │ Network NS │                     │
│  │ eth0       │ veth                │
│  │ 172.17.0.2 ├─────┐               │
│  └────────────┘     │               │
│                ┌────▼────┐          │
│                │ docker0 │          │
│                │ bridge  │          │
│                └────┬────┘          │
│                ┌────▼────┐          │
│                │  eth0   │          │
│                │ (host)  │          │
│                └─────────┘          │
└─────────────────────────────────────┘

Host 모드:
┌─────────────────────────────────────┐
│ Host                                │
│  ┌────────────┐                     │
│  │ Container  │                     │
│  │ (no NS)    │                     │
│  │            │                     │
│  └────────────┘                     │
│         │                           │
│    직접 사용                          │
│         │                           │
│    ┌────▼────┐                      │
│    │  eth0   │                      │
│    │ (host)  │                      │
│    └─────────┘                      │
└─────────────────────────────────────┘

차이:
- Bridge: 컨테이너 → veth → bridge → host eth0
- Host: 컨테이너가 host eth0 직접 사용
```

#### Host 모드 확인

```bash
# 1. Bridge 모드 (기본)
docker run -d --name bridge-test nginx

# 네트워크 네임스페이스 확인
docker inspect bridge-test | grep NetworkMode
# "NetworkMode": "default"  (bridge)

# 컨테이너 내부 인터페이스
docker exec bridge-test ip addr
# 1: lo: <LOOPBACK,UP>
# 2: eth0@if123: <BROADCAST,MULTICAST,UP>
#     inet 172.17.0.2/16  ← 독립된 IP

# 호스트 인터페이스
ip addr show docker0
# docker0: ... inet 172.17.0.1/16

docker rm -f bridge-test

# 2. Host 모드
docker run -d --name host-test --network host nginx

# 네트워크 모드 확인
docker inspect host-test | grep NetworkMode
# "NetworkMode": "host"

# 컨테이너 내부 인터페이스
docker exec host-test ip addr
# 1: lo: <LOOPBACK,UP>
# 2: eth0: <BROADCAST,MULTICAST,UP>  ← 호스트와 동일!
#     inet 10.0.0.100/24
# 3: docker0: ...
# (호스트의 모든 인터페이스 보임)

docker rm -f host-test
```

---

### 2. 성능 비교

#### 지연시간 (Latency)

```bash
# 준비: 테스트 서버
mkdir host-network-test
cd host-network-test

cat > server.js << 'EOF'
const http = require('http');
const server = http.createServer((req, res) => {
  res.writeHead(200);
  res.end('OK');
});
server.listen(8080, '0.0.0.0');
console.log('Server running on port 8080');
EOF

cat > Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY server.js .
CMD ["node", "server.js"]
EOF

docker build -t test-server .

# 1. Bridge 모드 테스트
docker run -d --name bridge-server -p 8080:8080 test-server

# 성능 측정 (10000 requests)
ab -n 10000 -c 100 http://localhost:8080/

# 결과:
# Requests per second:    15234.12 [#/sec]
# Time per request:       6.564 [ms]

docker rm -f bridge-server

# 2. Host 모드 테스트
docker run -d --name host-server --network host test-server

# 성능 측정
ab -n 10000 -c 100 http://localhost:8080/

# 결과:
# Requests per second:    18542.35 [#/sec]  (+22%)
# Time per request:       5.393 [ms]        (-18%)

docker rm -f host-server

# 비교:
# Host 모드가 약 20-25% 빠름
# (veth, bridge, NAT 오버헤드 제거)
```

#### 처리량 (Throughput)

```bash
# iperf3로 네트워크 대역폭 측정

# 서버 (Host 모드)
docker run -d --name iperf-server-host \
  --network host \
  networkstatic/iperf3 -s

# 클라이언트 (Bridge 모드)
docker run --rm \
  networkstatic/iperf3 -c <HOST_IP> -t 10

# 결과:
# [ ID] Interval           Transfer     Bitrate
# [  5]   0.00-10.00  sec  8.32 GBytes  7.14 Gbits/sec

docker rm -f iperf-server-host

# 서버 (Bridge 모드)
docker run -d --name iperf-server-bridge \
  -p 5201:5201 \
  networkstatic/iperf3 -s

# 클라이언트
docker run --rm \
  networkstatic/iperf3 -c <HOST_IP> -t 10

# 결과:
# [ ID] Interval           Transfer     Bitrate
# [  5]   0.00-10.00  sec  7.89 GBytes  6.78 Gbits/sec

docker rm -f iperf-server-bridge

# 비교:
# Host: 7.14 Gbits/sec
# Bridge: 6.78 Gbits/sec
# Host가 약 5% 더 빠름
```

---

### 3. 포트 관리

#### 포트 충돌

```bash
# Host 모드에서는 포트 공유 불가

# 1. 첫 번째 컨테이너 (성공)
docker run -d --name web1 --network host nginx
# nginx는 80 포트 사용

# 2. 두 번째 컨테이너 (실패)
docker run -d --name web2 --network host nginx
# Error: Address already in use

# 호스트의 80 포트를 이미 web1이 사용 중

# 확인
docker logs web2
# nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

# 3. 포트를 변경해야 함
docker run -d --name web2 --network host \
  -e NGINX_PORT=8080 \
  nginx

# 또는 애플리케이션 설정으로 포트 변경

docker rm -f web1 web2
```

#### 포트 바인딩 불필요

```bash
# Bridge 모드: -p 필요
docker run -d --name bridge-nginx -p 8080:80 nginx
curl http://localhost:8080  # ✅ 성공

docker rm -f bridge-nginx

# Host 모드: -p 무시됨
docker run -d --name host-nginx --network host nginx

# 경고 발생:
# WARNING: Published ports are discarded when using host network mode

# 바로 접근 가능
curl http://localhost:80  # ✅ 성공
# 호스트의 80 포트에서 바로 리스닝

docker rm -f host-nginx
```

---

### 4. 보안 고려사항

#### 네트워크 격리 없음

```bash
# 1. Bridge 모드 (격리됨)
docker run -d --name isolated nginx

# 호스트의 다른 서비스 접근 불가
docker exec isolated curl http://localhost:22
# Connection refused or timeout

docker rm -f isolated

# 2. Host 모드 (격리 없음)
docker run -d --name exposed --network host nginx

# 호스트의 모든 서비스 접근 가능
docker exec exposed curl http://localhost:22
# SSH banner (위험!)

docker exec exposed netstat -tlnp
# 호스트의 모든 리스닝 포트 확인 가능

docker rm -f exposed

# 보안 위험:
# - 컨테이너가 손상되면 호스트 네트워크 노출
# - 호스트의 다른 서비스 공격 가능
# - 네트워크 스니핑 가능
```

#### 권한 에스컬레이션

```bash
# Host 모드 + 권한 있는 컨테이너
docker run -it --rm \
  --network host \
  --cap-add=NET_ADMIN \
  alpine sh

# 컨테이너 내부에서
ip link  # 호스트의 모든 인터페이스
ip link set eth0 down  # 호스트 네트워크 중단 가능!

# 매우 위험!
```

---

### 5. 사용 시나리오

#### 적합한 경우

```
✅ 네트워크 집약적 애플리케이션:
- 고성능 웹 서버 (초당 수만 요청)
- 실시간 스트리밍
- 고속 데이터 전송

✅ 포트 스캐너, 네트워크 모니터링:
- 모든 인터페이스 접근 필요
- 예: nmap, tcpdump, Prometheus node_exporter

✅ 멀티캐스트/브로드캐스트:
- mDNS, SSDP
- Bridge 모드에서 제한적

✅ 개발/테스트 환경:
- 빠른 반복 개발
- 로컬 서비스 접근
```

#### 부적합한 경우

```
❌ 프로덕션 웹 애플리케이션:
- 보안 위험
- 포트 관리 복잡
- Bridge로도 충분한 성능

❌ 멀티 테넌트 환경:
- 격리 필수
- Host 모드는 격리 없음

❌ 컨테이너 오케스트레이션:
- Kubernetes, Swarm
- 포트 충돌 문제

❌ 마이크로서비스:
- 서비스 간 격리 필요
- 복잡한 네트워크 토폴로지
```

---

## 💻 실습

### 실습 1: Bridge vs Host 성능 비교

#### 준비

```bash
# 테스트 이미지 빌드
mkdir perf-test
cd perf-test

cat > server.py << 'EOF'
from http.server import HTTPServer, BaseHTTPRequestHandler
import time

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/plain')
        self.end_headers()
        self.wfile.write(b'OK')
    
    def log_message(self, format, *args):
        pass  # 로그 비활성화 (성능 테스트용)

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 8080), Handler)
    print('Server started on port 8080')
    server.serve_forever()
EOF

cat > Dockerfile << 'EOF'
FROM python:3.11-alpine
WORKDIR /app
COPY server.py .
CMD ["python", "server.py"]
EOF

docker build -t perf-test .
```

#### Bridge 모드 테스트

```bash
# 서버 시작
docker run -d --name bridge-perf -p 8080:8080 perf-test

# ApacheBench로 테스트
ab -n 50000 -c 100 http://localhost:8080/

# 결과 기록:
# Requests per second:    12453.21 [#/sec]
# Time per request:       8.030 [ms]
# Transfer rate:          1234.56 [Kbytes/sec]

# 정리
docker rm -f bridge-perf
```

#### Host 모드 테스트

```bash
# 서버 시작
docker run -d --name host-perf --network host perf-test

# 동일한 테스트
ab -n 50000 -c 100 http://localhost:8080/

# 결과 기록:
# Requests per second:    15234.87 [#/sec]  (+22%)
# Time per request:       6.564 [ms]         (-18%)
# Transfer rate:          1512.34 [Kbytes/sec] (+22%)

# 정리
docker rm -f host-perf
```

#### 결과 비교

```
┌──────────────────┬────────────┬───────────┬─────────┐
│ 지표              │ Bridge     │ Host      │ 개선율    │
├──────────────────┼────────────┼───────────┼─────────┤
│ Requests/sec     │ 12,453     │ 15,234    │ +22%    │
├──────────────────┼────────────┼───────────┼─────────┤
│ Time/request (ms)│ 8.030      │ 6.564     │ -18%    │
├──────────────────┼────────────┼───────────┼─────────┤
│ Transfer (KB/s)  │ 1,234      │ 1,512     │ +22%    │
└──────────────────┴────────────┴───────────┴─────────┘

결론:
- Host 모드가 약 20-25% 빠름
- 네트워크 집약적일수록 차이 증가
```

---

### 실습 2: 포트 충돌 시나리오

#### 충돌 발생

```bash
# 1. 호스트에서 nginx 실행 (80 포트)
docker run -d --name nginx-host --network host nginx

# 확인
curl http://localhost:80
# Welcome to nginx!

# 2. 동일 포트로 두 번째 컨테이너 시도
docker run -d --name nginx-host2 --network host nginx

# 컨테이너는 시작되지만 nginx는 실패
docker ps -a
# nginx-host2   Up (unhealthy)

# 로그 확인
docker logs nginx-host2
# nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)

# 정리
docker rm -f nginx-host nginx-host2
```

#### 해결 방법

```bash
# 방법 1: 애플리케이션 포트 변경
docker run -d --name web1 --network host \
  nginx:alpine sh -c \
  "echo 'server { listen 8080; location / { return 200 \"Web1\"; } }' > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"

docker run -d --name web2 --network host \
  nginx:alpine sh -c \
  "echo 'server { listen 8081; location / { return 200 \"Web2\"; } }' > /etc/nginx/conf.d/default.conf && nginx -g 'daemon off;'"

# 각각 다른 포트로 접근
curl http://localhost:8080  # Web1
curl http://localhost:8081  # Web2

# 정리
docker rm -f web1 web2

# 방법 2: Bridge 모드 사용 (권장)
docker run -d --name web1 -p 8080:80 nginx
docker run -d --name web2 -p 8081:80 nginx

curl http://localhost:8080
curl http://localhost:8081

docker rm -f web1 web2
```

---

### 실습 3: 네트워크 모니터링 도구

#### Prometheus Node Exporter

```bash
# Host 모드로 실행 (호스트 메트릭 수집)
docker run -d \
  --name node-exporter \
  --network host \
  --pid host \
  -v /:/host:ro,rslave \
  prom/node-exporter \
  --path.rootfs=/host

# 메트릭 확인
curl http://localhost:9100/metrics | head -20

# 출력:
# # HELP node_cpu_seconds_total ...
# node_cpu_seconds_total{cpu="0",mode="idle"} 123456.78
# node_cpu_seconds_total{cpu="0",mode="system"} 1234.56
# ...

# Host 모드가 필요한 이유:
# - 호스트의 모든 인터페이스 메트릭 수집
# - /proc, /sys 접근
# - 정확한 시스템 모니터링

# 정리
docker rm -f node-exporter
```

#### tcpdump (패킷 캡처)

```bash
# Host 모드로 실행
docker run -it --rm \
  --network host \
  --cap-add=NET_ADMIN \
  nicolaka/netshoot \
  tcpdump -i eth0 -n port 80

# 다른 터미널에서 트래픽 생성
curl http://localhost:80

# tcpdump 출력:
# 14:32:45.123456 IP 127.0.0.1.54321 > 127.0.0.1.80: Flags [S], seq ...
# 14:32:45.123789 IP 127.0.0.1.80 > 127.0.0.1.54321: Flags [S.], seq ...

# Host 모드가 필요한 이유:
# - 호스트의 실제 인터페이스 접근
# - 모든 패킷 캡처
```

---

### 실습 4: 개발 환경에서 Host 모드

#### 로컬 서비스 접근

```bash
# 시나리오: 개발 중인 API가 로컬 Redis 사용

# 1. 로컬 Redis 실행 (호스트)
docker run -d --name redis-local -p 6379:6379 redis:alpine

# 2. API 서버 (Bridge 모드)
docker run -it --rm \
  -e REDIS_HOST=172.17.0.1 \
  node:18-alpine sh

# 코드에서 172.17.0.1:6379로 연결 (불편)

# 3. API 서버 (Host 모드)
docker run -it --rm \
  --network host \
  -e REDIS_HOST=localhost \
  node:18-alpine sh

# 코드에서 localhost:6379로 연결 (편리)
# 호스트와 동일한 네트워크 스택

# 정리
docker rm -f redis-local
```

---

## 🔥 실전 적용

### 시나리오 1: 고성능 웹 서버

**상황:**
- 초당 10만 요청 처리
- 지연시간 최소화 필요
- 단일 서버 환경

**솔루션: Host 네트워크**

```bash
# Nginx 역방향 프록시
docker run -d \
  --name nginx-proxy \
  --network host \
  --restart always \
  -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:alpine

# nginx.conf (샘플)
cat > /etc/nginx/nginx.conf << 'EOF'
events {
    worker_connections 10000;
}

http {
    upstream backend {
        server 127.0.0.1:8080;
        server 127.0.0.1:8081;
        server 127.0.0.1:8082;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://backend;
        }
    }
}
EOF

# 백엔드 서버들 (Host 모드)
for port in 8080 8081 8082; do
  docker run -d \
    --name backend-$port \
    --network host \
    -e PORT=$port \
    myapp:latest
done

# 장점:
# - 최소 지연시간
# - 최대 처리량
# - 포트 바인딩 간단
```

**결과:**
```
Bridge 모드:
- 처리량: 85,000 req/s
- 지연시간: 12ms (p95)

Host 모드:
- 처리량: 105,000 req/s (+23%)
- 지연시간: 9.5ms (p95) (-21%)
```

---

### 시나리오 2: 네트워크 모니터링 스택

**상황:**
- 호스트 및 컨테이너 모니터링
- 메트릭 수집 및 시각화
- 정확한 네트워크 통계 필요

**솔루션: Host 네트워크 모니터링**

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Node Exporter (Host 모드)
  node-exporter:
    image: prom/node-exporter:latest
    network_mode: host
    pid: host
    volumes:
      - /:/host:ro,rslave
    command:
      - '--path.rootfs=/host'
    restart: always

  # cAdvisor (호스트 컨테이너 메트릭)
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    network_mode: host
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: always

  # Prometheus (Bridge 모드)
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    restart: always

  # Grafana (Bridge 모드)
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    restart: always

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
  
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['localhost:8080']
```

```bash
docker-compose up -d

# 접근:
# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000
```

---

### 시나리오 3: 개발 환경 최적화

**상황:**
- 로컬 개발 환경
- 여러 마이크로서비스
- 빠른 반복 개발 필요

**솔루션: Host 네트워크 개발 스택**

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  # Frontend (Host 모드)
  frontend:
    build: ./frontend
    network_mode: host
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - API_URL=http://localhost:8080
    command: npm run dev
    # 포트 3000에서 실행

  # Backend API (Host 모드)
  api:
    build: ./api
    network_mode: host
    volumes:
      - ./api:/app
    environment:
      - DATABASE_URL=postgresql://localhost:5432/myapp
      - REDIS_URL=redis://localhost:6379
    command: npm run dev
    # 포트 8080에서 실행

  # Database (Bridge 모드, 포트 퍼블리시)
  postgres:
    image: postgres:14-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=devpass
    volumes:
      - postgres-data:/var/lib/postgresql/data

  # Redis (Bridge 모드, 포트 퍼블리시)
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

volumes:
  postgres-data:
```

**장점:**
```
- Hot reload 빠름
- localhost로 모든 서비스 접근
- IDE 디버거 연결 쉬움
- 포트 관리 간단
```

---

## ⚡ Host 네트워크 체크리스트

### 사용 결정

```
□ 성능이 정말 중요한가? (20%+ 차이)
□ 네트워크 격리가 필요 없는가?
□ 포트 충돌 관리 가능한가?
□ 보안 위험을 감수할 수 있는가?
□ 대안(Bridge)으로 불충분한가?
```

### 보안 강화

```
□ 컨테이너 이미지 신뢰 확인
□ Read-only 파일시스템
□ 최소 권한 실행 (non-root)
□ Capabilities 제한
□ Seccomp 프로파일 적용
```

### 운영

```
□ 포트 충돌 모니터링
□ 성능 측정 및 비교
□ 로깅 및 감사
□ 장애 격리 계획
□ 롤백 전략
```

### 문서화

```
□ Host 모드 사용 이유 명시
□ 포트 할당 문서화
□ 보안 예외 승인
□ 성능 벤치마크 기록
□ 대체 방안 검토
```

---

## 🚫 안티패턴

### 1. 무분별한 Host 모드 사용

```yaml
# ❌ 모든 서비스를 Host 모드로
services:
  web:
    network_mode: host
  api:
    network_mode: host
  db:
    network_mode: host
# 포트 충돌, 보안 위험

# ✅ 필요한 것만 Host 모드
services:
  web:
    ports:
      - "80:80"
  api:
    ports:
      - "8080:8080"
  db:
    # 포트 퍼블리시 안 함 (내부만)
```

### 2. 프로덕션에서 Host 모드

```bash
# ❌ 프로덕션 웹 앱에 Host 모드
docker run -d \
  --network host \
  --name production-app \
  myapp:latest
# 보안 위험, 격리 없음

# ✅ Bridge + 튜닝
docker run -d \
  -p 80:80 \
  --name production-app \
  --sysctl net.core.somaxconn=10000 \
  myapp:latest
# 충분한 성능 + 격리
```

### 3. Host 모드 + 높은 권한

```bash
# ❌ 매우 위험
docker run -it --rm \
  --network host \
  --privileged \
  alpine sh
# 호스트 완전 장악 가능

# ✅ 최소 권한
docker run -it --rm \
  --network host \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --read-only \
  alpine sh
```

### 4. 포트 하드코딩

```python
# ❌ 포트 하드코딩
app.run(host='0.0.0.0', port=80)
# Host 모드에서 충돌

# ✅ 환경변수로 포트 설정
import os
port = int(os.getenv('PORT', 8080))
app.run(host='0.0.0.0', port=port)
```

---

## 🎓 핵심 정리

### 1. Host 네트워크 특징

```
장점:
+ 최고 성능 (NAT 없음)
+ 지연시간 최소
+ 포트 바인딩 간단
+ 호스트 서비스 직접 접근

단점:
- 격리 없음 (보안 위험)
- 포트 충돌
- 멀티 테넌트 불가
- 오케스트레이션 제한
```

### 2. 성능 개선

```
네트워크 집약적 워크로드:
- 처리량: +20-25%
- 지연시간: -15-20%

일반 워크로드:
- 처리량: +5-10%
- 지연시간: -5-10%

체감 개선 작음
→ Bridge로도 충분한 경우 많음
```

### 3. 사용 가이드

```
Host 모드 사용:
✅ 네트워크 모니터링 도구
✅ 고성능 필수 (100k+ req/s)
✅ 멀티캐스트/브로드캐스트
✅ 개발 환경 (편의)

Bridge 모드 사용:
✅ 일반 웹 애플리케이션
✅ 마이크로서비스
✅ 멀티 테넌트
✅ 프로덕션 (권장)
```

### 4. 핵심 명령어

```bash
# Host 모드 실행
docker run --network host <image>

# 네트워크 모드 확인
docker inspect <container> | grep NetworkMode

# 성능 테스트
ab -n 10000 -c 100 <url>
iperf3 -c <host>

# 포트 사용 확인
netstat -tlnp
ss -tlnp
```

---

## 📚 참고 자료

- [Docker Host Networking](https://docs.docker.com/network/host/)
- [Network Performance](https://docs.docker.com/network/performance/)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Container Networking Benchmarks](https://www.kernel.org/doc/Documentation/networking/)

---

## 🤔 생각해볼 문제

1. Host 모드에서 컨테이너가 127.0.0.1로 바인딩하면 호스트에서 접근 가능할까?
2. Host 모드 컨테이너와 Bridge 모드 컨테이너가 통신할 수 있을까?
3. Host 모드에서도 성능 오버헤드가 있는 부분은?

> 💡 **답변**: 1) 가능 - Host 모드는 호스트의 네트워크 스택을 직접 사용하므로 127.0.0.1 바인딩은 호스트의 localhost와 동일, 호스트에서 localhost로 접근 가능, 2) 가능 - Host 모드 컨테이너는 호스트 IP를 사용하고, Bridge 컨테이너는 호스트 IP로 NAT되므로 통신 가능, 다만 Host 컨테이너에서 Bridge 컨테이너의 내부 IP(172.17.x.x)로는 직접 접근 불가, 호스트의 퍼블리시된 포트로 접근해야 함, 3) 파일시스템 I/O, 시스템콜 오버헤드(여전히 격리된 프로세스), CPU/메모리 제한(cgroups), 디스크 I/O 제한, 로깅 드라이버 - 네트워크만 Host 모드이고 나머지는 여전히 컨테이너

---

<div align="center">

**[⬅️ 이전: Bridge Network](./02-Bridge-Network.md)** | **[다음: Overlay Network ➡️](./04-Overlay-Network.md)**

</div>
