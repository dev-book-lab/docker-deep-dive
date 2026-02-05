# 05. Macvlan Network - 맥브이랜 네트워크

## 🎯 이 챕터에서 배울 것

- **Macvlan**의 개념과 동작 원리
- 물리 네트워크와의 **직접 통합**
- **VLAN 태깅**과 네트워크 세그멘테이션
- 실무 **사용 시나리오**와 제약사항

## 📌 왜 중요한가?

**"Macvlan은 컨테이너를 물리 네트워크의 1등 시민으로 만듭니다."**

```
Bridge 네트워크:
┌─────────────┐
│ Container   │
│ 172.17.0.2  │ ← 내부 IP
└──────┬──────┘
       │ NAT
   ┌───▼────┐
   │ Host   │
   │10.0.0.1│ ← 외부 IP
   └────────┘

Macvlan 네트워크:
┌─────────────┐
│ Container   │
│ 10.0.0.100  │ ← 물리 네트워크 IP
└──────┬──────┘
       │ 직접 연결
   ┌───▼────────┐
   │ Physical   │
   │ Network    │
   └────────────┘

차이:
- Bridge: NAT, 간접 통신
- Macvlan: 직접 통신, 물리 네트워크처럼
```

**실무 영향:**
- 성능: NAT 오버헤드 제거
- 통합: 레거시 시스템과 쉬운 연동
- 관리: 기존 네트워크 도구 사용 가능
- 제약: 호스트-컨테이너 통신 제한

---

## 🔬 Deep Dive

### 1. Macvlan이란?

#### 기본 개념

```
Macvlan:
- MAC address를 가진 가상 네트워크 인터페이스
- 하나의 물리 인터페이스에 여러 MAC 주소
- 각 컨테이너가 고유한 MAC 주소
- 물리 네트워크에서 별도 디바이스처럼 보임

구조:
┌──────────────────────────────────────────┐
│ Physical Switch                          │
│  ┌────┬────┬────┬────┐                   │
│  │MAC1│MAC2│MAC3│MAC4│                   │
│  └─┬──┴─┬──┴─┬──┴─┬──┘                   │
└────┼────┼────┼────┼──────────────────────┘
     │    │    │    │
┌────┼────┼────┼────┼──────────────────────┐
│ Host                                     │
│    │    │    │    │                      │
│    └────┴────┴────┘                      │
│         eth0 (Physical)                  │
│    ┌─────┬─────┬─────┐                   │
│    │     │     │     │                   │
│ ┌──▼──┐ ┌▼───┐ ┌▼───┐ ┌▼───┐             │
│ │C1   │ │C2  │ │C3  │ │C4  │             │
│ │MAC1 │ │MAC2│ │MAC3│ │MAC4│             │
│ │.100 │ │.101│ │.102│ │.103│             │
│ └─────┘ └────┘ └────┘ └────┘             │
└──────────────────────────────────────────┘

특징:
- 컨테이너가 물리 네트워크의 IP 사용
- 스위치에서 각 컨테이너를 별도 디바이스로 인식
- DHCP 서버에서 IP 할당 가능
```

#### Macvlan 모드

```
4가지 모드:

1. Bridge Mode (일반적)
   - 같은 Macvlan의 컨테이너끼리 통신
   - 외부 네트워크와 통신

2. VEPA (Virtual Ethernet Port Aggregator)
   - 모든 트래픽이 외부 스위치 경유
   - 하드웨어 스위치 정책 적용

3. Private Mode
   - 컨테이너 간 통신 차단
   - 외부와만 통신

4. Passthru Mode
   - 물리 인터페이스 1:1 매핑
   - 컨테이너 1개만 연결 가능

일반적으로 Bridge 모드 사용
```

---

### 2. Macvlan 기본 사용

#### 네트워크 생성

```bash
# 물리 인터페이스 확인
ip link show
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>

# Macvlan 네트워크 생성
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net

# 네트워크 확인
docker network inspect macvlan-net

# 출력:
# {
#     "Name": "macvlan-net",
#     "Driver": "macvlan",
#     "IPAM": {
#         "Config": [
#             {
#                 "Subnet": "192.168.1.0/24",
#                 "Gateway": "192.168.1.1"
#             }
#         ]
#     },
#     "Options": {
#         "parent": "eth0"
#     }
# }
```

#### 컨테이너 실행

```bash
# 컨테이너 시작
docker run -d \
  --name web1 \
  --network macvlan-net \
  --ip 192.168.1.100 \
  nginx:alpine

docker run -d \
  --name web2 \
  --network macvlan-net \
  --ip 192.168.1.101 \
  nginx:alpine

# IP 확인
docker exec web1 ip addr show eth0
# eth0: ...
#     inet 192.168.1.100/24

docker exec web2 ip addr show eth0
# eth0: ...
#     inet 192.168.1.101/24

# 물리 네트워크의 다른 디바이스에서 접근
# (같은 네트워크의 다른 컴퓨터에서)
curl http://192.168.1.100
# Welcome to nginx!

# 컨테이너가 물리 네트워크의 일부처럼 동작!
```

#### 호스트-컨테이너 통신 문제

```bash
# ⚠️ 호스트에서 컨테이너 접근 시도
ping 192.168.1.100
# 실패! (Macvlan 제약사항)

# 이유:
# - Macvlan은 호스트-컨테이너 직접 통신 차단
# - 보안 및 네트워크 격리 목적

# 해결 방법 1: 외부 디바이스 경유
# (물리 스위치/라우터 경유)

# 해결 방법 2: Macvlan 서브인터페이스 생성
sudo ip link add macvlan-shim link eth0 type macvlan mode bridge
sudo ip addr add 192.168.1.254/32 dev macvlan-shim
sudo ip link set macvlan-shim up
sudo ip route add 192.168.1.100/32 dev macvlan-shim

# 이제 호스트에서 접근 가능
ping 192.168.1.100
# 성공!
```

---

### 3. VLAN 태깅

#### 802.1Q VLAN

```
VLAN:
- 물리 네트워크를 논리적으로 분할
- VLAN ID (1-4094)
- 네트워크 세그멘테이션

예시:
┌──────────────────────────────────┐
│ Physical Switch                  │
│  VLAN 10: 192.168.10.0/24        │
│  VLAN 20: 192.168.20.0/24        │
│  VLAN 30: 192.168.30.0/24        │
└────────────┬─────────────────────┘
             │ Trunk Port (802.1Q)
┌────────────▼─────────────────────┐
│ Host eth0                        │
│  eth0.10 (VLAN 10)               │
│  eth0.20 (VLAN 20)               │
│  eth0.30 (VLAN 30)               │
└──────────────────────────────────┘
```

#### VLAN 서브인터페이스 생성

```bash
# VLAN 10용 서브인터페이스
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip link set eth0.10 up

# VLAN 20용 서브인터페이스
sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip link set eth0.20 up

# 확인
ip link show | grep vlan
# eth0.10@eth0: <BROADCAST,MULTICAST,UP>
# eth0.20@eth0: <BROADCAST,MULTICAST,UP>

# Macvlan 네트워크 (VLAN 10)
docker network create -d macvlan \
  --subnet=192.168.10.0/24 \
  --gateway=192.168.10.1 \
  -o parent=eth0.10 \
  vlan10-net

# Macvlan 네트워크 (VLAN 20)
docker network create -d macvlan \
  --subnet=192.168.20.0/24 \
  --gateway=192.168.20.1 \
  -o parent=eth0.20 \
  vlan20-net

# VLAN 10 컨테이너
docker run -d \
  --name web-vlan10 \
  --network vlan10-net \
  --ip 192.168.10.100 \
  nginx:alpine

# VLAN 20 컨테이너
docker run -d \
  --name db-vlan20 \
  --network vlan20-net \
  --ip 192.168.20.100 \
  postgres:alpine

# VLAN 격리:
# web-vlan10 ❌ db-vlan20 (다른 VLAN)
# 스위치 라우팅 필요
```

---

### 4. Macvlan 모드 비교

#### Bridge 모드

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  -o parent=eth0 \
  -o macvlan_mode=bridge \
  macvlan-bridge

docker run -d --name c1 --network macvlan-bridge nginx
docker run -d --name c2 --network macvlan-bridge nginx

# c1 ↔ c2 통신 가능
# c1/c2 ↔ 외부 통신 가능
```

#### Private 모드

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  -o parent=eth0 \
  -o macvlan_mode=private \
  macvlan-private

docker run -d --name c1 --network macvlan-private nginx
docker run -d --name c2 --network macvlan-private nginx

# c1 ❌ c2 (차단)
# c1/c2 ↔ 외부 통신 가능
```

#### VEPA 모드

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  -o parent=eth0 \
  -o macvlan_mode=vepa \
  macvlan-vepa

# 모든 트래픽이 외부 스위치 경유
# 스위치에서 정책 적용 가능
# (Hairpin mode 필요)
```

---

### 5. IP 관리 전략

#### 정적 IP 할당

```bash
# IP 범위 지정
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --ip-range=192.168.1.128/25 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-static

# 컨테이너에 고정 IP
docker run -d \
  --name web \
  --network macvlan-static \
  --ip 192.168.1.150 \
  nginx

# IP 확인
docker inspect web | grep IPAddress
# "IPAddress": "192.168.1.150"
```

#### DHCP 사용

```bash
# DHCP 네트워크
docker network create -d macvlan \
  -o parent=eth0 \
  macvlan-dhcp

# DHCP로 IP 할당
docker run -d \
  --name web-dhcp \
  --network macvlan-dhcp \
  nginx

# 네트워크의 DHCP 서버에서 IP 할당
docker exec web-dhcp ip addr show eth0
# inet 192.168.1.200/24 (DHCP 할당)
```

---

## 💻 실습

### 실습 1: 기본 Macvlan 설정

#### 환경 준비

```bash
# 물리 인터페이스 확인
ip link show
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>

# 네트워크 정보
ip addr show eth0
# inet 192.168.1.50/24

# 게이트웨이
ip route
# default via 192.168.1.1 dev eth0
```

#### Macvlan 네트워크 생성

```bash
# Macvlan 네트워크
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.200/29 \
  -o parent=eth0 \
  my-macvlan

# 상세 정보
docker network inspect my-macvlan

# 컨테이너 시작
docker run -d \
  --name nginx1 \
  --network my-macvlan \
  --ip 192.168.1.201 \
  nginx:alpine

docker run -d \
  --name nginx2 \
  --network my-macvlan \
  --ip 192.168.1.202 \
  nginx:alpine

# 컨테이너 간 통신
docker exec nginx1 ping -c 2 192.168.1.202
# 성공!

# 외부 통신
docker exec nginx1 ping -c 2 8.8.8.8
# 성공!

# 물리 네트워크에서 접근
# (다른 컴퓨터에서)
curl http://192.168.1.201
# Welcome to nginx!
```

---

### 실습 2: VLAN 태깅

#### VLAN 서브인터페이스 생성

```bash
# VLAN 서브인터페이스
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip link set eth0.100 up

sudo ip link add link eth0 name eth0.200 type vlan id 200
sudo ip link set eth0.200 up

# 확인
ip -d link show eth0.100
# eth0.100@eth0: ...
#     vlan protocol 802.1Q id 100

ip -d link show eth0.200
# eth0.200@eth0: ...
#     vlan protocol 802.1Q id 200
```

#### VLAN별 Macvlan 네트워크

```bash
# VLAN 100 (Frontend)
docker network create -d macvlan \
  --subnet=10.100.0.0/24 \
  --gateway=10.100.0.1 \
  -o parent=eth0.100 \
  frontend-vlan

# VLAN 200 (Backend)
docker network create -d macvlan \
  --subnet=10.200.0.0/24 \
  --gateway=10.200.0.1 \
  -o parent=eth0.200 \
  backend-vlan

# Frontend 서비스
docker run -d \
  --name web \
  --network frontend-vlan \
  --ip 10.100.0.10 \
  nginx:alpine

# Backend 서비스
docker run -d \
  --name db \
  --network backend-vlan \
  --ip 10.200.0.10 \
  postgres:alpine

# VLAN 격리 확인
docker exec web ping -c 1 10.200.0.10
# timeout (VLAN 격리됨)

# 정리
docker rm -f web db
docker network rm frontend-vlan backend-vlan
sudo ip link delete eth0.100
sudo ip link delete eth0.200
```

---

### 실습 3: 호스트-컨테이너 통신 설정

#### 문제 확인

```bash
# Macvlan 네트워크
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.220/30 \
  -o parent=eth0 \
  test-macvlan

# 컨테이너 시작
docker run -d \
  --name test \
  --network test-macvlan \
  --ip 192.168.1.221 \
  nginx:alpine

# 호스트에서 접근 시도
ping -c 1 192.168.1.221
# timeout (실패)

curl http://192.168.1.221
# timeout (실패)
```

#### Shim 인터페이스로 해결

```bash
# Shim 인터페이스 생성
sudo ip link add macvlan-shim link eth0 type macvlan mode bridge
sudo ip addr add 192.168.1.250/32 dev macvlan-shim
sudo ip link set macvlan-shim up

# 라우팅 추가
sudo ip route add 192.168.1.221/32 dev macvlan-shim

# 재시도
ping -c 2 192.168.1.221
# 성공! ✅

curl http://192.168.1.221
# Welcome to nginx! ✅

# 정리
docker rm -f test
docker network rm test-macvlan
sudo ip link delete macvlan-shim
```

---

### 실습 4: Macvlan vs Bridge 성능 비교

#### Bridge 네트워크

```bash
# Bridge 네트워크
docker network create bridge-test

# 컨테이너 시작
docker run -d \
  --name bridge-server \
  --network bridge-test \
  -p 8080:80 \
  nginx:alpine

# 성능 테스트
ab -n 10000 -c 100 http://localhost:8080/

# 결과:
# Requests per second:    12450.23 [#/sec]
# Time per request:       8.032 [ms]

docker rm -f bridge-server
docker network rm bridge-test
```

#### Macvlan 네트워크

```bash
# Macvlan 네트워크
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.230/30 \
  -o parent=eth0 \
  macvlan-test

# 컨테이너 시작
docker run -d \
  --name macvlan-server \
  --network macvlan-test \
  --ip 192.168.1.231 \
  nginx:alpine

# Shim 설정 (호스트 접근용)
sudo ip link add macvlan-shim link eth0 type macvlan mode bridge
sudo ip addr add 192.168.1.250/32 dev macvlan-shim
sudo ip link set macvlan-shim up
sudo ip route add 192.168.1.231/32 dev macvlan-shim

# 성능 테스트
ab -n 10000 -c 100 http://192.168.1.231/

# 결과:
# Requests per second:    14823.47 [#/sec]  (+19%)
# Time per request:       6.747 [ms]         (-16%)

# 정리
docker rm -f macvlan-server
docker network rm macvlan-test
sudo ip link delete macvlan-shim
```

#### 결과 비교

```
┌─────────────────┬────────────┬───────────┬─────────┐
│ 지표             │ Bridge     │ Macvlan   │ 개선율    │
├─────────────────┼────────────┼───────────┼─────────┤
│ Requests/sec    │ 12,450     │ 14,823    │ +19%    │
├─────────────────┼────────────┼───────────┼─────────┤
│ Time/req (ms)   │ 8.032      │ 6.747     │ -16%    │
└─────────────────┴────────────┴───────────┴─────────┘

Macvlan 장점:
- NAT 오버헤드 제거
- 직접 물리 네트워크 접근
- 15-20% 성능 향상
```

---

## 🔥 실전 적용

### 시나리오 1: 레거시 시스템 통합

**상황:**
- 기존 물리 서버들 (192.168.10.0/24)
- 새로운 컨테이너 서비스
- 동일 네트워크에서 통신 필요

**솔루션:**

```bash
# Macvlan 네트워크 (기존 네트워크와 동일)
docker network create -d macvlan \
  --subnet=192.168.10.0/24 \
  --gateway=192.168.10.1 \
  --ip-range=192.168.10.200/29 \
  -o parent=eth0 \
  legacy-integration

# 컨테이너 서비스 (API)
docker run -d \
  --name new-api \
  --network legacy-integration \
  --ip 192.168.10.201 \
  myapp:latest

# 레거시 서버에서 접근
# Legacy Server (192.168.10.50) → API (192.168.10.201)
curl http://192.168.10.201/api/v1/health
# 직접 통신! NAT 없음

# 장점:
# - 기존 방화벽 규칙 사용
# - IP 기반 접근 제어 유지
# - 네트워크 모니터링 도구 사용 가능
```

---

### 시나리오 2: 멀티 테넌트 격리

**상황:**
- 여러 고객사 서비스
- VLAN으로 격리
- 각 고객사별 독립 네트워크

**솔루션:**

```bash
# VLAN 서브인터페이스
for vlan in 10 20 30; do
  sudo ip link add link eth0 name eth0.$vlan type vlan id $vlan
  sudo ip link set eth0.$vlan up
done

# 고객사 A (VLAN 10)
docker network create -d macvlan \
  --subnet=10.10.0.0/24 \
  -o parent=eth0.10 \
  tenant-a

docker run -d \
  --name tenant-a-web \
  --network tenant-a \
  --ip 10.10.0.10 \
  nginx

# 고객사 B (VLAN 20)
docker network create -d macvlan \
  --subnet=10.20.0.0/24 \
  -o parent=eth0.20 \
  tenant-b

docker run -d \
  --name tenant-b-web \
  --network tenant-b \
  --ip 10.20.0.10 \
  nginx

# 고객사 C (VLAN 30)
docker network create -d macvlan \
  --subnet=10.30.0.0/24 \
  -o parent=eth0.30 \
  tenant-c

docker run -d \
  --name tenant-c-web \
  --network tenant-c \
  --ip 10.30.0.10 \
  nginx

# VLAN 격리:
# Tenant A ❌ Tenant B (완전 격리)
# Tenant B ❌ Tenant C (완전 격리)

# 스위치에서 VLAN 정책 적용
# - 각 VLAN별 QoS
# - 각 VLAN별 보안 규칙
# - 각 VLAN별 대역폭 제한
```

---

### 시나리오 3: IoT 디바이스 관리

**상황:**
- 수백 개의 IoT 디바이스
- 각 디바이스별 관리 컨테이너
- 물리 네트워크 직접 접근 필요

**솔루션:**

```bash
# Macvlan 네트워크 (IoT 네트워크)
docker network create -d macvlan \
  --subnet=192.168.50.0/24 \
  --gateway=192.168.50.1 \
  --ip-range=192.168.50.100/25 \
  -o parent=eth0 \
  iot-network

# 디바이스별 관리 컨테이너
for device_id in {1..10}; do
  ip=$((100 + device_id))
  docker run -d \
    --name device-manager-$device_id \
    --network iot-network \
    --ip 192.168.50.$ip \
    -e DEVICE_ID=$device_id \
    iot-manager:latest
done

# 각 컨테이너가 물리 네트워크의 IP
# IoT 디바이스에서 직접 접근 가능
# 192.168.50.101, 192.168.50.102, ...

# 장점:
# - mDNS, UPnP 등 프로토콜 지원
# - 브로드캐스트/멀티캐스트 지원
# - 기존 IoT 프로토콜 그대로 사용
```

---

## ⚡ Macvlan 체크리스트

### 네트워크 계획

```
□ 물리 네트워크 토폴로지 확인
□ IP 대역 할당 계획
□ VLAN 필요성 검토
□ 게이트웨이/라우팅 확인
□ DHCP 사용 여부 결정
```

### 인프라 준비

```
□ 물리 스위치 설정
□ Promiscuous 모드 활성화
□ VLAN 트렁크 포트 설정
□ 방화벽 규칙 조정
□ MAC 주소 테이블 크기 확인
```

### 컨테이너 배포

```
□ IP 충돌 방지
□ Shim 인터페이스 설정 (호스트 통신)
□ DNS 설정 확인
□ 라우팅 테이블 검증
□ 네트워크 성능 측정
```

### 모니터링

```
□ MAC 주소 테이블 모니터링
□ 네트워크 트래픽 분석
□ IP 할당 추적
□ 성능 메트릭 수집
□ 장애 로그 수집
```

---

## 🚫 안티패턴

### 1. 호스트 통신 미고려

```bash
# ❌ Shim 없이 배포
docker network create -d macvlan ... my-macvlan
docker run --network my-macvlan ...
# 호스트에서 접근 불가!

# ✅ Shim 인터페이스 설정
sudo ip link add macvlan-shim link eth0 type macvlan mode bridge
sudo ip addr add 192.168.1.250/32 dev macvlan-shim
sudo ip link set macvlan-shim up
```

### 2. IP 충돌

```bash
# ❌ 물리 네트워크 IP와 충돌
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  ...
docker run --ip 192.168.1.50 ...
# 물리 서버가 이미 192.168.1.50 사용 중 → 충돌!

# ✅ IP 범위 분리
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --ip-range=192.168.1.200/29 \
  ...
# 192.168.1.200-207만 컨테이너 사용
```

### 3. VLAN 미설정

```bash
# ❌ VLAN 서브인터페이스 없이 사용
docker network create -d macvlan \
  -o parent=eth0 \
  ...
# VLAN 태깅 안 됨

# ✅ VLAN 서브인터페이스 사용
sudo ip link add link eth0 name eth0.100 type vlan id 100
docker network create -d macvlan \
  -o parent=eth0.100 \
  ...
```

### 4. Promiscuous 모드 미활성화

```bash
# ❌ 스위치에서 차단
# 스위치 설정: Port Security Enabled
# Macvlan 동작 안 함

# ✅ Promiscuous 모드 허용
# 스위치 설정: Promiscuous Mode Allowed
# 또는 포트 보안 비활성화
```

---

## 🎓 핵심 정리

### 1. Macvlan 개념

```
특징:
- 물리 네트워크 직접 연결
- 컨테이너별 고유 MAC 주소
- NAT 없음
- VLAN 지원

장점:
+ 최고 성능 (NAT 오버헤드 제거)
+ 레거시 통합 용이
+ 브로드캐스트/멀티캐스트 지원
+ 기존 네트워크 도구 사용

단점:
- 호스트-컨테이너 통신 제한
- Promiscuous 모드 필요
- 복잡한 설정
- 클라우드 환경 제약
```

### 2. 사용 시나리오

```
적합:
✅ 레거시 시스템 통합
✅ 물리 네트워크 IP 필요
✅ 고성능 필요
✅ VLAN 격리

부적합:
❌ 호스트-컨테이너 빈번한 통신
❌ 클라우드 환경 (AWS, GCP)
❌ 동적 IP 환경
❌ 단순한 컨테이너 통신
```

### 3. VLAN 태깅

```
설정:
1. VLAN 서브인터페이스 생성
2. Macvlan parent로 지정
3. 컨테이너 배포

격리:
- VLAN별 독립 네트워크
- 스위치 정책 적용
- 멀티 테넌트 지원
```

### 4. 핵심 명령어

```bash
# Macvlan 네트워크
docker network create -d macvlan \
  -o parent=<interface>

# VLAN 서브인터페이스
sudo ip link add link eth0 name eth0.100 \
  type vlan id 100

# Shim 인터페이스
sudo ip link add macvlan-shim link eth0 \
  type macvlan mode bridge

# 확인
ip -d link show
docker network inspect
```

---

## 📚 참고 자료

- [Docker Macvlan Networks](https://docs.docker.com/network/macvlan/)
- [Linux Macvlan](https://www.kernel.org/doc/html/latest/networking/macvlan.html)
- [802.1Q VLAN Tagging](https://en.wikipedia.org/wiki/IEEE_802.1Q)
- [Macvlan Driver](https://github.com/moby/moby/blob/master/libnetwork/drivers/macvlan/macvlan.go)

---

## 🤔 생각해볼 문제

1. Macvlan에서 호스트-컨테이너 통신이 차단되는 이유는?
2. Macvlan이 클라우드 환경(AWS, GCP)에서 지원되지 않는 이유는?
3. Macvlan Bridge 모드와 VEPA 모드의 실질적 차이는?

> 💡 **답변**: 1) 커널 네트워크 스택의 보안 기능 - 같은 물리 인터페이스에서 생성된 Macvlan 인터페이스끼리는 직접 통신 차단, 외부 스위치를 경유해야 통신 가능 (hairpin 또는 shim 인터페이스 필요), 2) 클라우드는 가상화된 네트워크이며 보안상 Promiscuous 모드를 허용하지 않음, 각 VM/인스턴스는 정해진 MAC 주소만 사용 가능, 하이퍼바이저 레벨에서 제한, 3) Bridge는 로컬 스위칭(같은 호스트 내), VEPA는 모든 트래픽을 외부 스위치로 전송 (hairpin 필요), VEPA는 물리 스위치의 정책/ACL/QoS를 적용하려는 엔터프라이즈 환경용, 실제로는 Bridge 모드가 대부분 사용됨

---

<div align="center">

**[⬅️ 이전: Overlay Network](./04-Overlay-Network.md)** | **[다음: Custom Networks ➡️](./06-Custom-Networks.md)**

</div>
