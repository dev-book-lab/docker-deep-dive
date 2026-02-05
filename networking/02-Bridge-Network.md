# 02. Bridge Network - 브리지 네트워크

## 🎯 이 챕터에서 배울 것

- Docker **기본 브리지** 네트워크의 동작 원리
- **사용자 정의 브리지**의 장점과 사용법
- **내장 DNS** 서비스와 서비스 디스커버리
- 네트워크 **격리**와 **통신 제어**

## 📌 왜 중요한가?

**"브리지 네트워크는 Docker 컨테이너 네트워킹의 기본이자 핵심입니다."**

```
기본 브리지 vs 사용자 정의 브리지:

기본 브리지 (docker0):
- IP 주소로만 통신
- 모든 컨테이너가 접근 가능
- 제한적인 기능
- 레거시 방식

사용자 정의 브리지:
- 컨테이너 이름으로 통신 ✅
- 네트워크 격리 ✅
- 고급 설정 가능 ✅
- 권장 방식 ✅
```

**실무 영향:**
- 서비스 디스커버리: 이름 기반 자동 연결
- 보안: 네트워크 레벨 격리
- 유지보수: 명확한 네트워크 구조
- 확장성: 쉬운 컨테이너 추가/제거

---

## 🔬 Deep Dive

### 1. 기본 브리지 네트워크 (docker0)

#### 특징

```
docker0 브리지:
- Docker 설치 시 자동 생성
- 기본 네트워크 (--network 미지정 시)
- 172.17.0.0/16 대역 (기본값)
- 모든 컨테이너가 공유

구조:
┌─────────────────────────────────────────┐
│ Host                                    │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ Cont A  │  │ Cont B  │  │ Cont C  │  │
│  │.17.0.2  │  │.17.0.3  │  │.17.0.4  │  │
│  └────┬────┘  └────┬────┘  └────┬────┘  │
│       └────────┬────────────────┘       │
│                │                        │
│          ┌─────▼──────┐                 │
│          │  docker0   │                 │
│          │ 172.17.0.1 │                 │
│          └────────────┘                 │
└─────────────────────────────────────────┘
```

#### 기본 브리지 확인

```bash
# docker0 확인
ip addr show docker0
# 4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet 172.17.0.1/16 scope global docker0

# 브리지 정보
docker network inspect bridge

# 출력:
# {
#     "Name": "bridge",
#     "Driver": "bridge",
#     "IPAM": {
#         "Config": [
#             {
#                 "Subnet": "172.17.0.0/16",
#                 "Gateway": "172.17.0.1"
#             }
#         ]
#     },
#     "Containers": {}
# }
```

#### 기본 브리지 사용

```bash
# 컨테이너 시작 (기본 브리지)
docker run -d --name web1 nginx
docker run -d --name web2 nginx

# IP 확인
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web1
# 172.17.0.2

docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web2
# 172.17.0.3

# IP로 통신 (성공)
docker exec web1 ping -c 2 172.17.0.3
# PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
# 64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.123 ms

# ❌ 이름으로 통신 (실패)
docker exec web1 ping -c 2 web2
# ping: web2: Name or service not known

# 정리
docker rm -f web1 web2
```

---

### 2. 사용자 정의 브리지 네트워크

#### 생성 및 기본 사용

```bash
# 사용자 정의 브리지 생성
docker network create mynet

# 네트워크 확인
docker network ls
# NETWORK ID     NAME      DRIVER    SCOPE
# abc123...      bridge    bridge    local
# def456...      mynet     bridge    local

# 상세 정보
docker network inspect mynet

# 출력:
# {
#     "Name": "mynet",
#     "Driver": "bridge",
#     "IPAM": {
#         "Config": [
#             {
#                 "Subnet": "172.18.0.0/16",
#                 "Gateway": "172.18.0.1"
#             }
#         ]
#     }
# }

# 컨테이너를 사용자 정의 네트워크에 연결
docker run -d --name app1 --network mynet nginx
docker run -d --name app2 --network mynet nginx

# ✅ 이름으로 통신 (성공!)
docker exec app1 ping -c 2 app2
# PING app2 (172.18.0.3) 56(84) bytes of data.
# 64 bytes from app2.mynet (172.18.0.3): icmp_seq=1 ...

# DNS 조회
docker exec app1 nslookup app2
# Server:		127.0.0.11
# Address:	127.0.0.11#53
# 
# Non-authoritative answer:
# Name:	app2
# Address: 172.18.0.3

# 정리
docker rm -f app1 app2
docker network rm mynet
```

#### 고급 옵션으로 생성

```bash
# 서브넷, 게이트웨이, IP 범위 지정
docker network create \
  --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  --ip-range 192.168.100.128/25 \
  custom-net

# 설정 확인
docker network inspect custom-net | grep -A 5 IPAM
# "IPAM": {
#     "Config": [
#         {
#             "Subnet": "192.168.100.0/24",
#             "IPRange": "192.168.100.128/25",
#             "Gateway": "192.168.100.1"
#         }
#     ]
# }

# 컨테이너에 고정 IP 할당
docker run -d \
  --name web \
  --network custom-net \
  --ip 192.168.100.150 \
  nginx

# IP 확인
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web
# 192.168.100.150

# 정리
docker rm -f web
docker network rm custom-net
```

---

### 3. 내장 DNS 서버

#### DNS 동작 원리

```
사용자 정의 브리지 네트워크의 DNS:

1. 각 컨테이너는 내장 DNS 서버 사용
   - 127.0.0.11:53

2. DNS 쿼리 흐름:
   Container → 127.0.0.11 → Docker DNS → IP 주소

3. Docker DNS는 자동으로:
   - 컨테이너 이름 → IP 매핑
   - 네트워크 별명(alias) 지원
   - 동적 업데이트 (컨테이너 추가/제거)

┌──────────────────┐
│ Container: app1  │
│                  │
│ /etc/resolv.conf │
│ nameserver       │
│ 127.0.0.11       │ ← Docker DNS
└────────┬─────────┘
         │ DNS query: app2?
         ↓
┌────────▼─────────┐
│ Docker DNS       │
│ 127.0.0.11       │
│                  │
│ app1 → 172.18.0.2│
│ app2 → 172.18.0.3│
└────────┬─────────┘
         │ Response: 172.18.0.3
         ↓
┌──────────────────┐
│ Container: app1  │
│ Connect to       │
│ 172.18.0.3       │
└──────────────────┘
```

#### DNS 실습

```bash
# 네트워크 생성
docker network create testnet

# 컨테이너 시작
docker run -d --name db --network testnet postgres:alpine
docker run -d --name api --network testnet nginx
docker run -d --name web --network testnet nginx

# DNS 서버 확인
docker exec api cat /etc/resolv.conf
# nameserver 127.0.0.11
# options ndots:0

# 이름으로 조회
docker exec api nslookup db
# Server:		127.0.0.11
# Address:	127.0.0.11#53
# 
# Non-authoritative answer:
# Name:	db
# Address: 172.19.0.2

# ping으로 확인
docker exec api ping -c 1 db
docker exec api ping -c 1 web

# 네트워크 별명(alias) 사용
docker run -d \
  --name cache \
  --network testnet \
  --network-alias redis \
  --network-alias cache-server \
  redis:alpine

# 별명으로 접근
docker exec api nslookup redis
docker exec api nslookup cache-server
# 둘 다 같은 IP 반환!

# 정리
docker rm -f db api web cache
docker network rm testnet
```

---

### 4. 네트워크 격리

#### 네트워크 간 격리

```bash
# 두 개의 독립 네트워크
docker network create frontend
docker network create backend

# Frontend 컨테이너
docker run -d --name web --network frontend nginx
docker run -d --name proxy --network frontend nginx

# Backend 컨테이너
docker run -d --name db --network backend postgres:alpine
docker run -d --name cache --network backend redis:alpine

# ❌ 다른 네트워크 접근 불가
docker exec web ping -c 1 db
# ping: db: Name or service not known

docker exec db ping -c 1 web
# ping: web: Name or service not known

# ✅ 같은 네트워크 내에서는 가능
docker exec web ping -c 1 proxy
# 성공!

docker exec db ping -c 1 cache
# 성공!

# 정리
docker rm -f web proxy db cache
docker network rm frontend backend
```

#### 다중 네트워크 연결

```bash
# 네트워크 생성
docker network create frontend
docker network create backend

# 컨테이너를 두 네트워크에 모두 연결
docker run -d --name api --network frontend nginx

# 추가 네트워크 연결
docker network connect backend api

# 네트워크 확인
docker inspect api | grep -A 20 Networks

# 출력:
# "Networks": {
#     "frontend": {
#         "IPAddress": "172.20.0.2",
#         ...
#     },
#     "backend": {
#         "IPAddress": "172.21.0.2",
#         ...
#     }
# }

# 이제 양쪽 네트워크의 컨테이너와 통신 가능
docker run -d --name web --network frontend nginx
docker run -d --name db --network backend postgres:alpine

docker exec api ping -c 1 web  # ✅ 성공
docker exec api ping -c 1 db   # ✅ 성공

# 네트워크 연결 해제
docker network disconnect backend api

# 정리
docker rm -f api web db
docker network rm frontend backend
```

---

### 5. 포트 매핑과 외부 접근

#### 포트 퍼블리싱

```bash
# 포트 매핑 없음 (외부 접근 불가)
docker run -d --name web1 nginx

# 호스트에서 접근 시도
curl http://localhost:80
# Connection refused

# 컨테이너 IP로 접근 (호스트에서만 가능)
WEB1_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web1)
curl http://$WEB1_IP
# Welcome to nginx! ← 성공

# ✅ 포트 매핑 (외부 접근 가능)
docker run -d --name web2 -p 8080:80 nginx

# 외부에서 접근
curl http://localhost:8080
# Welcome to nginx! ← 성공

# 포트 확인
docker port web2
# 80/tcp -> 0.0.0.0:8080

# iptables 규칙 확인
sudo iptables -t nat -L DOCKER -n | grep 8080
# DNAT  tcp dpt:8080 to:172.17.0.3:80

# 정리
docker rm -f web1 web2
```

#### 여러 포트 매핑

```bash
# 여러 포트 퍼블리시
docker run -d \
  --name app \
  -p 8080:80 \
  -p 8443:443 \
  -p 3000:3000 \
  nginx

# 모든 포트 확인
docker port app
# 80/tcp -> 0.0.0.0:8080
# 443/tcp -> 0.0.0.0:8443
# 3000/tcp -> 0.0.0.0:3000

# 특정 인터페이스에만 바인딩
docker run -d \
  --name secure \
  -p 127.0.0.1:9000:80 \
  nginx

# localhost에서만 접근 가능
curl http://localhost:9000  # ✅ 성공
curl http://<외부IP>:9000    # ❌ 접근 불가

# 정리
docker rm -f app secure
```

---

### 6. 네트워크 드라이버 옵션

#### MTU 설정

```bash
# MTU 조정 (기본 1500)
docker network create \
  --driver bridge \
  --opt com.docker.network.driver.mtu=1450 \
  jumbo-net

# 확인
docker network inspect jumbo-net | grep mtu
# "com.docker.network.driver.mtu": "1450"

# 정리
docker network rm jumbo-net
```

#### ICC (Inter-Container Communication) 제어

```bash
# ICC 비활성화 (컨테이너 간 통신 차단)
docker network create \
  --driver bridge \
  --opt com.docker.network.bridge.enable_icc=false \
  isolated-net

# 테스트
docker run -d --name c1 --network isolated-net nginx
docker run -d --name c2 --network isolated-net nginx

# 통신 차단됨
docker exec c1 ping -c 1 c2
# timeout 또는 Network unreachable

# 정리
docker rm -f c1 c2
docker network rm isolated-net
```

#### IP Masquerade 제어

```bash
# IP Masquerade 비활성화
docker network create \
  --driver bridge \
  --opt com.docker.network.bridge.enable_ip_masquerade=false \
  no-nat-net

# 컨테이너는 외부 통신 불가 (NAT 없음)
docker run -d --name test --network no-nat-net nginx

docker exec test ping -c 1 8.8.8.8
# timeout (NAT 없어서 실패)

# 정리
docker rm -f test
docker network rm no-nat-net
```

---

## 💻 실습

### 실습 1: 기본 브리지 vs 사용자 정의 브리지

#### 비교 실습

```bash
# 1. 기본 브리지 테스트
echo "=== 기본 브리지 (docker0) ==="

docker run -d --name default1 nginx
docker run -d --name default2 nginx

# IP로 통신
DEFAULT2_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' default2)
docker exec default1 ping -c 2 $DEFAULT2_IP
# ✅ 성공

# 이름으로 통신
docker exec default1 ping -c 2 default2
# ❌ 실패: Name or service not known

docker rm -f default1 default2

# 2. 사용자 정의 브리지 테스트
echo "=== 사용자 정의 브리지 ==="

docker network create custom-bridge

docker run -d --name custom1 --network custom-bridge nginx
docker run -d --name custom2 --network custom-bridge nginx

# 이름으로 통신
docker exec custom1 ping -c 2 custom2
# ✅ 성공!

# DNS 조회
docker exec custom1 nslookup custom2
# ✅ IP 주소 반환

# 정리
docker rm -f custom1 custom2
docker network rm custom-bridge

# 결론:
# 기본 브리지: IP만 가능
# 사용자 정의: 이름으로 가능 (권장!)
```

---

### 실습 2: 마이크로서비스 네트워크 구성

#### 3-Tier 아키텍처

```bash
# 1. 네트워크 생성
docker network create frontend
docker network create backend

# 2. Database (backend만)
docker run -d \
  --name postgres \
  --network backend \
  -e POSTGRES_PASSWORD=secret \
  postgres:alpine

# 3. API (frontend + backend)
docker run -d \
  --name api \
  --network frontend \
  nginx

docker network connect backend api

# 4. Web (frontend만)
docker run -d \
  --name web \
  --network frontend \
  -p 8080:80 \
  nginx

# 5. 연결 테스트

# Web → API (같은 frontend)
docker exec web ping -c 1 api
# ✅ 성공

# API → Database (같은 backend)
docker exec api ping -c 1 postgres
# ✅ 성공

# Web → Database (다른 네트워크)
docker exec web ping -c 1 postgres
# ❌ 실패: Name or service not known
# 보안상 좋음!

# 6. 네트워크 구조 시각화
echo "=== 네트워크 구조 ==="
echo "Frontend: $(docker network inspect -f '{{range .Containers}}{{.Name}} {{end}}' frontend)"
echo "Backend: $(docker network inspect -f '{{range .Containers}}{{.Name}} {{end}}' backend)"

# 출력:
# Frontend: web api
# Backend: api postgres

# 7. 정리
docker rm -f web api postgres
docker network rm frontend backend
```

---

### 실습 3: 네트워크 별명과 로드 밸런싱

#### 라운드 로빈 DNS

```bash
# 1. 네트워크 생성
docker network create app-net

# 2. 같은 별명으로 여러 컨테이너
docker run -d \
  --name web1 \
  --network app-net \
  --network-alias webapp \
  nginx

docker run -d \
  --name web2 \
  --network app-net \
  --network-alias webapp \
  nginx

docker run -d \
  --name web3 \
  --network app-net \
  --network-alias webapp \
  nginx

# 3. DNS 조회
docker run --rm \
  --network app-net \
  alpine \
  nslookup webapp

# 출력: 3개의 IP 주소 모두 반환
# Name:	webapp
# Address: 172.22.0.2
# Address: 172.22.0.3
# Address: 172.22.0.4

# 4. 라운드 로빈 테스트
docker run --rm \
  --network app-net \
  alpine \
  sh -c 'for i in $(seq 1 10); do wget -qO- webapp | grep "<title>"; done'

# 5. 특정 컨테이너 제거 후
docker stop web2

# DNS는 자동으로 업데이트
docker run --rm \
  --network app-net \
  alpine \
  nslookup webapp
# web2의 IP는 제외됨!

# 6. 정리
docker rm -f web1 web2 web3
docker network rm app-net
```

---

### 실습 4: 포트 충돌 해결

#### 동적 포트 할당

```bash
# 1. 고정 포트 (충돌 가능)
docker run -d --name web1 -p 8080:80 nginx
docker run -d --name web2 -p 8080:80 nginx
# Error: port is already allocated

docker rm -f web1

# 2. 동적 포트 할당
docker run -d --name app1 -p 80 nginx
docker run -d --name app2 -p 80 nginx
docker run -d --name app3 -p 80 nginx

# 할당된 포트 확인
docker port app1
# 80/tcp -> 0.0.0.0:32768

docker port app2
# 80/tcp -> 0.0.0.0:32769

docker port app3
# 80/tcp -> 0.0.0.0:32770

# 3. 스크립트로 접근
for container in app1 app2 app3; do
  PORT=$(docker port $container | cut -d: -f2)
  echo "$container: http://localhost:$PORT"
  curl -s http://localhost:$PORT | grep "<title>"
done

# 출력:
# app1: http://localhost:32768
# <title>Welcome to nginx!</title>
# app2: http://localhost:32769
# <title>Welcome to nginx!</title>
# app3: http://localhost:32770
# <title>Welcome to nginx!</title>

# 4. 정리
docker rm -f app1 app2 app3
```

---

## 🔥 실전 적용

### 시나리오 1: 개발 환경 구성

**상황:**
- Frontend, Backend, Database 분리
- 각 서비스는 이름으로 통신
- Database는 외부 접근 차단

**구성:**

```bash
# 1. 네트워크 생성
docker network create dev-frontend
docker network create dev-backend

# 2. Database (backend만, 외부 접근 불가)
docker run -d \
  --name dev-db \
  --network dev-backend \
  -e POSTGRES_PASSWORD=devpass \
  -v dev-db-data:/var/lib/postgresql/data \
  postgres:14-alpine

# 3. Backend API (frontend + backend)
docker run -d \
  --name dev-api \
  --network dev-frontend \
  -e DATABASE_URL=postgresql://postgres:devpass@dev-db:5432/myapp \
  node:18-alpine \
  sh -c 'while true; do sleep 3600; done'

docker network connect dev-backend dev-api

# 4. Frontend (frontend만, 포트 퍼블리시)
docker run -d \
  --name dev-web \
  --network dev-frontend \
  -p 3000:80 \
  -e API_URL=http://dev-api:8080 \
  nginx:alpine

# 5. 검증

# API → Database
docker exec dev-api ping -c 1 dev-db
# ✅ 성공

# Web → API
docker exec dev-web ping -c 1 dev-api
# ✅ 성공

# Web → Database (차단되어야 함)
docker exec dev-web ping -c 1 dev-db
# ❌ 실패: 네트워크 격리됨

# 외부 → Web
curl http://localhost:3000
# ✅ 성공

# 외부 → Database (차단되어야 함)
# 포트가 퍼블리시 안 됨 → 접근 불가 ✅

# 정리
docker rm -f dev-web dev-api dev-db
docker network rm dev-frontend dev-backend
docker volume rm dev-db-data
```

---

### 시나리오 2: Blue-Green 배포

**상황:**
- 무중단 배포를 위한 Blue/Green 환경
- 네트워크 별명으로 트래픽 전환

**구성:**

```bash
# 1. 네트워크 생성
docker network create app-net

# 2. Blue 환경 (현재 프로덕션)
docker run -d \
  --name blue-v1 \
  --network app-net \
  --network-alias app \
  -e VERSION=1.0 \
  nginx:alpine

# 3. 로드 밸런서 (nginx)
docker run -d \
  --name lb \
  --network app-net \
  -p 8080:80 \
  nginx:alpine

# lb 설정 (간소화)
docker exec lb sh -c 'echo "upstream backend { server app:80; } 
server { 
  location / { proxy_pass http://backend; } 
}" > /etc/nginx/conf.d/default.conf'
docker exec lb nginx -s reload

# 4. 현재 상태 확인
curl http://localhost:8080

# 5. Green 환경 배포 (새 버전)
docker run -d \
  --name green-v2 \
  --network app-net \
  -e VERSION=2.0 \
  nginx:alpine

# 테스트
docker exec green-v2 nginx -v
# 문제 없으면 전환

# 6. 트래픽 전환
# Blue에서 별명 제거
docker network disconnect app-net blue-v1
docker network connect app-net blue-v1

# Green에 별명 추가
docker network disconnect app-net green-v2
docker run -d \
  --name green-v2-alias \
  --network app-net \
  --network-alias app \
  -e VERSION=2.0 \
  nginx:alpine

# 7. 즉시 전환됨
curl http://localhost:8080
# VERSION=2.0

# 8. 롤백 필요시
docker stop green-v2-alias
docker network disconnect app-net blue-v1
docker network connect app-net blue-v1 --alias app
# Blue로 즉시 복귀

# 정리
docker rm -f lb blue-v1 green-v2 green-v2-alias
docker network rm app-net
```

---

### 시나리오 3: 서비스 메쉬 시뮬레이션

**상황:**
- 여러 마이크로서비스
- Sidecar 패턴으로 로깅/모니터링

**구성:**

```bash
# 1. 네트워크
docker network create service-mesh

# 2. 서비스 A + 사이드카
docker run -d \
  --name service-a \
  --network service-mesh \
  nginx:alpine

docker run -d \
  --name service-a-sidecar \
  --network container:service-a \
  alpine:latest \
  sh -c 'while true; do echo "Logging from service-a"; sleep 10; done'

# 3. 서비스 B + 사이드카
docker run -d \
  --name service-b \
  --network service-mesh \
  nginx:alpine

docker run -d \
  --name service-b-sidecar \
  --network container:service-b \
  alpine:latest \
  sh -c 'while true; do echo "Logging from service-b"; sleep 10; done'

# 4. 사이드카는 같은 네트워크 네임스페이스 공유
docker exec service-a-sidecar ip addr
# service-a와 동일한 IP!

# 5. 서비스 간 통신
docker exec service-a ping -c 1 service-b
# ✅ 성공

# 6. 로그 확인
docker logs service-a-sidecar
docker logs service-b-sidecar

# 정리
docker rm -f service-a service-a-sidecar service-b service-b-sidecar
docker network rm service-mesh
```

---

## ⚡ 브리지 네트워크 체크리스트

### 네트워크 설계

```
□ 사용자 정의 브리지 사용 (docker0 금지)
□ 서비스별 네트워크 분리
□ 명확한 네트워크 명명 규칙
□ 서브넷 계획 (IP 대역 충돌 방지)
□ 문서화
```

### 보안

```
□ 네트워크 격리 (frontend/backend)
□ 불필요한 포트 퍼블리시 금지
□ ICC 제어 (필요시)
□ 최소 권한 원칙
□ 감사 로그
```

### 성능

```
□ MTU 최적화
□ DNS 캐싱 활용
□ 불필요한 네트워크 홉 제거
□ 네트워크 별명으로 로드 밸런싱
□ 모니터링 설정
```

### 운영

```
□ 네트워크 명명 규칙 준수
□ 네트워크 별명 문서화
□ 컨테이너-네트워크 매핑 관리
□ 정기적인 네트워크 정리
□ 네트워크 사용량 모니터링
```

---

## 🚫 안티패턴

### 1. 기본 브리지 사용

```bash
# ❌ 기본 브리지 (레거시)
docker run -d --name app nginx
# 이름 기반 통신 불가
# 모든 컨테이너가 접근 가능

# ✅ 사용자 정의 브리지
docker network create mynet
docker run -d --name app --network mynet nginx
# 이름 기반 통신 가능
# 네트워크 격리
```

### 2. 네트워크 격리 없음

```bash
# ❌ 모두 같은 네트워크
docker network create app-net
docker run -d --network app-net --name web nginx
docker run -d --network app-net --name api nginx
docker run -d --network app-net --name db postgres
# DB가 web에서 접근 가능 (보안 위험)

# ✅ 네트워크 분리
docker network create frontend
docker network create backend
docker run -d --network frontend --name web nginx
docker run -d --network frontend --name api nginx
docker network connect backend api
docker run -d --network backend --name db postgres
# DB는 api만 접근 가능
```

### 3. 하드코딩된 IP

```bash
# ❌ IP 주소 하드코딩
docker run -d --name api nginx
# 코드에서: http://172.18.0.3:8080
# IP 변경 시 문제 발생

# ✅ 서비스 이름 사용
docker network create mynet
docker run -d --name api --network mynet nginx
# 코드에서: http://api:8080
# IP 변경되어도 동작
```

### 4. 불필요한 포트 퍼블리시

```bash
# ❌ 모든 서비스 포트 오픈
docker run -d -p 5432:5432 --name db postgres
# 외부에서 DB 직접 접근 가능 (위험)

# ✅ 필요한 것만 퍼블리시
docker network create app-net
docker run -d --network app-net --name db postgres
# 포트 오픈 안 함, 내부 통신만
docker run -d --network app-net -p 8080:80 --name web nginx
# Web만 외부 접근 허용
```

---

## 🎓 핵심 정리

### 1. 기본 vs 사용자 정의

```
기본 브리지 (docker0):
- 레거시 방식
- IP 주소만 사용
- 모든 컨테이너 공유
- 권장하지 않음

사용자 정의 브리지:
- 현대적 방식
- 이름 기반 통신
- 네트워크 격리
- 권장 ✅
```

### 2. 내장 DNS

```
사용자 정의 브리지:
- 자동 DNS 서버 (127.0.0.11)
- 컨테이너 이름 → IP
- 네트워크 별명 지원
- 동적 업데이트

기본 브리지:
- DNS 없음
- --link (deprecated)
```

### 3. 네트워크 격리

```
보안 레이어:
1. 네트워크 분리 (frontend/backend)
2. 포트 제한 (필요한 것만 퍼블리시)
3. 다중 네트워크 연결 (API만)

접근 제어:
Web → API → Database
Web ❌ Database
```

### 4. 핵심 명령어

```bash
# 네트워크 관리
docker network create/ls/rm/inspect

# 컨테이너 연결
--network <name>
docker network connect/disconnect

# 포트 매핑
-p <host>:<container>
docker port

# 네트워크 별명
--network-alias <name>
```

---

## 📚 참고 자료

- [Docker Bridge Networks](https://docs.docker.com/network/bridge/)
- [Docker Embedded DNS](https://docs.docker.com/config/containers/container-networking/#dns-services)
- [Network Drivers](https://docs.docker.com/network/drivers/)
- [Docker Networking Best Practices](https://docs.docker.com/network/)

---

## 🤔 생각해볼 문제

1. 사용자 정의 브리지에서만 DNS가 동작하는 이유는?
2. 컨테이너를 여러 네트워크에 연결하면 라우팅은 어떻게 될까?
3. 네트워크 별명으로 여러 컨테이너를 연결하면 로드 밸런싱이 될까?

> 💡 **답변**: 1) 기본 브리지(docker0)는 레거시 호환성 유지를 위해 옛날 방식(--link) 사용, 사용자 정의 브리지는 새로운 네트워크 스택으로 처음부터 DNS 서버(127.0.0.11) 내장 설계, 2) 여러 네트워크에 연결된 컨테이너는 각 네트워크에 별도 IP를 가지며, 라우팅 테이블에 각 네트워크로의 경로가 자동 추가됨, 목적지에 따라 적절한 인터페이스로 패킷 전송, 3) DNS는 라운드 로빈으로 여러 IP를 반환하지만 실제 로드 밸런싱은 클라이언트가 캐싱하면 한 서버로만 갈 수 있음, 진정한 로드 밸런싱은 별도 LB(nginx, haproxy) 필요

---

<div align="center">

**[⬅️ 이전: Network Fundamentals](./01-Network-Fundamentals.md)** | **[다음: Host Network ➡️](./03-Host-Network.md)**

</div>
