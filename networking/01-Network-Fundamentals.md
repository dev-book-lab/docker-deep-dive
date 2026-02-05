# 01. Network Fundamentals - 네트워크 기초

## 🎯 이 챕터에서 배울 것

- **컨테이너 네트워킹**의 기본 원리
- **veth pair**와 **bridge**의 동작 방식
- **iptables**를 통한 패킷 라우팅
- 컨테이너 간 통신의 **완전한 패킷 흐름**

## 📌 왜 중요한가?

**"네트워크를 이해하지 못하면 컨테이너 문제의 80%를 해결할 수 없습니다."**

```
네트워크 이해 없이:
- 컨테이너 간 통신 안 됨
- 외부 접근 불가
- 성능 문제 해결 못 함
- 보안 구성 불가

네트워크 이해 후:
- 모든 통신 경로 파악
- 문제 즉시 해결
- 성능 최적화
- 보안 강화
```

**실무 영향:**
- 트러블슈팅: 네트워크 문제 90% 자체 해결
- 성능: 불필요한 hop 제거로 지연시간 감소
- 보안: 정확한 방화벽 규칙 설정
- 설계: 효율적인 네트워크 아키텍처

---

## 🔬 Deep Dive

### 1. 컨테이너 네트워킹 개요

#### 기본 개념

```
컨테이너 네트워킹 핵심 구성 요소:

1. Network Namespace
   - 각 컨테이너의 독립적인 네트워크 스택
   - 자체 IP, 라우팅 테이블, 방화벽

2. veth pair (Virtual Ethernet)
   - 가상 네트워크 케이블
   - 한쪽은 컨테이너, 한쪽은 호스트

3. Bridge
   - 가상 스위치
   - 여러 veth를 연결

4. iptables
   - 패킷 필터링 및 NAT
   - 포트 포워딩

5. 라우팅
   - 패킷 경로 결정
   - 외부 통신 처리
```

#### 전체 구조

```
┌─────────────────────────────────────────────────────────┐
│ Host                                                    │
│                                                         │
│  ┌──────────────┐         ┌──────────────┐              │
│  │ Container A  │         │ Container B  │              │
│  │              │         │              │              │
│  │ eth0         │         │ eth0         │              │
│  │ 172.17.0.2   │         │ 172.17.0.3   │              │
│  └──────┬───────┘         └──────┬───────┘              │
│         │ veth1a                 │ veth1b               │
│         │                        │                      │
│         └────────┬───────────────┘                      │
│                  │                                      │
│            ┌─────▼──────┐                               │
│            │  docker0   │ (bridge)                      │
│            │ 172.17.0.1 │                               │
│            └─────┬──────┘                               │
│                  │                                      │
│            ┌─────▼──────┐                               │
│            │   eth0     │ (host interface)              │
│            │ 10.0.0.100 │                               │
│            └─────┬──────┘                               │
└──────────────────┼──────────────────────────────────────┘
                   │
              ┌────▼────┐
              │ Internet│
              └─────────┘

통신 경로:
Container A → veth1a → docker0 → veth1b → Container B
Container A → veth1a → docker0 → eth0 → Internet
```

---

### 2. Network Namespace

#### Namespace란?

```
Linux Network Namespace:
- 독립적인 네트워크 스택
- 컨테이너별로 격리된 네트워크 환경

각 Namespace는 가짐:
- 네트워크 인터페이스
- IP 주소
- 라우팅 테이블
- iptables 규칙
- 소켓
```

#### Namespace 실습

```bash
# 현재 네트워크 네임스페이스 확인
ip netns list

# 새 네임스페이스 생성
sudo ip netns add demo-ns

# 생성 확인
ip netns list
# demo-ns

# 네임스페이스 내부에서 명령 실행
sudo ip netns exec demo-ns ip addr
# 1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
# (아무 인터페이스도 없음)

# 네임스페이스 내부 쉘 시작
sudo ip netns exec demo-ns bash

# 이제 demo-ns 안에 있음
ip addr
# 1: lo만 있음

# loopback 활성화
ip link set lo up

exit

# 네임스페이스 삭제
sudo ip netns delete demo-ns
```

---

### 3. veth pair (Virtual Ethernet)

#### veth의 원리

```
veth pair = 가상 네트워크 케이블

특징:
- 항상 쌍으로 생성됨
- 한쪽에서 보낸 패킷이 다른 쪽에서 나옴
- 터널처럼 동작

사용:
┌──────────────┐         ┌──────────────┐
│  Namespace A │         │  Namespace B │
│              │         │              │
│   veth-a ◄───┼─────────┼───► veth-b   │
│              │  pair   │              │
└──────────────┘         └──────────────┘

veth-a로 보낸 패킷 → veth-b에서 수신
```

#### veth 생성 및 연결

```bash
# 1. 두 개의 네임스페이스 생성
sudo ip netns add ns1
sudo ip netns add ns2

# 2. veth pair 생성
sudo ip link add veth1 type veth peer name veth2

# 3. 확인
ip link show type veth
# veth1@veth2: ...
# veth2@veth1: ...

# 4. veth를 각 네임스페이스에 할당
sudo ip link set veth1 netns ns1
sudo ip link set veth2 netns ns2

# 5. 각 veth에 IP 할당
sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth1
sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth2

# 6. 인터페이스 활성화
sudo ip netns exec ns1 ip link set veth1 up
sudo ip netns exec ns2 ip link set veth2 up

# 7. 통신 테스트
sudo ip netns exec ns1 ping -c 2 10.0.0.2
# PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
# 64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.045 ms
# 성공! ✅

# 정리
sudo ip netns delete ns1
sudo ip netns delete ns2
```

---

### 4. Linux Bridge

#### Bridge의 역할

```
Linux Bridge = 가상 스위치

기능:
- 여러 네트워크 인터페이스 연결
- MAC 주소 기반 패킷 전달
- Layer 2 스위칭

구조:
┌─────────┐  ┌─────────┐  ┌─────────┐
│ veth-1  │  │ veth-2  │  │ veth-3  │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  │
            ┌─────▼──────┐
            │  br0       │ (bridge)
            │  (switch)  │
            └────────────┘

패킷 전달:
veth-1 → br0 → (MAC 학습) → veth-2
```

#### Bridge 생성 및 설정

```bash
# 1. Bridge 생성
sudo ip link add br0 type bridge

# 2. IP 할당 및 활성화
sudo ip addr add 10.0.0.1/24 dev br0
sudo ip link set br0 up

# 3. 네임스페이스와 veth pair 생성
for i in 1 2 3; do
  # 네임스페이스
  sudo ip netns add ns$i
  
  # veth pair
  sudo ip link add veth$i type veth peer name veth${i}-br
  
  # 한쪽은 네임스페이스로
  sudo ip link set veth$i netns ns$i
  sudo ip netns exec ns$i ip addr add 10.0.0.$((i+1))/24 dev veth$i
  sudo ip netns exec ns$i ip link set veth$i up
  
  # 다른 쪽은 bridge에 연결
  sudo ip link set veth${i}-br master br0
  sudo ip link set veth${i}-br up
done

# 4. Bridge 상태 확인
ip link show master br0
# veth1-br@if...: ...master br0...
# veth2-br@if...: ...master br0...
# veth3-br@if...: ...master br0...

# 5. 통신 테스트
sudo ip netns exec ns1 ping -c 1 10.0.0.3
# 성공! ns1 ↔ ns2, ns1 ↔ ns3 모두 통신 가능

# Bridge MAC 학습 테이블 확인
sudo bridge fdb show br br0
# 각 veth의 MAC 주소가 학습됨

# 정리
sudo ip link delete br0
for i in 1 2 3; do
  sudo ip netns delete ns$i
done
```

---

### 5. iptables와 NAT

#### iptables 기본

```
iptables = Linux 방화벽

체인 (Chain):
- PREROUTING: 패킷 도착 직후
- INPUT: 로컬 프로세스로 향하는 패킷
- FORWARD: 라우팅될 패킷
- OUTPUT: 로컬 프로세스에서 나가는 패킷
- POSTROUTING: 패킷 송신 직전

테이블 (Table):
- filter: 패킷 필터링 (허용/차단)
- nat: 주소 변환 (SNAT, DNAT)
- mangle: 패킷 수정

흐름:
[Packet In] → PREROUTING → FORWARD → POSTROUTING → [Packet Out]
                    │                      ↑
                    ↓                      │
                  INPUT → [Local Process] → OUTPUT
```

#### NAT (Network Address Translation)

```
SNAT (Source NAT):
- 출발지 IP 변경
- 컨테이너 → 외부 통신

DNAT (Destination NAT):
- 목적지 IP/포트 변경
- 외부 → 컨테이너 (포트 포워딩)

Masquerade:
- 동적 SNAT
- 출발지 IP를 호스트 IP로 변경

예시:
컨테이너 (172.17.0.2) → 인터넷 (8.8.8.8)
POSTROUTING: 172.17.0.2 → 10.0.0.100 (호스트 IP)
```

#### iptables 규칙 확인

```bash
# NAT 규칙 확인
sudo iptables -t nat -L -n -v

# Chain POSTROUTING
# ...
# MASQUERADE  all  --  172.17.0.0/16  0.0.0.0/0
# (컨테이너 → 외부: IP 변환)

# Chain DOCKER
# DNAT  tcp  --  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:80
# (외부 → 컨테이너: 포트 포워딩)

# 필터 규칙 확인
sudo iptables -t filter -L -n -v

# Chain FORWARD
# DOCKER-ISOLATION  ...
# DOCKER  ...
# ACCEPT  ...
```

---

### 6. 완전한 패킷 흐름 분석

#### 시나리오 1: 컨테이너 간 통신

```
Container A (172.17.0.2) → Container B (172.17.0.3)

1. Container A에서 패킷 생성
   src: 172.17.0.2, dst: 172.17.0.3

2. veth-a (Container A 측)
   패킷이 veth로 전달

3. veth-a-br (Bridge 측)
   Bridge에 도착

4. docker0 (Bridge)
   MAC 테이블 확인
   172.17.0.3 → veth-b-br

5. veth-b-br
   패킷 전달

6. veth-b (Container B 측)
   Container B로 전달

7. Container B 수신
   성공! ✅

특징:
- Layer 2 스위칭
- NAT 불필요
- 빠름 (direct)
```

#### 시나리오 2: 컨테이너 → 인터넷

```
Container A (172.17.0.2) → Google DNS (8.8.8.8)

1. Container A에서 패킷 생성
   src: 172.17.0.2, dst: 8.8.8.8

2. veth-a → veth-a-br → docker0
   Bridge에 도착

3. docker0의 라우팅 결정
   목적지 8.8.8.8는 docker0 네트워크 밖
   → 호스트 네트워크로 포워딩

4. iptables FORWARD 체인
   DOCKER-ISOLATION, DOCKER 체인 통과
   ACCEPT 규칙 매칭

5. iptables POSTROUTING 체인
   MASQUERADE 규칙 적용
   src: 172.17.0.2 → 10.0.0.100 (호스트 IP)

6. 호스트 eth0
   패킷 송신: src 10.0.0.100, dst 8.8.8.8

7. 인터넷으로 전달
   성공! ✅

응답 패킷:
8.8.8.8 → 10.0.0.100
PREROUTING: conntrack 확인
FORWARD: ACCEPT
→ docker0 → veth-a-br → veth-a → Container A

특징:
- NAT 적용 (MASQUERADE)
- conntrack으로 응답 추적
```

#### 시나리오 3: 외부 → 컨테이너 (포트 포워딩)

```
외부 (203.0.113.50) → Host:8080 → Container:80

1. 외부에서 패킷 도착
   src: 203.0.113.50, dst: 10.0.0.100:8080

2. iptables PREROUTING
   DOCKER 체인의 DNAT 규칙 적용
   dst: 10.0.0.100:8080 → 172.17.0.2:80

3. 라우팅 결정
   dst 172.17.0.2는 docker0 네트워크
   → docker0로 포워딩

4. iptables FORWARD
   DOCKER 체인 통과
   ACCEPT

5. docker0 → veth-a-br → veth-a
   Container A로 전달

6. Container A 수신
   src: 203.0.113.50, dst: 172.17.0.2:80
   성공! ✅

응답 패킷:
Container A → ... → POSTROUTING (SNAT)
src: 172.17.0.2 → 10.0.0.100
→ 외부로 송신

특징:
- DNAT (포트 포워딩)
- SNAT (응답 시)
```

---

## 💻 실습

### 실습 1: veth pair로 직접 통신

#### 기본 veth 연결

```bash
# 네임스페이스 생성
sudo ip netns add red
sudo ip netns add blue

# veth pair 생성
sudo ip link add veth-red type veth peer name veth-blue

# 각 네임스페이스에 할당
sudo ip link set veth-red netns red
sudo ip link set veth-blue netns blue

# IP 설정
sudo ip netns exec red ip addr add 192.168.1.1/24 dev veth-red
sudo ip netns exec blue ip addr add 192.168.1.2/24 dev veth-blue

# 인터페이스 활성화
sudo ip netns exec red ip link set veth-red up
sudo ip netns exec red ip link set lo up
sudo ip netns exec blue ip link set veth-blue up
sudo ip netns exec blue ip link set lo up

# 통신 테스트
sudo ip netns exec red ping -c 3 192.168.1.2
# PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
# 64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=0.051 ms
# 성공! ✅

# 라우팅 테이블 확인
sudo ip netns exec red ip route
# 192.168.1.0/24 dev veth-red scope link

# 패킷 캡처 (별도 터미널)
sudo ip netns exec red tcpdump -i veth-red -n
# 패킷 흐름 실시간 확인

# 정리
sudo ip netns delete red
sudo ip netns delete blue
```

---

### 실습 2: Bridge로 여러 컨테이너 연결

#### Bridge 기반 네트워크 구성

```bash
# 1. Bridge 생성
sudo ip link add dev mybr0 type bridge
sudo ip addr add 192.168.100.1/24 dev mybr0
sudo ip link set mybr0 up

# 2. 3개의 "컨테이너" 시뮬레이션
for i in 1 2 3; do
  # 네임스페이스
  sudo ip netns add container$i
  
  # veth pair
  sudo ip link add veth$i type veth peer name veth${i}-br
  
  # 컨테이너 측 설정
  sudo ip link set veth$i netns container$i
  sudo ip netns exec container$i ip link set veth$i up
  sudo ip netns exec container$i ip link set lo up
  sudo ip netns exec container$i ip addr add 192.168.100.$((i+1))/24 dev veth$i
  
  # 게이트웨이 설정
  sudo ip netns exec container$i ip route add default via 192.168.100.1
  
  # Bridge 측 설정
  sudo ip link set veth${i}-br master mybr0
  sudo ip link set veth${i}-br up
done

# 3. Bridge 확인
ip -d link show mybr0
# mybr0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
#     bridge ...

brctl show mybr0  # 또는
bridge link show
# veth1-br, veth2-br, veth3-br이 연결됨

# 4. 통신 테스트
# container1 → container2
sudo ip netns exec container1 ping -c 2 192.168.100.3
# 성공!

# container2 → container3
sudo ip netns exec container2 ping -c 2 192.168.100.4
# 성공!

# 5. Bridge MAC 테이블
sudo bridge fdb show br mybr0
# MAC 주소와 포트 매핑 확인

# 6. 패킷 추적
# 터미널 1: 브리지에서 캡처
sudo tcpdump -i mybr0 -n icmp

# 터미널 2: ping 실행
sudo ip netns exec container1 ping -c 1 192.168.100.3

# 터미널 1 출력:
# 192.168.100.2 > 192.168.100.3: ICMP echo request
# 192.168.100.3 > 192.168.100.2: ICMP echo reply

# 정리
sudo ip link delete mybr0
for i in 1 2 3; do
  sudo ip netns delete container$i
done
```

---

### 실습 3: NAT 및 외부 통신

#### 컨테이너에서 인터넷 접근

```bash
# 1. Bridge와 컨테이너 생성
sudo ip link add dev mybr0 type bridge
sudo ip addr add 10.10.0.1/24 dev mybr0
sudo ip link set mybr0 up

sudo ip netns add test-ns
sudo ip link add veth0 type veth peer name veth0-br
sudo ip link set veth0 netns test-ns
sudo ip link set veth0-br master mybr0
sudo ip link set veth0-br up

sudo ip netns exec test-ns ip addr add 10.10.0.2/24 dev veth0
sudo ip netns exec test-ns ip link set veth0 up
sudo ip netns exec test-ns ip link set lo up
sudo ip netns exec test-ns ip route add default via 10.10.0.1

# 2. 현재 상태: 외부 통신 안 됨
sudo ip netns exec test-ns ping -c 1 8.8.8.8
# Network unreachable 또는 timeout

# 3. IP 포워딩 활성화
sudo sysctl -w net.ipv4.ip_forward=1

# 4. NAT 규칙 추가
sudo iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -j MASQUERADE

# 5. 다시 테스트
sudo ip netns exec test-ns ping -c 3 8.8.8.8
# PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
# 64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=10.2 ms
# 성공! ✅

# 6. NAT 규칙 확인
sudo iptables -t nat -L POSTROUTING -n -v
# MASQUERADE  all  --  10.10.0.0/24  0.0.0.0/0

# 7. conntrack 확인
sudo conntrack -L | grep 10.10.0.2
# icmp ... src=10.10.0.2 dst=8.8.8.8 ...
# (NAT 세션 추적 확인)

# 정리
sudo iptables -t nat -D POSTROUTING -s 10.10.0.0/24 -j MASQUERADE
sudo ip link delete mybr0
sudo ip netns delete test-ns
```

---

### 실습 4: 포트 포워딩 (DNAT)

#### 외부에서 컨테이너 서비스 접근

```bash
# 1. "웹 서버" 컨테이너 생성
sudo ip netns add webserver

sudo ip link add veth-web type veth peer name veth-web-br

sudo ip link add dev mybr0 type bridge
sudo ip addr add 10.20.0.1/24 dev mybr0
sudo ip link set mybr0 up

sudo ip link set veth-web netns webserver
sudo ip link set veth-web-br master mybr0
sudo ip link set veth-web-br up

sudo ip netns exec webserver ip addr add 10.20.0.2/24 dev veth-web
sudo ip netns exec webserver ip link set veth-web up
sudo ip netns exec webserver ip link set lo up
sudo ip netns exec webserver ip route add default via 10.20.0.1

# 2. 간단한 웹 서버 실행
sudo ip netns exec webserver python3 -m http.server 8000 &
WEB_PID=$!

# 3. 현재 상태: 외부에서 접근 불가
curl http://10.20.0.2:8000
# No route to host

# 4. IP 포워딩
sudo sysctl -w net.ipv4.ip_forward=1

# 5. 포트 포워딩 규칙 (DNAT)
# Host:9000 → Container:8000
HOST_IP=$(ip -4 addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -1)
sudo iptables -t nat -A PREROUTING -p tcp --dport 9000 -j DNAT --to-destination 10.20.0.2:8000

# 6. FORWARD 체인 허용
sudo iptables -A FORWARD -d 10.20.0.2 -p tcp --dport 8000 -j ACCEPT

# 7. NAT (MASQUERADE)
sudo iptables -t nat -A POSTROUTING -s 10.20.0.0/24 -j MASQUERADE

# 8. 테스트
curl http://localhost:9000
# <!DOCTYPE HTML> ...
# 성공! ✅

# 9. 규칙 확인
sudo iptables -t nat -L PREROUTING -n -v
# DNAT  tcp dpt:9000 to:10.20.0.2:8000

# 정리
kill $WEB_PID
sudo iptables -t nat -F PREROUTING
sudo iptables -t nat -F POSTROUTING
sudo iptables -F FORWARD
sudo ip link delete mybr0
sudo ip netns delete webserver
```

---

## 🔥 실전 적용

### 시나리오 1: Docker 네트워크 디버깅

**문제: 컨테이너가 인터넷에 연결 안 됨**

```bash
# 1. 컨테이너 시작
docker run -d --name test-web nginx

# 2. 인터넷 접근 테스트
docker exec test-web ping -c 2 8.8.8.8
# Timeout... 실패!

# 3. 진단 시작

# 컨테이너 IP 확인
docker inspect test-web | grep IPAddress
# "IPAddress": "172.17.0.2"

# 호스트에서 컨테이너로 ping
ping -c 2 172.17.0.2
# 성공 → veth/bridge는 정상

# IP 포워딩 확인
sysctl net.ipv4.ip_forward
# net.ipv4.ip_forward = 0  ← 문제 발견!

# 4. 해결
sudo sysctl -w net.ipv4.ip_forward=1

# 5. 재시도
docker exec test-web ping -c 2 8.8.8.8
# 성공! ✅

# 영구 설정
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### 시나리오 2: 컨테이너 간 통신 차단

**문제: 같은 브리지의 컨테이너가 서로 통신 안 됨**

```bash
# 1. 두 컨테이너 시작
docker run -d --name c1 nginx
docker run -d --name c2 nginx

# 2. IP 확인
C1_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' c1)
C2_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' c2)

echo "c1: $C1_IP, c2: $C2_IP"

# 3. 통신 테스트
docker exec c1 ping -c 2 $C2_IP
# Timeout... 실패!

# 4. 진단

# Bridge 확인
docker network inspect bridge | grep "com.docker.network.bridge.name"
# "com.docker.network.bridge.name": "docker0"

# iptables FORWARD 규칙 확인
sudo iptables -L FORWARD -n -v
# Chain FORWARD (policy DROP)  ← 문제!
# 기본 정책이 DROP

# 5. 해결: ACCEPT 규칙 추가
sudo iptables -I FORWARD -i docker0 -o docker0 -j ACCEPT

# 6. 재시도
docker exec c1 ping -c 2 $C2_IP
# 성공! ✅

# 정리
docker rm -f c1 c2
```

---

### 시나리오 3: 포트 충돌 디버깅

**문제: 포트 포워딩이 안 됨**

```bash
# 1. 컨테이너 시작 (포트 포워딩)
docker run -d -p 8080:80 --name web nginx

# 2. 접근 시도
curl http://localhost:8080
# Connection refused... 실패!

# 3. 진단

# 컨테이너 상태 확인
docker ps
# web ... Up ... 0.0.0.0:8080->80/tcp  ← 포트 매핑은 정상

# 포트 리스닝 확인
sudo netstat -tlnp | grep 8080
# tcp ... 0.0.0.0:8080 ... docker-proxy
# 리스닝은 하고 있음

# iptables NAT 규칙 확인
sudo iptables -t nat -L DOCKER -n
# DNAT  tcp dpt:8080 to:172.17.0.2:80  ← 규칙은 있음

# 컨테이너 내부에서 nginx 확인
docker exec web curl localhost:80
# Welcome to nginx!  ← nginx는 동작 중

# FORWARD 규칙 확인
sudo iptables -L FORWARD -n -v | grep 172.17.0.2
# (규칙 없음)  ← 문제 발견!

# 4. 해결
sudo iptables -I FORWARD -d 172.17.0.2 -p tcp --dport 80 -j ACCEPT

# 5. 재시도
curl http://localhost:8080
# Welcome to nginx!  ← 성공! ✅

# 정리
docker rm -f web
```

---

## ⚡ 네트워킹 트러블슈팅 체크리스트

### 컨테이너 → 인터넷

```
□ IP 포워딩 활성화 (sysctl net.ipv4.ip_forward)
□ MASQUERADE 규칙 (iptables -t nat -L POSTROUTING)
□ FORWARD 체인 허용
□ DNS 설정 (/etc/resolv.conf)
□ 라우팅 테이블 (ip route)
```

### 외부 → 컨테이너

```
□ 포트 포워딩 규칙 (iptables -t nat -L DOCKER)
□ FORWARD 체인 허용
□ 방화벽 (ufw, firewalld)
□ 컨테이너 서비스 리스닝 확인
□ 보안 그룹 (클라우드 환경)
```

### 컨테이너 ↔ 컨테이너

```
□ 같은 브리지 네트워크
□ FORWARD 체인 허용
□ Bridge 활성화 상태
□ veth 연결 상태
□ 네트워크 네임스페이스 확인
```

### 디버깅 명령어

```bash
# 네트워크 인터페이스
ip addr
ip link
ip -s link  # 통계

# 라우팅
ip route
ip route get 8.8.8.8

# Bridge
bridge link show
bridge fdb show

# iptables
iptables -t nat -L -n -v
iptables -t filter -L -n -v
iptables -L DOCKER -n

# 연결 추적
conntrack -L
conntrack -L -p tcp --dport 80

# 패킷 캡처
tcpdump -i docker0 -n
tcpdump -i any -n host 172.17.0.2

# 네임스페이스
ip netns list
ip netns exec <ns> <command>
```

---

## 🚫 안티패턴

### 1. IP 포워딩 비활성화

```bash
# ❌ 컨테이너가 외부 통신 못 함
sysctl net.ipv4.ip_forward=0

# ✅ 반드시 활성화
sysctl -w net.ipv4.ip_forward=1
```

### 2. 잘못된 iptables 정책

```bash
# ❌ 모든 FORWARD 차단
iptables -P FORWARD DROP
# 컨테이너 통신 불가

# ✅ 선택적 허용
iptables -P FORWARD DROP
iptables -A FORWARD -i docker0 -o docker0 -j ACCEPT
iptables -A FORWARD -i docker0 ! -o docker0 -j ACCEPT
```

### 3. Bridge IP 충돌

```bash
# ❌ 호스트 네트워크와 충돌
# Host: 192.168.1.0/24
# Bridge: 192.168.1.0/24
# 라우팅 문제 발생!

# ✅ 다른 대역 사용
# Host: 192.168.1.0/24
# Bridge: 172.17.0.0/16
```

### 4. DNS 설정 누락

```bash
# ❌ DNS 없음
docker run --dns="" nginx
# 도메인 resolv 불가

# ✅ DNS 설정
docker run --dns=8.8.8.8 --dns=8.8.4.4 nginx
```

---

## 🎓 핵심 정리

### 1. 핵심 구성 요소

```
Network Namespace:
- 독립적인 네트워크 스택
- 컨테이너별 격리

veth pair:
- 가상 네트워크 케이블
- Namespace 간 연결

Bridge:
- 가상 스위치
- Layer 2 스위칭

iptables:
- 패킷 필터링
- NAT (MASQUERADE, DNAT)
```

### 2. 통신 경로

```
컨테이너 간:
Container A → veth → bridge → veth → Container B
(Layer 2, NAT 불필요)

컨테이너 → 인터넷:
Container → veth → bridge → iptables (MASQUERADE) → eth0 → Internet
(SNAT 적용)

인터넷 → 컨테이너:
Internet → eth0 → iptables (DNAT) → bridge → veth → Container
(포트 포워딩)
```

### 3. 핵심 명령어

```bash
# 네임스페이스
ip netns add/delete/exec

# 인터페이스
ip link add/set/show
ip addr add/show

# 라우팅
ip route add/show

# Bridge
bridge link/fdb

# iptables
iptables -t nat/filter -L/A/D
```

### 4. 디버깅 순서

```
1. 컨테이너 IP 확인
2. 호스트 → 컨테이너 ping
3. 컨테이너 → 게이트웨이 ping
4. 컨테이너 → 인터넷 ping
5. IP 포워딩 확인
6. iptables 규칙 확인
7. tcpdump로 패킷 추적
```

---

## 📚 참고 자료

- [Linux Network Namespaces](https://man7.org/linux/man-pages/man7/network_namespaces.7.html)
- [veth - Virtual Ethernet Device](https://man7.org/linux/man-pages/man4/veth.4.html)
- [Linux Bridge](https://wiki.linuxfoundation.org/networking/bridge)
- [iptables Tutorial](https://www.netfilter.org/documentation/HOWTO/packet-filtering-HOWTO.html)
- [Docker Networking](https://docs.docker.com/network/)

---

## 🤔 생각해볼 문제

1. veth pair 없이 컨테이너를 네트워크에 연결할 수 있을까?
2. Bridge 대신 Router를 사용하면 어떻게 될까?
3. MASQUERADE와 SNAT의 차이는 무엇이고 언제 각각 사용할까?

> 💡 **답변**: 1) 불가능 - veth pair는 Namespace 간 유일한 연결 수단, macvlan 같은 다른 방식은 있지만 결국 가상 인터페이스 필요, 2) Bridge는 Layer 2(MAC), Router는 Layer 3(IP) - Router를 사용하면 각 컨테이너가 서로 다른 서브넷에 있어야 하고 라우팅 테이블 필요, 오버헤드 증가, 3) MASQUERADE는 동적 IP(DHCP)에 사용, 인터페이스 IP 자동 감지, SNAT는 정적 IP에 사용, 명시적 IP 지정, 성능은 SNAT이 약간 더 좋음(IP 조회 불필요)

---

<div align="center">

**[⬅️ 이전 섹션: Images](../images/07-Custom-Base-Images.md)** | **[다음: Bridge Network ➡️](./02-Bridge-Network.md)**

</div>
