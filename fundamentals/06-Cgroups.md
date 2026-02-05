# 06. Cgroups - 리소스 제한과 관리

## 🎯 이 챕터에서 배울 것

- Linux **Cgroups (Control Groups)**의 개념과 역할
- **CPU, Memory, I/O** 등 리소스 제어 메커니즘
- **Cgroup v1 vs v2** 차이점과 마이그레이션
- 실제 리소스 제한 설정 및 모니터링

## 📌 왜 중요한가?

**"컨테이너가 시스템 전체를 먹어버리지 않게 하려면?"**

```
문제 상황:
- 컨테이너 하나가 CPU 100% 사용 → 다른 컨테이너 느려짐
- 메모리 누수 → OOM Killer가 중요한 프로세스 종료
- 디스크 I/O 폭주 → 전체 시스템 느려짐

해결: Cgroups로 리소스 제한!
```

**실무 영향:**
- 안정성: 리소스 격리로 안정적 운영
- 비용: 효율적 리소스 할당으로 비용 절감
- 성능: 공정한 리소스 분배

---

## 🔬 Deep Dive

### 1. Cgroups란?

#### 개념

**Control Groups = 프로세스 그룹의 리소스를 제한/격리/모니터링**

```
Namespace vs Cgroups:
├─ Namespace: "무엇이 보이는가?" (격리)
│  └─ 프로세스, 네트워크, 파일시스템 격리
└─ Cgroups: "얼마나 사용할 수 있는가?" (제한)
   └─ CPU, 메모리, I/O 제한

┌─────────────────────────────────┐
│     Process Group               │
│  ┌──────┐ ┌──────┐ ┌──────┐     │
│  │ PID1 │ │ PID2 │ │ PID3 │     │
│  └──────┘ └──────┘ └──────┘     │
└─────────────────────────────────┘
         ↓ Cgroup 적용
┌─────────────────────────────────┐
│  리소스 제한:                      │
│  ├─ CPU: 50%                    │
│  ├─ Memory: 512MB               │
│  └─ I/O: 10MB/s                 │
└─────────────────────────────────┘
```

#### 역사

```
2006: Paul Menage (Google)이 제안
2008: Linux 2.6.24에 포함
2016: Cgroup v2 도입 (단일 계층)
2021: Cgroup v2 대부분 배포판 기본값

Docker:
2013~: Cgroup v1 사용
2020~: Cgroup v2 지원 시작
```

---

### 2. Cgroup Subsystems (Controllers)

#### 주요 컨트롤러

| Controller | 제어 대상 | 주요 파라미터 |
|------------|----------|--------------|
| **cpu** | CPU 시간 | shares, quota, period |
| **cpuset** | CPU 코어 할당 | cpus, mems |
| **memory** | 메모리 사용량 | limit, soft_limit, swap |
| **blkio** | 블록 I/O | weight, throttle |
| **devices** | 디바이스 접근 | allow/deny 규칙 |
| **freezer** | 프로세스 일시정지 | state |
| **net_cls** | 네트워크 분류 | classid |
| **pids** | 프로세스 수 제한 | max |

---

### 3. CPU 제어

#### CPU Shares (상대적 우선순위)

```
CPU Shares = 상대적 CPU 할당 비율

예시:
Container A: cpu.shares = 1024 (기본값)
Container B: cpu.shares = 512
Container C: cpu.shares = 2048

CPU 경합 시 할당:
A : B : C = 1024 : 512 : 2048 = 2 : 1 : 4
→ A는 28.6%, B는 14.3%, C는 57.1%

CPU 여유 시:
→ 제한 없음, 모두 100% 사용 가능!
```

#### CPU Quota (절대적 제한)

```
CPU Quota = 절대적 CPU 사용량 제한

cpu.cfs_quota_us: 사용 가능한 CPU 시간
cpu.cfs_period_us: 기간 (보통 100ms = 100000us)

예시:
period = 100000us (100ms)
quota = 50000us (50ms)
→ 100ms 중 50ms만 사용 = 50% 제한 (0.5 CPU)

멀티코어:
quota = 200000us, period = 100000us
→ 2개 코어 100% = 2 CPU
```

#### CPUset (코어 고정)

```
특정 CPU 코어에 프로세스 고정

cpuset.cpus = "0,1"  → 코어 0, 1만 사용
cpuset.cpus = "0-3"  → 코어 0~3 사용

장점:
├─ 캐시 효율 증가
├─ NUMA 최적화
└─ 예측 가능한 성능

단점:
├─ 유연성 감소
└─ 코어 유휴 가능
```

---

### 4. Memory 제어

#### Memory Limit

```
메모리 사용량 상한 설정

memory.limit_in_bytes: 최대 메모리
memory.soft_limit_in_bytes: 소프트 제한 (권장)
memory.memsw.limit_in_bytes: 메모리 + Swap

제한 초과 시:
1. Soft limit 초과: 메모리 회수 시도
2. Hard limit 초과: OOM Killer 발동!
```

#### OOM Killer

```
Out Of Memory Killer = 메모리 부족 시 프로세스 종료

동작:
1. 메모리 한계 도달
2. 메모리 회수 시도 (페이지 스왑, 캐시 정리)
3. 실패 시 OOM Killer 발동
4. 스코어 계산 (oom_score)
5. 가장 높은 스코어 프로세스 종료 (SIGKILL)

oom_score 계산:
├─ 메모리 사용량 높을수록 +
├─ 실행 시간 짧을수록 +
├─ oom_score_adj 값
└─ root 프로세스는 -

Docker에서:
oom-kill-disable: OOM Killer 비활성화 (위험!)
oom-score-adj: 우선순위 조정
```

#### Memory Reservation

```
메모리 예약 (부드러운 제한)

memory.soft_limit_in_bytes: 권장 제한
→ 초과해도 바로 종료 안 됨
→ 메모리 부족 시 우선 회수 대상

사용 예:
limit: 1GB (절대 한계)
soft_limit: 800MB (권장)
→ 평소 800MB 사용
→ 필요 시 1GB까지 버스트
```

---

### 5. I/O 제어

#### Block I/O Weight

```
I/O 우선순위 (상대적)

blkio.weight: 100 ~ 1000 (기본 500)

Container A: 1000 (높음)
Container B: 500 (보통)
Container C: 100 (낮음)

I/O 경합 시:
A : B : C = 10 : 5 : 1
→ A가 가장 많은 I/O 대역폭 획득
```

#### I/O Throttle (절대 제한)

```
I/O 처리량 절대 제한

blkio.throttle.read_bps_device: 읽기 제한 (Bytes/sec)
blkio.throttle.write_bps_device: 쓰기 제한 (Bytes/sec)
blkio.throttle.read_iops_device: 읽기 IOPS 제한
blkio.throttle.write_iops_device: 쓰기 IOPS 제한

예:
echo "8:0 10485760" > blkio.throttle.read_bps_device
→ /dev/sda (8:0) 읽기를 10MB/s로 제한
```

---

### 6. Cgroup v1 vs v2

#### 구조 차이

**Cgroup v1: 다중 계층 (Multiple Hierarchies)**
```
/sys/fs/cgroup/
├─ cpu/
│  ├─ docker/
│  │  └─ container-id/
│  └─ systemd/
├─ memory/
│  ├─ docker/
│  │  └─ container-id/
│  └─ systemd/
└─ blkio/
   └─ docker/
      └─ container-id/

문제:
- 각 subsystem마다 별도 계층
- 일관성 없는 구조
- 관리 복잡
```

**Cgroup v2: 단일 계층 (Unified Hierarchy)**
```
/sys/fs/cgroup/
└─ docker/
   └─ container-id/
      ├─ cpu.max (cpu 제어)
      ├─ memory.max (memory 제어)
      └─ io.max (io 제어)

장점:
- 단일 계층 구조
- 일관된 인터페이스
- 관리 간편
- 더 나은 격리
```

#### 주요 변경사항

| 기능 | v1 | v2 |
|------|----|----|
| **계층** | 다중 | 단일 |
| **CPU** | cpu.shares, cpu.cfs_quota_us | cpu.weight, cpu.max |
| **Memory** | memory.limit_in_bytes | memory.max, memory.high |
| **I/O** | blkio.weight | io.weight, io.max |
| **Pressure Stall** | 없음 | 있음 (PSI) |
| **Thread 모드** | 없음 | 있음 |

---

## 💻 실습

### 실습 1: CPU 제한

```bash
# 1. CPU Shares (상대적)
# Container A: 기본 (1024 shares)
docker run -d --name cpu-normal stress --cpu 2

# Container B: 낮은 우선순위 (512 shares)
docker run -d --name cpu-low --cpu-shares 512 stress --cpu 2

# Container C: 높은 우선순위 (2048 shares)
docker run -d --name cpu-high --cpu-shares 2048 stress --cpu 2

# CPU 사용률 확인
docker stats --no-stream
# NAME       CPU %
# cpu-high   57%  (가장 많이 사용)
# cpu-normal 29%
# cpu-low    14%  (가장 적게 사용)

# 2. CPU Quota (절대적)
# 0.5 CPU (50%)
docker run -d --name cpu-50 --cpus 0.5 stress --cpu 2

# 2 CPU (200%)
docker run -d --name cpu-200 --cpus 2 stress --cpu 4

docker stats --no-stream
# NAME     CPU %
# cpu-50   50%   (제한됨)
# cpu-200  200%  (2 코어)

# 3. CPUset (코어 고정)
# 코어 0, 1만 사용
docker run -d --name cpu-pinned --cpuset-cpus="0,1" stress --cpu 4

# 확인
docker exec cpu-pinned taskset -c -p 1
# pid 1's current affinity list: 0,1

# 정리
docker rm -f cpu-normal cpu-low cpu-high cpu-50 cpu-200 cpu-pinned
```

### 실습 2: Memory 제한

```bash
# 1. Memory Limit
# 128MB 제한
docker run -d --name mem-128 --memory=128m ubuntu \
  bash -c 'while true; do echo "x" >> /tmp/file; done'

# 메모리 사용량 모니터링
docker stats mem-128 --no-stream
# NAME     MEM USAGE / LIMIT
# mem-128  128MiB / 128MiB  (한계 도달)

# OOM 확인 (초과 시 종료됨)
docker logs mem-128
# (출력 없음 - 종료됨)

docker inspect mem-128 | jq '.[0].State'
# "OOMKilled": true

# 2. Memory Reservation (Soft Limit)
docker run -d --name mem-reserve \
  --memory=256m \
  --memory-reservation=128m \
  stress --vm 1 --vm-bytes 200M

# 200MB 사용 (128MB 초과하지만 256MB 이하)
# → 종료 안 됨

# 3. Swap 제한
# Swap 비활성화
docker run -d --name no-swap \
  --memory=128m \
  --memory-swap=128m \
  stress --vm 1 --vm-bytes 256M

# memory-swap = memory → Swap 0
# 128MB 초과 시 바로 OOM

# 4. OOM Kill 비활성화 (위험!)
docker run -d --name no-oom \
  --memory=128m \
  --oom-kill-disable \
  stress --vm 1 --vm-bytes 256M

# OOM 발생해도 종료 안 됨
# → 시스템 전체 메모리 부족 가능!

# 정리
docker rm -f mem-128 mem-reserve no-swap no-oom
```

### 실습 3: I/O 제한

```bash
# 1. I/O 쓰기 제한
# 10MB/s로 제한
docker run -it --rm \
  --device-write-bps /dev/sda:10mb \
  ubuntu dd if=/dev/zero of=/tmp/test bs=1M count=100 oflag=direct

# 결과: ~10초 소요 (10MB/s)

# 2. I/O 읽기 제한
# 5MB/s로 제한
docker run -it --rm \
  --device-read-bps /dev/sda:5mb \
  ubuntu dd if=/dev/sda of=/dev/null bs=1M count=100 iflag=direct

# 3. IOPS 제한
# 100 IOPS
docker run -it --rm \
  --device-write-iops /dev/sda:100 \
  ubuntu \
  bash -c 'for i in {1..1000}; do echo $i > /tmp/file$i; done'

# 4. I/O Weight (상대적)
# 높은 우선순위
docker run -d --name io-high --blkio-weight 1000 \
  ubuntu dd if=/dev/zero of=/tmp/test bs=1M count=1000

# 낮은 우선순위
docker run -d --name io-low --blkio-weight 100 \
  ubuntu dd if=/dev/zero of=/tmp/test bs=1M count=1000

# 정리
docker rm -f io-high io-low
```

### 실습 4: Cgroup 직접 확인

```bash
# 1. 컨테이너 실행
docker run -d --name cgroup-test \
  --cpus=0.5 \
  --memory=256m \
  nginx

# 2. Cgroup 경로 찾기 (v1)
CONTAINER_ID=$(docker inspect -f '{{.Id}}' cgroup-test)
CGROUP_PATH="/sys/fs/cgroup"

# 3. CPU 제한 확인
cat $CGROUP_PATH/cpu/docker/$CONTAINER_ID/cpu.cfs_quota_us
# 50000  (50ms / 100ms = 50%)

cat $CGROUP_PATH/cpu/docker/$CONTAINER_ID/cpu.cfs_period_us
# 100000

# 4. Memory 제한 확인
cat $CGROUP_PATH/memory/docker/$CONTAINER_ID/memory.limit_in_bytes
# 268435456  (256MB)

# 5. 현재 메모리 사용량
cat $CGROUP_PATH/memory/docker/$CONTAINER_ID/memory.usage_in_bytes

# 6. 메모리 통계
cat $CGROUP_PATH/memory/docker/$CONTAINER_ID/memory.stat
# cache 1234567
# rss 2345678
# ...

# 7. Cgroup v2 (최신 시스템)
ls /sys/fs/cgroup/system.slice/docker-$CONTAINER_ID.scope/
# cpu.max
# memory.max
# io.max
# ...

cat /sys/fs/cgroup/system.slice/docker-$CONTAINER_ID.scope/cpu.max
# 50000 100000  (50% CPU)

# 정리
docker rm -f cgroup-test
```

---

## 🔥 실전 적용

### 시나리오 1: 리소스 보장 (QoS)

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Critical: 높은 우선순위
  database:
    image: postgres
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
    # CPU shares: 높음
    cpu_shares: 2048

  # Normal: 중간 우선순위
  api:
    image: myapi
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
    cpu_shares: 1024

  # Low priority: 낮은 우선순위 (배치 작업)
  batch:
    image: mybatch
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.1'
          memory: 128M
    cpu_shares: 512
```

### 시나리오 2: OOM 대응 전략

```bash
# 1. 메모리 사용량 모니터링
docker run -d --name web \
  --memory=512m \
  --memory-reservation=384m \
  nginx

# 2. 메모리 알림 설정
CONTAINER_ID=$(docker inspect -f '{{.Id}}' web)
CGROUP="/sys/fs/cgroup/memory/docker/$CONTAINER_ID"

# 90% 도달 시 알림
echo $((512 * 1024 * 1024 * 90 / 100)) > $CGROUP/memory.soft_limit_in_bytes

# 3. OOM Score 조정 (중요 컨테이너)
docker run -d --name critical-app \
  --memory=1g \
  --oom-score-adj=-500 \
  myapp
# → OOM 시 나중에 종료됨

# 4. Health Check 추가
docker run -d --name monitored-app \
  --memory=512m \
  --health-cmd='test $(ps aux | wc -l) -lt 100 || exit 1' \
  --health-interval=30s \
  --health-retries=3 \
  myapp

# 메모리 누수 감지 시 자동 재시작
```

### 시나리오 3: 멀티테넌트 환경

```bash
# 테넌트별 리소스 격리

# 테넌트 A (프리미엄)
docker run -d --name tenant-a-app \
  --cpus=2 \
  --memory=2g \
  --cpu-shares=2048 \
  --blkio-weight=1000 \
  tenant-a:latest

# 테넌트 B (스탠다드)
docker run -d --name tenant-b-app \
  --cpus=1 \
  --memory=1g \
  --cpu-shares=1024 \
  --blkio-weight=500 \
  tenant-b:latest

# 테넌트 C (프리)
docker run -d --name tenant-c-app \
  --cpus=0.5 \
  --memory=512m \
  --cpu-shares=512 \
  --blkio-weight=100 \
  tenant-c:latest

# 결과:
# - CPU 경합 시: A=50%, B=25%, C=12.5%
# - I/O 경합 시: A=62.5%, B=31.25%, C=6.25%
# - 메모리: 각각 절대 제한
```

### 시나리오 4: 개발 vs 프로덕션

```bash
# 개발: 느슨한 제한
docker run -d --name dev-app \
  --memory=1g \
  --memory-swap=2g \
  dev-image:latest
# → Swap 허용, OOM 가능성 낮음

# 프로덕션: 엄격한 제한
docker run -d --name prod-app \
  --memory=512m \
  --memory-swap=512m \
  --memory-reservation=384m \
  --oom-score-adj=-500 \
  prod-image:latest
# → Swap 제한, 메모리 예약, OOM 우선순위 낮음
```

---

## ⚡ 모니터링과 디버깅

### 1. 실시간 모니터링

```bash
# Docker Stats
docker stats --no-stream --format \
  "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.BlockIO}}"

# Cgroup 직접 모니터링
watch -n 1 'cat /sys/fs/cgroup/memory/docker/*/memory.usage_in_bytes'

# CPU Throttling 확인
cat /sys/fs/cgroup/cpu/docker/*/cpu.stat
# nr_periods 100
# nr_throttled 50  ← 50% throttled!
# throttled_time 5000000000
```

### 2. 리소스 부족 탐지

```bash
# CPU Throttling 로그
docker inspect <container> | jq '.[0].HostConfig.CpuQuota'

# Memory Pressure
cat /sys/fs/cgroup/memory/docker/*/memory.pressure_level
# low / medium / critical

# OOM Events
journalctl -k | grep -i "out of memory"
journalctl -u docker | grep -i "oom"
```

---

## 🚫 안티패턴

### 1. 제한 없는 컨테이너

```bash
# ❌ 위험: 제한 없음
docker run -d unlimited-app
# → CPU, 메모리 제한 없음
# → 시스템 전체 마비 가능

# ✅ 안전: 항상 제한 설정
docker run -d \
  --cpus=1 \
  --memory=512m \
  safe-app
```

### 2. OOM Kill 비활성화

```bash
# ❌ 매우 위험
docker run -d \
  --memory=128m \
  --oom-kill-disable \
  dangerous-app
# → 메모리 누수 시 호스트 전체 OOM

# ✅ 올바른 방법
docker run -d \
  --memory=128m \
  --memory-reservation=96m \
  --health-check ... \
  safe-app
```

### 3. 과도한 Swap 사용

```bash
# ❌ 성능 저하
docker run -d \
  --memory=256m \
  --memory-swap=10g \
  slow-app
# → 과도한 Swap → 매우 느림

# ✅ Swap 제한
docker run -d \
  --memory=512m \
  --memory-swap=512m \
  fast-app
# → Swap 없음 (또는 메모리와 동일)
```

---

## 🎓 핵심 정리

### Cgroups 역할

```
리소스 제한 (Limit):
├─ CPU: 얼마나 사용할 수 있는가
├─ Memory: 최대 메모리 사용량
└─ I/O: 디스크 읽기/쓰기 속도

리소스 우선순위 (Priority):
├─ CPU Shares: 상대적 CPU 할당
├─ I/O Weight: 상대적 I/O 할당
└─ OOM Score: 종료 우선순위

모니터링 (Accounting):
├─ 현재 사용량 추적
├─ 통계 수집
└─ 리소스 히스토리
```

### 실무 가이드

```
✅ 항상 설정:
├─ --memory: 메모리 제한 (필수!)
├─ --cpus: CPU 제한
└─ --oom-score-adj: 중요도

⚠️ 주의사항:
├─ 너무 낮은 제한 → 성능 저하
├─ 너무 높은 제한 → 격리 실패
└─ 모니터링 필수

🔧 튜닝:
├─ 프로파일링으로 적정 값 찾기
├─ Memory Reservation 활용
└─ CPU Shares로 우선순위 조정
```

---

## 📚 참고 자료

- [Linux Control Groups](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [Docker Runtime Options](https://docs.docker.com/config/containers/resource_constraints/)
- [Cgroups v2](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [OOM Killer](https://www.kernel.org/doc/gorman/html/understand/understand016.html)

---

**🤔 생각해볼 문제**

1. `--cpus=0.5`와 `--cpu-shares=512`의 차이는?
2. Memory Limit을 초과하면 항상 OOM Killer가 발동할까?
3. Kubernetes에서 `requests`와 `limits`는 Cgroups의 어떤 기능을 사용할까?

> 💡 **답변**: 1) 절대/상대 제한, 2) 아니오 (Swap 가능, soft limit), 3) requests=reservation, limits=limit

---

<div align="center">

**[⬅️ 이전: Namespaces](./05-Namespaces.md)** | **[다음: Docker Engine ➡️](./07-Docker-Engine.md)**

</div>
