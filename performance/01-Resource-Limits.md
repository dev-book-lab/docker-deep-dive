# 01. Resource Limits - 리소스 제한 전략

## 🎯 이 챕터에서 배울 것

- **CPU 제한** - shares, quota, cpuset 전략
- **Memory 제한** - limit, reservation, swap 설정
- **리소스 격리** - cgroup 기반 제어
- **제한 전략** - 워크로드별 최적 설정
- **모니터링** - 리소스 사용량 추적

## 📌 왜 중요한가?

**"리소스 제한은 안정적인 멀티 테넌시와 예측 가능한 성능의 기반입니다."**

```
리소스 제한 없이 vs 리소스 제한 적용:

리소스 제한 없는 환경:
┌─────────────────────────────────────┐
│ Host (16 CPU, 32GB RAM)             │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ Container A (메모리 누수)          │ │
│ │ CPU: 1600% (16 cores)           │ │
│ │ Memory: 28GB (88%)              │ │
│ └─────────────────────────────────┘ │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ Container B,C,D (정상)           │ │
│ │ CPU: 0% (Starved!)              │ │
│ │ Memory: OOM Killed              │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘

문제:
❌ Noisy Neighbor - 한 컨테이너가 전체 독점
❌ 서비스 다운 - OOM Killer가 무작위 종료
❌ 예측 불가 - 성능이 다른 컨테이너에 의존
❌ 비용 낭비 - 오버 프로비저닝 필요

리소스 제한 적용:
┌─────────────────────────────────────┐
│ Host (16 CPU, 32GB RAM)             │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ Container A (제한 적용)           │ │
│ │ CPU: 400% (4 cores max)         │ │
│ │ Memory: 8GB (제한)               │ │
│ │ → 초과 시 Throttling             │ │
│ └─────────────────────────────────┘ │
│                                     │
│ ┌─────────────────────────────────┐ │
│ │ Container B,C,D (보호됨)          │ │
│ │ CPU: 각 400% (보장)               │ │
│ │ Memory: 각 8GB (안정)             │ │
│ └─────────────────────────────────┘ │
│                                     │
│ 여유: 0 CPU, 0 RAM (효율적 활용)        │
└─────────────────────────────────────┘

장점:
✅ 격리 - 서로 영향 없음
✅ 안정성 - 예측 가능한 성능
✅ 효율 - 리소스 최대 활용
✅ 비용 - 오버 프로비저닝 불필요

리소스 제한 핵심 개념:

1. CPU 제한 3가지 방식:

   a) Shares (상대적 가중치):
   ┌──────────────────────────────┐
   │ --cpu-shares=1024 (기본값)     │
   └──────────────────────────────┘
   
   경쟁 상황에서만 적용:
   A (2048) vs B (1024) vs C (1024)
   → A: 50%, B: 25%, C: 25%
   
   유휴 시에는 제한 없음:
   A만 실행 중 → A가 100% 사용 가능
   
   b) Quota (절대적 제한):
   ┌──────────────────────────────┐
   │ --cpus=2.5                   │
   │ = --cpu-period=100000        │
   │   --cpu-quota=250000         │
   └──────────────────────────────┘
   
   항상 적용:
   유휴 여부와 관계없이 최대 2.5 cores
   
   c) Cpuset (코어 고정):
   ┌──────────────────────────────┐
   │ --cpuset-cpus="0,1"          │
   │ --cpuset-cpus="0-3,6-7"      │
   └──────────────────────────────┘
   
   특정 CPU 코어에 바인딩
   NUMA 환경에 유용

2. Memory 제한 계층:

   ┌─────────────────────────────┐
   │ Hard Limit                  │
   │ --memory=1g                 │
   │ 초과 시 OOM Killer            │
   └──────────┬──────────────────┘
              │
   ┌──────────▼──────────────────┐
   │ Soft Limit                  │
   │ --memory-reservation=512m   │
   │ 메모리 압박 시 우선 회수          │
   └──────────┬──────────────────┘
              │
   ┌──────────▼──────────────────┐
   │ Swap Limit                  │
   │ --memory-swap=2g            │
   │ (Memory + Swap = 2g)        │
   └─────────────────────────────┘

   설정 조합:
   --memory=1g --memory-swap=2g
   → RAM: 1GB, Swap: 1GB
   
   --memory=1g --memory-swap=1g
   → RAM: 1GB, Swap: 0GB (Swap 비활성화)
   
   --memory=1g --memory-swap=-1
   → RAM: 1GB, Swap: 무제한 (비권장!)

3. 실무 전략 비교:

   보수적 (Production):
   ┌──────────────────────────────┐
   │ CPU: --cpus=2 (고정)          │
   │ Memory: --memory=4g          │
   │ Swap: --memory-swap=4g       │
   │ (Swap 비활성화)                │
   └──────────────────────────────┘
   
   장점: 예측 가능, 안정적
   단점: 리소스 낭비 가능
   
   적극적 (Development):
   ┌───────────────────────────────┐
   │ CPU: --cpu-shares=512         │
   │ Memory: --memory=2g           │
   │ Reservation: -m-reservation=1g│
   │ Swap: --memory-swap=4g        │
   └───────────────────────────────┘
   
   장점: 유연, 효율적
   단점: 경쟁 시 성능 변동

실무 시나리오:

시나리오 1 - 메모리 누수:
┌─────────────────────────────────────┐
│ 09:00 - 애플리케이션 시작               │
│ Memory: 500MB                       │
└────────────┬────────────────────────┘
             │ (시간 경과)
┌────────────▼────────────────────────┐
│ 12:00 - 메모리 증가                    │
│ Memory: 2GB → 3GB → 4GB             │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Without Limit:                      │
│ Memory: 8GB → 16GB → 24GB           │
│ → Host OOM → 다른 컨테이너 종료         │
└─────────────────────────────────────┘

┌────────────────────────────────────┐
│ With Limit (--memory=4g):          │
│ Memory: 3.8GB → 4GB (limit)        │
│ → 해당 컨테이너만 OOM Killed           │
│ → 자동 재시작 (--restart=always)      │
│ → 다른 컨테이너 정상 동작                │
└────────────────────────────────────┘

시나리오 2 - CPU 버스트:
┌─────────────────────────────────────┐
│ 정상 상태: CPU 10%                    │
│ (4개 컨테이너 각 10% = 40% 사용)        │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Container A: 갑자기 100% 필요          │
│                                     │
│ Without Limit:                      │
│ A: 100% (다른 컨테이너 Starved)        │
│ B,C,D: 0-5% (응답 지연)               │
└─────────────────────────────────────┘

┌────────────────────────────────────┐
│ With Shares (각 1024):              │
│ A: 40% (25% 보장 + 15% 추가)         │
│ B,C,D: 각 20% (정상 동작)             │
│                                    │
│ With Quota (각 --cpus=2):           │
│ A: 200% (2 cores, 제한됨)            │
│ B,C,D: 각 200% (보장됨)              │
└────────────────────────────────────┘
```

---

## 🔧 실습 1: CPU 제한

### Step 1: CPU Shares (상대적 가중치)

```bash
# 기본값 확인
docker run --rm alpine cat /sys/fs/cgroup/cpu/cpu.shares
# 1024

# CPU shares 설정
docker run -d --name high-priority \
  --cpu-shares=2048 \
  alpine sh -c 'while true; do :; done'

docker run -d --name low-priority \
  --cpu-shares=512 \
  alpine sh -c 'while true; do :; done'

# 리소스 사용 확인
docker stats --no-stream

# 출력 (CPU 경쟁 시):
# CONTAINER      CPU %
# high-priority  66.67%    (2048 / 3072)
# low-priority   33.33%    (512 / 3072)

# 정리
docker rm -f high-priority low-priority
```

**경쟁 없을 때 vs 있을 때:**
```bash
# 실험: 하나만 실행
docker run -d --name solo --cpu-shares=512 alpine sh -c 'while true; do :; done'
docker stats --no-stream solo
# CPU %: ~100% (shares 무시, 전체 사용)

# 경쟁 상황 생성
docker run -d --name competitor --cpu-shares=1024 alpine sh -c 'while true; do :; done'
docker stats --no-stream

# 출력:
# solo:       33%   (512/1536)
# competitor: 67%   (1024/1536)

docker rm -f solo competitor
```

### Step 2: CPU Quota (절대적 제한)

```bash
# CPU 2개로 제한
docker run -d --name limited \
  --cpus=2 \
  alpine sh -c 'while true; do :; done'

# 확인
docker stats --no-stream limited
# CPU %: ~200% (2 cores)

# 내부 확인
docker exec limited cat /sys/fs/cgroup/cpu/cpu.cfs_period_us
# 100000 (100ms)

docker exec limited cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
# 200000 (200ms per 100ms = 2 cores)

# 0.5 CPU 제한
docker run -d --name half \
  --cpus=0.5 \
  alpine sh -c 'while true; do :; done'

docker stats --no-stream half
# CPU %: ~50% (0.5 cores)

# 수동 설정 (같은 효과)
docker run -d --name manual \
  --cpu-period=100000 \
  --cpu-quota=50000 \
  alpine sh -c 'while true; do :; done'

docker rm -f limited half manual
```

### Step 3: CPU Pinning (코어 고정)

```bash
# 시스템 CPU 정보
lscpu | grep "^CPU(s)"
# CPU(s): 8

# CPU 0,1에 고정
docker run -d --name pinned-0-1 \
  --cpuset-cpus="0,1" \
  alpine sh -c 'while true; do :; done'

# CPU 2-5에 고정
docker run -d --name pinned-2-5 \
  --cpuset-cpus="2-5" \
  alpine sh -c 'while true; do :; done'

# 확인
docker exec pinned-0-1 cat /sys/fs/cgroup/cpuset/cpuset.cpus
# 0-1

docker exec pinned-2-5 cat /sys/fs/cgroup/cpuset/cpuset.cpus
# 2-5

# 프로세스 확인 (htop으로)
htop
# pinned-0-1은 CPU 0,1만 사용
# pinned-2-5는 CPU 2,3,4,5만 사용

docker rm -f pinned-0-1 pinned-2-5
```

**NUMA 환경에서 최적화:**
```bash
# NUMA 노드 확인
numactl --hardware

# 출력:
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3
# node 1 cpus: 4 5 6 7

# Node 0에 바인딩
docker run -d --name numa-node-0 \
  --cpuset-cpus="0-3" \
  --cpuset-mems="0" \
  myapp:latest

# Node 1에 바인딩
docker run -d --name numa-node-1 \
  --cpuset-cpus="4-7" \
  --cpuset-mems="1" \
  myapp:latest
```

---

## 🔧 실습 2: Memory 제한

### Step 1: Hard Limit (필수)

```bash
# 512MB 제한
docker run -d --name mem-512m \
  --memory=512m \
  alpine sh -c 'tail -f /dev/null'

# 확인
docker stats --no-stream mem-512m

# 메모리 초과 테스트
docker exec mem-512m sh -c 'yes | tr \\n x | head -c 1024m | grep n'
# Killed (OOM)

docker logs mem-512m --tail 5
# (empty - 프로세스가 종료됨)

# 호스트에서 OOM 확인
dmesg | grep -i oom | tail -5
# Memory cgroup out of memory: Killed process ...

docker rm -f mem-512m
```

### Step 2: Soft Limit (Reservation)

```bash
# Hard: 1GB, Soft: 512MB
docker run -d --name mem-reserved \
  --memory=1g \
  --memory-reservation=512m \
  alpine sh -c 'tail -f /dev/null'

# cgroup 확인
docker exec mem-reserved cat /sys/fs/cgroup/memory/memory.limit_in_bytes
# 1073741824 (1GB)

docker exec mem-reserved cat /sys/fs/cgroup/memory/memory.soft_limit_in_bytes
# 536870912 (512MB)

# 동작:
# 평소: 1GB까지 사용 가능
# Host 메모리 부족 시: 512MB 이상 사용 중이면 우선 회수

docker rm -f mem-reserved
```

### Step 3: Swap 제어

```bash
# Swap 비활성화 (권장 - Production)
docker run -d --name no-swap \
  --memory=1g \
  --memory-swap=1g \
  alpine sh -c 'tail -f /dev/null'

# 확인
docker exec no-swap cat /sys/fs/cgroup/memory/memory.memsw.limit_in_bytes
# 1073741824 (Memory + Swap = 1GB, 즉 Swap 0)

# Swap 1GB 허용
docker run -d --name with-swap \
  --memory=1g \
  --memory-swap=2g \
  alpine sh -c 'tail -f /dev/null'

docker exec with-swap cat /sys/fs/cgroup/memory/memory.memsw.limit_in_bytes
# 2147483648 (2GB total)

# Swap 무제한 (비권장!)
docker run -d --name unlimited-swap \
  --memory=1g \
  --memory-swap=-1 \
  alpine sh -c 'tail -f /dev/null'

docker rm -f no-swap with-swap unlimited-swap
```

### Step 4: OOM Killer 동작 확인

```bash
# OOM 발생 시 종료 (기본)
docker run --name oom-kill \
  --memory=100m \
  alpine sh -c 'yes | tr \\n x | head -c 200m | grep n'

# 종료 코드 확인
docker inspect oom-kill --format '{{.State.ExitCode}}'
# 137 (128 + 9 = SIGKILL)

docker inspect oom-kill --format '{{.State.OOMKilled}}'
# true

# OOM 발생 시에도 종료 방지 (비권장)
docker run --name oom-no-kill \
  --memory=100m \
  --oom-kill-disable \
  alpine sh -c 'yes | tr \\n x | head -c 200m | grep n'

# 주의: 컨테이너는 살아있지만 프로세스는 멈춤 (Frozen)
docker stats --no-stream oom-no-kill
# MEM USAGE: 100MB (limit)

docker rm -f oom-kill oom-no-kill
```

---

## 🔧 실습 3: 복합 리소스 제한

### Step 1: Production 워크로드

```bash
# 웹 서버 (안정성 중시)
docker run -d --name web \
  --cpus=2 \
  --memory=2g \
  --memory-swap=2g \
  --memory-reservation=1g \
  --restart=always \
  --health-cmd='wget -q --spider http://localhost || exit 1' \
  --health-interval=30s \
  nginx:alpine

# 확인
docker stats --no-stream web

# CONTAINER  CPU %   MEM USAGE / LIMIT
# web        0.5%    10MB / 2GB
```

### Step 2: Batch 워크로드

```bash
# 데이터 처리 (처리량 중시)
docker run -d --name batch-job \
  --cpu-shares=2048 \
  --memory=8g \
  --memory-swap=16g \
  data-processor:latest

# CPU는 경쟁 시에만 제한 (유휴 시 전체 사용)
# Memory는 8GB로 제한
# Swap 8GB 추가 (대용량 데이터 처리)
```

### Step 3: 리소스 제한 템플릿

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 고우선순위 서비스
  api:
    image: api:latest
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 4G
        reservations:
          cpus: '2'
          memory: 2G
    restart: always

  # 저우선순위 서비스
  worker:
    image: worker:latest
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
    restart: on-failure

  # 데이터베이스 (격리)
  database:
    image: postgres:15
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G
        reservations:
          cpus: '2'
          memory: 4G
    cpuset: "0-3"  # CPU 0-3 전용
    mem_swappiness: 0  # Swap 최소화
```

---

## 🔧 실습 4: 모니터링 및 튜닝

### Step 1: 실시간 모니터링

```bash
# docker stats (실시간)
docker stats

# 출력:
# CONTAINER  CPU %  MEM USAGE / LIMIT   MEM %   NET I/O       BLOCK I/O
# web        2.5%   256MB / 2GB         12.8%   1.2MB / 890KB  5MB / 0B
# worker     45.2%  1.8GB / 2GB         90.0%   100KB / 50KB   100MB / 50MB
# db         8.1%   3.2GB / 8GB         40.0%   500KB / 300KB  200MB / 100MB

# 특정 컨테이너만
docker stats web worker --no-stream

# JSON 형식
docker stats --no-stream --format "{{json .}}" | jq .
```

### Step 2: cAdvisor 설치

```bash
# cAdvisor 실행
docker run -d \
  --name=cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  --detach=true \
  gcr.io/cadvisor/cadvisor:latest

# 웹 UI 접속
# http://localhost:8080

# Metrics 확인
curl http://localhost:8080/metrics | grep container_cpu
```

### Step 3: 리소스 사용 분석

```bash
# 컨테이너별 리소스 사용 로그
cat > monitor.sh <<'EOF'
#!/bin/bash

while true; do
  echo "=== $(date) ==="
  docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
  echo ""
  sleep 60
done > resource-usage.log &
EOF

chmod +x monitor.sh
./monitor.sh

# 1시간 후 분석
tail -100 resource-usage.log

# 평균 계산 (Python)
cat > analyze.py <<'EOF'
import re

cpu_sum = 0
mem_sum = 0
count = 0

with open('resource-usage.log') as f:
    for line in f:
        if 'web' in line:
            cpu = re.search(r'(\d+\.\d+)%', line)
            if cpu:
                cpu_sum += float(cpu.group(1))
                count += 1

print(f"Average CPU: {cpu_sum/count:.2f}%")
EOF

python3 analyze.py
```

---

## 💡 주요 명령어 정리

### CPU 제한

```bash
# Shares (상대적)
docker run --cpu-shares=1024 IMAGE

# Quota (절대적)
docker run --cpus=2.5 IMAGE
docker run --cpu-period=100000 --cpu-quota=250000 IMAGE

# Cpuset (코어 고정)
docker run --cpuset-cpus="0,1" IMAGE
docker run --cpuset-cpus="0-3" IMAGE
```

### Memory 제한

```bash
# Hard limit
docker run --memory=1g IMAGE
docker run -m 1g IMAGE

# Soft limit
docker run --memory-reservation=512m IMAGE

# Swap 제어
docker run --memory=1g --memory-swap=2g IMAGE  # Swap 1GB
docker run --memory=1g --memory-swap=1g IMAGE  # Swap 0GB
docker run --memory=1g --memory-swap=-1 IMAGE  # Swap 무제한

# OOM Killer
docker run --oom-kill-disable IMAGE  # 비권장
docker run --oom-score-adj=-500 IMAGE  # 우선순위 조정
```

### 모니터링

```bash
# 실시간 통계
docker stats
docker stats --no-stream
docker stats --format "{{.Name}}: {{.CPUPerc}}"

# 컨테이너 정보
docker inspect CONTAINER | jq '.[0].HostConfig.Memory'
docker inspect CONTAINER | jq '.[0].HostConfig.NanoCpus'
```

---

## 🎓 연습 문제

### 문제 1: CPU Shares vs Quota

다음 시나리오에서 각 컨테이너의 CPU 사용률을 예측하세요:

```bash
# 8 CPU 시스템
docker run -d --name A --cpu-shares=1024 IMAGE
docker run -d --name B --cpu-shares=512 IMAGE
docker run -d --name C --cpus=2 IMAGE

# 모든 컨테이너가 100% CPU를 요구할 때
```

<details>
<summary>정답 보기</summary>

```
Container C: 200% (2 cores 고정)
나머지 6 cores를 A,B가 shares 비율로 분배:

A: 1024 / (1024+512) = 66.67% of 6 cores = 400%
B: 512 / (1024+512) = 33.33% of 6 cores = 200%

정답:
A: 400%
B: 200%
C: 200%
```

</details>

### 문제 2: Memory Limit 설정

2GB RAM 컨테이너에 최적의 설정을 선택하세요:

```bash
# 옵션 A
--memory=2g --memory-swap=2g

# 옵션 B
--memory=2g --memory-swap=4g

# 옵션 C
--memory=2g --memory-reservation=1g --memory-swap=2g
```

<details>
<summary>정답 보기</summary>

**Production (안정성 중시):** 옵션 A
- Swap 비활성화로 예측 가능한 성능
- OOM 발생 시 명확한 실패

**Batch (처리량 중시):** 옵션 B
- Swap 2GB 추가로 대용량 처리
- 느려지지만 OOM 방지

**일반적 권장:** 옵션 C
- Soft limit으로 유연성
- Swap 비활성화로 안정성
- 메모리 부족 시 우선 회수 대상

</details>

### 문제 3: 리소스 튜닝

다음 상황에서 어떻게 튜닝할지 설명하세요:

```bash
docker stats
# CONTAINER  CPU %   MEM %
# web        5%      90%
# worker     150%    30%
```

<details>
<summary>정답 보기</summary>

**문제 분석:**
- web: CPU 여유, Memory 부족
- worker: CPU 부족, Memory 여유

**해결책:**
```bash
# web - Memory 증가
docker update --memory=4g web

# worker - CPU 증가
docker update --cpus=3 worker

# 또는 재생성
docker stop web worker
docker run -d --name web --cpus=1 --memory=4g nginx
docker run -d --name worker --cpus=3 --memory=2g worker-image
```

</details>

---

## 📌 핵심 요약

### 리소스 제한 전략

| 워크로드 | CPU 전략 | Memory 전략 | 목표 |
|---------|---------|------------|------|
| **Web** | Quota (고정) | Hard limit | 예측 가능 |
| **Batch** | Shares (상대) | Soft + Hard | 처리량 |
| **Database** | Cpuset (격리) | Hard + No swap | 안정성 |
| **Cache** | Shares (낮음) | Soft limit | 유연성 |

### CPU 제한 비교

```
Shares:
- 상대적 가중치
- 경쟁 시에만 적용
- 유휴 시 100% 사용 가능
- 유연함

Quota:
- 절대적 제한
- 항상 적용
- 유휴 시에도 제한
- 예측 가능

Cpuset:
- 코어 고정
- NUMA 최적화
- 격리 필요 시
- 성능 일관성
```

### Memory 제한 Best Practices

```yaml
Production:
  memory: 4g
  memory-swap: 4g      # Swap 비활성화
  memory-reservation: 2g

Development:
  memory: 2g
  memory-swap: 4g      # Swap 2GB
  memory-reservation: 1g

Database:
  memory: 8g
  memory-swap: 8g      # Swap 비활성화
  mem-swappiness: 0    # Kernel swap 최소화
```

### 모니터링 체크리스트

- [ ] CPU 사용률 90% 초과 시 확인
- [ ] Memory 사용률 80% 초과 시 확인
- [ ] OOM Kill 발생 시 즉시 대응
- [ ] Swap 사용 시 원인 분석
- [ ] CPU Throttling 빈도 확인
- [ ] 리소스 제한 주기적 검토

---

## 📚 참고 자료

- [Docker Resource Constraints](https://docs.docker.com/config/containers/resource_constraints/)
- [cgroup Documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/)
- [Understanding Docker CPU Limits](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)
- [Memory Management in Docker](https://docs.docker.com/config/containers/resource_constraints/#memory)

---

## 🤔 생각해볼 문제

1. CPU Shares를 2048로 설정했는데도 10%만 사용한다면 어떤 상황일까?
2. Memory Swap을 활성화하면 성능이 왜 저하될까?
3. Cpuset을 사용하면 어떤 장단점이 있을까?

> 💡 **답변**: 1) CPU Shares는 경쟁 상황에서만 작동, 다른 컨테이너가 CPU를 사용하지 않으면 shares 값과 무관하게 실제 필요한 만큼만 사용, 애플리케이션 자체가 10%만 필요하거나, I/O 대기 상태일 수 있음, `docker stats`로 확인 시 shares는 제한이 아닌 가중치임을 기억, 2) Swap은 디스크 기반 메모리, RAM 대비 100-1000배 느림, Swap 사용 시 Page Fault 발생 → 디스크 I/O 대기, 빈번한 swap in/out은 thrashing 유발, Database는 특히 치명적 (메모리 직접 접근 전제), Production에서는 swap 비활성화 권장 (--memory-swap=<memory>), 3) 장점: NUMA 환경에서 메모리 로컬리티 향상 (같은 NUMA 노드 사용), CPU 캐시 히트율 증가, 다른 워크로드와 완전 격리, 단점: 유연성 감소 (고정된 코어만 사용), 코어 수 변경 시 재시작 필요, 불균형 발생 가능 (일부 코어만 과부하), 활용 사례: 고성능 데이터베이스, 레이턴시 민감 애플리케이션, NUMA 아키텍처 서버

---

<div align="center">

**[다음: CPU Management ➡️](./02-CPU-Management.md)**

</div>
