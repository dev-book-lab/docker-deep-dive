# 03. Network Debugging - 네트워크 디버깅

## 🎯 이 챕터에서 배울 것

- **네트워크 기본**: 컨테이너 네트워킹
- **연결 테스트**: ping, curl, telnet
- **패킷 분석**: tcpdump, wireshark
- **DNS 문제**: nslookup, dig
- **포트 확인**: netstat, ss, lsof
- **실전 기법**: 네트워크 장애 해결

## 📌 왜 중요한가?

**"컨테이너 네트워크 문제는 가장 흔하면서도 디버깅하기 어려운 문제입니다."**

```
Network Debugging의 핵심:

Common Network Problems:
┌─────────────────────────────────────────────────┐
│ 1. "Connection Refused"                         │
│    - 서비스가 시작 안 됨                             │
│    - 잘못된 포트                                   │
│    - 방화벽                                       │
│                                                 │
│ 2. "Connection Timeout"                         │
│    - 네트워크 단절                                 │
│    - 방화벽 차단                                   │
│    - 잘못된 IP                                    │
│                                                 │
│ 3. "Name Resolution Failed"                     │
│    - DNS 문제                                    │
│    - /etc/hosts                                 │
│    - 잘못된 도메인                                 │
│                                                 │
│ 4. "No Route to Host"                           │
│    - 라우팅 문제                                   │
│    - 네트워크 분리                                 │
│    - IP 충돌                                     │
└─────────────────────────────────────────────────┘

Docker Networking Modes:
┌─────────────────────────────────────────────────┐
│ 1. bridge (기본)                                 │
│    - 가상 네트워크                                 │
│    - 컨테이너 간 통신                               │
│    - NAT로 외부 통신                               │
│                                                 │
│ 2. host                                         │
│    - 호스트 네트워크 직접 사용                        │
│    - 포트 충돌 주의                                │
│    - 최고 성능                                    │
│                                                 │
│ 3. none                                         │
│    - 네트워크 없음                                 │
│    - 완전 격리                                    │
│                                                 │
│ 4. container                                    │
│    - 다른 컨테이너 네트워크 공유                       │
│    - 사이드카 패턴                                  │
└─────────────────────────────────────────────────┘

OSI 7 Layer Debugging:
┌─────────────────────────────────────────────────┐
│ Layer 7 (Application): curl, wget               │
│   ↓                                             │
│ Layer 4 (Transport): netstat, ss                │
│   ↓                                             │
│ Layer 3 (Network): ping, traceroute             │
│   ↓                                             │
│ Layer 2 (Data Link): tcpdump, arp               │
│   ↓                                             │
│ Layer 1 (Physical): 물리적 연결                    │
└─────────────────────────────────────────────────┘

Network Debugging Flow:
┌─────────────────────────────────────────────────┐
│ 1. 네트워크 모드 확인                                │
│    docker inspect myapp | grep NetworkMode      │
│                                                 │
│ 2. IP 주소 확인                                   │
│    docker exec myapp ip addr                    │
│                                                 │
│ 3. 라우팅 확인                                     │
│    docker exec myapp ip route                   │
│                                                 │
│ 4. DNS 확인                                      │
│    docker exec myapp nslookup google.com        │
│                                                 │
│ 5. 연결 테스트                                     │
│    docker exec myapp curl -v http://api/health  │
│                                                 │
│ 6. 포트 확인                                      │
│    docker exec myapp netstat -tlnp              │
│                                                 │
│ 7. 패킷 캡처 (심층)                                │
│    docker exec myapp tcpdump -i any             │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **다운타임 최소화**: 네트워크 문제 빠른 해결
- **성능 최적화**: 병목 지점 발견
- **보안**: 비정상 트래픽 감지
- **안정성**: 간헐적 문제 원인 파악

---

## 🔬 Deep Dive

### 1. 기본 연결 테스트

#### ping (ICMP)

```bash
# 컨테이너 내부에서
docker exec myapp ping google.com

# 특정 횟수
docker exec myapp ping -c 4 google.com

# IPv4만
docker exec myapp ping -4 google.com

# Timeout 설정
docker exec myapp ping -W 2 google.com
```

#### curl (HTTP)

```bash
# GET 요청
docker exec myapp curl http://api:8080/health

# Verbose (상세 정보)
docker exec myapp curl -v http://api:8080

# Header 확인
docker exec myapp curl -I http://api:8080

# Timeout
docker exec myapp curl --max-time 5 http://api:8080

# DNS 확인 포함
docker exec myapp curl -v http://api:8080 2>&1 | grep "Trying"
```

#### telnet (포트 확인)

```bash
# 포트 열림 확인
docker exec myapp telnet api 8080

# 성공: Connected
# 실패: Connection refused

# nc (netcat) 사용
docker exec myapp nc -zv api 8080
```

---

### 2. DNS 문제 해결

#### nslookup

```bash
# 기본 조회
docker exec myapp nslookup google.com

# 특정 DNS 서버 사용
docker exec myapp nslookup google.com 8.8.8.8

# 레코드 타입 지정
docker exec myapp nslookup -type=A google.com
```

#### dig

```bash
# 상세 DNS 조회
docker exec myapp dig google.com

# 간단한 답변만
docker exec myapp dig +short google.com

# 역방향 조회
docker exec myapp dig -x 8.8.8.8

# Trace
docker exec myapp dig +trace google.com
```

---

## 🔧 실습 1: 기본 네트워크 진단

### Step 1: 네트워크 설정 확인

```bash
# 1. 컨테이너 네트워크 모드
docker inspect myapp | jq '.[0].HostConfig.NetworkMode'

# 2. IP 주소
docker exec myapp ip addr show

# 3. 라우팅 테이블
docker exec myapp ip route

# 4. DNS 설정
docker exec myapp cat /etc/resolv.conf

# 5. Hosts 파일
docker exec myapp cat /etc/hosts
```

### Step 2: 연결 테스트

```bash
# 1. 외부 연결 (인터넷)
docker exec myapp ping -c 3 8.8.8.8

# 2. DNS 해석
docker exec myapp ping -c 3 google.com

# 3. 다른 컨테이너 (서비스 이름)
docker exec myapp ping -c 3 database

# 4. 특정 포트
docker exec myapp curl http://database:5432
docker exec myapp nc -zv database 5432
```

---

## 🔧 실습 2: tcpdump를 이용한 패킷 분석

### Step 1: 기본 사용법

```bash
# tcpdump 설치 (alpine)
docker exec myapp apk add tcpdump

# 모든 인터페이스
docker exec myapp tcpdump -i any

# 특정 인터페이스
docker exec myapp tcpdump -i eth0

# 패킷 개수 제한
docker exec myapp tcpdump -i any -c 10

# 파일로 저장
docker exec myapp tcpdump -i any -w /tmp/capture.pcap

# 파일 복사 (호스트로)
docker cp myapp:/tmp/capture.pcap ./capture.pcap

# Wireshark로 분석
wireshark capture.pcap
```

### Step 2: 필터링

```bash
# 특정 포트 (HTTP)
docker exec myapp tcpdump -i any port 80

# 특정 호스트
docker exec myapp tcpdump -i any host 10.0.0.5

# 여러 조건 (AND)
docker exec myapp tcpdump -i any 'host 10.0.0.5 and port 80'

# 여러 조건 (OR)
docker exec myapp tcpdump -i any 'port 80 or port 443'

# TCP만
docker exec myapp tcpdump -i any tcp

# SYN 패킷만 (연결 시작)
docker exec myapp tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0'
```

### Step 3: 실전 디버깅

```bash
# 문제: API 호출이 느림
# 1. 패킷 캡처 시작
docker exec -d myapp tcpdump -i any port 8080 -w /tmp/slow.pcap

# 2. 문제 재현
docker exec myapp curl http://api:8080/slow-endpoint

# 3. 캡처 중지 및 복사
docker exec myapp pkill tcpdump
docker cp myapp:/tmp/slow.pcap ./

# 4. Wireshark 분석
# - TCP 재전송 확인
# - 응답 시간 확인
# - 패킷 손실 확인
```

---

## 🔧 실습 3: 포트 및 연결 상태 확인

### Step 1: netstat

```bash
# 모든 연결
docker exec myapp netstat -a

# TCP 연결
docker exec myapp netstat -t

# Listening 포트
docker exec myapp netstat -tln

# 프로세스 포함
docker exec myapp netstat -tlnp

# 통계
docker exec myapp netstat -s

# 라우팅 테이블
docker exec myapp netstat -r
```

### Step 2: ss (modern netstat)

```bash
# TCP Listening
docker exec myapp ss -tln

# UDP
docker exec myapp ss -uln

# 프로세스 포함
docker exec myapp ss -tlnp

# Established 연결
docker exec myapp ss -tn state established

# 연결 수
docker exec myapp ss -s
```

### Step 3: lsof (파일 디스크립터)

```bash
# 특정 포트 사용 프로세스
docker exec myapp lsof -i :8080

# TCP 연결
docker exec myapp lsof -i tcp

# 특정 프로세스의 네트워크
docker exec myapp lsof -p 1 -a -i
```

---

## 🔧 실습 4: DNS 문제 해결

### Step 1: DNS 설정 확인

```bash
# 1. resolv.conf
docker exec myapp cat /etc/resolv.conf
# nameserver 8.8.8.8

# 2. 직접 DNS 쿼리
docker exec myapp nslookup google.com
docker exec myapp nslookup google.com 8.8.8.8

# 3. dig 상세 정보
docker exec myapp dig google.com

# 4. /etc/hosts 확인
docker exec myapp cat /etc/hosts
```

### Step 2: DNS 문제 패턴

```bash
# 문제 1: DNS 서버 응답 없음
docker exec myapp nslookup google.com
# ;; connection timed out

# 해결: DNS 서버 변경
docker run --dns 8.8.8.8 myapp

# 문제 2: 서비스 이름 해석 안 됨
docker exec myapp ping database
# ping: unknown host database

# 확인: 같은 네트워크인가?
docker network inspect bridge

# 해결: 같은 네트워크 사용
docker network create mynetwork
docker run --network mynetwork --name db postgres
docker run --network mynetwork myapp
```

---

## 🔧 실습 5: Docker 네트워크 디버깅

### Step 1: 네트워크 검사

```bash
# 네트워크 목록
docker network ls

# 네트워크 상세 정보
docker network inspect bridge

# 컨테이너 네트워크 확인
docker inspect myapp | jq '.[0].NetworkSettings'

# IP 주소
docker inspect myapp | jq '.[0].NetworkSettings.IPAddress'

# 게이트웨이
docker inspect myapp | jq '.[0].NetworkSettings.Gateway'
```

### Step 2: 네트워크 생성 및 연결

```bash
# 커스텀 네트워크 생성
docker network create --driver bridge mynetwork

# 서브넷 지정
docker network create \
  --subnet=172.20.0.0/16 \
  --gateway=172.20.0.1 \
  mynetwork

# 컨테이너 연결
docker network connect mynetwork myapp

# 연결 해제
docker network disconnect mynetwork myapp

# 네트워크와 함께 시작
docker run --network mynetwork myapp
```

### Step 3: Host 네트워크

```bash
# Host 네트워크 사용
docker run --network host myapp

# 장점: 최고 성능, 포트 매핑 불필요
# 단점: 포트 충돌 가능, 격리 없음

# 확인
docker exec myapp ip addr
# 호스트와 동일한 IP
```

---

## 🔧 실습 6: 네트워크 문제 해결 시나리오

### 시나리오 1: Connection Refused

```bash
# 증상
docker exec app curl http://api:8080
# curl: (7) Failed to connect to api port 8080: Connection refused

# 디버깅
# 1. 서비스 실행 중인가?
docker exec api ps aux | grep java

# 2. 포트 Listening?
docker exec api netstat -tln | grep 8080
# 없으면 → 서비스가 다른 포트 또는 시작 안 됨

# 3. 방화벽?
docker exec api iptables -L

# 4. 같은 네트워크?
docker network inspect bridge | grep -A 20 Containers
```

### 시나리오 2: Connection Timeout

```bash
# 증상
docker exec app curl --max-time 5 http://api:8080
# curl: (28) Connection timeout

# 디버깅
# 1. 네트워크 연결?
docker exec app ping api
# 실패 → 네트워크 문제

# 2. 라우팅?
docker exec app ip route
docker exec app traceroute api

# 3. 패킷 도달?
docker exec api tcpdump -i any port 8080
# 패킷 안 옴 → 네트워크 경로 문제

# 4. 방화벽?
docker exec api iptables -L -n
```

### 시나리오 3: Intermittent Failures

```bash
# 증상: 간헐적 실패

# 1. DNS 캐시?
docker exec app cat /etc/resolv.conf

# 2. 로드 밸런싱?
for i in {1..10}; do
  docker exec app curl -s http://api:8080 | grep -q "OK" && echo "Success" || echo "Fail"
done

# 3. 연결 풀 고갈?
docker exec app netstat -an | grep ESTABLISHED | wc -l

# 4. 간헐적 네트워크 문제
docker exec app ping -i 0.1 api | tee ping.log
# 패킷 손실 확인
```

---

## 💡 주요 명령어 정리

```
┌──────────────────────┬────────────────────────────┐
│ 명령어                 │ 용도                        │
├──────────────────────┼────────────────────────────┤
│ ping                 │ 연결 테스트 (ICMP)            │
├──────────────────────┼────────────────────────────┤
│ curl                 │ HTTP 테스트                  │
├──────────────────────┼────────────────────────────┤
│ tcpdump              │ 패킷 캡처                    │
├──────────────────────┼────────────────────────────┤
│ nslookup/dig         │ DNS 조회                    │
├──────────────────────┼────────────────────────────┤
│ netstat/ss           │ 연결 상태                    │
├──────────────────────┼────────────────────────────┤
│ ip addr/route        │ 네트워크 설정                 │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. Layer by layer (OSI)
2. 간단한 것부터 (ping)
3. 패킷 캡처 (마지막)
4. 로그 확인 병행
5. 재현 가능하게
```

---

## 🎓 연습 문제

### 문제 1: 컨테이너끼리 통신이 안 된다면?

<details>
<summary>정답 보기</summary>

**체크리스트:**

**1. 같은 네트워크?**
```bash
docker network inspect bridge
# 두 컨테이너 모두 있는지 확인
```

**2. 서비스 이름 사용?**
```bash
# ❌ IP로 (변경될 수 있음)
curl http://172.17.0.2:8080

# ✅ 서비스 이름
curl http://api:8080
```

**3. 포트 맞는가?**
```bash
docker exec api netstat -tln
# Listening 포트 확인
```

**4. 방화벽?**
```bash
docker exec api iptables -L
```

**해결:**
```bash
# 같은 네트워크 사용
docker network create app-network
docker run --network app-network --name api myapi
docker run --network app-network --name web myweb

# 확인
docker exec web curl http://api:8080
```

</details>

### 문제 2: 외부에서 컨테이너에 접근이 안 된다면?

<details>
<summary>정답 보기</summary>

**원인:**

**1. 포트 매핑 안 됨**
```bash
# 확인
docker ps
# PORTS 컬럼 확인

# 해결
docker run -p 8080:8080 myapp
```

**2. 0.0.0.0 바인딩**
```bash
# ❌ 잘못된 바인딩
app.run(host='127.0.0.1', port=8080)
# localhost만 접근 가능

# ✅ 올바른 바인딩
app.run(host='0.0.0.0', port=8080)
# 모든 인터페이스
```

**3. 방화벽**
```bash
# 호스트 방화벽 확인
sudo iptables -L
sudo ufw status

# 필요 시 포트 개방
sudo ufw allow 8080
```

**4. 네트워크 모드**
```bash
# Host 네트워크 사용
docker run --network host myapp
# 포트 매핑 불필요
```

</details>

### 문제 3: 네트워크 성능이 느리다면?

<details>
<summary>정답 보기</summary>

**측정:**

**1. 대역폭**
```bash
# iperf3 설치
docker exec server iperf3 -s
docker exec client iperf3 -c server
```

**2. 레이턴시**
```bash
docker exec app ping -c 100 api
# RTT min/avg/max
```

**3. 패킷 손실**
```bash
docker exec app ping -c 100 -i 0.1 api | grep loss
```

**최적화:**

**1. Host 네트워크**
```bash
docker run --network host myapp
# 최고 성능, NAT 없음
```

**2. MTU 조정**
```bash
docker network create \
  --opt com.docker.network.driver.mtu=9000 \
  mynetwork
```

**3. --net=container (사이드카)**
```bash
docker run --name app myapp
docker run --net=container:app sidecar
# 같은 네트워크 스택, localhost 통신
```

**4. 불필요한 NAT 제거**
```bash
# Bridge → Host or Overlay
```

</details>

---

## 📌 핵심 요약

```
Network Debugging 핵심:
1. Layer by layer (ping → curl → tcpdump)
2. 네트워크 모드 확인
3. DNS 문제 자주 발생
4. 같은 네트워크 사용
5. 패킷 캡처 (마지막 수단)

Best Practices:
✅ 서비스 이름 사용
✅ 같은 네트워크
✅ 0.0.0.0 바인딩
✅ 포트 확인
✅ DNS 설정 확인
```

---

## 📚 참고 자료

- [Docker Networking](https://docs.docker.com/network/)
- [tcpdump Tutorial](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Wireshark User Guide](https://www.wireshark.org/docs/)

---

## 🤔 생각해볼 문제

1. 프로덕션에서 tcpdump를 사용해도 되는가?
2. Service Mesh (Istio)를 사용하면 디버깅이 쉬워지는가?
3. IPv6 컨테이너 네트워킹은?

> 💡 **답변**:
> 
> **1) 프로덕션 tcpdump:**
> 
> ```
> ⚠️ 주의:
> - CPU 영향 (5-10%)
> - 디스크 공간 (빠르게 증가)
> - 민감 정보 (패킷 내용)
> 
> 권장:
> - 짧은 시간 (1-2분)
> - 필터 사용 (특정 포트만)
> - 복제본 중 1개만
> - Off-peak 시간
> ```
> 
> **2) Service Mesh 디버깅:**
> 
> ```
> 장점:
> - 자동 트레이싱
> - 트래픽 시각화
> - 메트릭 자동 수집
> 
> 단점:
> - 복잡도 증가
> - 추가 레이턴시
> - 리소스 오버헤드
> 
> 권장: 마이크로서비스 많을 때
> ```
> 
> **3) IPv6:**
> 
> ```bash
> # IPv6 활성화
> {
>   "ipv6": true,
>   "fixed-cidr-v6": "2001:db8:1::/64"
> }
> 
> # 네트워크 생성
> docker network create \
>   --ipv6 \
>   --subnet=2001:db8:1::/64 \
>   mynetwork
> 
> # 주의: 아직 일부 제한
> ```

---

<div align="center">

**[⬅️ 이전: Log Analysis](02-Log-Analysis.md)** | **[다음: Performance Issues ➡️](./04-Performance-Issues.md)**

</div>
