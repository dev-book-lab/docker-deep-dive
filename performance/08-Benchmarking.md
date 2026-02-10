# 08. Benchmarking - 성능 테스트 방법론

## 🎯 이 챕터에서 배울 것

- **부하 테스트** - Apache Bench, wrk, Gatling
- **스트레스 테스트** - 한계점 찾기
- **내구성 테스트** - 장시간 안정성
- **벤치마크 설계** - 재현 가능한 테스트
- **성능 회귀 방지** - CI/CD 통합

## 📌 왜 중요한가?

**"벤치마킹은 성능을 측정하고 개선을 검증하는 객관적 방법입니다."**

```
추측 vs 측정:

추측 기반:
┌─────────────────────────────────────┐
│ 개발자: "이제 더 빠를 거야"              │
│ → 배포                               │
│ → 사용자: "더 느려졌어요!"               │
│ → 롤백                               │
└─────────────────────────────────────┘

측정 기반:
┌─────────────────────────────────────┐
│ Before:                             │
│ - 처리량: 1,000 req/s                │
│ - P95: 100ms                        │
│ - 에러율: 0.1%                        │
└────────────┬────────────────────────┘
             │ (최적화)
┌────────────▼────────────────────────┐
│ After:                              │
│ - 처리량: 2,500 req/s (+150%)        │
│ - P95: 40ms (-60%)                  │
│ - 에러율: 0.05% (-50%)                │
│ → ✅ 개선 확인 → 배포                  │
└─────────────────────────────────────┘

벤치마킹 핵심 개념:

1. 테스트 유형:

   부하 테스트 (Load Test):
   ┌──────────────────────────────┐
   │ 정상 트래픽 시뮬레이션             │
   │ 1,000 req/s (예상 피크)        │
   │ 목표: 안정적 동작 확인            │
   └──────────────────────────────┘
   
   스트레스 테스트 (Stress Test):
   ┌──────────────────────────────┐
   │ 한계까지 부하 증가                │
   │ 1K → 2K → 5K → 10K req/s     │
   │ 목표: 병목 및 실패점 찾기          │
   └──────────────────────────────┘
   
   내구성 테스트 (Endurance Test):
   ┌──────────────────────────────┐
   │ 장시간 정상 부하                 │
   │ 1,000 req/s × 24시간          │
   │ 목표: 메모리 누수 탐지            │
   └──────────────────────────────┘
   
   스파이크 테스트 (Spike Test):
   ┌──────────────────────────────┐
   │ 갑작스런 부하 급증                │
   │ 100 → 10,000 → 100 req/s     │
   │ 목표: 오토스케일링 검증            │
   └──────────────────────────────┘

2. 성능 메트릭:
   
   Throughput (처리량):
   - Requests/sec
   - Transactions/sec
   - 높을수록 좋음
   
   Latency (응답시간):
   - Mean (평균)
   - P50 (중앙값)
   - P95, P99 (백분위수)
   - 낮을수록 좋음
   
   Error Rate (에러율):
   - HTTP 4xx, 5xx %
   - Timeout %
   - 0%가 이상적
   
   Resource Usage:
   - CPU %
   - Memory %
   - Network I/O
   - Disk I/O

3. 벤치마크 패턴:
   
   점진적 부하:
   ┌──────────────────────────────┐
   │ RPS                          │
   │   ▲                          │
   │ 5K│        ┌─────────────    │
   │ 4K│      ┌─┘                 │
   │ 3K│    ┌─┘                   │
   │ 2K│  ┌─┘                     │
   │ 1K│┌─┘                       │
   │   └──────────────────────→   │
   │        Time                  │
   └──────────────────────────────┘
   
   스파이크:
   ┌──────────────────────────────┐
   │ RPS                          │
   │   ▲                          │
   │10K│    ┌─┐                   │
   │   │    │ │                   │
   │   │    │ │                   │
   │ 1K│────┘ └──────────         │
   │   └──────────────────────→   │
   │        Time                  │
   └──────────────────────────────┘

4. 결과 해석:
   
   좋은 결과:
   ┌──────────────────────────────┐
   │ Latency vs Load              │
   │ Latency                      │
   │   ▲                          │
   │200│                          │
   │ms │                ─────     │
   │100│        ────────          │
   │ 50│────────                  │
   │   └──────────────────────→   │
   │   1K  2K  3K  4K  5K   RPS   │
   │                              │
   │ → 5K까지 선형적 증가             │
   │ → 안정적                       │
   └──────────────────────────────┘
   
   나쁜 결과:
   ┌──────────────────────────────┐
   │ Latency vs Load              │
   │ Latency                      │
   │   ▲                          │
   │ 2s│              ┌───        │
   │   │         ┌────┘           │
   │500│    ┌────┘                │
   │ms │────┘                     │
   │   └──────────────────────→   │
   │   1K  2K  3K  4K  5K   RPS   │
   │                              │
   │ → 3K부터 급증 (병목)            │
   │ → 2.5K가 한계                 │
   └──────────────────────────────┘

실무 시나리오:

시나리오 1 - 배포 전 검증:
┌─────────────────────────────────────┐
│ 최적화: DB 쿼리 개선                    │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Before Benchmark:                   │
│ - 1,000 req/s                       │
│ - P95: 200ms                        │
│ - Error: 0%                         │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ After Benchmark:                    │
│ - 1,500 req/s (+50%)                │
│ - P95: 80ms (-60%)                  │
│ - Error: 0%                         │
│ → ✅ 개선 확인 → 배포                  │
└─────────────────────────────────────┘

시나리오 2 - 용량 계획:
┌─────────────────────────────────────┐
│ 질문: "블랙프라이데이 대비 몇 대?"         │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Benchmark 결과:                      │
│ 1대: 2,000 req/s (한계)               │
│                                     │
│ 예상 트래픽: 10,000 req/s              │
│ 필요: 5대 (여유 20%)                   │
│ → 6대 준비                            │
└─────────────────────────────────────┘

시나리오 3 - 병목 발견:
┌─────────────────────────────────────┐
│ Stress Test:                        │
│ 1K → 2K → 3K req/s                  │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2.5K에서 에러 급증:                    │
│ - DB 연결 풀 고갈                      │
│ - CPU 100%                          │
│                                     │
│ 병목: DB 연결 수 (10개)                │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 수정: 연결 풀 50개                     │
│ → 5K req/s까지 안정                   │
└─────────────────────────────────────┘
```

---

## 🔧 실습 1: Apache Bench (ab)

### Step 1: 기본 벤치마크

```bash
# 테스트 서버 실행
docker run -d --name nginx-bench \
  -p 8080:80 \
  nginx:alpine

# Apache Bench로 테스트
docker run --rm --network host \
  alpine sh -c '
    apk add apache2-utils
    ab -n 10000 -c 100 http://localhost:8080/
  '

# 출력:
# Concurrency Level:      100
# Time taken for tests:   2.345 seconds
# Complete requests:      10000
# Failed requests:        0
# Total transferred:      8200000 bytes
# HTML transferred:       6150000 bytes
# Requests per second:    4264.39 [#/sec] (mean)
# Time per request:       23.450 [ms] (mean)
# Time per request:       0.234 [ms] (mean, across all concurrent requests)
#
# Percentage of the requests served within a certain time (ms)
#   50%     20
#   66%     22
#   75%     24
#   80%     25
#   90%     28
#   95%     32
#   98%     38
#   99%     42
#  100%     65 (longest request)

docker rm -f nginx-bench
```

### Step 2: POST 요청 벤치마크

```bash
# API 서버 실행 (예시)
docker run -d --name api-bench \
  -p 3000:3000 \
  node:18-alpine sh -c '
    cat > server.js <<EOF
const http = require("http");
http.createServer((req, res) => {
  let body = "";
  req.on("data", chunk => body += chunk);
  req.on("end", () => {
    res.writeHead(200, {"Content-Type": "application/json"});
    res.end(JSON.stringify({status: "ok", received: body}));
  });
}).listen(3000);
EOF
    node server.js
  '

# POST 벤치마크
echo '{"test": "data"}' > post-data.json

docker run --rm --network host \
  -v $(pwd)/post-data.json:/data.json \
  alpine sh -c '
    apk add apache2-utils
    ab -n 1000 -c 10 -p /data.json -T application/json http://localhost:3000/
  '

docker rm -f api-bench
```

---

## 🔧 실습 2: wrk (고급)

### Step 1: wrk 기본

```bash
# wrk로 벤치마크
docker run --rm --network host \
  williamyeh/wrk \
  -t 4 -c 100 -d 30s http://localhost:8080/

# 출력:
# Running 30s test @ http://localhost:8080/
#   4 threads and 100 connections
#   Thread Stats   Avg      Stdev     Max   +/- Stdev
#     Latency    23.45ms   12.34ms  200.00ms   85.67%
#     Req/Sec     1.05k   123.45   1.50k    73.33%
#   126789 requests in 30.03s, 104.23MB read
# Requests/sec:   4221.23
# Transfer/sec:      3.47MB
```

### Step 2: Lua 스크립트 (동적 요청)

```bash
# Lua 스크립트
cat > script.lua <<'EOF'
-- 랜덤 URL 생성
request = function()
  local id = math.random(1, 10000)
  return wrk.format("GET", "/api/users/" .. id)
end

-- 응답 처리
response = function(status, headers, body)
  if status ~= 200 then
    print("Error: " .. status)
  end
end
EOF

# 스크립트 실행
docker run --rm --network host \
  -v $(pwd)/script.lua:/script.lua \
  williamyeh/wrk \
  -t 4 -c 100 -d 30s -s /script.lua http://localhost:8080/
```

### Step 3: 결과 분석

```bash
# 상세 통계 수집
cat > report.lua <<'EOF'
done = function(summary, latency, requests)
  io.write("------------------------------
")
  io.write(string.format("Requests: %d
", summary.requests))
  io.write(string.format("Duration: %.2fs
", summary.duration / 1000000))
  io.write(string.format("Bytes: %d
", summary.bytes))
  io.write(string.format("Errors: %d
", summary.errors.connect +
                                          summary.errors.read +
                                          summary.errors.write +
                                          summary.errors.status +
                                          summary.errors.timeout))
  io.write("
Latency Distribution:
")
  io.write(string.format("  50%%: %.2fms
", latency:percentile(50)))
  io.write(string.format("  75%%: %.2fms
", latency:percentile(75)))
  io.write(string.format("  90%%: %.2fms
", latency:percentile(90)))
  io.write(string.format("  95%%: %.2fms
", latency:percentile(95)))
  io.write(string.format("  99%%: %.2fms
", latency:percentile(99)))
end
EOF

docker run --rm --network host \
  -v $(pwd)/report.lua:/report.lua \
  williamyeh/wrk \
  -t 4 -c 100 -d 30s -s /report.lua http://localhost:8080/
```

---

## 🔧 실습 3: Gatling (시나리오 기반)

### Step 1: Gatling 시나리오

```scala
// simulation/BasicSimulation.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class BasicSimulation extends Simulation {
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")

  val scn = scenario("Basic Load Test")
    .exec(
      http("Get Homepage")
        .get("/")
        .check(status.is(200))
    )
    .pause(1)
    .exec(
      http("Get API")
        .get("/api/users")
        .check(status.is(200))
    )

  setUp(
    scn.inject(
      nothingFor(4 seconds),
      atOnceUsers(10),
      rampUsers(100) during (10 seconds),
      constantUsersPerSec(20) during (30 seconds)
    )
  ).protocols(httpProtocol)
}
```

### Step 2: Gatling 실행

```bash
# Dockerfile
cat > Dockerfile.gatling <<'EOF'
FROM denvazh/gatling:latest
COPY simulation /opt/gatling/user-files/simulations/
EOF

docker build -t gatling-test -f Dockerfile.gatling .

# 실행
docker run --rm --network host \
  gatling-test \
  -s BasicSimulation

# 결과: HTML 리포트 생성
# results/basicsimulation-<timestamp>/index.html
```

---

## 🔧 실습 4: 스트레스 테스트

### Step 1: 점진적 부하 증가

```bash
# wrk로 점진적 증가
cat > stress-test.sh <<'EOF'
#!/bin/bash

for CONNECTIONS in 10 50 100 200 500 1000; do
  echo "=== Testing with $CONNECTIONS connections ==="
  docker run --rm --network host \
    williamyeh/wrk \
    -t 4 -c $CONNECTIONS -d 30s http://localhost:8080/ \
    | grep "Requests/sec"
  
  sleep 10
done
EOF

chmod +x stress-test.sh
./stress-test.sh

# 출력:
# === Testing with 10 connections ===
# Requests/sec:   1234.56
# === Testing with 50 connections ===
# Requests/sec:   4567.89
# === Testing with 100 connections ===
# Requests/sec:   5678.90
# === Testing with 200 connections ===
# Requests/sec:   5234.56  (감소 시작)
# === Testing with 500 connections ===
# Requests/sec:   3456.78  (계속 감소)
# === Testing with 1000 connections ===
# Requests/sec:   2345.67  (병목)
```

### Step 2: 병목 지점 분석

```bash
# 모니터링과 함께 스트레스 테스트
docker run -d --name stress-app \
  -p 8080:80 \
  nginx:alpine

# 백그라운드 모니터링
docker stats stress-app > stats.log &

# 스트레스 테스트
docker run --rm --network host \
  williamyeh/wrk \
  -t 8 -c 1000 -d 60s http://localhost:8080/

# stats.log 분석
grep stress-app stats.log

# 출력:
# CONTAINER  CPU %  MEM USAGE / LIMIT  NET I/O
# stress-app 95%    100MB / 512MB      1GB / 500MB
#            ↑ CPU 병목

docker rm -f stress-app
```

---

## 🔧 실습 5: 내구성 테스트

### Step 1: 24시간 테스트

```bash
# 24시간 지속 테스트
cat > endurance-test.sh <<'EOF'
#!/bin/bash

START_TIME=$(date +%s)
DURATION=$((24 * 60 * 60))  # 24시간

echo "timestamp,rps,latency_p95,errors,memory_mb" > endurance-results.csv

while true; do
  CURRENT_TIME=$(date +%s)
  ELAPSED=$((CURRENT_TIME - START_TIME))
  
  if [ $ELAPSED -ge $DURATION ]; then
    echo "Test completed!"
    break
  fi
  
  # 1분마다 벤치마크
  RESULT=$(docker run --rm --network host \
    williamyeh/wrk \
    -t 2 -c 50 -d 10s http://localhost:8080/ 2>&1)
  
  RPS=$(echo "$RESULT" | grep "Requests/sec" | awk '{print $2}')
  
  # 메모리 확인
  MEM=$(docker stats --no-stream app --format "{{.MemUsage}}" | awk '{print $1}' | sed 's/MiB//')
  
  echo "$CURRENT_TIME,$RPS,0,0,$MEM" >> endurance-results.csv
  
  sleep 60
done
EOF

chmod +x endurance-test.sh
./endurance-test.sh &

# 다음날 결과 분석
cat > analyze-endurance.py <<'EOF'
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('endurance-results.csv')
df['hours'] = (df['timestamp'] - df['timestamp'].iloc[0]) / 3600

fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(12, 8))

# RPS
ax1.plot(df['hours'], df['rps'])
ax1.set_xlabel('Hours')
ax1.set_ylabel('Requests/sec')
ax1.set_title('Throughput Over Time')

# Memory
ax2.plot(df['hours'], df['memory_mb'])
ax2.set_xlabel('Hours')
ax2.set_ylabel('Memory (MB)')
ax2.set_title('Memory Usage Over Time')

plt.tight_layout()
plt.savefig('endurance-report.png')
EOF

python3 analyze-endurance.py
```

---

## 🔧 실습 6: CI/CD 통합

### Step 1: GitLab CI 벤치마크

```yaml
# .gitlab-ci.yml
stages:
  - build
  - benchmark
  - deploy

build:
  stage: build
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA

benchmark:
  stage: benchmark
  script:
    # 앱 시작
    - docker run -d --name bench-app -p 8080:80 myapp:$CI_COMMIT_SHA
    - sleep 10
    
    # 벤치마크
    - |
      docker run --rm --network host \
        williamyeh/wrk \
        -t 4 -c 100 -d 30s http://localhost:8080/ \
        | tee benchmark-result.txt
    
    # 결과 파싱
    - RPS=$(grep "Requests/sec" benchmark-result.txt | awk '{print $2}')
    - P95=$(grep "95%" benchmark-result.txt | awk '{print $2}' | sed 's/ms//')
    
    # 기준 확인
    - |
      if (( $(echo "$RPS < 1000" | bc -l) )); then
        echo "❌ Performance regression: RPS $RPS < 1000"
        exit 1
      fi
    
    - |
      if (( $(echo "$P95 > 100" | bc -l) )); then
        echo "❌ Performance regression: P95 ${P95}ms > 100ms"
        exit 1
      fi
    
    - echo "✅ Performance OK: RPS=$RPS, P95=${P95}ms"
    
    # 정리
    - docker rm -f bench-app
  
  artifacts:
    paths:
      - benchmark-result.txt
    expire_in: 30 days

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/myapp myapp=myapp:$CI_COMMIT_SHA
  only:
    - main
  when: manual
```

### Step 2: GitHub Actions 벤치마크

```yaml
# .github/workflows/benchmark.yml
name: Performance Benchmark

on:
  pull_request:
    branches: [ main ]

jobs:
  benchmark:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker image
        run: docker build -t myapp:test .
      
      - name: Start application
        run: |
          docker run -d --name app -p 8080:80 myapp:test
          sleep 10
      
      - name: Run benchmark
        run: |
          docker run --rm --network host \
            williamyeh/wrk \
            -t 4 -c 100 -d 30s http://localhost:8080/ \
            | tee benchmark.txt
      
      - name: Parse results
        id: results
        run: |
          RPS=$(grep "Requests/sec" benchmark.txt | awk '{print $2}')
          P95=$(grep "95%" benchmark.txt | awk '{print $2}' | sed 's/ms//')
          
          echo "rps=$RPS" >> $GITHUB_OUTPUT
          echo "p95=$P95" >> $GITHUB_OUTPUT
      
      - name: Check performance
        run: |
          if (( $(echo "${{ steps.results.outputs.rps }} < 1000" | bc -l) )); then
            echo "::error::RPS below threshold"
            exit 1
          fi
          
          if (( $(echo "${{ steps.results.outputs.p95 }} > 100" | bc -l) )); then
            echo "::error::P95 latency above threshold"
            exit 1
          fi
      
      - name: Comment PR
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🚀 Performance Benchmark
              
              | Metric | Value |
              |--------|-------|
              | Throughput | ${{ steps.results.outputs.rps }} req/s |
              | P95 Latency | ${{ steps.results.outputs.p95 }} ms |
              
              ✅ All checks passed!`
            })
```

---

## 💡 주요 명령어 정리

### Apache Bench

```bash
# 기본
ab -n 10000 -c 100 http://localhost/

# POST
ab -n 1000 -c 10 -p data.json -T application/json http://localhost/

# Keep-Alive
ab -n 10000 -c 100 -k http://localhost/
```

### wrk

```bash
# 기본
wrk -t 4 -c 100 -d 30s http://localhost/

# 스크립트
wrk -t 4 -c 100 -d 30s -s script.lua http://localhost/

# Header
wrk -t 4 -c 100 -d 30s -H "Authorization: Bearer token" http://localhost/
```

### Docker 통합

```bash
# ab
docker run --rm --network host alpine sh -c '
  apk add apache2-utils
  ab -n 10000 -c 100 http://localhost/
'

# wrk
docker run --rm --network host williamyeh/wrk \
  -t 4 -c 100 -d 30s http://localhost/
```

---

## 🎓 연습 문제

### 문제 1: 처리량 한계 찾기

다음 벤치마크 결과에서 최대 처리량을 찾으세요:

```
10 conn:  1,000 req/s, P95: 20ms
50 conn:  4,000 req/s, P95: 50ms
100 conn: 5,500 req/s, P95: 90ms
200 conn: 5,800 req/s, P95: 180ms
500 conn: 5,200 req/s, P95: 500ms
```

<details>
<summary>정답 보기</summary>

**최대 처리량: 약 5,500-5,800 req/s (100-200 connections)**

분석:
- 100 conn: 5,500 req/s, P95: 90ms (최적)
- 200 conn: 5,800 req/s, P95: 180ms (레이턴시 2배)
- 500 conn: 5,200 req/s (처리량 감소)

권장:
- 운영: 100 connections (안정적)
- 최대: 200 connections (레이턴시 타협 가능 시)
- 500은 오버로드 (처리량도 감소, 레이턴시도 나쁨)

</details>

### 문제 2: 성능 회귀 탐지

Before: 2,000 req/s, P95: 50ms
After: 1,500 req/s, P95: 80ms

회귀 여부를 판단하세요.

<details>
<summary>정답 보기</summary>

**성능 회귀 확인 ❌**

증거:
- 처리량: 2,000 → 1,500 (25% 감소)
- P95: 50ms → 80ms (60% 증가)

조치:
```bash
# 1. 롤백 고려
# 2. 원인 분석
#    - Profiling 실행
#    - 리소스 사용 확인
#    - 코드 변경 검토

# 3. 기준 설정
# 처리량 변화: ±10% 허용
# 레이턴시 변화: ±20% 허용

# 이 경우 둘 다 초과 → 배포 중지
```

</details>

### 문제 3: 벤치마크 설계

블로그 사이트의 벤치마크 시나리오를 설계하세요.

<details>
<summary>정답 보기</summary>

```scala
// Gatling 시나리오
class BlogSimulation extends Simulation {
  val httpProtocol = http.baseUrl("http://localhost")

  val scn = scenario("Blog User Journey")
    // 1. 홈페이지
    .exec(http("Homepage")
      .get("/")
      .check(status.is(200)))
    .pause(2)
    
    // 2. 글 목록
    .exec(http("Post List")
      .get("/posts")
      .check(status.is(200)))
    .pause(3)
    
    // 3. 글 읽기
    .exec(http("Read Post")
      .get("/posts/${random_id}")
      .check(status.is(200)))
    .pause(10)
    
    // 4. 댓글 쓰기
    .exec(http("Post Comment")
      .post("/comments")
      .body(StringBody("""{"text":"Great!"}"""))
      .check(status.is(201)))

  setUp(
    // 정상 트래픽
    scn.inject(
      rampUsers(100) during (60 seconds),
      constantUsersPerSec(50) during (5 minutes)
    ),
    // 피크 트래픽
    scn.inject(
      nothingFor(6 minutes),
      rampUsers(500) during (60 seconds)
    )
  ).protocols(httpProtocol)
}
```

</details>

---

## 📌 핵심 요약

### 벤치마킹 도구 비교

| 도구 | 장점 | 단점 | 용도 |
|-----|------|------|------|
| **ab** | 간단, 빠름 | 기능 제한 | 간단한 테스트 |
| **wrk** | 강력, Lua | 설정 복잡 | 고급 테스트 |
| **Gatling** | 시나리오 | 무거움 | 사용자 흐름 |
| **JMeter** | GUI, 다양 | 리소스 많음 | 종합 테스트 |

### 테스트 유형

```
Load Test:
- 정상 트래픽
- 안정성 확인

Stress Test:
- 한계 탐색
- 병목 발견

Endurance Test:
- 24시간+
- 메모리 누수

Spike Test:
- 급격한 부하
- 스케일링 검증
```

### 성능 기준 예시

```yaml
API 서버:
  throughput: 1000 req/s
  p95_latency: 100ms
  p99_latency: 500ms
  error_rate: 0.1%

웹 사이트:
  throughput: 5000 req/s
  p95_latency: 200ms
  p99_latency: 1000ms
  error_rate: 0.5%

데이터베이스:
  throughput: 10000 queries/s
  p95_latency: 10ms
  p99_latency: 50ms
  error_rate: 0%
```

### CI/CD 통합 체크리스트

- [ ] 자동 벤치마크 실행
- [ ] 성능 기준 설정
- [ ] 회귀 탐지
- [ ] 결과 보고 (PR 코멘트)
- [ ] Artifact 저장
- [ ] 트렌드 추적
- [ ] 알림 설정

---

## 📚 참고 자료

- [Apache Bench Manual](https://httpd.apache.org/docs/2.4/programs/ab.html)
- [wrk GitHub](https://github.com/wg/wrk)
- [Gatling Documentation](https://gatling.io/docs/)
- [Performance Testing Guide](https://martinfowler.com/articles/practical-test-pyramid.html#PerformanceTests)

---

## 🤔 생각해볼 문제

1. 벤치마크 결과가 매번 다른 이유는?
2. Production과 같은 환경에서 테스트해야 하는 이유는?
3. P95, P99를 왜 측정할까?

> 💡 **답변**: 1) 벤치마크 결과 변동 원인, 시스템 상태, 다른 프로세스 실행 중, CPU/메모리 사용 중, 디스크 I/O 경쟁, 네트워크, 대역폭 변동, 패킷 손실, 라우팅 변경, 캐시, Warm vs Cold cache, 이전 요청 영향, GC (Java/Node.js), GC pause 타이밍, Heap 크기, 해결: 여러 번 실행 (최소 3-5번), 평균/중앙값 사용, 아웃라이어 제거, 안정된 환경 (전용 서버), 2) Production 환경 중요성, 하드웨어, Dev: 노트북 (4 core, 16GB), Prod: 서버 (32 core, 128GB), 10배 차이, 네트워크, Dev: localhost (빠름), Prod: 인터넷 (레이턴시), 데이터, Dev: 1000 rows, Prod: 10M rows, 쿼리 성능 차이, 부하, Dev: 1 user, Prod: 1000 users, 동시성 문제 발견 못 함, Staging, Production과 동일 스펙, 실제 데이터 복사, 안전한 테스트, 3) 백분위수 중요성, Mean (평균), 평균은 이상치에 민감, 1개의 느린 요청 → 평균 증가, 사용자 경험 반영 안 됨, P50 (중앙값), 절반은 이보다 빠름, 절반은 느림, 대표값으로 좋음, P95, 95%는 이보다 빠름, 5%는 느림, "거의 모든 사용자" 경험, P99, 99%는 이보다 빠름, 1%는 느림, "최악의 경험" 파악, SLA 설정, 예: P95 < 100ms 보장, 1000명 중 950명은 100ms 이하, 50명은 느릴 수 있음

---

<div align="center">

**[⬅️ 이전: Profiling](./07-Profiling.md)** | **[홈으로 🏠](../README.md)**

</div>
