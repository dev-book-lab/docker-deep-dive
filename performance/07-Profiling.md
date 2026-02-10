# 07. Profiling - 애플리케이션 프로파일링

## 🎯 이 챕터에서 배울 것

- **CPU Profiling** - 함수별 CPU 사용 분석
- **Memory Profiling** - 메모리 할당 패턴
- **I/O Profiling** - 파일/네트워크 I/O 분석
- **Flame Graphs** - 시각화 도구
- **Production Profiling** - 운영 환경 프로파일링

## 📌 왜 중요한가?

**"프로파일링은 성능 병목을 정확히 찾아 최적화하는 핵심 도구입니다."**

```
추측 기반 최적화 vs 데이터 기반 최적화:

추측 기반:
┌─────────────────────────────────────┐
│ 개발자: "DB 쿼리가 느린 것 같아"          │
│ → DB 인덱스 추가 (3일 작업)             │
│ → 성능 개선: 5%                       │
│                                     │
│ 실제 병목: JSON 파싱 (70% CPU)         │
│ → 발견 못 함                          │
└─────────────────────────────────────┘

데이터 기반:
┌─────────────────────────────────────┐
│ Profiler 실행 (5분)                   │
│ → CPU 분석:                          │
│   - JSON.parse(): 70%               │
│   - DB query: 10%                   │
│   - Others: 20%                     │
│                                     │
│ 최적화: JSON 파서 변경 (1일)            │
│ → 성능 개선: 60%                      │
└─────────────────────────────────────┘

차이:
❌ 추측: 3일 작업 → 5% 개선
✅ 프로파일링: 1일 작업 → 60% 개선

프로파일링 핵심 개념:

1. CPU Profiling:
   
   Flame Graph:
   ┌─────────────────────────────────┐
   │ main() ─────────────────────────│ 100%
   ├─────────────────────────────────┤
   │ processRequest() ───────────────│ 90%
   ├─────────────────────────────────┤
   │ parseJSON() ────│ queryDB() ────│ 70%│20%
   ├─────────────────┼───────────────┤
   │ parse()│validate│ execute()     │
   └─────────────────┴───────────────┘
   
   해석:
   - 넓을수록 시간 많이 소비
   - parseJSON()이 70% → 최우선 최적화
   - queryDB()는 20% → 차순위

2. Memory Profiling:
   
   할당 패턴:
   ┌──────────────────────────────┐
   │ Function A: 500MB (50%)      │
   │ - 1M allocations             │
   │ - 평균 500 bytes              │
   │ → 작은 객체 많음 (GC 부담)       │
   └──────────────────────────────┘
   
   ┌──────────────────────────────┐
   │ Function B: 300MB (30%)      │
   │ - 10 allocations             │
   │ - 평균 30MB                   │
   │ → 큰 객체 (메모리 누수?)         │
   └──────────────────────────────┘

3. I/O Profiling:
   
   File I/O:
   ┌──────────────────────────────┐
   │ read(): 1000 calls           │
   │ → 각 4KB 읽기                  │
   │ → 총 4MB                      │
   │ → 시간: 10초                   │
   │                              │
   │ 개선: BufferedReader          │
   │ → 64KB 버퍼                   │
   │ → 시간: 1초 (10배 빠름)         │
   └──────────────────────────────┘

4. Profiling 도구:
   
   언어별:
   - Python: cProfile, memory_profiler
   - Java: JProfiler, VisualVM
   - Node.js: 0x, clinic
   - Go: pprof
   - C/C++: perf, gprof
   
   범용:
   - perf (Linux)
   - Flame Graphs
   - async-profiler

실무 시나리오:

시나리오 1 - API 레이턴시:
┌─────────────────────────────────────┐
│ 문제: P95 레이턴시 500ms               │
│ 목표: 100ms 미만                      │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Profiling 결과:                      │
│                                     │
│ Flame Graph:                        │
│ ┌─────────────────────────────────┐ │
│ │ handleRequest() ───────────────││ │
│ ├────────────────────────────────┤│ │
│ │ serialize() ─────│ other ──────││ │
│ │     80%          │     20%     ││ │
│ └────────────────────────────────┘│ │
│                                     │
│ 병목: JSON 직렬화                      │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 최적화:                              │
│ JSON.stringify → MessagePack        │
│                                     │
│ 결과:                                │
│ P95: 500ms → 80ms (6배 개선)          │
└─────────────────────────────────────┘

시나리오 2 - 메모리 누수:
┌─────────────────────────────────────┐
│ 문제: 메모리 24시간마다 2GB 증가          │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Memory Profiler:                    │
│                                     │
│ Top Allocations:                    │
│ 1. EventListener: 1.5GB             │
│    - 100,000 listeners              │
│    - 미제거됨                         │
│                                     │
│ 2. Cache: 500MB                     │
│    - TTL 없음                        │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 수정:                                │
│ 1. removeEventListener() 추가        │
│ 2. Cache TTL 1시간 설정               │
│                                     │
│ 결과: 메모리 안정화 (1GB 유지)           │
└─────────────────────────────────────┘

시나리오 3 - CPU 100%:
┌─────────────────────────────────────┐
│ 문제: CPU 항상 100%                   │
│ 처리량: 100 req/s                     │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ CPU Profiler:                       │
│                                     │
│ Function Calls:                     │
│ - sortArray(): 90%                  │
│   - O(n²) 알고리즘                    │
│   - 매 요청마다 1000개 정렬             │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 최적화:                              │
│ - O(n log n) 알고리즘 사용             │
│ - 결과 캐싱                           │
│                                     │
│ 결과:                                │
│ CPU: 20%                            │
│ 처리량: 500 req/s (5배)               │
└─────────────────────────────────────┘
```

---

## 🔧 실습 1: Python Profiling

### Step 1: cProfile (CPU)

```bash
# 프로파일링할 Python 코드
cat > app.py <<'EOF'
import time

def slow_function():
    """느린 함수"""
    total = 0
    for i in range(1000000):
        total += i
    return total

def fast_function():
    """빠른 함수"""
    return sum(range(1000000))

def main():
    for _ in range(10):
        slow_function()
    for _ in range(10):
        fast_function()

if __name__ == '__main__':
    main()
EOF

# Dockerfile
cat > Dockerfile <<'EOF'
FROM python:3.11-slim
COPY app.py /app/
WORKDIR /app
CMD ["python", "-m", "cProfile", "-s", "cumulative", "app.py"]
EOF

docker build -t py-profile .
docker run --rm py-profile

# 출력:
#          20 function calls in 1.234 seconds
#
#    Ordered by: cumulative time
#
#    ncalls  tottime  percall  cumtime  percall filename:lineno(function)
#        1    0.001    0.001    1.234    1.234 app.py:1(<module>)
#        1    0.002    0.002    1.233    1.233 app.py:15(main)
#       10    1.100    0.110    1.100    0.110 app.py:3(slow_function)
#       10    0.020    0.002    0.020    0.002 app.py:10(fast_function)
#                                              ↑ slow_function이 병목
```

### Step 2: memory_profiler (메모리)

```bash
# 메모리 프로파일링 코드
cat > mem_app.py <<'EOF'
from memory_profiler import profile

@profile
def memory_hog():
    """메모리 많이 사용"""
    big_list = [i for i in range(10**6)]
    return big_list

@profile
def memory_efficient():
    """메모리 적게 사용"""
    return sum(i for i in range(10**6))

if __name__ == '__main__':
    memory_hog()
    memory_efficient()
EOF

# Dockerfile
cat > Dockerfile.mem <<'EOF'
FROM python:3.11-slim
RUN pip install memory_profiler
COPY mem_app.py /app/
WORKDIR /app
CMD ["python", "mem_app.py"]
EOF

docker build -t py-mem-profile -f Dockerfile.mem .
docker run --rm py-mem-profile

# 출력:
# Line #    Mem usage    Increment  Occurrences   Line Contents
# ============================================================
#      3   10.5 MiB     10.5 MiB           1   @profile
#      4                                       def memory_hog():
#      5   48.2 MiB     37.7 MiB           1       big_list = [i for i in range(10**6)]
#                                                            ↑ 37.7MB 할당
#      6   48.2 MiB      0.0 MiB           1       return big_list
#
#      8   48.2 MiB      0.0 MiB           1   @profile
#      9                                       def memory_efficient():
#     10   48.2 MiB      0.0 MiB           1       return sum(i for i in range(10**6))
#                                                            ↑ 거의 메모리 안 씀
```

### Step 3: py-spy (샘플링)

```bash
# 실행 중인 Python 프로세스 프로파일링
cat > long_app.py <<'EOF'
import time

def process_data():
    while True:
        # 무거운 작업
        sum(i**2 for i in range(10**6))
        time.sleep(0.1)

if __name__ == '__main__':
    process_data()
EOF

# 백그라운드 실행
docker run -d --name py-app \
  -v $(pwd)/long_app.py:/app.py \
  python:3.11-slim \
  python /app.py

# py-spy로 프로파일링
docker run --rm --cap-add SYS_PTRACE \
  --pid container:py-app \
  -v $(pwd):/output \
  python:3.11-slim sh -c '
    pip install py-spy &&
    py-spy record -o /output/profile.svg --pid 1 --duration 10
  '

# profile.svg 확인 (Flame Graph)
# 브라우저로 열기

docker rm -f py-app
```

---

## 🔧 실습 2: Java Profiling

### Step 1: JProfiler / VisualVM

```bash
# Java 애플리케이션
cat > App.java <<'EOF'
public class App {
    public static void slowMethod() {
        int sum = 0;
        for (int i = 0; i < 10000000; i++) {
            sum += i;
        }
    }
    
    public static void fastMethod() {
        // 빠른 로직
    }
    
    public static void main(String[] args) throws Exception {
        while (true) {
            slowMethod();
            fastMethod();
            Thread.sleep(100);
        }
    }
}
EOF

# Dockerfile
cat > Dockerfile.java <<'EOF'
FROM openjdk:11-slim
COPY App.java /app/
WORKDIR /app
RUN javac App.java
CMD ["java", "-agentlib:hprof=cpu=samples", "App"]
EOF

docker build -t java-profile -f Dockerfile.java .
docker run --rm java-profile

# java.hprof.txt 생성됨
```

### Step 2: async-profiler

```bash
# async-profiler (프로덕션 친화적)
docker run -d --name java-app \
  -v $(pwd):/app \
  openjdk:11-slim \
  java App

# async-profiler 실행
docker exec java-app sh -c '
  wget https://github.com/jvm-profiling-tools/async-profiler/releases/download/v2.9/async-profiler-2.9-linux-x64.tar.gz
  tar xzf async-profiler-2.9-linux-x64.tar.gz
  ./async-profiler-2.9-linux-x64/profiler.sh -d 30 -f /app/flamegraph.html 1
'

# flamegraph.html 확인

docker rm -f java-app
```

---

## 🔧 실습 3: Node.js Profiling

### Step 1: 0x (Flame Graph)

```bash
# Node.js 애플리케이션
cat > app.js <<'EOF'
function slowFunction() {
  let sum = 0;
  for (let i = 0; i < 10000000; i++) {
    sum += i;
  }
  return sum;
}

function fastFunction() {
  return 42;
}

setInterval(() => {
  slowFunction();
  fastFunction();
}, 100);
EOF

# Dockerfile
cat > Dockerfile.node <<'EOF'
FROM node:18-slim
RUN npm install -g 0x
COPY app.js /app/
WORKDIR /app
CMD ["0x", "app.js"]
EOF

docker build -t node-profile -f Dockerfile.node .
docker run --rm -v $(pwd):/output node-profile

# Flame Graph HTML 생성됨
```

### Step 2: clinic.js

```bash
# clinic doctor (종합 진단)
cat > Dockerfile.clinic <<'EOF'
FROM node:18-slim
RUN npm install -g clinic
COPY app.js /app/
WORKDIR /app
CMD ["clinic", "doctor", "--", "node", "app.js"]
EOF

docker build -t node-clinic -f Dockerfile.clinic .
docker run --rm -v $(pwd):/output node-clinic

# 리포트 생성:
# - CPU
# - Memory
# - Event Loop
# - I/O
```

---

## 🔧 실습 4: Go Profiling

### Step 1: pprof

```go
// main.go
package main

import (
    "net/http"
    _ "net/http/pprof"
    "time"
)

func slowFunction() {
    sum := 0
    for i := 0; i < 10000000; i++ {
        sum += i
    }
}

func main() {
    // pprof HTTP endpoint
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    
    for {
        slowFunction()
        time.Sleep(100 * time.Millisecond)
    }
}
```

```bash
# Dockerfile
cat > Dockerfile.go <<'EOF'
FROM golang:1.21-alpine
COPY main.go /app/
WORKDIR /app
RUN go build -o app main.go
EXPOSE 6060
CMD ["./app"]
EOF

docker build -t go-profile -f Dockerfile.go .
docker run -d --name go-app -p 6060:6060 go-profile

# CPU 프로파일 수집
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 대화형 분석
# (pprof) top
# (pprof) list slowFunction
# (pprof) web  # Flame Graph (graphviz 필요)

# 메모리 프로파일
go tool pprof http://localhost:6060/debug/pprof/heap

docker rm -f go-app
```

---

## 🔧 실습 5: Linux perf (범용)

### Step 1: perf 설치 및 사용

```bash
# perf 설치 (호스트)
sudo apt-get install -y linux-tools-common linux-tools-generic

# 프로파일링할 앱
docker run -d --name perf-test \
  --cap-add SYS_ADMIN \
  alpine sh -c 'while true; do :; done'

# PID 확인
PID=$(docker inspect perf-test --format '{{.State.Pid}}')

# perf 수집 (30초)
sudo perf record -F 99 -p $PID -g -- sleep 30

# 리포트
sudo perf report

# 출력:
# Samples: 2K of event 'cycles:ppp'
# Overhead  Command  Shared Object      Symbol
#   99.50%  sh       [kernel.kallsyms]  [k] native_safe_halt
#    0.30%  sh       libc.so.6          [.] __GI___libc_write

docker rm -f perf-test
```

### Step 2: Flame Graph 생성

```bash
# FlameGraph 도구 설치
git clone https://github.com/brendangregg/FlameGraph
cd FlameGraph

# perf 데이터 수집
sudo perf record -F 99 -a -g -- sleep 30

# Flame Graph 생성
sudo perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > flamegraph.svg

# 브라우저로 flamegraph.svg 열기
```

---

## 🔧 실습 6: Production Profiling

### Step 1: 안전한 프로파일링

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    image: myapp:latest
    environment:
      - ENABLE_PROFILING=true
      - PROFILING_SAMPLE_RATE=0.01  # 1%만 샘플링
    ports:
      - "8080:8080"
      - "6060:6060"  # pprof endpoint
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G

  profiler:
    image: google/cloud-profiler
    environment:
      - SERVICE_NAME=myapp
      - SERVICE_VERSION=1.0
      - ENABLE_HEAP_PROFILING=true
    depends_on:
      - app
```

### Step 2: Continuous Profiling

```bash
# Pyroscope (Continuous Profiling)
cat > docker-compose.profiler.yml <<'EOF'
version: '3.8'

services:
  pyroscope:
    image: pyroscope/pyroscope:latest
    ports:
      - "4040:4040"
    command:
      - server

  app:
    image: myapp:latest
    environment:
      - PYROSCOPE_SERVER_ADDRESS=http://pyroscope:4040
      - PYROSCOPE_APPLICATION_NAME=myapp
    depends_on:
      - pyroscope
EOF

docker-compose -f docker-compose.profiler.yml up -d

# Pyroscope UI
# http://localhost:4040
# - Flame Graph (실시간)
# - 시간별 비교
# - Diff 기능
```

---

## 💡 주요 명령어 정리

### Python

```bash
# cProfile
python -m cProfile -s cumulative app.py

# memory_profiler
python -m memory_profiler app.py

# py-spy (실행 중 프로세스)
py-spy record -o profile.svg --pid PID
```

### Java

```bash
# jmap (힙 덤프)
jmap -dump:format=b,file=heap.bin PID

# jstack (스레드 덤프)
jstack PID

# async-profiler
./profiler.sh -d 30 -f flamegraph.html PID
```

### Node.js

```bash
# 0x
0x app.js

# clinic
clinic doctor -- node app.js
clinic bubbleprof -- node app.js
```

### Go

```bash
# pprof
go tool pprof http://localhost:6060/debug/pprof/profile
go tool pprof http://localhost:6060/debug/pprof/heap
```

### Linux

```bash
# perf
perf record -F 99 -p PID -g -- sleep 30
perf report

# Flame Graph
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

---

## 🎓 연습 문제

### 문제 1: CPU 병목 찾기

다음 Flame Graph에서 최우선 최적화 대상을 찾으세요:

```
main() ──────────────────────── 100%
  ├─ processRequest() ─────────  95%
  │   ├─ parseJSON() ──────────  60%
  │   └─ queryDB() ────────────  35%
  └─ other ───────────────────    5%
```

<details>
<summary>정답 보기</summary>

**최우선: parseJSON() (60%)**

이유:
- 전체 시간의 60% 소비
- queryDB() (35%)보다 큼
- 최대 효과 가능

최적화 방법:
1. 더 빠른 JSON 파서 (simdjson)
2. JSON 스키마 단순화
3. MessagePack/Protocol Buffers 사용
4. 결과 캐싱

</details>

### 문제 2: 메모리 누수 탐지

메모리 프로파일러 결과:

```
Function A: 100MB (정상)
Function B: 2GB (24시간 전: 500MB)
Function C: 50MB (정상)
```

<details>
<summary>정답 보기</summary>

**메모리 누수: Function B**

증거:
- 24시간 동안 4배 증가 (500MB → 2GB)
- 지속적 증가 패턴

조사 방법:
```python
from memory_profiler import profile

@profile
def function_b():
    # 라인별 메모리 사용 확인
    pass

# 또는 힙 덤프 비교
# 시작 시 덤프 vs 24시간 후 덤프
```

일반적 원인:
- Event listener 미제거
- 전역 캐시 무제한 증가
- 순환 참조
- 파일 핸들 미해제

</details>

### 문제 3: Profiling Overhead

Production 환경에서 안전하게 프로파일링하는 방법은?

<details>
<summary>정답 보기</summary>

**안전한 프로파일링 전략:**

1. **샘플링 프로파일러 사용**
```bash
# 1% 트래픽만 프로파일링
PROFILING_SAMPLE_RATE=0.01

# 또는 특정 request에만
if random.random() < 0.01:
    profiler.start()
```

2. **낮은 샘플링 주파수**
```bash
# 99Hz (기본) 대신 낮춤
perf record -F 49 -p PID

# CPU overhead: 1-2%
```

3. **짧은 기간**
```bash
# 30초만
perf record -F 99 -p PID -g -- sleep 30
```

4. **Continuous Profiling**
```
Pyroscope, Datadog Profiler
- Always-on
- <1% overhead
- 자동 집계
```

</details>

---

## 📌 핵심 요약

### 프로파일링 도구 선택

| 언어 | CPU | Memory | Production |
|-----|-----|--------|------------|
| **Python** | cProfile | memory_profiler | py-spy |
| **Java** | async-profiler | JProfiler | Flight Recorder |
| **Node.js** | 0x | clinic | N|Solid |
| **Go** | pprof | pprof | pprof |
| **C/C++** | perf | Valgrind | perf |

### Flame Graph 해석

```
넓이 = CPU 시간 비율
높이 = Call Stack 깊이

넓은 부분 = 최적화 대상
평평한 부분 = 효율적
뾰족한 부분 = 깊은 재귀
```

### 프로파일링 워크플로우

```
1. 측정
   ↓
2. 병목 식별 (Flame Graph)
   ↓
3. 가설 수립
   ↓
4. 최적화
   ↓
5. 재측정
   ↓
6. 반복
```

### Production 프로파일링 Best Practices

- [ ] 샘플링 프로파일러 사용
- [ ] 낮은 샘플링 레이트 (1-5%)
- [ ] 짧은 수집 기간 (30-60초)
- [ ] 비피크 시간대 수행
- [ ] Canary 배포에서 먼저 테스트
- [ ] Overhead 모니터링 (<2%)
- [ ] 자동화 (Continuous Profiling)

---

## 📚 참고 자료

- [Brendan Gregg's Blog](https://www.brendangregg.com/flamegraphs.html)
- [Python Profilers](https://docs.python.org/3/library/profile.html)
- [Go pprof](https://go.dev/blog/pprof)
- [Java Flight Recorder](https://docs.oracle.com/javacomponents/jmc-5-5/jfr-runtime-guide/)

---

## 🤔 생각해볼 문제

1. Flame Graph에서 넓은 부분이 왜 최적화 대상일까?
2. 메모리 프로파일링은 언제 해야 할까?
3. Production에서 프로파일링이 왜 조심스러울까?

> 💡 **답변**: 1) Flame Graph 해석, X축(너비) = CPU 시간 비율, 넓을수록 = 더 많은 시간 소비, 예: 60% 너비 → 전체 시간의 60%, Y축(높이) = Call stack 깊이, 최적화 효과, 넓은 부분 개선 = 큰 효과, 예: 60% → 30%로 줄이면 전체 50% 빠름, 좁은 부분 개선 = 작은 효과, 예: 5% → 0%로 줄여도 5%만 빠름, 우선순위, 1순위: 가장 넓은 부분, 2순위: 최적화 용이성 고려, 2) 메모리 프로파일링 시점, 증상 발생 시, 메모리 사용량 지속 증가, OOM Killer 빈번, Swap 사용 증가, 정기적 점검, 주간/월간 프로파일링, 메모리 트렌드 모니터링, 개발 단계, 새 기능 추가 후, 알고리즘 변경 후, 대용량 데이터 처리 전, 도구, Python: memory_profiler, Java: jmap, heapdump, Go: pprof heap, Node.js: clinic, 3) Production 프로파일링 위험성, Performance Overhead, CPU: 추가 1-5% 사용, 응답 시간 증가 가능, Sampling profiler 권장, Memory Overhead, 프로파일 데이터 저장, Heap dump = 전체 메모리 복사, Disk I/O, 프로파일 데이터 쓰기, 로그 증가, 시스템 영향, 안전한 방법, 샘플링 레이트 낮춤 (1%), 짧은 기간 (30초), 비피크 시간, Canary/Staging 먼저, Continuous Profiler 사용 (<1% overhead)

---

<div align="center">

**[⬅️ 이전: Logging](./06-Logging.md)** | **[다음: Benchmarking ➡️](./08-Benchmarking.md)**

</div>
