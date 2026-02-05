# 07. DNS Resolution - DNS 해석

## 🎯 이 챕터에서 배울 것

- Docker **내장 DNS 서버** 동작 원리
- 컨테이너 간 **서비스 디스커버리**
- DNS 기반 **로드 밸런싱**
- 외부 DNS 서버 **연동** 및 커스터마이징

## 📌 왜 중요한가?

**"DNS는 컨테이너 세계의 전화번호부입니다."**

```
IP 주소로 통신 (불편):
Container A → 172.17.0.3 (Container B)
- IP가 변경되면?
- 수동 관리 필요
- 확장성 없음

서비스 이름으로 통신 (편리):
Container A → "database" (자동 해석)
- IP 변경 무관
- 자동 디스커버리
- 로드 밸런싱

DNS 해석 과정:
┌─────────────┐
│ Container A │
└──────┬──────┘
       │ 1. "database" 조회
       ▼
┌──────────────────┐
│ Docker DNS       │
│ 127.0.0.11:53    │
└──────┬───────────┘
       │ 2. IP 반환 (172.17.0.3)
       ▼
┌─────────────┐
│ Container B │
│ 172.17.0.3  │
└─────────────┘
```

**실무 영향:**
- 개발: 환경별 설정 간소화 (dev/staging/prod)
- 운영: 동적 스케일링 지원
- 마이그레이션: IP 변경에 유연
- 마이크로서비스: 서비스 디스커버리 핵심

---

## 🔬 Deep Dive

### 1. Docker 내장 DNS

#### 기본 구조

```
Docker 내장 DNS:
- 모든 사용자 정의 네트워크에 자동 활성화
- 각 컨테이너: 127.0.0.11:53
- 기본 bridge 네트워크: DNS 없음 (레거시)

구조:
┌──────────────────────────────────────┐
│ Container                            │
│  ┌────────────────────────────────┐  │
│  │ /etc/resolv.conf               │  │
│  │ nameserver 127.0.0.11          │  │
│  │ options ndots:0                │  │
│  └────────────────────────────────┘  │
│                 │                    │
│                 ▼                    │
│  ┌────────────────────────────────┐  │
│  │ Docker Embedded DNS Resolver   │  │
│  │ 127.0.0.11:53                  │  │
│  └────────────────────────────────┘  │
└───────────────┬──────────────────────┘
                │
    ┌───────────▼───────────┐
    │ Docker DNS Database   │
    ├───────────────────────┤
    │ web     → 172.17.0.2  │
    │ db      → 172.17.0.3  │
    │ cache   → 172.17.0.4  │
    └───────────────────────┘
                │
    ┌───────────▼───────────┐
    │ Upstream DNS          │
    │ 8.8.8.8, 8.8.4.4      │
    └───────────────────────┘
```

#### DNS 설정 확인

```bash
# 네트워크 생성
docker network create mynet

# 컨테이너 시작
docker run -d --name web --network mynet nginx:alpine
docker run -d --name db --network mynet postgres:alpine

# DNS 설정 확인
docker exec web cat /etc/resolv.conf

# 출력:
# nameserver 127.0.0.11
# options ndots:0

# DNS 조회
docker exec web nslookup db

# 출력:
# Server:		127.0.0.11
# Address:	127.0.0.11:53
# 
# Non-authoritative answer:
# Name:	db
# Address: 172.18.0.3

# ping으로 확인
docker exec web ping -c 2 db
# PING db (172.18.0.3) 56(84) bytes of data.
# 64 bytes from db.mynet (172.18.0.3): icmp_seq=1 ttl=64 time=0.123 ms

# 정리
docker rm -f web db
docker network rm mynet
```

---

### 2. 컨테이너 이름 해석

#### 자동 DNS 등록

```bash
# 네트워크 생성
docker network create app-net

# 컨테이너 시작 (이름으로 자동 등록)
docker run -d --name frontend --network app-net nginx:alpine
docker run -d --name backend --network app-net nginx:alpine
docker run -d --name database --network app-net postgres:alpine

# 이름으로 통신
docker exec frontend ping -c 1 backend
# PING backend (172.19.0.3) ...
# ✅ 성공

docker exec frontend ping -c 1 database
# PING database (172.19.0.4) ...
# ✅ 성공

docker exec backend ping -c 1 database
# PING database (172.19.0.4) ...
# ✅ 성공

# DNS 조회
docker exec frontend nslookup backend
# Name:	backend
# Address: 172.19.0.3

docker exec frontend nslookup database
# Name:	database
# Address: 172.19.0.4

# 정리
docker rm -f frontend backend database
docker network rm app-net
```

#### 네트워크 별칭

```bash
# 네트워크 생성
docker network create web-net

# 별칭 설정
docker run -d \
  --name web1 \
  --network web-net \
  --network-alias web \
  nginx:alpine

docker run -d \
  --name web2 \
  --network web-net \
  --network-alias web \
  nginx:alpine

# DNS 조회 (라운드 로빈)
docker run --rm --network web-net alpine nslookup web

# 출력:
# Name:	web
# Address: 172.20.0.2
# Name:	web
# Address: 172.20.0.3

# 두 IP가 모두 반환됨!

# 연결 테스트
docker run --rm --network web-net curlimages/curl curl web

# web1 또는 web2에 랜덤하게 연결

# 정리
docker rm -f web1 web2
docker network rm web-net
```

---

### 3. 서비스 디스커버리

#### 동적 서비스 등록

```bash
# 네트워크 생성
docker network create service-net

# 서비스 시작
docker run -d \
  --name api-v1 \
  --network service-net \
  --network-alias api \
  myapp:v1

# API 호출
docker run --rm --network service-net curlimages/curl \
  curl http://api/health
# ✅ api-v1 응답

# 새 버전 추가 (무중단 배포)
docker run -d \
  --name api-v2 \
  --network service-net \
  --network-alias api \
  myapp:v2

# DNS 조회
docker run --rm --network service-net alpine nslookup api
# Name:	api
# Address: 172.21.0.2  (api-v1)
# Address: 172.21.0.3  (api-v2)

# 트래픽이 두 버전으로 분산!

# 구버전 제거
docker rm -f api-v1

# DNS 조회
docker run --rm --network service-net alpine nslookup api
# Name:	api
# Address: 172.21.0.3  (api-v2만)

# 정리
docker rm -f api-v2
docker network rm service-net
```

#### 여러 네트워크 연결

```bash
# 네트워크 2개 생성
docker network create frontend-net
docker network create backend-net

# API Gateway (양쪽 네트워크)
docker run -d \
  --name gateway \
  --network frontend-net \
  nginx:alpine

docker network connect backend-net gateway

# Frontend (frontend-net)
docker run -d \
  --name web \
  --network frontend-net \
  nginx:alpine

# Backend (backend-net)
docker run -d \
  --name db \
  --network backend-net \
  postgres:alpine

# Web → Gateway (가능)
docker exec web ping -c 1 gateway
# ✅ 성공

# Gateway → DB (가능)
docker exec gateway ping -c 1 db
# ✅ 성공

# Web → DB (불가능)
docker exec web ping -c 1 db
# ❌ 실패 (다른 네트워크)

# DNS 확인
docker exec web nslookup gateway
# Address: 172.22.0.2

docker exec gateway nslookup db
# Address: 172.23.0.2

# 정리
docker rm -f web gateway db
docker network rm frontend-net backend-net
```

---

### 4. 외부 DNS 설정

#### 커스텀 DNS 서버

```bash
# 방법 1: 개별 컨테이너
docker run -d \
  --name web \
  --dns 1.1.1.1 \
  --dns 8.8.8.8 \
  nginx:alpine

# 확인
docker exec web cat /etc/resolv.conf
# nameserver 127.0.0.11  ← Docker DNS (우선)
# nameserver 1.1.1.1     ← 커스텀
# nameserver 8.8.8.8

docker rm -f web

# 방법 2: Docker daemon 전체
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "dns": ["1.1.1.1", "8.8.8.8"]
}
EOF

sudo systemctl restart docker

# 모든 컨테이너가 이 DNS 사용

# 방법 3: 네트워크별
docker network create \
  --dns 1.1.1.1 \
  custom-dns-net

docker run -d --network custom-dns-net nginx

# 해당 네트워크의 컨테이너만 적용
```

#### DNS 검색 도메인

```bash
# 검색 도메인 설정
docker run -d \
  --name app \
  --dns-search example.com \
  --dns-search internal.local \
  nginx:alpine

# 확인
docker exec app cat /etc/resolv.conf
# nameserver 127.0.0.11
# search example.com internal.local

# 사용
docker exec app ping -c 1 server
# server.example.com 으로 자동 해석
# 없으면 server.internal.local 시도

docker rm -f app
```

---

### 5. DNS 라운드 로빈

#### 로드 밸런싱

```bash
# 네트워크 생성
docker network create lb-net

# 백엔드 서버 3대
for i in {1..3}; do
  docker run -d \
    --name backend-$i \
    --network lb-net \
    --network-alias backend \
    nginx:alpine
done

# DNS 조회
docker run --rm --network lb-net alpine nslookup backend

# 출력:
# Name:	backend
# Address: 172.24.0.2
# Address: 172.24.0.3
# Address: 172.24.0.4

# 부하 테스트
docker run --rm --network lb-net alpine sh -c '
  for i in $(seq 1 10); do
    wget -qO- http://backend/ | grep -o "Server: backend-[0-9]"
  done
'

# 결과:
# Server: backend-1
# Server: backend-2
# Server: backend-3
# Server: backend-1
# Server: backend-2
# ...
# (라운드 로빈 분산)

# 정리
docker rm -f backend-1 backend-2 backend-3
docker network rm lb-net
```

---

## 💻 실습

### 실습 1: 마이크로서비스 DNS

#### 서비스 구성

```bash
# 네트워크 생성
docker network create microservices

# 데이터베이스
docker run -d \
  --name postgres \
  --network microservices \
  --network-alias db \
  -e POSTGRES_PASSWORD=secret \
  postgres:alpine

# Redis
docker run -d \
  --name redis \
  --network microservices \
  --network-alias cache \
  redis:alpine

# API 서버
docker run -d \
  --name api \
  --network microservices \
  --network-alias api \
  -e DATABASE_URL=postgresql://postgres:secret@db:5432/myapp \
  -e REDIS_URL=redis://cache:6379 \
  nginx:alpine

# Web 프론트엔드
docker run -d \
  --name web \
  --network microservices \
  -p 8080:80 \
  -e API_URL=http://api:80 \
  nginx:alpine
```

#### DNS 검증

```bash
# Web에서 API 접근
docker exec web nslookup api
# Name:	api
# Address: 172.25.0.3

docker exec web wget -qO- http://api
# ✅ 성공

# API에서 DB 접근
docker exec api nslookup db
# Name:	db
# Address: 172.25.0.2

docker exec api nc -zv db 5432
# db (172.25.0.2:5432) open
# ✅ 성공

# API에서 Cache 접근
docker exec api nslookup cache
# Name:	cache
# Address: 172.25.0.4

docker exec api nc -zv cache 6379
# cache (172.25.0.4:6379) open
# ✅ 성공
```

#### 동적 확장

```bash
# API 서버 추가 (스케일 아웃)
docker run -d \
  --name api-2 \
  --network microservices \
  --network-alias api \
  nginx:alpine

docker run -d \
  --name api-3 \
  --network microservices \
  --network-alias api \
  nginx:alpine

# DNS 조회
docker exec web nslookup api
# Name:	api
# Address: 172.25.0.3
# Address: 172.25.0.5
# Address: 172.25.0.6

# 부하 분산 확인
docker exec web sh -c '
  for i in $(seq 1 9); do
    wget -qO- http://api | head -1
  done
'

# 정리
docker rm -f web api api-2 api-3 redis postgres
docker network rm microservices
```

---

### 실습 2: Blue-Green 배포

#### Blue 환경 (현재)

```bash
# 네트워크
docker network create prod-net

# Blue 버전
docker run -d \
  --name app-blue-1 \
  --network prod-net \
  --network-alias app \
  nginx:1.20-alpine

docker run -d \
  --name app-blue-2 \
  --network prod-net \
  --network-alias app \
  nginx:1.20-alpine

# DNS 확인
docker run --rm --network prod-net alpine nslookup app
# Name:	app
# Address: 172.26.0.2  (blue-1)
# Address: 172.26.0.3  (blue-2)
```

#### Green 환경 (새 버전)

```bash
# Green 버전 (별칭 없이 시작)
docker run -d \
  --name app-green-1 \
  --network prod-net \
  nginx:1.21-alpine

docker run -d \
  --name app-green-2 \
  --network prod-net \
  nginx:1.21-alpine

# Green 테스트 (이름으로 직접)
docker run --rm --network prod-net curlimages/curl \
  curl http://app-green-1
# ✅ 테스트 성공
```

#### 트래픽 전환

```bash
# Blue → Green 전환
# 1. Blue 별칭 제거
docker network disconnect prod-net app-blue-1
docker network disconnect prod-net app-blue-2

# 2. Green 별칭 추가
docker network connect prod-net app-green-1 --alias app
docker network connect prod-net app-green-2 --alias app

# DNS 확인
docker run --rm --network prod-net alpine nslookup app
# Name:	app
# Address: 172.26.0.4  (green-1)
# Address: 172.26.0.5  (green-2)

# 즉시 전환 완료!

# 정리
docker rm -f app-blue-1 app-blue-2 app-green-1 app-green-2
docker network rm prod-net
```

---

### 실습 3: DNS 디버깅

#### DNS 조회 도구

```bash
# 네트워크 생성
docker network create debug-net

# 서비스
docker run -d --name service1 --network debug-net nginx
docker run -d --name service2 --network debug-net nginx

# 디버그 컨테이너
docker run -it --rm \
  --network debug-net \
  nicolaka/netshoot

# 컨테이너 내부에서 DNS 테스트
```

```bash
# 1. nslookup
nslookup service1
# Name:	service1
# Address: 172.27.0.2

# 2. dig (상세 정보)
dig service1

# 출력:
# ;; ANSWER SECTION:
# service1.		600	IN	A	172.27.0.2

# 3. host
host service1
# service1 has address 172.27.0.2

# 4. DNS 서버 확인
cat /etc/resolv.conf
# nameserver 127.0.0.11

# 5. DNS 캐시 확인
dig service1 +noall +stats
# Query time: 0 msec  (캐시됨)

# 6. DNS 추적
dig service1 +trace

# 7. 역방향 조회
dig -x 172.27.0.2

# 8. 모든 레코드
dig service1 ANY
```

#### DNS 문제 해결

```bash
# 문제 1: DNS 해석 실패
docker exec service1 ping service2
# ping: bad address 'service2'

# 원인: 기본 bridge 네트워크 (DNS 없음)
docker inspect service1 | grep NetworkMode
# "NetworkMode": "default"

# 해결: 사용자 정의 네트워크 사용
docker network create custom-net
docker network connect custom-net service1
docker network connect custom-net service2

docker exec service1 ping -c 1 service2
# ✅ 성공!

# 문제 2: 외부 DNS 실패
docker exec service1 ping google.com
# timeout

# 원인: DNS 서버 설정 문제
docker exec service1 cat /etc/resolv.conf
# nameserver 127.0.0.11
# nameserver 0.0.0.0  ← 잘못된 설정

# 해결: 올바른 DNS 설정
docker run -d --name fixed \
  --network custom-net \
  --dns 8.8.8.8 \
  nginx

docker exec fixed ping -c 1 google.com
# ✅ 성공!

# 정리
docker rm -f service1 service2 fixed
docker network rm debug-net custom-net
```

---

### 실습 4: Docker Compose DNS

#### docker-compose.yml

```yaml
version: '3.8'

services:
  # Frontend
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    networks:
      - frontend
    depends_on:
      - api
    environment:
      - API_URL=http://api:8080

  # API Gateway
  api:
    image: nginx:alpine
    networks:
      - frontend
      - backend
    depends_on:
      - service1
      - service2
    environment:
      - SERVICE1_URL=http://service1:8080
      - SERVICE2_URL=http://service2:8080

  # Backend Services
  service1:
    image: nginx:alpine
    networks:
      backend:
        aliases:
          - svc1
          - backend-service
    deploy:
      replicas: 2

  service2:
    image: nginx:alpine
    networks:
      backend:
        aliases:
          - svc2
          - backend-service
    deploy:
      replicas: 2

  # Database
  database:
    image: postgres:14-alpine
    networks:
      - backend
    environment:
      - POSTGRES_PASSWORD=secret

  # Cache
  redis:
    image: redis:alpine
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # 외부 접근 차단
```

#### DNS 동작 확인

```bash
# 서비스 시작
docker-compose up -d

# Web → API
docker-compose exec web nslookup api
# Name:	api
# Address: 172.28.0.3

# API → Service1 (별칭)
docker-compose exec api nslookup backend-service
# Name:	backend-service
# Address: 172.29.0.2
# Address: 172.29.0.3
# Address: 172.29.0.4
# Address: 172.29.0.5

# Service1 → Database
docker-compose exec service1 nslookup database
# Name:	database
# Address: 172.29.0.6

# 연결 테스트
docker-compose exec web wget -qO- http://api
# ✅ 성공

docker-compose exec api wget -qO- http://backend-service
# ✅ 성공 (라운드 로빈)

# 정리
docker-compose down
```

---

## 🔥 실전 적용

### 시나리오 1: 다단계 애플리케이션

**구조:**
```
Internet → Nginx (Reverse Proxy)
           ↓
       Web Server (3대)
           ↓
       API Gateway (2대)
           ↓
    ┌──────┴──────┬──────────┐
    ↓             ↓          ↓
 Auth Service  Order Service  Payment Service
    ↓             ↓          ↓
    └──────┬──────┴──────────┘
           ↓
      PostgreSQL + Redis
```

**docker-compose.yml:**

```yaml
version: '3.8'

networks:
  public:
  app:
  data:
    internal: true

services:
  # Reverse Proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    networks:
      - public
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    environment:
      - BACKEND=http://web:80

  # Web Tier
  web:
    image: myapp/web:latest
    deploy:
      replicas: 3
    networks:
      public:
        aliases:
          - web
      app:
        aliases:
          - web
    environment:
      - API_URL=http://gateway:8080

  # API Gateway
  gateway:
    image: myapp/gateway:latest
    deploy:
      replicas: 2
    networks:
      app:
        aliases:
          - gateway
    environment:
      - AUTH_URL=http://auth:8080
      - ORDER_URL=http://order:8080
      - PAYMENT_URL=http://payment:8080

  # Microservices
  auth:
    image: myapp/auth:latest
    deploy:
      replicas: 2
    networks:
      - app
      - data
    environment:
      - DB_URL=postgresql://postgres:secret@db:5432/auth
      - REDIS_URL=redis://cache:6379

  order:
    image: myapp/order:latest
    deploy:
      replicas: 3
    networks:
      - app
      - data
    environment:
      - DB_URL=postgresql://postgres:secret@db:5432/order
      - REDIS_URL=redis://cache:6379

  payment:
    image: myapp/payment:latest
    deploy:
      replicas: 2
    networks:
      - app
      - data
    environment:
      - DB_URL=postgresql://postgres:secret@db:5432/payment

  # Data Tier
  db:
    image: postgres:14-alpine
    networks:
      - data
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - db-data:/var/lib/postgresql/data

  cache:
    image: redis:alpine
    networks:
      - data

volumes:
  db-data:
```

**DNS 장점:**
```
- 서비스 이름으로 통신 (IP 무관)
- 자동 로드 밸런싱 (replicas)
- 네트워크 격리 (public/app/data)
- 무중단 배포 가능
```

---

### 시나리오 2: 환경별 DNS 설정

**개발 환경:**

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    image: myapp:dev
    dns:
      - 8.8.8.8
    dns_search:
      - dev.example.com
      - local
    environment:
      - DB_HOST=db.dev.example.com
      - API_URL=http://api:3000
    networks:
      - dev-net

  db:
    image: postgres:14-alpine
    networks:
      dev-net:
        aliases:
          - db.dev.example.com

networks:
  dev-net:
```

**스테이징 환경:**

```yaml
# docker-compose.staging.yml
version: '3.8'

services:
  app:
    image: myapp:staging
    dns:
      - 1.1.1.1
    dns_search:
      - staging.example.com
    environment:
      - DB_HOST=db.staging.example.com
      - API_URL=http://api:8080
    networks:
      - staging-net

  db:
    image: postgres:14-alpine
    networks:
      staging-net:
        aliases:
          - db.staging.example.com

networks:
  staging-net:
```

**프로덕션 환경:**

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    image: myapp:latest
    dns:
      - 1.1.1.1
      - 1.0.0.1
    dns_search:
      - example.com
    environment:
      - DB_HOST=db.example.com
      - API_URL=https://api.example.com
    networks:
      - prod-net

  db:
    image: postgres:14-alpine
    networks:
      prod-net:
        aliases:
          - db.example.com

networks:
  prod-net:
    driver: overlay
```

---

## ⚡ DNS 체크리스트

### 네트워크 설계

```
□ 사용자 정의 네트워크 사용 (기본 bridge 피하기)
□ 네트워크 별칭 전략 수립
□ DNS 검색 도메인 정의
□ 네트워크 격리 계획
□ 외부 DNS 서버 선정
```

### 서비스 명명

```
□ 명확한 서비스 이름 (frontend, backend, db)
□ 환경별 접미사 (dev, staging, prod)
□ 버전 관리 (v1, v2)
□ 역할 기반 별칭 (api, cache, queue)
□ 도메인 규칙 준수
```

### 성능 최적화

```
□ DNS 캐시 활용
□ ndots 설정 최적화
□ 불필요한 검색 도메인 제거
□ TTL 적절히 설정
□ 라운드 로빈 부하 분산
```

### 모니터링

```
□ DNS 쿼리 로깅
□ 해석 실패 추적
□ 응답 시간 측정
□ 캐시 히트율 확인
□ 외부 DNS 가용성 체크
```

---

## 🚫 안티패턴

### 1. 기본 bridge 사용

```bash
# ❌ 기본 bridge (DNS 없음)
docker run -d --name app1 nginx
docker run -d --name app2 nginx

docker exec app1 ping app2
# ping: bad address 'app2'

# ✅ 사용자 정의 네트워크
docker network create mynet
docker run -d --name app1 --network mynet nginx
docker run -d --name app2 --network mynet nginx

docker exec app1 ping app2
# ✅ 성공!
```

### 2. IP 하드코딩

```yaml
# ❌ IP 하드코딩
services:
  app:
    environment:
      - DB_HOST=172.18.0.3  # IP 변경 시 문제

# ✅ 서비스 이름 사용
services:
  app:
    environment:
      - DB_HOST=database
```

### 3. 링크 사용 (레거시)

```bash
# ❌ --link (deprecated)
docker run -d --name db postgres
docker run -d --name app --link db nginx

# ✅ 네트워크 사용
docker network create mynet
docker run -d --name db --network mynet postgres
docker run -d --name app --network mynet nginx
```

### 4. DNS 설정 누락

```yaml
# ❌ 외부 DNS 미설정
services:
  app:
    image: myapp
# 외부 도메인 해석 실패 가능

# ✅ 명시적 DNS 설정
services:
  app:
    image: myapp
    dns:
      - 1.1.1.1
      - 8.8.8.8
```

---

## 🎓 핵심 정리

### 1. Docker DNS 구조

```
내장 DNS:
- 127.0.0.11:53
- 사용자 정의 네트워크에서 자동 활성화
- 컨테이너 이름 자동 등록
- 외부 DNS로 폴백

설정 파일:
- /etc/resolv.conf
- nameserver, search, options
```

### 2. 서비스 디스커버리

```
자동 등록:
- 컨테이너 이름
- 네트워크 별칭 (--network-alias)
- Compose 서비스 이름

해석:
- 단일 IP (컨테이너 1개)
- 여러 IP (라운드 로빈)
- 네트워크별 격리
```

### 3. DNS 라운드 로빈

```
동작:
- 같은 별칭의 여러 컨테이너
- DNS가 모든 IP 반환
- 클라이언트가 선택

장점:
- 간단한 로드 밸런싱
- 자동 장애 복구
- 무중단 배포
```

### 4. 핵심 명령어

```bash
# DNS 확인
docker exec <container> cat /etc/resolv.conf
docker exec <container> nslookup <service>
docker exec <container> dig <service>

# 네트워크 별칭
docker run --network-alias <alias>
docker network connect --alias <alias>

# DNS 설정
docker run --dns <server>
docker run --dns-search <domain>

# Compose
docker-compose exec <service> nslookup <target>
```

---

## 📚 참고 자료

- [Docker Embedded DNS](https://docs.docker.com/config/containers/container-networking/#dns-services)
- [Docker Network Aliases](https://docs.docker.com/engine/reference/commandline/network_connect/#add-a-container-to-a-network)
- [Compose Networking](https://docs.docker.com/compose/networking/)
- [DNS Resolution](https://www.ietf.org/rfc/rfc1035.txt)

---

## 🤔 생각해볼 문제

1. Docker DNS가 127.0.0.11을 사용하는 이유는?
2. 네트워크 별칭과 컨테이너 이름의 차이는?
3. DNS 라운드 로빈의 한계는 무엇인가?

> 💡 **답변**: 1) 루프백 주소(127.0.0.0/8)로 컨테이너 내부에서만 접근 가능, 다른 컨테이너와 충돌하지 않음, 각 컨테이너가 독립된 네트워크 네임스페이스에서 동일한 127.0.0.11 사용 가능, 호스트나 다른 컨테이너에서 접근 불가로 보안 강화, 2) 컨테이너 이름은 고유해야 하지만(전역), 별칭은 같은 네트워크 내에서 여러 컨테이너가 공유 가능(로컬), 별칭으로 라운드 로빈 로드 밸런싱 가능, 하나의 컨테이너가 여러 별칭 가질 수 있음, 3) 클라이언트 캐싱으로 불균등 분산, 헬스 체크 없음(죽은 컨테이너도 반환), 세션 유지 불가(Sticky Session), 가중치 설정 불가, TCP 연결 수준 분산 불가(DNS 레벨만), 실시간 변경 반영 지연 - 프로덕션에서는 별도 로드 밸런서 권장

---

<div align="center">

**[⬅️ 이전: Custom Networks](./06-Custom-Networks.md)** | **[다음: Load Balancing ➡️](./08-Load-Balancing.md)**

</div>
