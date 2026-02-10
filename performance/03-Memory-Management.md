# 03. Memory Management - 메모리 관리

## 🎯 이 챕터에서 배울 것

- **메모리 누수 탐지** - 누수 패턴 분석 및 진단
- **OOM Killer** - Out of Memory 처리 전략
- **Swap 관리** - Swap 사용 최적화
- **메모리 프로파일링** - Valgrind, heaptrack 활용
- **cgroup Memory** - 메모리 제어 메커니즘

## 📌 왜 중요한가?

**"메모리 관리는 안정적인 서비스 운영과 예측 가능한 성능의 핵심입니다."**

```
메모리 관리 부재 vs 체계적 관리:

메모리 관리 없는 환경:
┌─────────────────────────────────────┐
│ Container A (메모리 누수)              │
│ 09:00 - Start: 500MB                │
│ 12:00 - Grow: 2GB                   │
│ 15:00 - Grow: 8GB                   │
│ 18:00 - OOM: 전체 시스템 다운           │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Host (16GB RAM)                     │
│ OOM Killer 동작:                     │
│ - Container A? (8GB 사용)            │
│ - Container B? (중요 서비스, 4GB)      │
│ - sshd? (관리 접속)                   │
│ → 무작위 선택 → 서비스 중단              │
└─────────────────────────────────────┘

결과:
❌ 예측 불가능한 장애
❌ 중요 서비스 종료 가능
❌ 복구 불가능 (SSH 종료 시)
❌ 데이터 손실 위험

체계적 메모리 관리:
┌─────────────────────────────────────┐
│ Container A (제한: 2GB)              │
│ 09:00 - Start: 500MB                │
│ 12:00 - Grow: 1.5GB                 │
│ 15:00 - Grow: 2GB (limit)           │
│ 15:01 - OOM: Container A만 재시작     │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Host (16GB RAM)                     │
│ - Container A: 재시작 (격리됨)         │
│ - Container B: 정상 동작 (4GB)        │
│ - sshd: 정상 동작                     │
│ → 다른 서비스 영향 없음                  │
└─────────────────────────────────────┘

결과:
✅ 격리된 장애
✅ 자동 복구
✅ 다른 서비스 보호
✅ 예측 가능한 동작

메모리 관리 핵심 개념:

1. 메모리 계층:
   
   ┌─────────────────────────────┐
   │ RSS (Resident Set Size)     │
   │ 실제 물리 메모리 사용            │
   └──────────┬──────────────────┘
              │
   ┌──────────▼──────────────────┐
   │ Cache                       │
   │ 페이지 캐시, 버퍼               │
   │ (회수 가능)                   │
   └──────────┬──────────────────┘
              │
   ┌──────────▼──────────────────┐
   │ Swap                        │
   │ 디스크 기반 가상 메모리           │
   │ (느림)                       │
   └─────────────────────────────┘

2. OOM Score:
   
   OOM Killer 선택 기준:
   
   Score = (Memory Usage %) × 10 + oom_score_adj
   
   예시:
   Container A: 80% × 10 + 0 = 800
   Container B: 40% × 10 - 500 = -100
   sshd: 1% × 10 - 1000 = -990
   
   → Container A가 먼저 종료됨
   
   oom_score_adj 범위:
   -1000 (절대 종료 안 함)
   0 (기본값)
   +1000 (먼저 종료)

3. 메모리 누수 패턴:
   
   정상:
   ┌──────────────────────────────┐
   │ Memory                       │
   │   ▲                          │
   │   │    ┌───┐                 │
   │   │    │   │  ┌───┐          │
   │   │ ┌──┘   └──┘   └──┐       │
   │   └──────────────────────→   │
   │        Time                  │
   └──────────────────────────────┘
   → 일정 범위 내 변동
   
   누수:
   ┌──────────────────────────────┐
   │ Memory                       │
   │   ▲                          │
   │   │              ┌───        │
   │   │         ┌────┘           │
   │   │    ┌────┘                │
   │   │ ┌──┘                     │
   │   └──────────────────────→   │
   │        Time                  │
   └──────────────────────────────┘
   → 지속적 증가

4. Swap의 영향:
   
   Without Swap:
   Memory Full → OOM Killer → 프로세스 종료
   
   With Swap:
   Memory Full → Swap 사용 → 성능 저하
   → Disk I/O 대기 → 응답 시간 증가
   
   Swap 사용 시 성능:
   RAM: 100ns
   SSD Swap: 100,000ns (1000배 느림)
   HDD Swap: 10,000,000ns (100,000배 느림)

실무 시나리오:

시나리오 1 - 메모리 누수 장애:
┌─────────────────────────────────────┐
│ 월요일 09:00 - 서비스 시작              │
│ Memory: 500MB                       │
│ Requests/s: 1000                    │
│ Latency: 50ms                       │
└────────────┬────────────────────────┘
             │ (시간 경과)
┌────────────▼────────────────────────┐
│ 화요일 15:00 - 메모리 증가              │
│ Memory: 3GB                         │
│ Requests/s: 800 (감소)               │
│ Latency: 200ms (증가)                │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 수요일 02:00 - OOM Killer             │
│ Memory: 8GB (limit)                 │
│ → OOM Killed                        │
│ → 서비스 다운 (30분)                   │
└─────────────────────────────────────┘

근본 원인: Connection pool 미반환
해결: 연결 timeout 설정

시나리오 2 - Swap Thrashing:
┌─────────────────────────────────────┐
│ Normal Operation                    │
│ RAM: 2GB / 4GB                      │
│ Swap: 0GB                           │
│ Latency: 50ms                       │
└────────────┬────────────────────────┘
             │ (트래픽 증가)
┌────────────▼────────────────────────┐
│ Peak Traffic                        │
│ RAM: 4GB / 4GB (full)               │
│ Swap: 2GB (사용 시작)                 │
│ Latency: 5000ms (100배 증가!)         │
│ Disk I/O: 100% (병목)                │
└─────────────────────────────────────┘

해결: Swap 비활성화 + OOM Killer 허용
→ 일부 요청 실패 > 전체 서비스 느림

시나리오 3 - OOM Score 조정:
┌─────────────────────────────────────┐
│ 기본 설정:                            │
│ API (4GB): oom_score_adj = 0        │
│ Worker (2GB): oom_score_adj = 0     │
│ Cache (1GB): oom_score_adj = 0      │
│                                     │
│ OOM 발생 시:                          │
│ → API 종료 (가장 많은 메모리)            │
│ → 서비스 중단                          │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ 최적화:                               │
│ API (4GB): oom_score_adj = -500     │
│ Worker (2GB): oom_score_adj = 0     │
│ Cache (1GB): oom_score_adj = 500    │
│                                     │
│ OOM 발생 시:                          │
│ → Cache 종료 (우선순위 낮음)             │
│ → API 보호 (서비스 유지)                │
└─────────────────────────────────────┘
```

---

## 🔧 실습 1: 메모리 누수 탐지

### Step 1: 메모리 누수 시뮬레이션

```bash
# 누수가 있는 Python 애플리케이션
cat > leak-app.py <<'EOF'
import time
import os

# 메모리 누수: 리스트가 계속 증가
leak_list = []

def leak_memory():
    # 매번 1MB씩 누수
    leak_list.append(' ' * 1024 * 1024)

while True:
    leak_memory()
    print(f"Memory usage: {len(leak_list)} MB")
    time.sleep(1)
EOF

# Dockerfile
cat > Dockerfile.leak <<'EOF'
FROM python:3.11-slim
COPY leak-app.py /app/
WORKDIR /app
CMD ["python", "leak-app.py"]
EOF

docker build -t leak-app -f Dockerfile.leak .

# 2GB 제한으로 실행
docker run -d --name leak-test \
  --memory=2g \
  leak-app

# 메모리 사용 모니터링
docker stats leak-test

# 출력:
# CONTAINER   MEM USAGE / LIMIT
# leak-test   100MB / 2GB
# leak-test   500MB / 2GB
# leak-test   1GB / 2GB
# leak-test   1.5GB / 2GB
# leak-test   2GB / 2GB (limit)
# → OOM Killed

# 로그 확인
docker logs leak-test --tail 20
# Memory usage: 1900 MB
# Memory usage: 1950 MB
# Killed
```

### Step 2: 메모리 사용 패턴 분석

```bash
# 메모리 사용 기록
cat > monitor-memory.sh <<'EOF'
#!/bin/bash

CONTAINER=$1
LOG_FILE="memory-${CONTAINER}.csv"

echo "Timestamp,RSS,Cache,Swap" > $LOG_FILE

while true; do
  STATS=$(docker stats $CONTAINER --no-stream --format "{{.MemUsage}}")
  TIMESTAMP=$(date +%s)
  
  # cgroup에서 상세 정보
  CID=$(docker inspect $CONTAINER --format '{{.Id}}')
  CGROUP="/sys/fs/cgroup/memory/docker/$CID"
  
  RSS=$(cat $CGROUP/memory.stat | grep "^rss " | awk '{print $2}')
  CACHE=$(cat $CGROUP/memory.stat | grep "^cache " | awk '{print $2}')
  SWAP=$(cat $CGROUP/memory.stat | grep "^swap " | awk '{print $2}')
  
  echo "$TIMESTAMP,$RSS,$CACHE,$SWAP" >> $LOG_FILE
  sleep 5
done
EOF

chmod +x monitor-memory.sh
./monitor-memory.sh leak-test &

# 1시간 후 분석
pkill -f monitor-memory

# 시각화 (Python)
cat > plot-memory.py <<'EOF'
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('memory-leak-test.csv')
df['RSS_MB'] = df['RSS'] / 1024 / 1024
df['Time'] = df['Timestamp'] - df['Timestamp'].iloc[0]

plt.figure(figsize=(12, 6))
plt.plot(df['Time'], df['RSS_MB'])
plt.xlabel('Time (seconds)')
plt.ylabel('Memory (MB)')
plt.title('Memory Leak Detection')

# 선형 회귀로 누수율 계산
from numpy import polyfit
slope, intercept = polyfit(df['Time'], df['RSS_MB'], 1)
plt.plot(df['Time'], slope * df['Time'] + intercept, 'r--', 
         label=f'Leak rate: {slope:.2f} MB/s')

plt.legend()
plt.savefig('memory-leak.png')
print(f"Memory leak rate: {slope:.2f} MB/s")
EOF

python3 plot-memory.py
```

### Step 3: Valgrind로 누수 분석

```bash
# C 프로그램 누수 예시
cat > leak.c <<'EOF'
#include <stdlib.h>
#include <unistd.h>

int main() {
    while(1) {
        // 메모리 할당 후 해제 안 함 (누수)
        void* ptr = malloc(1024 * 1024); // 1MB
        sleep(1);
    }
    return 0;
}
EOF

gcc -o leak leak.c

# Valgrind로 누수 탐지
docker run --rm \
  -v $(pwd):/app \
  -w /app \
  valgrind/valgrind \
  valgrind --leak-check=full --show-leak-kinds=all ./leak

# 출력:
# ==1== HEAP SUMMARY:
# ==1==     in use at exit: 10,485,760 bytes in 10 blocks
# ==1==   total heap usage: 10 allocs, 0 frees, 10,485,760 bytes allocated
# ==1==
# ==1== LEAK SUMMARY:
# ==1==    definitely lost: 10,485,760 bytes in 10 blocks
#                           ↑ 누수 발견!

docker rm -f leak-test
```

---

## 🔧 실습 2: OOM Killer 관리

### Step 1: OOM Killer 동작 확인

```bash
# 메모리 한계 초과 테스트
docker run --name oom-test \
  --memory=100m \
  alpine sh -c '
    echo "Allocating memory..."
    yes | tr \\n x | head -c 200m | grep n
  '

# 종료 코드 확인
docker inspect oom-test --format '{{.State.ExitCode}}'
# 137 (128 + 9 SIGKILL)

docker inspect oom-test --format '{{.State.OOMKilled}}'
# true

# 호스트 로그 확인
dmesg | grep -i oom | tail -20

# 출력:
# [12345.678] oom-kill:constraint=CONSTRAINT_MEMCG,
#             nodemask=(null),cpuset=docker-...,
#             mems_allowed=0,oom_memcg=/docker/...,
#             task_memcg=/docker/...,task=yes,pid=12345,uid=0
# [12345.679] Memory cgroup out of memory: Killed process 12345 (yes)

docker rm oom-test
```

### Step 2: OOM Score 조정

```bash
# 기본 OOM Score
docker run -d --name default-oom \
  --memory=1g \
  alpine sleep 3600

CID=$(docker inspect default-oom --format '{{.Id}}')
cat /sys/fs/cgroup/memory/docker/$CID/memory.oom_control

# 출력:
# oom_kill_disable 0
# under_oom 0
# oom_kill 0

# OOM Score 확인
PID=$(docker inspect default-oom --format '{{.State.Pid}}')
cat /proc/$PID/oom_score
# 예: 200

cat /proc/$PID/oom_score_adj
# 0

# 중요 컨테이너 (낮은 점수 = 보호)
docker run -d --name important \
  --memory=1g \
  --oom-score-adj=-500 \
  alpine sleep 3600

PID=$(docker inspect important --format '{{.State.Pid}}')
cat /proc/$PID/oom_score_adj
# -500

# 덜 중요한 컨테이너 (높은 점수 = 먼저 종료)
docker run -d --name disposable \
  --memory=1g \
  --oom-score-adj=500 \
  alpine sleep 3600

# OOM 발생 시:
# 1. disposable (oom_score_adj=500)
# 2. default-oom (oom_score_adj=0)
# 3. important (oom_score_adj=-500)

docker rm -f default-oom important disposable
```

### Step 3: OOM 방지 전략

```bash
# 전략 1: OOM Killer 비활성화 (비권장)
docker run -d --name no-oom-kill \
  --memory=1g \
  --oom-kill-disable \
  alpine sleep 3600

# 주의: 메모리 초과 시 프로세스가 멈춤 (frozen)
# 컨테이너는 살아있지만 응답 없음

# 전략 2: Memory Reservation (권장)
docker run -d --name with-reservation \
  --memory=2g \
  --memory-reservation=1g \
  myapp:latest

# 동작:
# - 평소: 2GB까지 사용 가능
# - Host 메모리 부족 시: 1GB로 축소 시도
# - 1GB 미만으로 못 줄이면: OOM

# 전략 3: Swap 사용
docker run -d --name with-swap \
  --memory=1g \
  --memory-swap=2g \
  myapp:latest

# - RAM: 1GB
# - Swap: 1GB
# - OOM 가능성 낮아짐 (대신 느려짐)

docker rm -f no-oom-kill with-reservation with-swap
```

---

## 🔧 실습 3: Swap 관리

### Step 1: Swap 동작 확인

```bash
# Swap 활성화
docker run -d --name swap-test \
  --memory=512m \
  --memory-swap=1g \
  alpine sh -c 'tail -f /dev/null'

# Swap 사용 강제
docker exec swap-test sh -c '
  # 700MB 할당 (RAM 512MB 초과)
  yes | tr \\n x | head -c 700m > /tmp/data &
  sleep 5
'

# cgroup 확인
CID=$(docker inspect swap-test --format '{{.Id}}')
CGROUP="/sys/fs/cgroup/memory/docker/$CID"

cat $CGROUP/memory.usage_in_bytes
# 536870912 (512MB - RAM limit)

cat $CGROUP/memory.memsw.usage_in_bytes
# 734003200 (700MB - RAM + Swap)

cat $CGROUP/memory.stat | grep swap
# swap 188743680 (180MB swap 사용)

docker rm -f swap-test
```

### Step 2: Swap 성능 영향 측정

```bash
# Swap 없음
docker run --rm \
  --memory=512m \
  --memory-swap=512m \
  sysbench:latest \
  sysbench memory --memory-total-size=400M run

# 출력:
# Total time: 1.5s
# transferred (266.67 MiB/sec)

# Swap 있음 (사용 중)
docker run --rm \
  --memory=512m \
  --memory-swap=2g \
  sysbench:latest \
  sysbench memory --memory-total-size=1.5G run

# 출력:
# Total time: 45.2s  (30배 느림!)
# transferred (33.18 MiB/sec)
```

### Step 3: Swappiness 조정

```bash
# Host swappiness 확인
cat /proc/sys/vm/swappiness
# 60 (기본값)

# 0 = Swap 최소화 (OOM 위험)
# 100 = Swap 적극 사용 (성능 저하)

# Container swappiness 조정
docker run -d --name low-swap \
  --memory=1g \
  --memory-swappiness=10 \
  myapp:latest

# 확인
CID=$(docker inspect low-swap --format '{{.Id}}')
cat /sys/fs/cgroup/memory/docker/$CID/memory.swappiness
# 10

docker rm -f low-swap
```

---

## 🔧 실습 4: 메모리 프로파일링

### Step 1: heaptrack (C/C++)

```bash
# heaptrack 설치
sudo apt-get install -y heaptrack

# 프로그램 프로파일링
cat > allocate.c <<'EOF'
#include <stdlib.h>
#include <unistd.h>

int main() {
    for(int i = 0; i < 1000; i++) {
        malloc(1024 * 1024); // 1MB 할당
        sleep(1);
    }
    return 0;
}
EOF

gcc -o allocate allocate.c

# heaptrack 실행
heaptrack ./allocate

# 분석 (GUI)
heaptrack_gui heaptrack.allocate.12345.gz

# 또는 CLI
heaptrack_print heaptrack.allocate.12345.gz | head -50

# 출력:
# MOST MEMORY CONSUMING LOCATIONS
# 1024.00MB in 1000 allocations from:
#   allocate.c:5 (main)
```

### Step 2: memory_profiler (Python)

```bash
# memory_profiler 설치
pip install memory_profiler

# 프로파일링할 코드
cat > profile_mem.py <<'EOF'
from memory_profiler import profile

@profile
def leak_function():
    leak_list = []
    for i in range(1000):
        leak_list.append(' ' * 1024 * 1024)  # 1MB
    return leak_list

if __name__ == '__main__':
    result = leak_function()
EOF

# 실행
python -m memory_profiler profile_mem.py

# 출력:
# Line #    Mem usage    Increment   Line Contents
# ================================================
#      3   10.5 MiB     10.5 MiB   @profile
#      4                           def leak_function():
#      5   10.5 MiB      0.0 MiB       leak_list = []
#      6  1010.5 MiB   1000.0 MiB       for i in range(1000):
#      7  1010.5 MiB      0.0 MiB           leak_list.append(' ' * 1024 * 1024)
```

### Step 3: pmap (프로세스 메모리 맵)

```bash
# 메모리 맵 확인
docker run -d --name pmap-test \
  --memory=1g \
  alpine sleep 3600

PID=$(docker inspect pmap-test --format '{{.State.Pid}}')

# 전체 메모리 맵
sudo pmap -x $PID

# 출력:
# Address           Kbytes     RSS   Dirty Mode
# 0000000000400000       4       4       0 r-x-- sleep
# 0000000000600000       4       4       4 rw--- sleep
# 00007f1234000000   65536   32768   32768 rw---   [ anon ]
# ...
# total kB         1048576  524288  262144

# 요약만
sudo pmap -X $PID | tail -5

docker rm -f pmap-test
```

---

## 🔧 실습 5: 메모리 제한 전략

### Step 1: Production 설정

```bash
# 웹 서버 (안정성 중시)
docker run -d --name production-web \
  --memory=2g \
  --memory-swap=2g \
  --memory-reservation=1g \
  --oom-score-adj=-500 \
  nginx:alpine

# - Hard limit: 2GB
# - Swap: 0 (비활성화)
# - Soft limit: 1GB (압박 시)
# - OOM 우선순위: 낮음 (보호)
```

### Step 2: Development 설정

```bash
# 개발 환경 (유연성)
docker run -d --name dev-app \
  --memory=1g \
  --memory-swap=2g \
  --memory-reservation=512m \
  myapp:dev

# - Hard limit: 1GB
# - Swap: 1GB (여유)
# - Soft limit: 512MB
# - OOM 우선순위: 기본
```

### Step 3: Batch 작업

```bash
# 대용량 처리
docker run -d --name batch-job \
  --memory=8g \
  --memory-swap=16g \
  --oom-score-adj=500 \
  data-processor:latest

# - Hard limit: 8GB
# - Swap: 8GB (대용량 허용)
# - OOM 우선순위: 높음 (먼저 종료 가능)
```

---

## 💡 주요 명령어 정리

### 메모리 제한

```bash
# Hard limit
docker run --memory=1g IMAGE
docker run -m 1g IMAGE

# Soft limit
docker run --memory-reservation=512m IMAGE

# Swap
docker run --memory=1g --memory-swap=2g IMAGE

# Swappiness
docker run --memory-swappiness=10 IMAGE

# OOM Score
docker run --oom-score-adj=-500 IMAGE
docker run --oom-kill-disable IMAGE  # 비권장
```

### 메모리 확인

```bash
# Stats
docker stats CONTAINER

# cgroup
cat /sys/fs/cgroup/memory/docker/CONTAINER_ID/memory.usage_in_bytes
cat /sys/fs/cgroup/memory/docker/CONTAINER_ID/memory.stat

# OOM
docker inspect CONTAINER --format '{{.State.OOMKilled}}'

# pmap
pmap -x PID
```

---

## 🎓 연습 문제

### 문제 1: 메모리 누수 진단

다음 메모리 사용 패턴에서 누수 여부를 판단하세요:

```
09:00 - 500MB
12:00 - 800MB
15:00 - 1.1GB
18:00 - 1.4GB
21:00 - 1.7GB
```

<details>
<summary>정답 보기</summary>

**진단: 메모리 누수 확실**

증거:
- 시간에 비례한 선형 증가
- 3시간마다 약 300MB 증가
- 누수율: 100MB/hour

해결 단계:
```bash
# 1. 메모리 프로파일링
python -m memory_profiler app.py

# 2. 누수 지점 확인
# 예: 리스트/딕셔너리 무한 증가
# 예: 파일 핸들 미해제
# 예: 이벤트 리스너 미제거

# 3. 임시 조치: 자동 재시작
docker update --restart=always CONTAINER

# 4. 근본 수정: 코드 수정
```

</details>

### 문제 2: OOM Score 설정

다음 컨테이너들의 OOM Score를 설정하세요:

- API Server (중요, 4GB)
- Worker (보통, 2GB)
- Cache (선택, 1GB)

<details>
<summary>정답 보기</summary>

```bash
# API (가장 보호)
docker run -d --name api \
  --memory=4g \
  --oom-score-adj=-500 \
  api:latest

# Worker (기본)
docker run -d --name worker \
  --memory=2g \
  --oom-score-adj=0 \
  worker:latest

# Cache (먼저 종료)
docker run -d --name cache \
  --memory=1g \
  --oom-score-adj=500 \
  redis:alpine

# OOM 발생 시 종료 순서:
# 1. Cache (adj=500)
# 2. Worker (adj=0)
# 3. API (adj=-500) - 마지막
```

</details>

### 문제 3: Swap 전략

Database 컨테이너에 Swap을 활성화해야 할까?

<details>
<summary>정답 보기</summary>

**답: 아니오 (Swap 비활성화 권장)**

이유:
```bash
# Database는 메모리 직접 접근 전제
# Swap 사용 시:
# - 쿼리 성능 100-1000배 저하
# - Disk I/O 병목
# - 예측 불가능한 레이턴시

# 올바른 설정:
docker run -d --name database \
  --memory=8g \
  --memory-swap=8g \        # Swap 0
  --memory-swappiness=0 \   # Swap 최소화
  postgres:15

# Swap 대신:
# 1. 충분한 RAM 할당
# 2. OOM 발생 시 재시작 허용
# 3. 쿼리 캐시 최적화
```

</details>

---

## 📌 핵심 요약

### 메모리 관리 전략

| 워크로드 | Memory | Swap | OOM Score | 목표 |
|---------|--------|------|-----------|------|
| **API** | 2-4GB | 0 | -500 | 안정성 |
| **Worker** | 1-2GB | 0 | 0 | 균형 |
| **Cache** | 1GB | 0 | 500 | 유연성 |
| **Batch** | 8GB+ | 허용 | 500 | 처리량 |
| **Database** | 8GB+ | 0 | -500 | 성능 |

### OOM Score 가이드

```
-1000: 절대 종료 안 함 (커널 프로세스)
-500:  매우 중요 (API, DB)
0:     기본값 (일반 앱)
+500:  낮은 우선순위 (Cache, Worker)
+1000: 먼저 종료 (임시 작업)
```

### Swap 사용 가이드라인

```
Swap 비활성화 (권장):
- Database
- 레이턴시 민감 애플리케이션
- Production 웹 서버

Swap 허용:
- Batch 작업
- 대용량 데이터 처리
- Development 환경
```

### 메모리 누수 체크리스트

- [ ] 시간에 따른 메모리 증가 패턴 확인
- [ ] 메모리 프로파일러 실행
- [ ] 리소스 해제 누락 검사
- [ ] Connection pool 설정 확인
- [ ] Event listener 정리 확인
- [ ] 캐시 만료 정책 확인

---

## 📚 참고 자료

- [Linux OOM Killer](https://www.kernel.org/doc/gorman/html/understand/understand016.html)
- [cgroup Memory Controller](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)
- [Valgrind User Manual](https://valgrind.org/docs/manual/manual.html)
- [Python memory_profiler](https://pypi.org/project/memory-profiler/)

---

## 🤔 생각해볼 문제

1. OOM Killer는 어떤 기준으로 프로세스를 선택할까?
2. Swap을 사용하면 왜 성능이 저하될까?
3. 메모리 누수를 조기에 발견하려면 어떻게 해야 할까?

> 💡 **답변**: 1) OOM Score 계산: Score = (Memory Usage %) × 10 + oom_score_adj, 높은 점수부터 종료, 메모리 많이 사용 + 높은 adj = 먼저 종료, 예: 80% 메모리 사용 + adj 500 = 800 + 500 = 1300, 커널 프로세스는 adj = -1000 (거의 종료 안 됨), 사용자가 adj 조정 가능 (--oom-score-adj), 목표: 시스템 전체 다운 방지, 덜 중요한 프로세스부터 종료, 2) RAM vs Swap 속도 비교, RAM: 100ns (나노초), SSD: 100,000ns (1000배 느림), HDD: 10,000,000ns (100,000배 느림), Swap 사용 시 Page Fault 발생 → Disk I/O 대기, 빈번한 swap in/out = thrashing, CPU는 놀고 Disk만 바쁨, Database 특히 치명적 (메모리 직접 접근 전제), 해결: Swap 비활성화, 충분한 RAM 할당, 3) 조기 발견 방법, 메모리 사용량 지속 모니터링 (Prometheus), 증가 패턴 분석 (선형 회귀), 임계값 알림 (80% 초과 시), 정기 프로파일링 (주간), 자동 재시작 정책, 로그 분석 (GC 빈도, 할당 패턴), 예방: Connection timeout, 캐시 TTL, Resource pooling, 코드 리뷰

---

<div align="center">

**[⬅️ 이전: CPU Management](./02-CPU-Management.md)** | **[다음: I/O Performance ➡️](./04-IO-Performance.md)**

</div>
