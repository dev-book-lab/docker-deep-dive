# 02. Log Analysis - 로그 분석 기법

## 🎯 이 챕터에서 배울 것

- **로그 수집**: docker logs, kubectl logs
- **로그 드라이버**: json-file, syslog, fluentd
- **중앙 집중식 로깅**: ELK, Loki, CloudWatch
- **로그 분석**: grep, awk, jq
- **실시간 모니터링**: tail, watch
- **실전 기법**: 패턴 인식, 이슈 추적

## 📌 왜 중요한가?

**"로그는 애플리케이션의 블랙박스이며, 문제 해결의 첫 번째 단서입니다."**

```
Log Analysis의 핵심:

Without Log Analysis:
┌─────────────────────────────────────────────────┐
│ 문제 발생                                         │
│    ↓                                            │
│ "무슨 일이 일어났지?" 🤔                            │
│    ↓                                            │
│ 추측과 시행착오                                     │
│    ↓                                            │
│ 시간 낭비 (수 시간)                                 │
└─────────────────────────────────────────────────┘

With Log Analysis:
┌─────────────────────────────────────────────────┐
│ 문제 발생                                         │
│    ↓                                            │
│ 로그 확인                                         │
│    ↓                                            │
│ ERROR: Connection timeout at 10:23:45           │
│ Stack trace: ...                                │
│ Request: POST /api/users                        │
│    ↓                                            │
│ 원인 파악 (수 분)                                  │
│    ↓                                            │
│ 해결                                             │
└─────────────────────────────────────────────────┘

Log Levels:
┌─────────────────────────────────────────────────┐
│ TRACE (가장 상세)                                 │
│   └─ 모든 실행 흐름                                │
│                                                 │
│ DEBUG                                           │
│   └─ 디버깅 정보                                   │
│                                                 │
│ INFO ★ (기본)                                    │
│   └─ 일반 정보                                    │
│                                                 │
│ WARN ⚠️                                         │
│   └─ 경고 (동작은 함)                              │
│                                                 │
│ ERROR ❌                                        │
│   └─ 에러 (일부 실패)                              │
│                                                 │
│ FATAL 💀                                        │
│   └─ 치명적 에러 (종료)                             │
└─────────────────────────────────────────────────┘

프로덕션 설정:
- 개발: DEBUG
- 스테이징: INFO
- 프로덕션: WARN

Log Format Best Practices:
┌─────────────────────────────────────────────────┐
│ Structured Logging (JSON)                       │
│                                                 │
│ {                                               │
│   "timestamp": "2024-01-15T10:23:45.123Z",      │
│   "level": "ERROR",                             │
│   "service": "api-server",                      │
│   "trace_id": "abc123",                         │
│   "user_id": "user-456",                        │
│   "message": "Database connection failed",      │
│   "error": {                                    │
│     "type": "TimeoutError",                     │
│     "stack": "..."                              │
│   },                                            │
│   "context": {                                  │
│     "endpoint": "/api/users",                   │
│     "method": "POST"                            │
│   }                                             │
│ }                                               │
│                                                 │
│ 장점:                                            │
│ ✅ 파싱 쉬움 (jq)                                 │
│ ✅ 검색 쉬움 (필드별)                               │
│ ✅ 집계 쉬움 (trace_id)                           │
└─────────────────────────────────────────────────┘

Log Aggregation:
┌─────────────────────────────────────────────────┐
│ Container 1 ──┐                                 │
│ Container 2 ──┼─→ Log Aggregator ──→ Storage    │
│ Container 3 ──┘     (Fluentd)       (ES/Loki)   │
│                                                 │
│ 장점:                                            │
│ - 중앙 집중식 검색                                  │
│ - 장기 보관                                       │
│ - 분석 및 시각화                                   │
│ - 컨테이너 재시작 후에도 보존                         │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **문제 해결 속도**: 수 시간 → 수 분
- **사전 예방**: 패턴 인식으로 장애 예측
- **감사 추적**: 누가, 언제, 무엇을
- **성능 분석**: 응답 시간, 처리량

---

## 🔬 Deep Dive

### 1. Docker 로그 기본

#### docker logs 명령어

```bash
# 기본 로그 출력
docker logs container-name

# 실시간 로그 (tail -f)
docker logs -f container-name

# 최근 N줄
docker logs --tail 100 container-name

# 타임스탬프 포함
docker logs -t container-name

# 특정 시간 이후
docker logs --since 2024-01-15T10:00:00 container-name
docker logs --since 1h container-name  # 1시간 전부터

# 특정 시간 이전
docker logs --until 2024-01-15T11:00:00 container-name

# 조합
docker logs --since 1h --tail 1000 -f container-name
```

#### 로그 드라이버

```bash
# 현재 로그 드라이버 확인
docker inspect -f '{{.HostConfig.LogConfig.Type}}' container-name

# json-file (기본)
docker run --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp

# syslog
docker run --log-driver syslog \
  --log-opt syslog-address=tcp://192.168.1.100:514 \
  myapp

# fluentd
docker run --log-driver fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag="docker.{{.Name}}" \
  myapp

# 로그 없음 (성능 최적화, 비권장)
docker run --log-driver none myapp
```

---

### 2. Kubernetes 로그

#### kubectl logs

```bash
# Pod 로그
kubectl logs pod-name

# 특정 컨테이너 (multi-container)
kubectl logs pod-name -c container-name

# 이전 컨테이너 로그 (재시작된 경우)
kubectl logs --previous pod-name

# 실시간
kubectl logs -f pod-name

# 여러 Pod (label selector)
kubectl logs -l app=myapp --all-containers=true

# 타임스탬프
kubectl logs --timestamps pod-name

# 특정 시간 이후
kubectl logs --since=1h pod-name
```

---

## 🔧 실습 1: 기본 로그 분석

### Step 1: 로그 필터링

```bash
# 에러만 필터링
docker logs myapp 2>&1 | grep ERROR

# 특정 패턴
docker logs myapp | grep "Connection timeout"

# 대소문자 무시
docker logs myapp | grep -i error

# 여러 패턴 (OR)
docker logs myapp | grep -E "ERROR|WARN"

# 여러 패턴 (AND)
docker logs myapp | grep ERROR | grep database

# 제외
docker logs myapp | grep -v DEBUG

# 줄 번호 포함
docker logs myapp | grep -n ERROR
```

### Step 2: 로그 카운팅

```bash
# 에러 개수
docker logs myapp | grep ERROR | wc -l

# 패턴별 개수
docker logs myapp | grep -c "Connection refused"

# 시간별 에러 개수
docker logs --since 1h myapp | grep ERROR | wc -l

# 분당 에러율
docker logs --since 1h myapp | \
  grep ERROR | \
  awk '{print $1}' | \
  uniq -c
```

---

## 🔧 실습 2: 구조화된 로그 분석 (JSON)

### Step 1: jq를 이용한 파싱

```bash
# JSON 로그 예시
docker logs myapp | head -1
# {"timestamp":"2024-01-15T10:23:45Z","level":"ERROR","message":"DB error"}

# jq로 파싱
docker logs myapp | jq .

# 특정 필드 추출
docker logs myapp | jq .message
docker logs myapp | jq '.timestamp, .level, .message'

# 필터링 (ERROR만)
docker logs myapp | jq 'select(.level == "ERROR")'

# 여러 조건
docker logs myapp | jq 'select(.level == "ERROR" and .service == "api")'

# 카운팅
docker logs myapp | jq 'select(.level == "ERROR")' | jq -s 'length'

# 그룹화 (trace_id별)
docker logs myapp | jq -s 'group_by(.trace_id) | map({trace_id: .[0].trace_id, count: length})'
```

### Step 2: 복잡한 분석

```bash
# 응답 시간 통계
docker logs myapp | \
  jq -r 'select(.response_time) | .response_time' | \
  awk '{sum+=$1; count++} END {print "Avg:", sum/count}'

# 상위 10개 느린 요청
docker logs myapp | \
  jq -r 'select(.response_time) | "\(.response_time) \(.endpoint)"' | \
  sort -rn | \
  head -10

# 엔드포인트별 에러율
docker logs myapp | \
  jq -r '"\(.endpoint) \(.level)"' | \
  awk '{endpoint[$1]++; if($2=="ERROR") errors[$1]++} 
       END {for(e in endpoint) print e, errors[e]/endpoint[e]*100"%"}'
```

---

## 🔧 실습 3: 실시간 로그 모니터링

### Step 1: tail과 grep 조합

```bash
# 실시간 에러 모니터링
docker logs -f myapp 2>&1 | grep --color ERROR

# 여러 패턴
docker logs -f myapp | grep -E --color "ERROR|WARN|FATAL"

# 타임스탬프 포함
docker logs -ft myapp | grep ERROR

# 컨텍스트 포함 (전후 2줄)
docker logs -f myapp | grep -C 2 ERROR
```

### Step 2: 로그 알림

```bash
# 에러 발생 시 알림
docker logs -f myapp | while read line; do
  echo "$line" | grep -q ERROR && \
    echo "ERROR detected: $line" | \
    mail -s "Alert: Error in myapp" admin@example.com
done

# Slack 알림
docker logs -f myapp | while read line; do
  if echo "$line" | grep -q "FATAL"; then
    curl -X POST https://hooks.slack.com/services/XXX \
      -d "{\"text\":\"FATAL: $line\"}"
  fi
done
```

---

## 🔧 실습 4: 중앙 집중식 로깅 (ELK)

### Step 1: Fluentd 설정

```yaml
# fluentd/fluent.conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<filter docker.**>
  @type parser
  key_name log
  <parse>
    @type json
  </parse>
</filter>

<match docker.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix docker
  include_tag_key true
  tag_key @log_name
</match>
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 애플리케이션
  app:
    image: myapp:latest
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.app
  
  # Fluentd
  fluentd:
    image: fluent/fluentd:latest
    ports:
      - "24224:24224"
    volumes:
      - ./fluentd:/fluentd/etc
    depends_on:
      - elasticsearch
  
  # Elasticsearch
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
  
  # Kibana
  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
```

### Step 2: Kibana에서 로그 검색

```
# Kibana Query Language (KQL)

# 에러만
level: ERROR

# 특정 서비스
service: "api-server" AND level: ERROR

# 시간 범위
@timestamp >= "2024-01-15T10:00:00" AND @timestamp <= "2024-01-15T11:00:00"

# 정규 표현식
message: /timeout/

# 존재 여부
_exists_: trace_id

# 범위
response_time > 1000

# 집계 (Visualization)
# - Count by level
# - Average response_time by endpoint
# - Top 10 errors
```

---

## 🔧 실습 5: Loki와 Grafana

### Step 1: Loki 설정

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
  filesystem:
    directory: /loki/chunks
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Loki
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/config.yaml
  
  # Promtail (로그 수집)
  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml
  
  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  loki-data:
  grafana-data:
```

### Step 2: LogQL 쿼리

```
# Loki Query Language (LogQL)

# 특정 컨테이너
{container="myapp"}

# 레이블 필터
{container="myapp", level="ERROR"}

# 정규 표현식
{container=~"api.*"}

# 로그 내용 필터
{container="myapp"} |= "error"
{container="myapp"} != "debug"

# JSON 파싱
{container="myapp"} | json | level="ERROR"

# 집계
sum(rate({container="myapp"} |= "error" [5m]))

# Rate (분당)
rate({container="myapp"} |= "error" [1m])
```

---

## 🔧 실습 6: 로그 패턴 분석

### Step 1: 일반적인 에러 패턴

```bash
# Connection 에러
docker logs myapp | grep -E "connection (refused|timeout|reset)"

# OOM (Out of Memory)
docker logs myapp | grep -E "out of memory|OOM"

# Permission 에러
docker logs myapp | grep -E "permission denied|access denied"

# File not found
docker logs myapp | grep -E "no such file|file not found"

# Database 에러
docker logs myapp | grep -E "database|sql|query"
```

### Step 2: 로그 통계

```bash
# 시간별 요청 수
docker logs myapp | \
  awk '{print $1}' | \
  cut -d'T' -f2 | \
  cut -d':' -f1 | \
  sort | \
  uniq -c

# 상위 10개 에러 메시지
docker logs myapp | \
  grep ERROR | \
  awk -F'"message":"' '{print $2}' | \
  cut -d'"' -f1 | \
  sort | \
  uniq -c | \
  sort -rn | \
  head -10

# HTTP 상태 코드 분포
docker logs myapp | \
  grep -oP 'status=\K\d+' | \
  sort | \
  uniq -c | \
  sort -rn
```

---

## 💡 주요 명령어 정리

```
┌──────────────────────┬────────────────────────────┐
│ 명령어                 │ 용도                        │
├──────────────────────┼────────────────────────────┤
│ docker logs          │ 컨테이너 로그 확인             │
├──────────────────────┼────────────────────────────┤
│ kubectl logs         │ Pod 로그 확인                │
├──────────────────────┼────────────────────────────┤
│ grep                 │ 패턴 검색                    │
├──────────────────────┼────────────────────────────┤
│ jq                   │ JSON 파싱                   │
├──────────────────────┼────────────────────────────┤
│ awk                  │ 텍스트 처리                   │
├──────────────────────┼────────────────────────────┤
│ fluentd              │ 로그 수집                    │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. 구조화된 로그 (JSON)
2. 적절한 로그 레벨
3. 중앙 집중식 로깅
4. 로그 보존 정책
5. 민감 정보 제외
```

---

## 🎓 연습 문제

### 문제 1: 컨테이너 재시작 후 로그는?

<details>
<summary>정답 보기</summary>

**기본 동작:**
```bash
# 현재 컨테이너 로그만
docker logs myapp

# 이전 컨테이너 로그 (없음!)
# 재시작하면 이전 로그 사라짐
```

**해결:**

**1. Kubernetes (이전 로그 보존)**
```bash
kubectl logs --previous pod-name
```

**2. 영구 볼륨 마운트**
```yaml
volumes:
  - ./logs:/var/log/app
```

**3. 로그 드라이버 (syslog, fluentd)**
```bash
docker run --log-driver syslog myapp
# 외부 시스템에 저장
```

**4. 중앙 집중식 로깅**
```
Container → Fluentd → Elasticsearch
# 컨테이너와 독립적으로 보관
```

</details>

### 문제 2: 로그가 너무 많아서 성능 문제가 생긴다면?

<details>
<summary>정답 보기</summary>

**문제: 로그 폭주**
```bash
# 초당 1000줄
# 디스크 가득 참
# I/O 병목
```

**해결:**

**1. 로그 레벨 조정**
```
개발: DEBUG
프로덕션: WARN 또는 ERROR
```

**2. 로그 로테이션**
```bash
docker run \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp
```

**3. 샘플링**
```python
# 10%만 로그
if random.random() < 0.1:
    logger.debug(...)
```

**4. 비동기 로깅**
```python
import logging.handlers

handler = logging.handlers.QueueHandler(queue)
logger.addHandler(handler)
```

**5. 구조화 + 집계**
```json
{
  "type": "request",
  "count": 1000,  // 1000개 집계
  "avg_time": 123
}
```

</details>

### 문제 3: 여러 서비스의 로그를 연관 지어 추적하려면?

<details>
<summary>정답 보기</summary>

**Trace ID 사용:**

**1. 요청마다 고유 ID 생성**
```python
# API Gateway
trace_id = str(uuid.uuid4())
headers = {'X-Trace-ID': trace_id}

# 다음 서비스로 전달
response = requests.get(url, headers=headers)
```

**2. 모든 로그에 포함**
```python
logger.info("Processing request", extra={
    'trace_id': trace_id,
    'service': 'api-server'
})
```

**3. 검색**
```bash
# 모든 서비스에서 같은 trace_id 검색
docker logs api | jq 'select(.trace_id == "abc-123")'
docker logs db | jq 'select(.trace_id == "abc-123")'

# Elasticsearch
trace_id: "abc-123"

# Distributed Tracing (Jaeger, Zipkin)
# 자동으로 추적 및 시각화
```

**결과:**
```
API Gateway (10:00:00.123) → trace_id: abc-123
  ↓
User Service (10:00:00.145) → trace_id: abc-123
  ↓
Database (10:00:00.189) → trace_id: abc-123
```

</details>

---

## 📌 핵심 요약

```
Log Analysis 핵심:
1. docker/kubectl logs
2. grep, jq, awk
3. 구조화된 로그 (JSON)
4. 중앙 집중식 (ELK, Loki)
5. Trace ID로 연관

Best Practices:
✅ JSON 로그
✅ 적절한 레벨
✅ Trace ID
✅ 로그 로테이션
✅ 중앙 집중식 저장
```

---

## 📚 참고 자료

- [Docker Logging](https://docs.docker.com/config/containers/logging/)
- [Fluentd Documentation](https://docs.fluentd.org/)
- [Elastic Stack](https://www.elastic.co/elastic-stack/)

---

## 🤔 생각해볼 문제

1. 프로덕션 로그에 사용자 비밀번호가 출력된다면?
2. 로그를 얼마나 오래 보관해야 하는가?
3. 로그만으로 성능 문제를 파악할 수 있는가?

> 💡 **답변**:
> 
> **1) 민감 정보 로깅:**
> 
> ```python
> # ❌ 나쁜 예
> logger.info(f"User login: {email}, {password}")
> 
> # ✅ 좋은 예
> logger.info(f"User login: {email}")
> 
> # ✅ 마스킹
> def mask(s):
>     return s[:2] + "***" + s[-2:]
> 
> logger.info(f"Card: {mask(card_number)}")
> ```
> 
> **2) 로그 보존 기간:**
> 
> ```
> Hot Storage (빠른 검색):
> - 최근 7일
> - Elasticsearch
> 
> Warm Storage (보관):
> - 30-90일
> - S3, Glacier
> 
> Cold Storage (규정 준수):
> - 1-7년
> - 압축, 암호화
> 
> 고려사항:
> - 법적 요구사항 (GDPR 등)
> - 디스크 비용
> - 검색 빈도
> ```
> 
> **3) 로그로 성능 파악:**
> 
> ```
> ✅ 가능:
> - 응답 시간 (로그에 기록)
> - 에러율
> - 처리량 (RPS)
> 
> ❌ 제한적:
> - CPU 사용률
> - 메모리 사용률
> - 네트워크 I/O
> 
> 권장:
> 로그 + 메트릭 (Prometheus)
> + Tracing (Jaeger)
> ```

---

<div align="center">

**[⬅️ 이전: Debugging Techniques](01-Debugging-Techniques.md)** | **[다음: Network Debugging ➡️](03-Network-Debugging.md)**

</div>
