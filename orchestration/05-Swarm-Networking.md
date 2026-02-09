# 05. Swarm Networking - Swarm 네트워킹

## 🎯 이 챕터에서 배울 것

- **Ingress Network** 와 로드 밸런싱
- **Overlay Network** 와 서비스 메시
- **서비스 디스커버리** (DNS)
- **네트워크 격리**와 보안

## 📌 왜 중요한가?

**"Swarm 네트워킹은 멀티 호스트 서비스 간 통신과 로드 밸런싱의 핵심입니다."**

```
단일 호스트 vs Swarm 네트워킹:

단일 호스트 (docker-compose):
┌─────────────────────────────────────┐
│ Host                                │
│ ┌─────────┐  ┌─────────┐            │
│ │   Web   │←→│   DB    │            │
│ └─────────┘  └─────────┘            │
│ Bridge Network (localhost)          │
└─────────────────────────────────────┘
❌ 단일 호스트 제한
❌ 외부 로드 밸런서 필요
❌ 수동 서비스 디스커버리

Swarm 네트워킹:
┌─────────────────────────────────────┐
│ Ingress Network (모든 노드)           │
│ :80 → 자동 로드 밸런싱                  │
└────────────┬────────────────────────┘
             │
    ┌────────┼──────────┐
    │        │          │
┌───▼────┐ ┌─▼──────┐ ┌─▼──────┐
│ Web.1  │ │ Web.2  │ │ Web.3  │
│(node1) │ │(node2) │ │(node3) │
└───┬────┘ └───┬────┘ └───┬────┘
    │          │          │
┌───▼──────────▼──────────▼────┐
│ Overlay Network              │
│ (멀티 호스트)                   │
└───┬──────────────────────────┘
    │
┌───▼────┐
│   DB   │
│(node1) │
└────────┘

✅ 멀티 호스트 통신
✅ 내장 로드 밸런싱
✅ 자동 서비스 디스커버리
✅ 암호화 통신

Swarm 네트워킹의 핵심 가치:

1. Ingress 로드 밸런싱:
   - 어느 노드로 접속해도 가능
   - 자동 라운드 로빈
   - 포트 하나로 여러 레플리카

2. Overlay 네트워크:
   - 멀티 호스트 통신
   - VXLAN 터널링
   - 자동 암호화

3. 서비스 디스커버리:
   - DNS 기반
   - VIP (Virtual IP)
   - 자동 업데이트

4. 네트워크 격리:
   - 서비스별 네트워크
   - 보안 강화
   - 마이크로서비스 패턴

실무 시나리오:

External Request:
┌─────────────────────────────────────┐
│ Client                              │
│ http://any-node-ip:80               │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Ingress Network (Routing Mesh)      │
│ - 어느 노드든 접속 가능                  │
│ - 자동 로드 밸런싱                      │
└────────────┬────────────────────────┘
             │
    ┌────────┼──────────┐
    │        │          │
┌───▼────┐ ┌─▼──────┐ ┌─▼──────┐
│ LB     │ │ LB     │ │ LB     │
└───┬────┘ └───┬────┘ └───┬────┘
    │          │          │
┌───▼──────────▼──────────▼────┐
│ Overlay Network: frontend    │
└───┬──────────┬──────────┬────┘
┌───▼────┐ ┌───▼────┐ ┌───▼────┐
│ Web.1  │ │ Web.2  │ │ Web.3  │
└───┬────┘ └───┬────┘ └───┬────┘
    │          │          │
┌───▼──────────▼──────────▼─────┐
│ Overlay Network: backend      │
└───┬──────────┬────────────────┘
┌───▼────┐ ┌───▼────┐
│ API.1  │ │ API.2  │
└───┬────┘ └───┬────┘
    │          │
┌───▼──────────▼────────────────┐
│ Overlay Network: database     │
└───┬───────────────────────────┘
┌───▼────┐
│   DB   │
└────────┘
```

**실무 영향:**
- 무중단 스케일링
- 간편한 로드 밸런싱
- 보안 강화
- 마이크로서비스 지원

---

## 🔬 Deep Dive

### 1. Ingress Network

#### 개념

```
Ingress Network (Routing Mesh):

External Client:
┌─────────────────────────────────────┐
│ Client → node1:8080                 │
│ Client → node2:8080                 │
│ Client → node3:8080                 │
│ (어느 노드든 접속 가능)                  │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Ingress Network                     │
│ - 모든 노드가 포트 리슨                  │
│ - IPVS 로드 밸런서                     │
│ - 라운드 로빈                          │
└────────────┬────────────────────────┘
             │
    ┌────────┼──────────┐
    │        │          │
┌───▼────┐ ┌─▼──────┐ ┌─▼──────┐
│ Task.1 │ │ Task.2 │ │ Task.3 │
│(node1) │ │(node2) │ │(node1) │
└────────┘ └────────┘ └────────┘

특징:
1. 모든 노드가 포트 리슨
2. 태스크 없는 노드도 라우팅
3. IPVS 기반 L4 로드 밸런싱
4. 라운드 로빈 알고리즘
```

#### Ingress 사용

```bash
# Swarm 초기화
docker swarm init

# Ingress 모드 서비스 (기본)
docker service create \
  --name web \
  --replicas 3 \
  --publish 8080:80 \
  nginx:alpine

# 어느 노드에서든 접속 가능
curl http://localhost:8080
curl http://node1:8080
curl http://node2:8080
curl http://node3:8080
# 모두 동일하게 동작!

# Ingress 네트워크 확인
docker network inspect ingress

# 서비스 VIP 확인
docker service inspect web \
  --format '{{.Endpoint.VirtualIPs}}'
# [{ingress 10.0.0.2/24}]

# 정리
docker service rm web
```

#### Ingress 동작 원리

```bash
# 1. 서비스 생성
docker service create \
  --name web \
  --replicas 2 \
  --publish 8080:80 \
  nginx:alpine

# 2. iptables 규칙 확인
sudo iptables -t nat -L -n | grep 8080
# DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 tcp dpt:8080 to:10.0.0.2:80

# 3. IPVS 확인 (로드 밸런서)
sudo ipvsadm -Ln
# TCP  10.0.0.2:80 rr
#   -> 10.0.1.3:80   Masq
#   -> 10.0.1.4:80   Masq

# 4. 요청 흐름
# Client:8080 → iptables DNAT → VIP:80 → IPVS → Task:80

# 정리
docker service rm web
```

#### Host 모드 (Ingress 비활성화)

```bash
# Host 모드 (직접 바인딩)
docker service create \
  --name web \
  --publish mode=host,target=80,published=8080 \
  nginx:alpine

# 주의: 같은 노드에 여러 레플리카 불가
docker service scale web=3
# Error: bind address already in use

# 각 노드마다 1개씩만 가능
docker service create \
  --name web \
  --mode global \
  --publish mode=host,target=80,published=8080 \
  nginx:alpine

# 각 노드의 8080 포트에 직접 바인딩
# node1:8080 → node1의 task만
# node2:8080 → node2의 task만

# 정리
docker service rm web
```

---

### 2. Overlay Network

#### 개념

```
Overlay Network:

┌────────────────────────────────────┐
│ Node 1                             │
│ ┌─────────┐  ┌─────────┐           │
│ │Container│  │Container│           │
│ │10.0.9.2 │  │10.0.9.3 │           │
│ └────┬────┘  └────┬────┘           │
│      │            │                │
│      └─────┬──────┘                │
│      ┌─────▼────────┐              │
│      │ br0 (bridge) │              │
│      └─────┬────────┘              │
│      ┌─────▼────────┐              │
│      │ VXLAN Tunnel │              │
│      └─────┬────────┘              │
└────────────┼───────────────────────┘
             │ Underlay (eth0)
             │
┌────────────┼───────────────────────┐
│      ┌─────▼────────┐              │
│      │ VXLAN Tunnel │              │
│      └─────┬────────┘              │
│    ┌───────▼────────┐              │
│    │ br0 (bridge)   │              │
│    └───────┬────────┘              │
│      │            │                │
│      └─────┬──────┘                │
│ ┌────┴────┐  ┌────┴────┐           │
│ │Container│  │Container│           │
│ │10.0.9.4 │  │10.0.9.5 │           │
│ └─────────┘  └─────────┘           │
│ Node 2                             │
└────────────────────────────────────┘

특징:
- VXLAN (Virtual eXtensible LAN)
- 멀티 호스트 L2 네트워크
- 자동 암호화 (선택)
- VNI (VXLAN Network Identifier)
```

#### Overlay Network 생성

```bash
# Swarm 초기화
docker swarm init

# Overlay 네트워크 생성
docker network create \
  --driver overlay \
  my-overlay

# 확인
docker network ls
# NETWORK ID     NAME         DRIVER    SCOPE
# abc123...      ingress      overlay   swarm
# def456...      my-overlay   overlay   swarm

# 네트워크 상세
docker network inspect my-overlay

# 암호화 옵션
docker network create \
  --driver overlay \
  --opt encrypted=true \
  secure-overlay

# Subnet 지정
docker network create \
  --driver overlay \
  --subnet 10.0.9.0/24 \
  --gateway 10.0.9.1 \
  custom-overlay

# 정리
docker network rm my-overlay secure-overlay custom-overlay
```

#### 서비스에 Overlay 연결

```bash
# 네트워크 생성
docker network create --driver overlay frontend
docker network create --driver overlay backend

# 서비스 생성 (frontend)
docker service create \
  --name web \
  --network frontend \
  --replicas 3 \
  nginx:alpine

# 서비스 생성 (frontend + backend)
docker service create \
  --name api \
  --network frontend \
  --network backend \
  --replicas 3 \
  node:18-alpine

# 서비스 생성 (backend)
docker service create \
  --name db \
  --network backend \
  postgres:14-alpine

# 네트워크 확인
docker service inspect web --format '{{.Spec.TaskTemplate.Networks}}'

# 정리
docker service rm web api db
docker network rm frontend backend
```

---

### 3. 서비스 디스커버리

#### VIP (Virtual IP)

```bash
# Swarm 초기화
docker swarm init

# 네트워크 생성
docker network create --driver overlay app-net

# 서비스 생성
docker service create \
  --name web \
  --network app-net \
  --replicas 3 \
  nginx:alpine

# VIP 확인
docker service inspect web \
  --format '{{range .Endpoint.VirtualIPs}}{{.Addr}}{{end}}'
# 10.0.9.2/24

# 서비스 생성 (클라이언트)
docker service create \
  --name client \
  --network app-net \
  alpine sleep 3600

# DNS 확인
docker exec $(docker ps -q -f name=client) nslookup web
# Server:    127.0.0.11
# Address:   127.0.0.11:53
# 
# Name:      web
# Address:   10.0.9.2  ← VIP

# Ping 테스트
docker exec $(docker ps -q -f name=client) ping -c 3 web
# PING web (10.0.9.2): 56 data bytes
# 64 bytes from 10.0.9.2: seq=0 ttl=64 time=0.123 ms

# 정리
docker service rm web client
docker network rm app-net
```

#### DNS Round Robin (dnsrr)

```bash
# VIP 대신 DNS Round Robin
docker service create \
  --name web \
  --network app-net \
  --replicas 3 \
  --endpoint-mode dnsrr \
  nginx:alpine

# DNS 확인
docker exec $(docker ps -q -f name=client) nslookup web
# Name:      web
# Address:   10.0.9.3  ← Task 1
# Address:   10.0.9.4  ← Task 2
# Address:   10.0.9.5  ← Task 3
# (여러 IP 반환)

# 정리
docker service rm web
```

#### 서비스 이름으로 통신

```yaml
# stack.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    networks:
      - frontend
    deploy:
      replicas: 3

  api:
    image: node:18-alpine
    networks:
      - frontend
      - backend
    environment:
      # 서비스 이름으로 접근
      DB_HOST: database
      CACHE_HOST: redis
    deploy:
      replicas: 5

  database:
    image: postgres:14-alpine
    networks:
      - backend
    environment:
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:alpine
    networks:
      - backend

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
```

```bash
# 배포
docker stack deploy -c stack.yml myapp

# API 컨테이너에서 확인
API_CONTAINER=$(docker ps -q -f name=myapp_api)
docker exec $API_CONTAINER ping -c 1 database
# PING database (10.0.9.5): 56 data bytes

docker exec $API_CONTAINER nslookup redis
# Name:      redis
# Address:   10.0.9.6

# 정리
docker stack rm myapp
```

---

### 4. 네트워크 격리

#### 멀티 네트워크 전략

```bash
# Swarm 초기화
docker swarm init

# 네트워크 생성 (3개 tier)
docker network create --driver overlay public
docker network create --driver overlay internal
docker network create --driver overlay database

# Public tier (외부 접근)
docker service create \
  --name lb \
  --network public \
  --publish 80:80 \
  nginx:alpine

# Internal tier (중간)
docker service create \
  --name api \
  --network public \
  --network internal \
  node:18-alpine

# Database tier (격리)
docker service create \
  --name db \
  --network internal \
  postgres:14-alpine

# 확인: lb → api 가능
# lb → db 불가 (네트워크 다름)

# 정리
docker service rm lb api db
docker network rm public internal database
```

#### Internal 네트워크

```bash
# Internal 네트워크 (외부 접근 차단)
docker network create \
  --driver overlay \
  --internal \
  secure-backend

# 서비스 생성
docker service create \
  --name db \
  --network secure-backend \
  postgres:14-alpine

# 외부에서 접근 불가
# 같은 네트워크의 서비스만 접근 가능

# 정리
docker service rm db
docker network rm secure-backend
```

---

## 💻 실습

### 실습 1: Ingress 로드 밸런싱

```bash
# Swarm 초기화
docker swarm init

# 웹 서비스 생성 (3 replicas)
docker service create \
  --name web \
  --replicas 3 \
  --publish 8080:80 \
  nginx:alpine

# 각 태스크 확인
docker service ps web

# 로드 밸런싱 테스트
for i in {1..10}; do
  curl -s http://localhost:8080 | grep title
done
# 동일한 응답 (로드 밸런싱됨)

# 특정 태스크 중지
TASK_ID=$(docker service ps web -q | head -1)
CONTAINER_ID=$(docker inspect $TASK_ID --format '{{.Status.ContainerStatus.ContainerID}}')
docker stop $CONTAINER_ID

# 자동 복구 확인
docker service ps web
# 새 태스크 생성됨

# 계속 동작 확인
curl http://localhost:8080
# ✅ 정상 동작

# 정리
docker service rm web
docker swarm leave --force
```

---

### 실습 2: Overlay 네트워크

```bash
# Swarm 초기화
docker swarm init

# Overlay 네트워크 생성
docker network create \
  --driver overlay \
  --subnet 10.10.0.0/24 \
  test-overlay

# 서비스 생성 (서버)
docker service create \
  --name server \
  --network test-overlay \
  alpine sleep 3600

# 서비스 생성 (클라이언트)
docker service create \
  --name client \
  --network test-overlay \
  alpine sleep 3600

# 서버 IP 확인
SERVER_IP=$(docker exec $(docker ps -q -f name=server) \
  ip addr show eth0 | grep 'inet ' | awk '{print $2}' | cut -d'/' -f1)

echo "Server IP: $SERVER_IP"

# 클라이언트에서 서버 Ping
docker exec $(docker ps -q -f name=client) ping -c 3 server
# PING server (10.10.0.x): 56 data bytes
# ✅ 통신 성공

# DNS 확인
docker exec $(docker ps -q -f name=client) nslookup server
# Name:      server
# Address:   10.10.0.x

# 정리
docker service rm server client
docker network rm test-overlay
docker swarm leave --force
```

---

### 실습 3: 네트워크 격리

```bash
# Swarm 초기화
docker swarm init

# 네트워크 생성 (frontend, backend)
docker network create --driver overlay frontend
docker network create --driver overlay backend

# Frontend 서비스 (public 접근)
docker service create \
  --name web \
  --network frontend \
  --publish 8080:80 \
  nginx:alpine

# Middle tier (frontend + backend)
docker service create \
  --name api \
  --network frontend \
  --network backend \
  alpine sleep 3600

# Backend 서비스 (격리)
docker service create \
  --name db \
  --network backend \
  alpine sleep 3600

# 확인: api → db 가능
docker exec $(docker ps -q -f name=api) ping -c 1 db
# ✅ 성공

# 확인: web → db 불가
docker exec $(docker ps -q -f name=web) ping -c 1 db
# ❌ 실패 (네트워크 격리)

# 정리
docker service rm web api db
docker network rm frontend backend
docker swarm leave --force
```

---

## 🔥 실전 시나리오

### 시나리오 1: 3-Tier 네트워크 격리

```yaml
# stack.yml
version: '3.8'

services:
  # Tier 1: Public (External)
  nginx:
    image: nginx:alpine
    networks:
      - public
    ports:
      - "80:80"
      - "443:443"
    deploy:
      replicas: 3
      placement:
        constraints:
          - node.role==worker

  # Tier 2: Application (Internal)
  api:
    image: node:18-alpine
    networks:
      - public
      - application
    environment:
      DB_HOST: database
      REDIS_HOST: cache
    deploy:
      replicas: 5
      placement:
        constraints:
          - node.role==worker

  # Tier 3: Data (Backend)
  database:
    image: postgres:14-alpine
    networks:
      - application
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.type==database

  cache:
    image: redis:alpine
    networks:
      - application
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    deploy:
      replicas: 1

networks:
  # Public: 외부 접근 가능
  public:
    driver: overlay
  
  # Application: 내부 전용
  application:
    driver: overlay
    internal: true

volumes:
  db-data:
  redis-data:
```

```bash
# 배포
docker stack deploy -c stack.yml webapp

# 확인
docker stack services webapp
docker stack ps webapp

# 네트워크 확인
docker network ls --filter driver=overlay

# 테스트
curl http://localhost/
# ✅ Nginx 응답

# API에서 DB 접근 확인
API_CONTAINER=$(docker ps -q -f name=webapp_api | head -1)
docker exec $API_CONTAINER ping -c 1 database
# ✅ 성공

# Nginx에서 DB 접근 (불가)
NGINX_CONTAINER=$(docker ps -q -f name=webapp_nginx | head -1)
docker exec $NGINX_CONTAINER ping -c 1 database 2>&1
# ❌ 실패 (네트워크 격리)

# 정리
docker stack rm webapp
```

---

### 시나리오 2: 마이크로서비스 메시

```yaml
# microservices.yml
version: '3.8'

services:
  # API Gateway
  gateway:
    image: nginx:alpine
    networks:
      - frontend
    ports:
      - "80:80"
    configs:
      - source: nginx-config
        target: /etc/nginx/nginx.conf
    deploy:
      replicas: 2

  # User Service
  user-service:
    image: node:18-alpine
    networks:
      - frontend
      - user-backend
    environment:
      DB_HOST: user-db
    deploy:
      replicas: 3

  user-db:
    image: postgres:14-alpine
    networks:
      - user-backend
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - user-db-data:/var/lib/postgresql/data

  # Order Service
  order-service:
    image: node:18-alpine
    networks:
      - frontend
      - order-backend
    environment:
      DB_HOST: order-db
      USER_SERVICE: http://user-service:3000
    deploy:
      replicas: 3

  order-db:
    image: postgres:14-alpine
    networks:
      - order-backend
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - order-db-data:/var/lib/postgresql/data

  # Product Service
  product-service:
    image: node:18-alpine
    networks:
      - frontend
      - product-backend
    environment:
      DB_HOST: product-db
    deploy:
      replicas: 3

  product-db:
    image: mongo:6
    networks:
      - product-backend
    volumes:
      - product-db-data:/data/db

networks:
  frontend:
    driver: overlay
  user-backend:
    driver: overlay
    internal: true
  order-backend:
    driver: overlay
    internal: true
  product-backend:
    driver: overlay
    internal: true

volumes:
  user-db-data:
  order-db-data:
  product-db-data:

configs:
  nginx-config:
    file: ./nginx.conf
```

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream user-service {
        server user-service:3000;
    }

    upstream order-service {
        server order-service:3000;
    }

    upstream product-service {
        server product-service:3000;
    }

    server {
        listen 80;

        location /users {
            proxy_pass http://user-service;
        }

        location /orders {
            proxy_pass http://order-service;
        }

        location /products {
            proxy_pass http://product-service;
        }
    }
}
```

```bash
# 배포
docker stack deploy -c microservices.yml ecommerce

# 확인
docker stack services ecommerce
docker network ls --filter driver=overlay

# 각 서비스별 격리된 네트워크 확인
docker service inspect ecommerce_user-service \
  --format '{{range .Spec.TaskTemplate.Networks}}{{.Target}} {{end}}'
# frontend user-backend

docker service inspect ecommerce_user-db \
  --format '{{range .Spec.TaskTemplate.Networks}}{{.Target}} {{end}}'
# user-backend

# 정리
docker stack rm ecommerce
rm nginx.conf
```

---

## 🚫 안티패턴

### 1. 모든 서비스 같은 네트워크

```yaml
# ❌ 모든 서비스가 같은 네트워크
services:
  web:
    networks: [default]
  api:
    networks: [default]
  db:
    networks: [default]
# DB가 외부 노출 가능

# ✅ 네트워크 격리
services:
  web:
    networks: [frontend]
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]

networks:
  frontend:
  backend:
    internal: true
```

### 2. Ingress 포트 충돌

```bash
# ❌ 여러 서비스가 같은 포트
docker service create --name web1 --publish 80:80 nginx
docker service create --name web2 --publish 80:80 nginx
# Error: port already allocated

# ✅ 다른 포트 또는 같은 서비스로 통합
docker service create --name web --replicas 6 --publish 80:80 nginx
```

### 3. 암호화 없는 민감 데이터

```bash
# ❌ 평문 통신
docker network create --driver overlay backend

# ✅ 암호화 통신
docker network create \
  --driver overlay \
  --opt encrypted=true \
  secure-backend
```

---

## 🎓 핵심 정리

### 1. 네트워크 타입

```
Ingress:
- 모든 노드 포트 리슨
- 자동 로드 밸런싱
- IPVS 기반

Overlay:
- 멀티 호스트 통신
- VXLAN 터널
- 서비스 디스커버리
```

### 2. 서비스 디스커버리

```
VIP (기본):
- Virtual IP
- 자동 로드 밸런싱
- DNS → VIP → Tasks

DNS RR:
- Round Robin
- DNS → 여러 IP
- 클라이언트 선택
```

### 3. 네트워크 명령어

```bash
# 네트워크 생성
docker network create --driver overlay <n>

# 암호화
docker network create --driver overlay --opt encrypted=true <n>

# Internal
docker network create --driver overlay --internal <n>

# 서비스에 연결
docker service create --network <n> <image>
```

### 4. Best Practices

```
✅ 네트워크 격리 (3-tier)
✅ Internal 네트워크 (DB)
✅ 암호화 통신 (민감)
✅ VIP 기본 사용
✅ DNS 이름 활용
```

---

## 📚 참고 자료

- [Swarm mode overlay networking](https://docs.docker.com/network/overlay/)
- [Use overlay networks](https://docs.docker.com/network/network-tutorial-overlay/)
- [Swarm mode routing mesh](https://docs.docker.com/engine/swarm/ingress/)
- [VXLAN Protocol](https://datatracker.ietf.org/doc/html/rfc7348)
- [Service discovery](https://docs.docker.com/network/#service-discovery)

---

## 🤔 생각해볼 문제

1. Ingress 네트워크가 없다면 Swarm은 어떻게 로드 밸런싱을 구현할까?
2. Overlay 네트워크를 암호화하면 성능이 얼마나 저하될까?
3. VIP 방식과 DNSRR 방식, 어떤 경우에 각각 사용해야 할까?

> 💡 **답변**: 1) Ingress 없으면 외부 로드 밸런서 필요(Nginx, HAProxy, ALB 등), 각 노드 IP를 백엔드로 등록, 태스크 없는 노드 접속 시 실패, 수동 헬스체크 설정 필요, Ingress 장점: 자동 라우팅, 어느 노드든 접속 가능, 태스크 없어도 전달, 2) 암호화 오버헤드 약 10-15%, IPSec 사용, CPU 사용량 증가, 민감한 데이터(금융, 의료)는 필수, 내부 네트워크는 선택적, 성능 중요 시 애플리케이션 레벨 암호화(TLS) 고려, 3) VIP(기본): 대부분 경우 사용, 자동 로드 밸런싱, 클라이언트는 단일 IP만 봄, DNSRR: 레거시 애플리케이션(DNS 캐싱 문제), 클라이언트 측 로드 밸런싱 필요, Cassandra/Redis Cluster 같은 분산 시스템(모든 노드 IP 필요), Java RMI 등 특수 프로토콜

---

<div align="center">

**[⬅️ 이전: Swarm Services](./04-Swarm-Services.md)** | **[다음: Rolling Updates ➡️](./06-Rolling-Updates.md)**

</div>
