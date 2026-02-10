# 02. CPU Management - CPU 관리

## 🎯 이 챕터에서 배울 것

- **CPU Affinity** - 특정 코어에 프로세스 고정
- **NUMA Awareness** - Non-Uniform Memory Access 최적화
- **CPU Throttling** - CPU 제한 시 발생하는 현상
- **Real-time Priority** - 실시간 스케줄링
- **Performance Tuning** - CPU 성능 최적화

## 📌 왜 중요한가?

**"CPU 관리는 레이턴시 최소화와 처리량 극대화의 핵심입니다."**

```
CPU 최적화 전 vs 후:

최적화 전:
┌─────────────────────────────────────┐
│ 8 CPU 시스템                         │
│                                     │
│ Container A (웹 서버):               │
│ CPU 0,1,2,3,4,5,6,7 (모두 사용)       │
│ → Context switching 빈번             │
│ → Cache miss 증가                    │
│ → 레이턴시 변동 심함                    │
│                                     │
│ Container B (데이터베이스):            │
│ CPU 0,1,2,3,4,5,6,7 (경쟁)           │
│ → CPU 대기 발생                       │
│ → 성능 예측 불가                       │
└─────────────────────────────────────┘

문제점:
❌ CPU 경쟁으로 레이턴시 증가
❌ Context switch overhead
❌ L1/L2 캐시 효율 저하
❌ NUMA 최적화 불가

최적화 후 (CPU Pinning + NUMA):
┌─────────────────────────────────────┐
│ NUMA Node 0 (CPU 0-3, Memory 16GB)  │
│ ┌─────────────────────────────────┐ │
│ │ Container A (웹):               │ │
│ │ CPU 0,1 (고정)                   │ │
│ │ Memory: Node 0 (로컬)            │ │
│ │ → Cache hit 증가                 │ │
│ │ → 일관된 레이턴시                   │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ NUMA Node 1 (CPU 4-7, Memory 16GB)  │
│ ┌─────────────────────────────────┐ │
│ │ Container B (DB):               │ │
│ │ CPU 4,5,6,7 (고정)               │ │
│ │ Memory: Node 1 (로컬)            │ │
│ │ → 메모리 대역폭 최대                │ │
│ │ → NUMA 원격 접근 제거              │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘

장점:
✅ 격리된 CPU 리소스
✅ 캐시 효율 증가 (20-30%)
✅ 레이턴시 감소 (40-60%)
✅ NUMA 로컬 메모리 접근

CPU 관리 핵심 개념:

1. CPU Affinity (친화성):
   
   No Affinity:
   ┌──────────────────────────────┐
   │ Process 1: CPU 0 → 2 → 5 → 1 │
   │ Process 2: CPU 3 → 0 → 6 → 2 │
   │ → Context switching          │
   │ → Cache thrashing            │
   └──────────────────────────────┘
   
   With Affinity:
   ┌──────────────────────────────┐
   │ Process 1: CPU 0,1 only      │
   │ Process 2: CPU 2,3 only      │
   │ → Stable performance         │
   │ → Better cache locality      │
   └──────────────────────────────┘

2. NUMA 아키텍처:
   
   Uniform Memory Access (UMA):
   ┌─────────────────────────────┐
   │ CPU 0,1,2,3                 │
   │      ↓ (같은 거리)            │
   │ Shared Memory (32GB)        │
   └─────────────────────────────┘
   → 모든 CPU가 메모리에 동일한 시간
   
   Non-Uniform Memory Access (NUMA):
   ┌─────────────────────────────┐
   │ Node 0                      │
   │ CPU 0,1 ↔ Memory 16GB       │
   │ (Local: 100ns)              │
   └──────────┬──────────────────┘
              │ (Remote: 200ns)
   ┌──────────▼──────────────────┐
   │ Node 1                      │
   │ CPU 2,3 ↔ Memory 16GB       │
   │ (Local: 100ns)              │
   └─────────────────────────────┘
   
   → 로컬 메모리: 빠름
   → 원격 메모리: 2배 느림
   → CPU + Memory를 같은 노드에 배치 필수

3. CPU Throttling:
   
   Normal:
   ┌──────────────────────────────┐
   │ Time: 100ms                  │
   │ Quota: 200ms (2 cores)       │
   │ Used: 150ms                  │
   │ → Running (75% utilization)  │
   └──────────────────────────────┘
   
   Throttled:
   ┌──────────────────────────────┐
   │ Time: 100ms                  │
   │ Quota: 200ms (2 cores)       │
   │ Used: 200ms (limit reached)  │
   │ → Throttled (100% used)      │
   │ → Wait next period           │
   └──────────────────────────────┘
   
   확인:
   /sys/fs/cgroup/cpu/cpu.stat
   - nr_periods: 1000
   - nr_throttled: 234 (23.4% 제한)
   - throttled_time: 5000000000 (5초)

4. 실시간 우선순위:
   
   일반 프로세스 (CFS):
   Nice: -20 to +19
   → 공정한 스케줄링
   → 우선순위는 상대적
   
   실시간 프로세스 (RT):
   Priority: 1-99
   → 절대적 우선순위
   → 일반 프로세스보다 먼저 실행
   → 주의: CPU 독점 가능

실무 시나리오:

시나리오 1 - 레이턴시 민감 애플리케이션:
┌─────────────────────────────────────┐
│ 거래 시스템 (초당 10만 거래)              │
│                                     │
│ 요구사항:                             │
│ - 평균 레이턴시: <1ms                  │
│ - P99 레이턴시: <5ms                  │
│ - Jitter: 최소화                      │
└─────────────────────────────────────┘

최적화 전:
CPU Affinity 없음
→ P99: 15ms (목표 초과)
→ Jitter: 10ms

최적화 후:
CPU Pinning (0,1) + NUMA Node 0
→ P99: 3ms (목표 달성)
→ Jitter: 1ms

시나리오 2 - NUMA 최적화:
┌─────────────────────────────────────┐
│ 2-socket 서버 (각 16 cores, 64GB)     │
│                                     │
│ 최적화 전:                            │
│ DB Container: CPU 0-31 (모든 코어)    │
│ → NUMA Remote Access 50%            │
│ → 메모리 대역폭: 40GB/s                │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ 최적화 후:                            │
│ DB Container: CPU 0-15 + NUMA Node 0│
│ → NUMA Local Access 95%             │
│ → 메모리 대역폭: 70GB/s (75% 향상)      │
└─────────────────────────────────────┘

시나리오 3 - CPU Throttling 문제:
┌─────────────────────────────────────┐
│ 09:00 - 정상 동작                     │ 
│ CPU: 150% (2 cores limit)           │
│ Latency: 50ms                       │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 12:00 - 트래픽 증가                    │
│ CPU: 200% (limit 도달)               │
│ Throttled: 30%                      │
│ Latency: 300ms (6배 증가!)            │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 해결: CPU limit 증가                  │
│ --cpus=2 → --cpus=3                 │
│ Throttled: 0%                       │
│ Latency: 50ms (정상)                 │
└─────────────────────────────────────┘
```

---

## 🔧 실습 1: CPU Affinity 설정

### Step 1: taskset으로 CPU 고정

```bash
# 시스템 CPU 확인
lscpu
# CPU(s): 8
# Thread(s) per core: 2
# Core(s) per socket: 4
# Socket(s): 1

# CPU 0,1에 컨테이너 고정
docker run -d --name pinned \
  --cpuset-cpus="0,1" \
  alpine sh -c 'while true; do :; done'

# 프로세스 확인
PID=$(docker inspect pinned --format '{{.State.Pid}}')
taskset -cp $PID
# pid 12345's current affinity list: 0,1

# CPU 사용 확인 (htop)
htop
# CPU 0,1만 100% 사용, 나머지는 0%

docker rm -f pinned
```

### Step 2: 동적으로 CPU 이동

```bash
# 처음에는 CPU 0,1
docker run -d --name dynamic \
  --cpuset-cpus="0,1" \
  alpine sh -c 'while true; do :; done'

# 실행 중 CPU 2,3으로 변경
docker update --cpuset-cpus="2,3" dynamic

# 확인
PID=$(docker inspect dynamic --format '{{.State.Pid}}')
taskset -cp $PID
# pid 12345's current affinity list: 2,3

# htop으로 확인 - CPU 2,3으로 이동됨

docker rm -f dynamic
```

### Step 3: 여러 컨테이너 분산

```bash
# 8 CPU 시스템에 4개 컨테이너 분산
docker run -d --name web1 --cpuset-cpus="0,1" nginx:alpine
docker run -d --name web2 --cpuset-cpus="2,3" nginx:alpine
docker run -d --name web3 --cpuset-cpus="4,5" nginx:alpine
docker run -d --name web4 --cpuset-cpus="6,7" nginx:alpine

# 각 컨테이너가 2개 코어 전용 사용
docker stats --no-stream

# CONTAINER  CPU %
# web1       0.5%   (on CPU 0,1)
# web2       0.5%   (on CPU 2,3)
# web3       0.5%   (on CPU 4,5)
# web4       0.5%   (on CPU 6,7)

# 부하 테스트
for i in 1 2 3 4; do
  docker exec web$i sh -c 'while true; do :; done' &
done

# 각 컨테이너가 자신의 CPU에서만 200% 사용

docker rm -f web1 web2 web3 web4
```

---

## 🔧 실습 2: NUMA 인식 설정

### Step 1: NUMA 토폴로지 확인

```bash
# numactl 설치
sudo apt-get install -y numactl

# NUMA 노드 확인
numactl --hardware

# 출력:
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7
# node 0 size: 32000 MB
# node 1 cpus: 8 9 10 11 12 13 14 15
# node 1 size: 32000 MB
# node distances:
# node   0   1
#   0:  10  20
#   1:  20  10

# 거리:
# 10 = Local (빠름)
# 20 = Remote (느림)

# 현재 NUMA 정책 확인
numactl --show
```

### Step 2: NUMA Node에 컨테이너 바인딩

```bash
# Node 0에 바인딩
docker run -d --name numa-node-0 \
  --cpuset-cpus="0-7" \
  --cpuset-mems="0" \
  postgres:15-alpine

# Node 1에 바인딩
docker run -d --name numa-node-1 \
  --cpuset-cpus="8-15" \
  --cpuset-mems="1" \
  redis:alpine

# cgroup 확인
docker exec numa-node-0 cat /sys/fs/cgroup/cpuset/cpuset.cpus
# 0-7

docker exec numa-node-0 cat /sys/fs/cgroup/cpuset/cpuset.mems
# 0

# NUMA 통계 확인
numastat -c numa-node-0

# 출력:
#                           Node 0    Node 1
# numa_hit              123456789         0
# numa_miss                     0    123456  (원격 접근)
# local_node            123456789         0
# other_node                    0    123456
```

### Step 3: NUMA 메모리 정책

```bash
# Preferred 정책 (선호하지만 강제 아님)
docker run -d --name numa-preferred \
  --cpuset-cpus="0-7" \
  --cpuset-mems="0" \
  myapp:latest

# Bind 정책 (강제)
# Docker는 기본적으로 bind 정책 사용
# --cpuset-mems로 지정한 노드만 사용

# 메모리 사용 확인
docker exec numa-preferred cat /proc/self/numa_maps | head -20

# 출력:
# 7f8a4c000000 default anon=12345 dirty=12345 N0=12345
# 7f8a4c100000 default anon=6789 dirty=6789 N0=6789
# N0 = Node 0에 할당됨
```

### Step 4: NUMA 성능 측정

```bash
# 벤치마크 스크립트
cat > numa-bench.sh <<'EOF'
#!/bin/bash

# Local 접근 (같은 NUMA 노드)
echo "=== Local NUMA Access ==="
docker run --rm \
  --cpuset-cpus="0-7" \
  --cpuset-mems="0" \
  sysbench:latest \
  sysbench memory --memory-oper=read run | grep "transferred"

# Remote 접근 (다른 NUMA 노드)
echo "=== Remote NUMA Access ==="
docker run --rm \
  --cpuset-cpus="0-7" \
  --cpuset-mems="1" \
  sysbench:latest \
  sysbench memory --memory-oper=read run | grep "transferred"
EOF

chmod +x numa-bench.sh
./numa-bench.sh

# 결과 예시:
# Local:  10000 MiB transferred (5000 MiB/sec)
# Remote: 10000 MiB transferred (3000 MiB/sec)
# → Remote는 40% 느림
```

---

## 🔧 실습 3: CPU Throttling 분석

### Step 1: Throttling 발생 확인

```bash
# CPU 2개로 제한
docker run -d --name throttle-test \
  --cpus=2 \
  alpine sh -c 'while true; do :; done'

# Throttling 통계 확인
CONTAINER_ID=$(docker inspect throttle-test --format '{{.Id}}')
cat /sys/fs/cgroup/cpu/docker/$CONTAINER_ID/cpu.stat

# 출력:
# nr_periods 5234
# nr_throttled 1234
# throttled_time 12340000000

# 계산:
# Throttled: 1234 / 5234 = 23.5%
# Throttled time: 12.34초

# 실시간 모니터링
watch -n 1 'cat /sys/fs/cgroup/cpu/docker/'$CONTAINER_ID'/cpu.stat'
```

### Step 2: Throttling 시각화

```bash
# 모니터링 스크립트
cat > monitor-throttle.sh <<'EOF'
#!/bin/bash

CONTAINER=$1
CONTAINER_ID=$(docker inspect $CONTAINER --format '{{.Id}}')
CGROUP_PATH="/sys/fs/cgroup/cpu/docker/$CONTAINER_ID"

echo "Time,nr_periods,nr_throttled,throttled_time"

while true; do
  STATS=$(cat $CGROUP_PATH/cpu.stat)
  NR_PERIODS=$(echo "$STATS" | grep nr_periods | awk '{print $2}')
  NR_THROTTLED=$(echo "$STATS" | grep nr_throttled | awk '{print $2}')
  THROTTLED_TIME=$(echo "$STATS" | grep throttled_time | awk '{print $2}')
  
  echo "$(date +%s),$NR_PERIODS,$NR_THROTTLED,$THROTTLED_TIME"
  sleep 1
done
EOF

chmod +x monitor-throttle.sh
./monitor-throttle.sh throttle-test > throttle.csv &

# 1분 후 종료 후 분석
pkill -f monitor-throttle

# 그래프 (Python)
cat > plot-throttle.py <<'EOF'
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('throttle.csv')
df['throttle_pct'] = (df['nr_throttled'] / df['nr_periods']) * 100

plt.figure(figsize=(10, 6))
plt.plot(df['Time'], df['throttle_pct'])
plt.xlabel('Time (s)')
plt.ylabel('Throttle %')
plt.title('CPU Throttling Over Time')
plt.savefig('throttle.png')
EOF

python3 plot-throttle.py
```

### Step 3: Throttling 해결

```bash
# 문제: 높은 throttling
docker stats throttle-test --no-stream
# CPU %: 200% (limit)
# Throttled: 45%

# 해결 1: CPU limit 증가
docker update --cpus=3 throttle-test

# 확인
docker stats throttle-test --no-stream
# CPU %: 200% (여유 있음)
# Throttled: 0%

# 해결 2: CPU period 증가 (덜 빈번한 체크)
docker run -d --name less-throttle \
  --cpu-period=200000 \
  --cpu-quota=400000 \
  alpine sh -c 'while true; do :; done'

# Period 2배 증가 → 같은 CPU(2 cores)지만 더 유연함

docker rm -f throttle-test less-throttle
```

---

## 🔧 실습 4: 실시간 우선순위

### Step 1: RT Priority 설정

```bash
# 일반 우선순위
docker run -d --name normal-priority \
  alpine sh -c 'while true; do :; done'

# 높은 우선순위 (RT)
docker run -d --name high-priority \
  --cpu-rt-runtime=950000 \
  --cpu-rt-period=1000000 \
  alpine sh -c 'while true; do :; done'

# 의미:
# RT runtime: 950000 us (0.95초)
# RT period: 1000000 us (1초)
# → 1초 중 0.95초를 RT로 실행 (95%)

# 주의: RT는 일반 프로세스보다 먼저 실행됨
# 95%는 매우 높음 → 다른 프로세스 starving 가능

docker rm -f normal-priority high-priority
```

### Step 2: Nice 값 조정

```bash
# 낮은 우선순위 (CPU 경쟁 시 후순위)
docker run -d --name low-priority \
  alpine sh -c 'nice -n 19 sh -c "while true; do :; done"'

# 높은 우선순위
docker run -d --name high-priority \
  alpine sh -c 'nice -n -20 sh -c "while true; do :; done"'

# CPU 경쟁 시:
# high-priority: 더 많은 CPU 시간
# low-priority: 적은 CPU 시간

# 확인
PID_LOW=$(docker inspect low-priority --format '{{.State.Pid}}')
PID_HIGH=$(docker inspect high-priority --format '{{.State.Pid}}')

ps -o pid,ni,cmd -p $PID_LOW,$PID_HIGH
# PID  NI   CMD
# 1234 19   sh -c while true...  (낮은 우선순위)
# 5678 -20  sh -c while true...  (높은 우선순위)

docker rm -f low-priority high-priority
```

---

## 🔧 실습 5: 성능 튜닝

### Step 1: CPU Governor 설정

```bash
# 현재 governor 확인
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# powersave (절전)
# performance (성능)
# ondemand (자동)

# Performance 모드로 변경 (최대 성능)
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
  echo performance | sudo tee $cpu
done

# 주파수 확인
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 3500000 (3.5 GHz)
```

### Step 2: IRQ Affinity

```bash
# 네트워크 IRQ 확인
cat /proc/interrupts | grep eth0

# IRQ를 특정 CPU에 고정
# (컨테이너는 다른 CPU 사용)
echo 1 | sudo tee /proc/irq/30/smp_affinity
# CPU 0에 IRQ 고정

echo fe | sudo tee /proc/irq/30/smp_affinity
# CPU 1-7에 IRQ 분산 (11111110 = 0xFE)
```

### Step 3: 컨테이너 성능 프로파일

```bash
# 레이턴시 민감 (웹 서버)
cat > latency-profile.yml <<'EOF'
version: '3.8'

services:
  web:
    image: nginx:alpine
    cpuset: "0,1"
    cpus: 2
    mem_limit: 1g
    mem_reservation: 512m
EOF

# 처리량 중시 (배치 작업)
cat > throughput-profile.yml <<'EOF'
version: '3.8'

services:
  batch:
    image: worker:latest
    cpu_shares: 2048
    cpus: 8
    mem_limit: 16g
EOF

# 균형 (일반 애플리케이션)
cat > balanced-profile.yml <<'EOF'
version: '3.8'

services:
  app:
    image: myapp:latest
    cpuset: "2,3,4,5"
    cpus: 4
    cpu_shares: 1024
    mem_limit: 4g
    mem_reservation: 2g
EOF
```

---

## 💡 주요 명령어 정리

### CPU Affinity

```bash
# Docker
docker run --cpuset-cpus="0,1" IMAGE
docker run --cpuset-cpus="0-3" IMAGE
docker update --cpuset-cpus="4,5" CONTAINER

# taskset (프로세스)
taskset -cp PID
taskset -cp 0,1 PID
```

### NUMA

```bash
# Docker
docker run --cpuset-cpus="0-7" --cpuset-mems="0" IMAGE

# numactl
numactl --hardware
numactl --show
numastat -c CONTAINER
```

### Throttling

```bash
# 확인
cat /sys/fs/cgroup/cpu/docker/CONTAINER_ID/cpu.stat

# CPU limit 조정
docker update --cpus=3 CONTAINER
```

### Priority

```bash
# RT Priority
docker run --cpu-rt-runtime=950000 --cpu-rt-period=1000000 IMAGE

# Nice
docker run IMAGE nice -n 19 COMMAND
docker run IMAGE nice -n -20 COMMAND
```

---

## 🎓 연습 문제

### 문제 1: NUMA 최적화

2-socket 서버(각 8 cores, 32GB)에서 데이터베이스 컨테이너를 최적으로 배치하세요.

<details>
<summary>정답 보기</summary>

```bash
# NUMA 확인
numactl --hardware
# node 0: cpu 0-7, memory 32GB
# node 1: cpu 8-15, memory 32GB

# 하나의 NUMA 노드에 배치
docker run -d --name database \
  --cpuset-cpus="0-7" \
  --cpuset-mems="0" \
  --memory=28g \
  postgres:15

# 이유:
# - 모든 CPU와 메모리가 같은 노드
# - NUMA remote 접근 제거
# - 메모리 대역폭 최대화
# - 레이턴시 최소화

# 검증
numastat -c database
# numa_miss가 거의 0에 가까워야 함
```

</details>

### 문제 2: CPU Throttling 진단

다음 통계에서 문제를 진단하고 해결하세요:

```
nr_periods: 10000
nr_throttled: 4500
throttled_time: 45000000000
```

<details>
<summary>정답 보기</summary>

**문제 진단:**
- Throttled: 45% (4500/10000)
- Throttled time: 45초
- 매우 높은 throttling → 성능 저하 심각

**해결책:**

```bash
# 1. CPU limit 확인
docker inspect CONTAINER | jq '.[0].HostConfig.NanoCpus'

# 2. CPU limit 2배 증가
docker update --cpus=4 CONTAINER

# 3. 또는 CPU period 증가 (더 유연)
docker stop CONTAINER
docker run -d --name CONTAINER \
  --cpu-period=200000 \
  --cpu-quota=800000 \
  IMAGE
# 같은 4 cores지만 period가 2배 → 덜 빈번한 체크

# 4. 검증
# throttled 비율이 10% 이하로 감소해야 함
```

</details>

### 문제 3: 레이턴시 최적화

P99 레이턴시를 5ms 이하로 줄이세요 (현재 15ms).

<details>
<summary>정답 보기</summary>

```bash
# 1. CPU Pinning (캐시 히트율 증가)
docker run -d --name latency-app \
  --cpuset-cpus="0,1" \
  --cpuset-mems="0" \
  myapp:latest

# 2. CPU Governor (최대 주파수)
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# 3. IRQ를 다른 CPU로
echo fe | sudo tee /proc/irq/30/smp_affinity
# CPU 1-7에 IRQ, 앱은 CPU 0 사용

# 4. Nice 값 조정
docker exec latency-app renice -n -20 -p 1

# 5. 검증
# 벤치마크로 P99 레이턴시 측정
# 목표: 15ms → 5ms 이하
```

</details>

---

## 📌 핵심 요약

### CPU 관리 기법 비교

| 기법 | 목적 | 효과 | 적용 대상 |
|-----|------|------|----------|
| **CPU Pinning** | 코어 고정 | Cache hit +30% | 레이턴시 민감 |
| **NUMA Binding** | 로컬 메모리 | 대역폭 +50% | 고성능 DB |
| **Throttling 조정** | 제한 완화 | 성능 회복 | 과부하 컨테이너 |
| **RT Priority** | 우선 실행 | 레이턴시 -60% | 실시간 앱 |

### NUMA Best Practices

```yaml
# 단일 NUMA 노드 사용 (권장)
cpuset-cpus: "0-7"
cpuset-mems: "0"

# 장점:
# - 100% 로컬 메모리 접근
# - 일관된 성능
# - 메모리 대역폭 최대

# 피해야 할 것:
# - 여러 NUMA 노드에 걸친 cpuset
# - cpuset-mems 미지정
```

### Throttling 가이드라인

```
Throttled %:
- 0-5%: 정상
- 5-20%: 주의 (모니터링 강화)
- 20-50%: 경고 (CPU limit 증가 검토)
- 50%+: 위험 (즉시 조치 필요)

해결 방법:
1. CPU limit 증가
2. CPU period 증가 (유연성)
3. 워크로드 분산
```

### 성능 최적화 체크리스트

- [ ] CPU Pinning 적용 (레이턴시 앱)
- [ ] NUMA 노드 단일 사용
- [ ] Throttling 모니터링 (<10%)
- [ ] CPU Governor performance 설정
- [ ] IRQ와 앱 CPU 분리
- [ ] 불필요한 Context Switch 제거

---

## 📚 참고 자료

- [Linux CPU Affinity](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html)
- [NUMA Deep Dive](https://www.kernel.org/doc/html/latest/vm/numa.html)
- [CPU Throttling in Containers](https://engineering.squarespace.com/blog/2017/understanding-linux-container-scheduling)
- [Real-time Priority](https://www.kernel.org/doc/Documentation/scheduler/sched-rt-group.txt)

---

## 🤔 생각해볼 문제

1. CPU Pinning은 왜 캐시 히트율을 높일까?
2. NUMA Remote 접근이 왜 느릴까?
3. CPU Throttling은 어떻게 감지할 수 있을까?

> 💡 **답변**: 1) CPU가 고정되면 프로세스가 항상 같은 코어에서 실행 → L1/L2 캐시에 데이터가 유지됨, CPU 이동 시 캐시 무효화(Cache Invalidation) → 메모리에서 다시 로드 필요 (100배 느림), 예: CPU 0 캐시에 데이터 → CPU 3으로 이동 → CPU 3 캐시에 없음 → 메모리 접근, Pinning → 같은 코어 → 캐시 히트 → 빠름, 실측: Cache miss 30% → 5% 감소, 2) NUMA는 메모리가 각 CPU에 로컬로 연결됨, Local 접근: CPU → Interconnect → 같은 소켓 메모리 (100ns), Remote 접근: CPU → Interconnect → QPI/UPI → 다른 소켓 메모리 (200ns), QPI/UPI 대역폭 제한 + 레이턴시 증가, 예: Node 0 CPU가 Node 1 메모리 접근 → 2배 느림, 해결: CPU와 메모리를 같은 NUMA 노드에 배치, 3) cgroup cpu.stat 파일 확인, nr_throttled / nr_periods = Throttling 비율, throttled_time = 제한된 총 시간, 실시간 모니터링: `watch cat /sys/fs/cgroup/cpu/docker/<ID>/cpu.stat`, 증상: CPU가 limit에 도달했는데 더 필요함 → Throttling, 응답 시간 증가, 처리량 감소, Prometheus/Grafana로 시각화 권장

---

<div align="center">

**[⬅️ 이전: Resource Limits](./01-Resource-Limits.md)** | **[다음: Memory Management ➡️](./03-Memory-Management.md)**

</div>
