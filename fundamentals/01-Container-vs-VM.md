# 01. Container vs VM - 컨테이너와 가상 머신의 근본적 차이

## 🎯 이 챕터에서 배울 것

- 컨테이너와 VM의 **근본적인 차이점**과 동작 원리
- **프로세스 격리** vs **하드웨어 가상화**의 본질적 차이
- 각 기술의 성능 특성과 적용 시나리오
- 실제 리소스 사용량 비교 실험

## 📌 왜 중요한가?

많은 개발자들이 "컨테이너는 가벼운 VM"이라고 오해합니다. 하지만 이는 근본적으로 다른 기술입니다.

**실전 시나리오:**
```
상황: 마이크로서비스 50개를 배포해야 합니다.
- VM 방식: 각 서비스마다 별도 VM → 막대한 리소스 필요
- 컨테이너 방식: 같은 커널 공유 → 효율적 리소스 활용

이 차이를 이해하지 못하면 아키텍처 설계 자체가 틀어집니다.
```

---

## 🔬 Deep Dive

### 1. 가상 머신 (Virtual Machine)의 원리

#### 구조
```
┌─────────────────────────────────────┐
│         Application 1               │
│         Application 2               │
├─────────────────────────────────────┤
│         Guest OS (Linux)            │  ← 전체 OS
├─────────────────────────────────────┤
│         Hypervisor (Type 1/2)       │  ← 가상화 레이어
├─────────────────────────────────────┤
│         Host OS (선택적)              │
├─────────────────────────────────────┤
│         Hardware                    │
└─────────────────────────────────────┘
```

#### 핵심 특징
1. **하드웨어 가상화**: CPU, 메모리, 디스크, 네트워크 등을 가상화
2. **전체 OS 실행**: 각 VM은 완전한 운영체제를 포함
3. **강력한 격리**: 하이퍼바이저 레벨에서 완전 격리
4. **무거운 리소스**: OS 부팅, 메모리 오버헤드 (수 GB)

#### Hypervisor 종류

**Type 1 (Bare-metal)**
```
Hardware → Hypervisor → VMs
예: VMware ESXi, Xen, Hyper-V
- 하드웨어에 직접 설치
- 높은 성능
- 주로 데이터센터/클라우드
```

**Type 2 (Hosted)**
```
Hardware → Host OS → Hypervisor → VMs
예: VirtualBox, VMware Workstation
- OS 위에서 실행
- 개발/테스트 환경
- 상대적으로 낮은 성능
```

---

### 2. 컨테이너의 원리

#### 구조
```
┌──────────────────────────────────────┐
│  App 1  │  App 2  │  App 3  │  App 4 │  ← 프로세스
├─────────┼─────────┼─────────┼────────┤
│    Container Runtime (Docker)        │  ← 얇은 레이어
├──────────────────────────────────────┤
│         Host OS Kernel               │  ← 공유 커널
├──────────────────────────────────────┤
│         Hardware                     │
└──────────────────────────────────────┘
```

#### 핵심 특징
1. **프로세스 격리**: OS 레벨 가상화 (커널 공유)
2. **빠른 시작**: 프로세스 시작과 동일 (밀리초)
3. **가벼움**: 수십 MB, OS 없이 앱만 포함
4. **Linux 기술 활용**: Namespaces, Cgroups, chroot

#### 격리 메커니즘 (자세한 내용은 05-Namespaces.md 참고)

**Namespaces (격리)**
```bash
# PID namespace: 프로세스 ID 격리
docker run -it ubuntu ps aux
# → 컨테이너 내부에서는 PID 1부터 시작

# Network namespace: 네트워크 스택 격리
docker run -it ubuntu ip addr
# → 컨테이너만의 네트워크 인터페이스

# Mount namespace: 파일시스템 격리
docker run -it ubuntu ls /
# → 컨테이너만의 루트 파일시스템
```

**Cgroups (리소스 제한)**
```bash
# CPU 제한
docker run --cpus="1.5" ubuntu

# 메모리 제한
docker run --memory="512m" ubuntu

# 실제 cgroup 설정 확인
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
```

---

## 🔄 직접 비교

### 아키텍처 비교

| 항목 | Virtual Machine | Container |
|------|----------------|-----------|
| **격리 레벨** | 하드웨어 레벨 | 프로세스 레벨 |
| **OS** | 각 VM마다 전체 OS | 호스트 커널 공유 |
| **시작 시간** | 분 단위 | 초 단위 (밀리초) |
| **크기** | GB 단위 | MB 단위 |
| **오버헤드** | 높음 (~20%) | 매우 낮음 (~1-2%) |
| **이식성** | 낮음 (전체 이미지) | 높음 (앱+의존성만) |
| **밀도** | 수십 개/호스트 | 수백~수천 개/호스트 |

### 리소스 사용 비교

**시나리오: 간단한 웹 서버 실행**

```bash
# VM 방식
메모리: 512MB (OS) + 50MB (앱) = 562MB
디스크: 10GB (OS) + 100MB (앱) = 10.1GB
시작: 30초

# 컨테이너 방식
메모리: 50MB (앱만)
디스크: 100MB (앱+레이어)
시작: 0.5초
```

---

## 💻 실습

### 실습 1: 시작 시간 비교

**VM 시작 시간 측정**
```bash
# VirtualBox VM 시작 (예시)
time VBoxManage startvm "MyVM" --type headless
# 결과: real 0m45.231s
```

**컨테이너 시작 시간 측정**
```bash
# Docker 컨테이너 시작
time docker run --rm alpine echo "Hello"
# 결과: real 0m0.423s

# 더 정확한 측정
for i in {1..10}; do
  time docker run --rm alpine echo "Hello" 2>&1 | grep real
done
```

### 실습 2: 메모리 오버헤드 비교

**컨테이너 메모리 사용**
```bash
# 기본 Alpine 이미지 실행
docker run -d --name test-container alpine sleep 3600

# 메모리 사용량 확인
docker stats test-container --no-stream
# CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %
# test-cont   0.00%   1.2MiB / 7.8GiB     0.02%

# 호스트에서 프로세스 확인
ps aux | grep "sleep 3600"
# 실제로는 호스트의 프로세스일 뿐
```

**VM과 비교**
```
VM: 최소 512MB (Guest OS) + 앱
Container: 1-2MB (앱만)

→ 약 250-500배 차이!
```

### 실습 3: 프로세스 관점에서 확인

**컨테이너는 호스트의 프로세스**
```bash
# 컨테이너 실행
docker run -d --name nginx-test nginx

# 호스트에서 nginx 프로세스 확인
ps aux | grep nginx
# root  12345  nginx: master process
# → 호스트에서 직접 보임!

# 컨테이너 내부에서 확인
docker exec nginx-test ps aux
# PID 1: nginx (컨테이너 내부 관점)

# 하지만 호스트에서는 PID 12345
# → 같은 프로세스를 다른 관점에서 보는 것
```

### 실습 4: 커널 공유 확인

**모두 같은 커널 사용**
```bash
# 호스트 커널 버전
uname -r
# 5.15.0-58-generic

# 컨테이너 1 커널 버전
docker run --rm ubuntu uname -r
# 5.15.0-58-generic (동일!)

# 컨테이너 2 (다른 배포판)
docker run --rm alpine uname -r
# 5.15.0-58-generic (동일!)

# → 모두 호스트 커널을 공유
```

---

## 🔥 실전 적용

### 시나리오 1: 마이크로서비스 아키텍처

**문제 상황:**
```
100개의 마이크로서비스 배포
각 서비스는 평균 100MB 메모리 사용
```

**VM 방식:**
```
100 VMs × (512MB OS + 100MB 앱) = 61.2GB
시작 시간: 100 VMs × 30초 = 50분
```

**컨테이너 방식:**
```
100 컨테이너 × 100MB = 10GB
시작 시간: 100 컨테이너 × 0.5초 = 50초
```

**결론:** 
- 메모리: 6배 절약
- 시작 시간: 60배 빠름
- 비용: 클라우드 인프라 비용 대폭 절감

### 시나리오 2: 개발 환경

**VM 방식 개발 환경:**
```bash
# Vagrant로 개발 환경 구성
vagrant up  # 2-3분 소요
vagrant ssh
cd /project
npm start

# 문제점:
# - 느린 시작 시간
# - 큰 디스크 사용 (수 GB)
# - 팀원 간 환경 차이 발생 가능
```

**컨테이너 방식 개발 환경:**
```bash
# Docker Compose로 즉시 시작
docker-compose up  # 수 초 소요

# Dockerfile로 환경 정의
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]

# 장점:
# - 빠른 시작
# - 팀원 간 완벽히 동일한 환경
# - Git으로 환경 관리 가능
```

### 시나리오 3: CI/CD 파이프라인

**VM 기반 CI:**
```yaml
# 각 빌드마다 VM 프로비저닝
- VM 생성: 2분
- OS 부팅: 1분
- 의존성 설치: 3분
- 빌드 실행: 5분
총 11분
```

**컨테이너 기반 CI:**
```yaml
# .gitlab-ci.yml 예시
image: node:18-alpine

stages:
  - build
  - test

build:
  script:
    - npm install
    - npm run build
  # 컨테이너 시작: 5초
  # 빌드 실행: 5분
  # 총 5분 5초
```

**결과:** 빌드 시간 50% 단축

---

## ⚡ 언제 무엇을 사용할까?

### VM을 사용해야 하는 경우

1. **강력한 격리가 필요할 때**
```
멀티테넌트 환경에서 완전한 격리
서로 다른 고객의 워크로드
```

2. **다른 커널이 필요할 때**
```
Windows 앱을 Linux 호스트에서 실행
특정 커널 버전이 필요한 레거시 앱
```

3. **전체 OS 기능이 필요할 때**
```
커널 모듈 로드
시스템 레벨 설정 변경
```

### 컨테이너를 사용해야 하는 경우

1. **마이크로서비스 아키텍처**
```
수많은 서비스를 효율적으로 실행
빠른 스케일링
```

2. **CI/CD 파이프라인**
```
빠른 빌드 환경 구성
일관된 테스트 환경
```

3. **개발 환경 표준화**
```
"내 컴퓨터에서는 되는데" 문제 해결
팀 전체의 동일한 환경
```

4. **클라우드 네이티브 애플리케이션**
```
무상태(stateless) 앱
수평 확장
```

### 하이브리드 접근

**Best Practice: VM + 컨테이너**
```
┌───────────────────────────────────┐
│         VM 1 (호스트)               │
│  ┌──────┐ ┌──────┐ ┌──────┐       │
│  │ C1   │ │ C2   │ │ C3   │       │
│  └──────┘ └──────┘ └──────┘       │
└───────────────────────────────────┘
┌───────────────────────────────────┐
│         VM 2 (호스트)               │
│  ┌──────┐ ┌──────┐ ┌──────┐       │
│  │ C4   │ │ C5   │ │ C6   │       │
│  └──────┘ └──────┘ └──────┘       │
└───────────────────────────────────┘

VM: 테넌트/환경 격리
Container: 앱 배포 및 확장
```

---

## 🚫 흔한 오해와 실수

### 오해 1: "컨테이너는 가벼운 VM이다"

**❌ 잘못된 이해:**
```
컨테이너 = VM의 경량 버전
```

**✅ 올바른 이해:**
```
컨테이너 = 프로세스 격리 기술
VM = 하드웨어 가상화 기술

근본적으로 다른 접근 방식!
```

### 오해 2: "컨테이너는 보안이 약하다"

**❌ 잘못된 생각:**
```
VM > 컨테이너 (보안 측면)
항상 VM이 더 안전하다
```

**✅ 올바른 이해:**
```
- 격리 수준: VM이 더 강력함 (O)
- 하지만 올바른 설정 시 컨테이너도 안전
- 보안은 기술이 아닌 구성에 달림
- 많은 기업들이 프로덕션에서 컨테이너 사용

보안 강화 방법:
- User namespaces
- Seccomp profiles
- AppArmor/SELinux
- 최소 권한 실행
```

### 실수 3: "컨테이너에서 영구 데이터 저장"

**❌ 잘못된 방식:**
```bash
# 컨테이너 내부에 데이터베이스 저장
docker run -d mysql
# 컨테이너 삭제 시 데이터 손실!
```

**✅ 올바른 방식:**
```bash
# Volume 사용
docker run -d -v mysql-data:/var/lib/mysql mysql
# 데이터는 호스트에 영구 저장
```

### 실수 4: "컨테이너를 VM처럼 사용"

**❌ 안티패턴:**
```dockerfile
FROM ubuntu
RUN apt-get update && apt-get install -y \
    ssh \
    supervisor \
    nginx \
    mysql \
    redis
# 하나의 컨테이너에 모든 것 넣기 (VM 사고방식)
```

**✅ 올바른 패턴:**
```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx
  mysql:
    image: mysql
  redis:
    image: redis
# 각 서비스는 별도 컨테이너
# 단일 책임 원칙 (Single Responsibility)
```

---

## 📊 성능 벤치마크

### CPU 성능 비교

**테스트: 소수 계산 (1,000,000번)**

```bash
# Native (호스트)
time python3 -c "
for i in range(1000000):
    pass
"
# real: 0m0.123s

# Container
time docker run --rm python:3.11-slim python3 -c "
for i in range(1000000):
    pass
"
# real: 0m0.127s (약 3% 오버헤드)

# VM (예시)
# real: 0m0.145s (약 18% 오버헤드)
```

**결론:** 컨테이너는 거의 네이티브 수준의 성능

### I/O 성능 비교

**테스트: 파일 쓰기 (1GB)**

```bash
# Native
dd if=/dev/zero of=testfile bs=1M count=1024
# 1.2 GB/s

# Container (Volume)
docker run --rm -v $(pwd):/data ubuntu \
  dd if=/dev/zero of=/data/testfile bs=1M count=1024
# 1.15 GB/s (약 4% 차이)

# Container (Overlay)
docker run --rm ubuntu \
  dd if=/dev/zero of=/testfile bs=1M count=1024
# 0.95 GB/s (약 20% 차이 - 레이어 오버헤드)
```

**학습 포인트:**
- Volume 사용 시 거의 네이티브 수준
- Overlay 파일시스템은 약간의 오버헤드
- 프로덕션에서는 Volume 사용 권장

### 네트워크 성능 비교

```bash
# iperf3 테스트

# Native
iperf3 -c server_ip
# 9.8 Gbps

# Container (bridge)
docker run --rm networkstatic/iperf3 -c server_ip
# 9.5 Gbps (약 3% 차이)

# Container (host network)
docker run --rm --network host networkstatic/iperf3 -c server_ip
# 9.8 Gbps (거의 동일)
```

---

## 🎓 핵심 정리

### VM vs Container 한눈에

```
VM (가상 머신)
├─ 하드웨어 가상화
├─ 완전한 OS 포함
├─ 강력한 격리
├─ 무겁고 느림
└─ 다른 OS 실행 가능

Container (컨테이너)
├─ OS 레벨 가상화
├─ 커널 공유
├─ 프로세스 격리
├─ 가볍고 빠름
└─ 같은 커널만 사용
```

### 핵심 포인트

1. **컨테이너는 프로세스다**
   - VM이 아닌 격리된 프로세스
   - 호스트 커널 공유
   - 밀리초 단위 시작

2. **적재적소 사용**
   - 강한 격리: VM
   - 효율성/속도: 컨테이너
   - 실무에서는 둘 다 사용

3. **성능 특성 이해**
   - 컨테이너: 거의 네이티브 수준
   - VM: 10-20% 오버헤드
   - 네트워크/I/O도 컨테이너가 유리

4. **올바른 사용법**
   - 컨테이너: 단일 프로세스, 무상태
   - VM: 전체 시스템, 상태 보존

---

## 📚 참고 자료

- [Linux Containers (LXC)](https://linuxcontainers.org/)
- [Docker Architecture](https://docs.docker.com/get-started/overview/)
- [Understanding VM vs Container](https://www.docker.com/resources/what-container/)
- [Container Performance Analysis (Netflix)](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)

---

**🤔 생각해볼 문제**

1. 왜 컨테이너는 Windows 컨테이너와 Linux 컨테이너를 동시에 실행할 수 없을까?
2. Kubernetes가 VM이 아닌 컨테이너를 오케스트레이션 단위로 선택한 이유는?
3. 서버리스(Lambda, Cloud Functions)가 컨테이너 기술을 사용하는 이유는?

> 💡 **힌트**: 모두 "커널 공유"와 "빠른 시작 시간"이 핵심입니다.   

---

<div align="center">

**[다음: Docker Architecture ➡️](./02-Docker-Architecture.md)**

</div>
