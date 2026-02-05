# 04. Overlay Network - 오버레이 네트워크

## 🎯 이 챕터에서 배울 것

- **Overlay 네트워크**의 개념과 동작 원리
- **VXLAN** 터널링 프로토콜
- **Docker Swarm**에서의 멀티 호스트 네트워킹
- 실전 **클러스터 네트워킹** 구성

## 📌 왜 중요한가?

**"Overlay 네트워크는 여러 호스트의 컨테이너를 하나의 네트워크로 연결합니다."**

```
단일 호스트 (Bridge):
┌─────────────────┐
│ Host 1          │
│ Container A ←─→ Container B 
└─────────────────┘

멀티 호스트 (Overlay):
┌─────────────────┐         ┌─────────────────┐
│ Host 1          │         │ Host 2          │
│ Container A ←───┼─────────┼───→ Container C │
└─────────────────┘  VXLAN  └─────────────────┘

Overlay:
- 물리 네트워크 위에 가상 네트워크
- 여러 호스트의 컨테이너 직접 통신
- 마치 같은 호스트처럼
```

**실무 영향:**
- 확장성: 수백 노드까지 확장 가능
- 유연성: 컨테이너를 어느 호스트에나 배치
- 간편성: 복잡한 라우팅 설정 불필요
- 오케스트레이션: Swarm, Kubernetes의 핵심

---

## 🔬 Deep Dive

### 1. Overlay 네트워크란?

#### 기본 개념

```
Overlay Network:
- Layer 2 네트워크를 Layer 3 위에 구현
- 물리 네트워크(underlay) 위의 가상 네트워크(overlay)
- 캡슐화(Encapsulation)로 구현

구조:
┌──────────────────────────────────────────────────────┐
│                   Overlay Network                    │
│           (Virtual Layer 2: 10.0.0.0/24)             │
│                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐      │
│  │ Container  │  │ Container  │  │ Container  │      │
│  │ 10.0.0.2   │  │ 10.0.0.3   │  │ 10.0.0.4   │      │
│  └─────┬──────┘  └──────┬─────┘  └───────┬────┘      │
│        │                │                │           │
└────────┼────────────────┼────────────────┼───────────┘
         │                │                │
      Encap            Encap            Encap
         │                │                │
┌────────┼────────────────┼────────────────┼────────────┐
│        ↓                ↓                ↓            │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐       │
│  │ Host 1   │     │ Host 2   │     │ Host 3   │       │
│  │192.168.1.│     │192.168.1.│     │192.168.1.│       │
│  │   .10    │     │   .11    │     │   .12    │       │
│  └────┬─────┘     └─────┬────┘     └──────┬───┘       │
│       └─────────────────┴─────────────────┘           │
│              Physical Network                         │
│           (Underlay: 192.168.1.0/24)                  │
└───────────────────────────────────────────────────────┘

캡슐화:
[Container IP Header][Container Data]
         ↓ (VXLAN 캡슐화)
[Host IP Header][VXLAN Header][Container IP Header][Container Data]
```

#### VXLAN (Virtual Extensible LAN)

```
VXLAN:
- Virtual Extensible LAN
- RFC 7348
- UDP 포트 4789
- 24-bit VNI (Virtual Network Identifier)
  → 16M 개의 네트워크 지원

VXLAN 패킷 구조:
┌─────────────────────────────────────────────┐
│ Outer Ethernet Header                       │
├─────────────────────────────────────────────┤
│ Outer IP Header (Host IP)                   │
├─────────────────────────────────────────────┤
│ Outer UDP Header (Dst: 4789)                │
├─────────────────────────────────────────────┤
│ VXLAN Header (VNI: 4096)                    │
├─────────────────────────────────────────────┤
│ Inner Ethernet Header (Container MAC)       │
├─────────────────────────────────────────────┤
│ Inner IP Header (Container IP)              │
├─────────────────────────────────────────────┤
│ TCP/UDP Data                                │
└─────────────────────────────────────────────┘

예시:
Container A (10.0.0.2) → Container B (10.0.0.3)
서로 다른 호스트

1. Container A가 패킷 생성
   src: 10.0.0.2, dst: 10.0.0.3

2. VXLAN 캡슐화
   Outer: src 192.168.1.10, dst 192.168.1.11
   Inner: src 10.0.0.2, dst 10.0.0.3

3. 물리 네트워크 전송

4. Host 2에서 디캡슐화

5. Container B에 전달
```

---

### 2. Docker Swarm과 Overlay

#### Swarm 모드 활성화

```bash
# Node 1 (Manager)
docker swarm init --advertise-addr 192.168.1.10

# 출력:
# Swarm initialized: current node (abc123...) is now a manager.
# To add a worker to this swarm, run the following command:
#     docker swarm join --token SWMTKN-1-... 192.168.1.10:2377

# Worker 추가 토큰 확인
docker swarm join-token worker

# Node 2, 3 (Workers)
docker swarm join --token SWMTKN-1-... 192.168.1.10:2377

# 클러스터 확인 (Manager에서)
docker node ls
# ID        HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc123... node1      Ready   Active        Leader
# def456... node2      Ready   Active        
# ghi789... node3      Ready   Active
```

#### Overlay 네트워크 생성

```bash
# Manager 노드에서 생성
docker network create \
  --driver overlay \
  --attachable \
  my-overlay

# 네트워크 확인
docker network ls
# NETWORK ID     NAME         DRIVER    SCOPE
# ...            my-overlay   overlay   swarm

# 상세 정보
docker network inspect my-overlay

# 출력:
# {
#     "Name": "my-overlay",
#     "Driver": "overlay",
#     "Scope": "swarm",
#     "IPAM": {
#         "Config": [
#             {
#                 "Subnet": "10.0.0.0/24",
#                 "Gateway": "10.0.0.1"
#             }
#         ]
#     }
# }
```

---

### 3. 멀티 호스트 통신

#### 서비스 배포

```bash
# 3개 레플리카로 서비스 생성
docker service create \
  --name web \
  --replicas 3 \
  --network my-overlay \
  nginx:alpine

# 서비스 확인
docker service ls
# ID        NAME  MODE        REPLICAS  IMAGE
# xyz123... web   replicated  3/3       nginx:alpine

# 레플리카 위치 확인
docker service ps web
# ID        NAME    NODE   DESIRED STATE  CURRENT STATE
# abc...    web.1   node1  Running        Running
# def...    web.2   node2  Running        Running
# ghi...    web.3   node3  Running        Running

# 각 노드에 1개씩 분산 배치됨
```

#### 서비스 간 통신

```bash
# API 서비스 생성
docker service create \
  --name api \
  --replicas 2 \
  --network my-overlay \
  nginx:alpine

# 테스트 컨테이너 (Manager에서)
docker run -it --rm \
  --network my-overlay \
  alpine sh

# 컨테이너 내부에서
ping web
# PING web (10.0.0.2) 56(84) bytes of data.
# 64 bytes from web.1.xyz123 (10.0.0.2): icmp_seq=1 ...

ping api
# PING api (10.0.0.5) 56(84) bytes of data.
# 64 bytes from api.1.abc456 (10.0.0.5): icmp_seq=1 ...

# DNS 조회
nslookup web
# Server:		127.0.0.11
# Address:	127.0.0.11#53
# 
# Non-authoritative answer:
# Name:	web
# Address: 10.0.0.2
# Address: 10.0.0.3
# Address: 10.0.0.4

# 서비스 이름으로 통신!
# 물리적으로 다른 호스트에 있어도 같은 네트워크처럼
```

---

### 4. Overlay 네트워크 내부 구조

#### VXLAN 인터페이스 확인

```bash
# Manager 노드에서 서비스 실행 후
docker service create \
  --name test \
  --network my-overlay \
  --replicas 1 \
  alpine sleep 3600

# 컨테이너 찾기
CONTAINER_ID=$(docker ps --filter "name=test" -q)

# 네트워크 네임스페이스 확인
docker inspect $CONTAINER_ID | grep SandboxKey
# "SandboxKey": "/var/run/docker/netns/abc123"

# 인터페이스 확인
sudo nsenter --net=/var/run/docker/netns/abc123 ip addr

# 출력:
# 1: lo: <LOOPBACK,UP>
# 2: eth0@if123: <BROADCAST,MULTICAST,UP>
#     inet 10.0.0.2/24  ← Overlay IP
# 3: eth1@if456: <BROADCAST,MULTICAST,UP>
#     inet 10.255.0.5/16  ← Swarm 관리용

# VXLAN 디바이스 확인 (호스트)
ip -d link show

# 출력:
# vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     vxlan id 4096 local 192.168.1.10 dev eth0 port 4789
```

#### 패킷 흐름 추적

```bash
# Manager 노드에서 VXLAN 패킷 캡처
sudo tcpdump -i eth0 -n udp port 4789 -v

# 다른 터미널에서 통신 테스트
docker run --rm --network my-overlay alpine ping -c 1 10.0.0.2

# tcpdump 출력:
# 192.168.1.10.12345 > 192.168.1.11.4789: VXLAN, flags [I] (0x08), vni 4096
# IP 10.0.0.3 > 10.0.0.2: ICMP echo request, id 1, seq 1

# 설명:
# - Outer: 192.168.1.10 → 192.168.1.11 (호스트 간)
# - VXLAN: VNI 4096 (네트워크 식별)
# - Inner: 10.0.0.3 → 10.0.0.2 (컨테이너 간)
```

---

### 5. Overlay 네트워크 고급 설정

#### 서브넷 및 게이트웨이 지정

```bash
docker network create \
  --driver overlay \
  --subnet 10.10.0.0/16 \
  --gateway 10.10.0.1 \
  --ip-range 10.10.10.0/24 \
  custom-overlay

# 확인
docker network inspect custom-overlay | grep -A 10 IPAM
# "IPAM": {
#     "Config": [
#         {
#             "Subnet": "10.10.0.0/16",
#             "IPRange": "10.10.10.0/24",
#             "Gateway": "10.10.0.1"
#         }
#     ]
# }
```

#### 암호화된 Overlay

```bash
# 데이터 플레인 암호화 (IPSec)
docker network create \
  --driver overlay \
  --opt encrypted \
  secure-overlay

# 서비스 생성
docker service create \
  --name secure-app \
  --network secure-overlay \
  nginx:alpine

# 패킷 캡처 시 암호화됨
sudo tcpdump -i eth0 -n udp port 4789
# ESP 패킷 (암호화됨)
```

#### 외부 접근 (Ingress)

```bash
# 포트 퍼블리시
docker service create \
  --name web \
  --publish published=8080,target=80 \
  --replicas 3 \
  --network my-overlay \
  nginx:alpine

# 어느 노드로든 접근 가능
curl http://192.168.1.10:8080  # Node 1
curl http://192.168.1.11:8080  # Node 2
curl http://192.168.1.12:8080  # Node 3

# 모두 동작! (Ingress 네트워크)
# 내부적으로 로드 밸런싱
```

---

## 💻 실습

### 실습 1: 3노드 Swarm 클러스터 구성

#### 환경 준비 (로컬 시뮬레이션)

```bash
# Docker Machine 또는 VM 3개 필요
# 여기서는 단순화를 위해 docker-in-docker 사용

# Manager 노드
docker run -d --privileged --name manager \
  --hostname manager \
  docker:dind

# Worker 노드 2개
docker run -d --privileged --name worker1 \
  --hostname worker1 \
  docker:dind

docker run -d --privileged --name worker2 \
  --hostname worker2 \
  docker:dind

# Manager에서 Swarm 초기화
docker exec manager docker swarm init

# Join 토큰 가져오기
JOIN_TOKEN=$(docker exec manager docker swarm join-token worker -q)
MANAGER_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' manager)

# Worker 노드 조인
docker exec worker1 docker swarm join \
  --token $JOIN_TOKEN ${MANAGER_IP}:2377

docker exec worker2 docker swarm join \
  --token $JOIN_TOKEN ${MANAGER_IP}:2377

# 클러스터 확인
docker exec manager docker node ls
# ID        HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
# abc...    manager   Ready   Active        Leader
# def...    worker1   Ready   Active        
# ghi...    worker2   Ready   Active
```

#### Overlay 네트워크 생성 및 테스트

```bash
# Overlay 네트워크 생성
docker exec manager docker network create \
  --driver overlay \
  --attachable \
  test-overlay

# 서비스 배포
docker exec manager docker service create \
  --name web \
  --replicas 3 \
  --network test-overlay \
  nginx:alpine

# 배포 확인
docker exec manager docker service ps web

# 각 노드에서 컨테이너 확인
docker exec manager docker ps --filter "name=web"
docker exec worker1 docker ps --filter "name=web"
docker exec worker2 docker ps --filter "name=web"

# 통신 테스트
docker exec manager docker run --rm \
  --network test-overlay \
  alpine ping -c 2 web

# 정리
docker exec manager docker service rm web
docker exec manager docker network rm test-overlay
docker rm -f manager worker1 worker2
```

---

### 실습 2: 서비스 간 통신

#### 마이크로서비스 배포

```bash
# Swarm 클러스터 준비 (이전 실습 참고)

# Overlay 네트워크
docker exec manager docker network create \
  --driver overlay \
  app-net

# Database 서비스
docker exec manager docker service create \
  --name db \
  --network app-net \
  --mount type=volume,src=db-data,dst=/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:alpine

# API 서비스
docker exec manager docker service create \
  --name api \
  --network app-net \
  --replicas 3 \
  -e DATABASE_URL=postgresql://postgres:secret@db:5432/myapp \
  nginx:alpine

# Web 서비스 (Ingress 포트)
docker exec manager docker service create \
  --name web \
  --network app-net \
  --replicas 2 \
  --publish published=8080,target=80 \
  nginx:alpine

# 통신 테스트
# API → Database
docker exec manager docker service logs api

# Web → API
docker exec manager docker run --rm \
  --network app-net \
  alpine wget -qO- http://api

# 외부 → Web
curl http://<MANAGER_IP>:8080
curl http://<WORKER1_IP>:8080
# 어느 노드로든 접근 가능!
```

---

### 실습 3: 암호화된 Overlay

#### 암호화 활성화

```bash
# 암호화된 네트워크 생성
docker network create \
  --driver overlay \
  --opt encrypted \
  secure-net

# 서비스 배포
docker service create \
  --name secure-app \
  --network secure-net \
  --replicas 2 \
  nginx:alpine

# 패킷 캡처 (다른 터미널)
sudo tcpdump -i eth0 -n -X 'udp port 4789'

# 트래픽 생성
docker run --rm --network secure-net \
  alpine wget -qO- http://secure-app

# tcpdump 출력:
# - ESP (Encapsulating Security Payload)
# - 데이터가 암호화됨
# - 일반 텍스트 보이지 않음

# 비암호화 네트워크와 비교
docker network create --driver overlay plain-net
docker service create --name plain-app --network plain-net nginx:alpine

# 패킷 캡처 시 평문 데이터 보임
```

---

### 실습 4: Overlay 네트워크 디버깅

#### 네트워크 문제 진단

```bash
# 1. 네트워크 연결 확인
docker network inspect my-overlay | grep -A 5 Containers

# 2. 서비스 상태
docker service ps web

# 3. 로그 확인
docker service logs web

# 4. VXLAN 인터페이스
ip -d link show type vxlan

# 5. 라우팅 테이블
ip route show

# 6. ARP 테이블
ip neighbor show

# 7. 방화벽 규칙
sudo iptables -t filter -L -n | grep 4789
sudo iptables -t nat -L -n

# 8. 포트 리스닝
sudo netstat -nlpu | grep 4789

# 9. 연결 추적
sudo conntrack -L | grep 4789

# 10. 패킷 캡처
sudo tcpdump -i any -n 'udp port 4789'
```

---

## 🔥 실전 적용

### 시나리오 1: 멀티 리전 배포

**상황:**
- 3개 리전에 각각 3개 노드
- 리전 간 낮은 지연시간 필요
- 고가용성 필수

**구성:**

```bash
# 각 리전에 Swarm 클러스터

# Region 1 (US-East)
docker swarm init --advertise-addr 10.1.0.10
docker node update --label-add region=us-east manager1

# Region 2 (EU-West)  
docker swarm join --token ... 10.1.0.10:2377
docker node update --label-add region=eu-west worker1

# Region 3 (AP-Southeast)
docker swarm join --token ... 10.1.0.10:2377
docker node update --label-add region=ap-se worker2

# Overlay 네트워크
docker network create \
  --driver overlay \
  --attachable \
  global-net

# 리전별 서비스 배포
docker service create \
  --name api-us \
  --network global-net \
  --constraint 'node.labels.region==us-east' \
  --replicas 3 \
  myapp:latest

docker service create \
  --name api-eu \
  --network global-net \
  --constraint 'node.labels.region==eu-west' \
  --replicas 3 \
  myapp:latest

docker service create \
  --name api-ap \
  --network global-net \
  --constraint 'node.labels.region==ap-se' \
  --replicas 3 \
  myapp:latest

# 글로벌 데이터베이스
docker service create \
  --name db-primary \
  --network global-net \
  --constraint 'node.labels.region==us-east' \
  postgres:14

# 리전 간 통신
# api-us ←→ api-eu (Overlay 네트워크)
# api-eu ←→ api-ap (자동 라우팅)
```

---

### 시나리오 2: Blue-Green 배포

**상황:**
- 무중단 배포
- 트래픽 전환 제어
- 롤백 가능

**구성:**

```bash
# Blue 환경 (현재 프로덕션)
docker service create \
  --name app-blue \
  --network overlay-prod \
  --label version=blue \
  --replicas 10 \
  --publish published=8080,target=80 \
  myapp:v1.0

# Green 환경 (새 버전) - 포트 미퍼블리시
docker service create \
  --name app-green \
  --network overlay-prod \
  --label version=green \
  --replicas 10 \
  myapp:v2.0

# 내부 테스트
docker run --rm --network overlay-prod \
  alpine wget -qO- http://app-green

# Green 검증 완료 후 트래픽 전환

# 1. Blue 포트 제거
docker service update --publish-rm 8080 app-blue

# 2. Green 포트 추가
docker service update --publish-add published=8080,target=80 app-green

# 즉시 전환! (downtime 0초)

# 문제 발생 시 롤백
docker service update --publish-rm 8080 app-green
docker service update --publish-add published=8080,target=80 app-blue

# 완료 후 Blue 제거
docker service rm app-blue
```

---

### 시나리오 3: 하이브리드 클라우드

**상황:**
- 온프레미스 + 클라우드
- 데이터 주권 준수
- 점진적 마이그레이션

**구성:**

```bash
# 온프레미스 (민감 데이터)
docker swarm init --advertise-addr 10.0.1.10
docker node update --label-add location=onprem manager1

# 클라우드 (컴퓨트)
docker swarm join --token ... 10.0.1.10:2377
docker node update --label-add location=cloud worker1

# 암호화된 Overlay (보안)
docker network create \
  --driver overlay \
  --opt encrypted \
  hybrid-net

# 온프레미스 데이터베이스
docker service create \
  --name db \
  --network hybrid-net \
  --constraint 'node.labels.location==onprem' \
  postgres:14

# 클라우드 API (스케일 아웃)
docker service create \
  --name api \
  --network hybrid-net \
  --constraint 'node.labels.location==cloud' \
  --replicas 20 \
  myapp:latest

# API ←→ Database
# - Overlay 네트워크로 연결
# - 암호화된 터널
# - 온프레미스 ↔ 클라우드 투명하게 통신
```

---

## ⚡ Overlay 네트워크 체크리스트

### 설계

```
□ 서브넷 계획 (충돌 방지)
□ 암호화 필요성 검토
□ 네트워크 토폴로지 설계
□ 네이밍 규칙 정의
□ 방화벽 규칙 (UDP 4789)
```

### 보안

```
□ 암호화 활성화 (민감 데이터)
□ Swarm 토큰 관리
□ TLS 인증서 회전
□ 네트워크 세그멘테이션
□ 접근 제어 정책
```

### 성능

```
□ MTU 최적화
□ 라우터 홉 최소화
□ 대역폭 모니터링
□ 지연시간 측정
□ 패킷 손실률 확인
```

### 운영

```
□ 노드 라벨 관리
□ 서비스 배치 전략
□ 헬스 체크 설정
□ 로깅 및 모니터링
□ 백업 및 복구 계획
```

### 문제 해결

```
□ VXLAN 인터페이스 확인
□ 패킷 캡처 및 분석
□ 방화벽 규칙 검증
□ DNS 해석 확인
□ 네트워크 격리 테스트
```

---

## 🚫 안티패턴

### 1. 불필요한 Overlay 사용

```bash
# ❌ 단일 호스트에서 Overlay
docker network create --driver overlay single-net
docker run --network single-net nginx
# 오버헤드만 증가

# ✅ 단일 호스트는 Bridge
docker network create mynet
docker run --network mynet nginx
```

### 2. 암호화 미사용 (민감 데이터)

```bash
# ❌ 평문 전송
docker network create --driver overlay payment-net
# 결제 정보가 암호화 없이 전송

# ✅ 암호화 사용
docker network create \
  --driver overlay \
  --opt encrypted \
  payment-net
```

### 3. 서브넷 충돌

```bash
# ❌ 기존 네트워크와 충돌
docker network create \
  --driver overlay \
  --subnet 192.168.1.0/24 \
  app-net
# 호스트 네트워크가 192.168.1.0/24 → 충돌!

# ✅ 별도 대역 사용
docker network create \
  --driver overlay \
  --subnet 10.20.0.0/16 \
  app-net
```

### 4. 방화벽 미설정

```bash
# ❌ 방화벽 막힘
docker network create --driver overlay test-net
docker service create --network test-net nginx
# UDP 4789 차단 → 통신 안 됨

# ✅ 방화벽 오픈
sudo ufw allow 4789/udp
sudo ufw allow 7946/tcp
sudo ufw allow 7946/udp
sudo ufw allow 2377/tcp
```

---

## 🎓 핵심 정리

### 1. Overlay 네트워크 개념

```
정의:
- 물리 네트워크 위의 가상 네트워크
- Layer 2 over Layer 3
- VXLAN 캡슐화

장점:
+ 멀티 호스트 통신
+ 확장성
+ 유연한 배치
+ 자동 라우팅

단점:
- 캡슐화 오버헤드 (5-10%)
- 설정 복잡
- 디버깅 어려움
```

### 2. VXLAN

```
프로토콜:
- UDP 포트 4789
- 24-bit VNI
- 16M 개 네트워크

패킷:
Outer IP → UDP → VXLAN → Inner Ethernet → Inner IP → Data

오버헤드:
- 50 bytes 추가
- MTU 고려 필요
```

### 3. Swarm 통합

```
자동 기능:
- 서비스 디스커버리
- 로드 밸런싱 (Ingress)
- 암호화 (옵션)
- 헬스 체크

네트워크:
- ingress (기본)
- 사용자 정의 overlay
- docker_gwbridge (호스트 연결)
```

### 4. 핵심 명령어

```bash
# Swarm 관리
docker swarm init
docker swarm join

# 네트워크 생성
docker network create --driver overlay

# 서비스 배포
docker service create --network

# 디버깅
ip -d link show type vxlan
tcpdump -i any 'udp port 4789'
```

---

## 📚 참고 자료

- [Docker Overlay Networks](https://docs.docker.com/network/overlay/)
- [VXLAN RFC 7348](https://datatracker.ietf.org/doc/html/rfc7348)
- [Docker Swarm](https://docs.docker.com/engine/swarm/)
- [Swarm Networking](https://docs.docker.com/engine/swarm/networking/)

---

## 🤔 생각해볼 문제

1. Overlay 네트워크에서 컨테이너 간 통신이 Bridge보다 느린 이유는?
2. VXLAN VNI는 왜 24-bit일까? (16M 개면 충분한가?)
3. Overlay 네트워크에서 브로드캐스트는 어떻게 처리될까?

> 💡 **답변**: 1) VXLAN 캡슐화/디캡슐화 오버헤드(CPU), 50바이트 추가 헤더(대역폭), 추가 네트워크 홉(물리 네트워크), UDP 체크섬 계산 - 일반적으로 5-10% 성능 저하, 2) VLAN은 12-bit(4096개)로 부족했고, 24-bit는 16,777,216개로 대규모 클라우드 환경에서도 충분, 각 테넌트/애플리케이션별로 독립된 네트워크 ID 할당 가능, 3) BUM(Broadcast, Unknown unicast, Multicast) 트래픽은 멀티캐스트 그룹 또는 헤드엔드 복제(head-end replication)로 처리 - Docker는 주로 유니캐스트 헤드엔드 복제 사용, 각 VTEP에 복사본 전송

---

<div align="center">

**[⬅️ 이전: Host Network](./03-Host-Network.md)** | **[다음: Macvlan Network ➡️](./05-Macvlan-Network.md)**

</div>
