# 04. Performance Issues - 성능 병목 찾기

## 🎯 이 챕터에서 배울 것

- **리소스 모니터링**: CPU, Memory, Disk, Network
- **프로파일링**: perf, flamegraph, pprof
- **병목 지점 찾기**: 느린 쿼리, 메모리 누수
- **컨테이너 제약**: cgroup limits
- **최적화 기법**: 이미지 크기, 레이어 캐싱
- **실전 기법**: 프로덕션 성능 분석

## 📌 왜 중요한가?

**"성능 문제는 사용자 경험과 비용에 직접적인 영향을 미칩니다."**

```
Performance Issues의 핵심:

Common Performance Problems:
┌─────────────────────────────────────────────────┐
│ 1. High CPU Usage (높은 CPU)                     │
│    - 무한 루프                                    │
│    - 비효율적 알고리즘                              │
│    - 과도한 계산                                  │
│                                                 │
│ 2. Memory Leak (메모리 누수)                      │
│    - 객체 미해제                                  │
│    - 캐시 무한 증가                               │
│    - 연결 미종료                                  │
│                                                │
│ 3. Disk I/O Bottleneck (디스크 병목)              │
│    - 느린 스토리지                                 │
│    - 과도한 로깅                                   │
│    - 대용량 파일                                   │
│                                                 │
│ 4. Network Latency (네트워크 지연)                 │
│    - 먼 거리 통신                                 │
│    - 직렬 요청                                    │
│    - 큰 페이로드                                   │
└─────────────────────────────────────────────────┘

Performance Monitoring:
┌─────────────────────────────────────────────────┐
│ System Level:                                   │
│  ┌────────────────────────────────────────┐     │
│  │ CPU: 80% (top, htop)                   │     │
│  │ Memory: 14GB/16GB (free, vmstat)       │     │
│  │ Disk I/O: 100MB/s (iostat)             │     │
│  │ Network: 50Mbps (iftop)                │     │
│  └────────────────────────────────────────┘     │
│                                                 │
│ Container Level:                                │
│  ┌────────────────────────────────────────┐     │
│  │ docker stats                           │     │
│  │ Container    CPU%   MEM                │     │
│  │ myapp        85%    1.2GB/2GB          │     │
│  └────────────────────────────────────────┘     │
│                                                 │
│ Application Level:                              │
│  ┌────────────────────────────────────────┐     │
│  │ Response Time: 2.5s (target: < 1s)     │     │
│  │ Throughput: 100 req/s                  │     │
│  │ Error Rate: 0.5%                       │     │
│  └────────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘

Performance Analysis Flow:
┌─────────────────────────────────────────────────┐
│ 1. 증상 확인                                      │
│    "API가 느려요" → 얼마나? 언제부터?                 │
│                                                 │
│ 2. 메트릭 수집                                     │
│    CPU, Memory, I/O, Network                    │
│                                                 │
│ 3. 프로파일링                                      │
│    어느 함수가 느린가?                               │
│                                                 │
│ 4. 병목 지점 찾기                                   │
│    데이터베이스? 네트워크? CPU?                       │
│                                                 │
│ 5. 최적화                                         │
│    캐싱, 인덱스, 병렬 처리                           │
│                                                 │
│ 6. 검증                                          │
│    개선됐는가? 측정!                                │
└─────────────────────────────────────────────────┘

Resource Limits:
┌─────────────────────────────────────────────────┐
│ Docker Resource Constraints                     │
│                                                 │
│ docker run -d \                                 │
│   --cpus=2.0 \          # CPU 제한               │
│   --memory=2g \         # 메모리 제한              │
│   --memory-swap=4g \    # Swap 포함              │
│   --pids-limit=100 \    # 프로세스 수 제한          │
│   myapp                                         │
│                                                 │
│ 제한 초과 시:                                      │
│ - CPU: Throttling (느려짐)                        │
│ - Memory: OOM Killer (종료)                      │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **사용자 만족도**: 빠른 응답 = 좋은 경험
- **비용**: CPU/메모리 효율 = 비용 절감
- **확장성**: 병목 제거 = 더 많은 사용자
- **안정성**: 리소스 누수 방지 = 장애 예방

---

## 🔬 Deep Dive

### 1. CPU 분석

#### top/htop (실시간 모니터링)

```bash
# 컨테이너 내부
docker exec myapp top

# 컨테이너 외부 (호스트에서)
docker stats myapp

# 상세 정보
docker exec myapp htop

# CPU 사용률 높은 프로세스
docker exec myapp ps aux --sort=-%cpu | head -10
```

#### perf (성능 프로파일링)

```bash
# perf 설치
docker exec myapp apt-get install linux-perf

# CPU 프로파일링 (10초)
docker exec myapp perf record -F 99 -p 1 -g -- sleep 10

# 리포트 생성
docker exec myapp perf report

# FlameGraph
docker exec myapp perf script | flamegraph.pl > flamegraph.svg
```

---

### 2. 메모리 분석

#### free/vmstat

```bash
# 메모리 사용량
docker exec myapp free -h

# 실시간 메모리 통계
docker exec myapp vmstat 1 10

# 프로세스별 메모리
docker exec myapp ps aux --sort=-%mem | head -10
```

#### /proc/meminfo

```bash
# 상세 메모리 정보
docker exec myapp cat /proc/meminfo

# 특정 프로세스 메모리 맵
docker exec myapp cat /proc/1/smaps
docker exec myapp pmap 1
```

---

## 🔧 실습 1: CPU 병목 찾기

### Step 1: CPU 사용률 확인

```bash
# 1. 실시간 모니터링
docker stats myapp --no-stream

# 2. 컨테이너 내부
docker exec myapp top -bn1

# 3. CPU 사용률 높은 프로세스
docker exec myapp ps aux --sort=-%cpu | head -5

# 4. 스레드별 CPU
docker exec myapp top -H -p 1
```

### Step 2: Python 프로파일링

```python
# app.py
import cProfile
import pstats
import io

def slow_function():
    """느린 함수"""
    total = 0
    for i in range(1000000):
        total += i
    return total

# 프로파일링
pr = cProfile.Profile()
pr.enable()

slow_function()

pr.disable()
s = io.StringIO()
ps = pstats.Stats(pr, stream=s).sort_stats('cumulative')
ps.print_stats()
print(s.getvalue())
```

```bash
# 실행
docker exec myapp python -m cProfile -o profile.stats app.py

# 분석
docker exec myapp python -m pstats profile.stats
>>> sort cumulative
>>> stats 10
```

### Step 3: FlameGraph

```bash
# perf 데이터 수집
docker exec myapp perf record -F 99 -p 1 -g -- sleep 30

# 파일 복사
docker cp myapp:/perf.data ./

# FlameGraph 생성
perf script -i perf.data | stackcollapse-perf.pl | flamegraph.pl > cpu-flamegraph.svg

# 브라우저로 확인
open cpu-flamegraph.svg
```

---

## 🔧 실습 2: 메모리 누수 찾기

### Step 1: 메모리 사용 추적

```bash
# 실시간 모니터링
watch -n 1 'docker stats myapp --no-stream'

# 메모리 증가 추세
for i in {1..60}; do
  docker stats myapp --no-stream | awk '{print $4}' >> mem.log
  sleep 1
done

# 그래프 (gnuplot)
gnuplot << EOF
set terminal png
set output 'memory.png'
plot 'mem.log' with lines
EOF
```

### Step 2: Python 메모리 프로파일링

```python
# memory_profiler 사용
from memory_profiler import profile

@profile
def memory_leak():
    """메모리 누수 예제"""
    data = []
    for i in range(1000000):
        data.append(str(i))  # 계속 증가
    return data

memory_leak()
```

```bash
# 실행
docker exec myapp python -m memory_profiler app.py

# 출력:
# Line #  Mem usage  Increment   Line Contents
# ================================================
# 3       38.5 MiB   38.5 MiB   @profile
# 4                             def memory_leak():
# 5       38.5 MiB    0.0 MiB       data = []
# 6      115.8 MiB   77.3 MiB       for i in range(1000000):
# 7      115.8 MiB    0.0 MiB           data.append(str(i))
```

### Step 3: Java Heap Dump

```bash
# Heap dump 생성
docker exec myapp jmap -dump:live,format=b,file=/tmp/heap.bin 1

# 복사
docker cp myapp:/tmp/heap.bin ./

# VisualVM 또는 Eclipse MAT로 분석
# - 큰 객체 찾기
# - Leak suspects
# - Dominator tree
```

---

## 🔧 실습 3: Disk I/O 분석

### Step 1: Disk 사용률

```bash
# 디스크 사용량
docker exec myapp df -h

# I/O 통계
docker exec myapp iostat -x 1 10

# 프로세스별 I/O
docker exec myapp iotop

# 파일 디스크립터
docker exec myapp lsof -p 1 | wc -l
```

### Step 2: 로그 병목

```bash
# 로그 파일 크기
docker exec myapp du -sh /var/log/*

# 실시간 로그 쓰기
docker exec myapp tail -f /var/log/app.log &
docker stats myapp  # BLOCK I/O 확인

# 로그 로테이션 확인
docker exec myapp ls -lh /var/log/app.log*
```

---

## 🔧 실습 4: 네트워크 성능

### Step 1: 대역폭 측정

```bash
# iperf3 서버
docker run -d --name iperf-server networkstatic/iperf3 -s

# iperf3 클라이언트
docker run --rm networkstatic/iperf3 -c iperf-server

# 결과:
# Bandwidth: 942 Mbits/sec
```

### Step 2: 레이턴시 측정

```bash
# ping 통계
docker exec myapp ping -c 100 api | tail -2

# 출력:
# rtt min/avg/max/mdev = 0.123/0.456/1.234/0.123 ms

# HTTP 응답 시간
docker exec myapp curl -w "\nTotal: %{time_total}s\n" -o /dev/null -s http://api/endpoint
```

---

## 🔧 실습 5: 컨테이너 리소스 제약 분석

### Step 1: CPU Throttling

```bash
# CPU 제한된 컨테이너 실행
docker run -d --name app --cpus=1.0 stress-ng --cpu 4 --timeout 60s

# Throttling 확인
docker stats app

# cgroup 확인
docker exec app cat /sys/fs/cgroup/cpu/cpu.cfs_throttled_usec
```

### Step 2: OOM (Out of Memory)

```python
# oom.py
data = []
while True:
    data.append(' ' * 10**6)  # 1MB씩 증가
    print(f"Memory: {len(data)} MB")
```

```bash
# 메모리 제한
docker run -d --name oom-test --memory=100m python:3.9 python oom.py

# 로그 확인
docker logs oom-test

# OOM Kill 확인
docker inspect oom-test | jq '.[0].State'
# "OOMKilled": true
```

---

## 🔧 실습 6: 이미지 최적화

### Step 1: 이미지 크기 분석

```bash
# 이미지 크기
docker images myapp

# 레이어별 크기
docker history myapp

# dive로 상세 분석
dive myapp
```

### Step 2: Multi-stage Build

```dockerfile
# Before (1.2GB)
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]

# After (150MB)
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/server.js ./
CMD ["node", "server.js"]
```

### Step 3: .dockerignore

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
test/
*.test.js
```

---

## 🔧 실습 7: 데이터베이스 쿼리 최적화

### Step 1: 느린 쿼리 찾기

```sql
-- PostgreSQL
-- Slow query log 활성화
ALTER SYSTEM SET log_min_duration_statement = 1000; -- 1초 이상
SELECT pg_reload_conf();

-- 로그 확인
docker exec postgres tail -f /var/log/postgresql/postgresql.log
```

```python
# Python with logging
import logging
import time

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def execute_query(query):
    start = time.time()
    result = db.execute(query)
    duration = time.time() - start
    
    if duration > 1.0:
        logger.warning(f"Slow query ({duration:.2f}s): {query}")
    
    return result
```

### Step 2: 쿼리 분석

```sql
-- EXPLAIN ANALYZE
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';

-- 결과:
-- Seq Scan on users (cost=0.00..1500.00 rows=1)
-- Planning Time: 0.5ms
-- Execution Time: 234.5ms

-- 인덱스 추가
CREATE INDEX idx_users_email ON users(email);

-- 다시 실행
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';

-- Index Scan using idx_users_email (cost=0.43..8.45 rows=1)
-- Execution Time: 0.8ms
```

---

## 💡 성능 최적화 체크리스트

```
┌──────────────────────┬────────────────────────────┐
│ 영역                  │ 최적화 기법                   │
├──────────────────────┼────────────────────────────┤
│ CPU                  │ 알고리즘 개선, 캐싱            │
├──────────────────────┼────────────────────────────┤
│ Memory               │ 객체 풀, 메모리 프로파일링       │
├──────────────────────┼────────────────────────────┤
│ Disk I/O             │ 버퍼링, 비동기 I/O            │
├──────────────────────┼────────────────────────────┤
│ Network              │ 연결 풀, 압축, CDN            │
├──────────────────────┼────────────────────────────┤
│ Database             │ 인덱스, 쿼리 최적화            │
├──────────────────────┼────────────────────────────┤
│ Container            │ Multi-stage, alpine 이미지  │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. 측정 → 분석 → 최적화 → 검증
2. 병목 지점 하나씩 해결
3. 캐싱 적극 활용
4. 비동기 처리
5. 리소스 제한 설정
```

---

## 🎓 연습 문제

### 문제 1: CPU 100%인데 애플리케이션은 느리다면?

<details>
<summary>정답 보기</summary>

**가능한 원인:**

**1. CPU Throttling**
```bash
# cgroup 제한 확인
docker exec myapp cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us
docker exec myapp cat /sys/fs/cgroup/cpu/cpu.cfs_period_us

# Throttling 횟수
docker exec myapp cat /sys/fs/cgroup/cpu/cpu.stat
# nr_throttled: 1234
# throttled_time: 567890000
```

**해결: CPU 제한 늘리기**
```bash
docker update --cpus=2.0 myapp
```

**2. 단일 스레드 병목**
```bash
# 스레드별 CPU
docker exec myapp top -H -p 1

# 한 스레드만 100%
# → 병렬 처리 필요
```

**3. 컨텍스트 스위칭**
```bash
docker exec myapp vmstat 1 10
# cs (context switch) 매우 높으면
# → 스레드 수 줄이기
```

</details>

### 문제 2: 메모리가 계속 증가한다면?

<details>
<summary>정답 보기</summary>

**진단:**

**1. 메모리 누수 vs 캐시**
```python
# 누수
data = []
def leak():
    data.append(something)  # 계속 증가

# 캐시 (정상)
@lru_cache(maxsize=1000)  # 제한됨
def cached_func():
    ...
```

**2. 프로파일링**
```bash
# Python
docker exec myapp python -m memory_profiler app.py

# Node.js
docker exec myapp node --inspect app.js
# Chrome DevTools로 Heap Snapshot
```

**3. GC 확인**
```bash
# Java
docker exec myapp jstat -gcutil 1 1000

# 메모리 회수 안 되면 → 누수
```

**해결:**
```python
# 약한 참조 사용
import weakref
cache = weakref.WeakValueDictionary()

# 명시적 해제
data.clear()

# 객체 크기 제한
from cachetools import LRUCache
cache = LRUCache(maxsize=1000)
```

</details>

### 문제 3: 데이터베이스 쿼리가 느리다면?

<details>
<summary>정답 보기</summary>

**진단:**

**1. EXPLAIN 분석**
```sql
EXPLAIN ANALYZE SELECT ...;

-- Seq Scan → 인덱스 없음
-- Index Scan → 인덱스 사용
```

**2. Missing Index**
```sql
-- 인덱스 추가
CREATE INDEX idx_column ON table(column);

-- 복합 인덱스
CREATE INDEX idx_multi ON table(col1, col2);
```

**3. N+1 Query**
```python
# ❌ N+1 문제
users = User.query.all()
for user in users:
    posts = user.posts  # 각 user마다 쿼리

# ✅ 해결: JOIN
users = User.query.options(
    joinedload(User.posts)
).all()
```

**4. 과도한 데이터**
```sql
-- ❌ 전체 조회
SELECT * FROM logs;  -- 1000만 행

-- ✅ 페이징
SELECT * FROM logs LIMIT 100 OFFSET 0;

-- ✅ 필요한 컬럼만
SELECT id, message FROM logs;
```

</details>

---

## 📌 핵심 요약

```
Performance Issues 핵심:
1. 측정 (Measure)
2. 분석 (Analyze)
3. 최적화 (Optimize)
4. 검증 (Verify)
5. 반복

Best Practices:
✅ 프로파일링 도구 사용
✅ 병목 지점 하나씩
✅ 캐싱 적극 활용
✅ 리소스 제한 설정
✅ 이미지 최적화
```

---

## 📚 참고 자료

- [Linux perf Tutorial](https://perf.wiki.kernel.org/index.php/Tutorial)
- [FlameGraph](https://www.brendangregg.com/flamegraphs.html)
- [Docker Performance](https://docs.docker.com/config/containers/resource_constraints/)

---

## 🤔 생각해볼 문제

1. 프로덕션에서 프로파일링을 해도 되는가?
2. 모든 성능 문제를 해결해야 하는가?
3. 컨테이너 vs VM 성능 차이는?

> 💡 **답변**:
> 
> **1) 프로덕션 프로파일링:**
> 
> ```
> ⚠️ 주의:
> - 오버헤드 (5-30%)
> - 복제본 중 일부만
> - 짧은 시간 (5-10분)
> 
> 안전한 방법:
> - Sampling (1%)
> - 특정 엔드포인트만
> - Off-peak 시간
> - 카나리 배포
> ```
> 
> **2) 80/20 Rule:**
> 
> ```
> - 20% 노력 → 80% 개선
> - 중요한 병목만 해결
> - 비용 대비 효과
> 
> 우선순위:
> 1. 사용자 영향 큰 것
> 2. 빈도 높은 것
> 3. 쉬운 것
> ```
> 
> **3) Container vs VM:**
> 
> ```
> Container: 거의 Native 성능
> - CPU: 99%
> - Memory: 99%
> - Disk: 90-95%
> - Network: 95-98%
> 
> VM: Overhead 있음
> - CPU: 95%
> - Memory: 90%
> - Hypervisor 오버헤드
> ```

---

<div align="center">

**[⬅️ 이전: Network Debugging](./03-Network-Debugging.md)** | **[다음: Common Problems ➡️](./05-Common-Problems.md)**

</div>
