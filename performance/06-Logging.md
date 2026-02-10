# 06. Logging - 중앙화 로깅

## 🎯 이 챕터에서 배울 것

- **로깅 드라이버** - json-file, syslog, fluentd
- **ELK Stack** - Elasticsearch, Logstash, Kibana
- **Fluentd** - 로그 수집 및 전달
- **로그 분석** - 패턴 탐지 및 알림
- **로그 보존** - 저장 및 아카이빙

## 📌 왜 중요한가?

**"중앙화 로깅은 분산 시스템의 문제 해결과 보안 감사의 핵심입니다."**

```
분산 로깅 vs 중앙화 로깅:

분산 로깅 (각 컨테이너):
┌─────────────────────────────────────┐
│ Container 1                         │
│ /var/log/app.log (2GB)              │
│ → 디스크 Full → 로그 유실               │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Container 2                         │
│ /var/log/app.log (500MB)            │
│ → 7일 후 자동 삭제                     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Container 3                         │
│ 재시작됨 → 로그 소실                    │
└─────────────────────────────────────┘

문제점:
❌ 로그 분산 (검색 어려움)
❌ 로그 유실 (디스크, 재시작)
❌ 디버깅 어려움 (여러 곳 확인)
❌ 보안 감사 불가
❌ 용량 관리 수동

중앙화 로깅:
┌─────────────────────────────────────┐
│ Container 1 ──┐                     │
│ Container 2 ──┼─→ Fluentd           │
│ Container 3 ──┘     ↓               │
│                 Elasticsearch       │
│                     ↓               │
│                  Kibana             │
│                (검색/시각화)           │
└─────────────────────────────────────┘

장점:
✅ 중앙 집중 검색
✅ 영구 보존
✅ 빠른 디버깅
✅ 보안 감사 추적
✅ 자동 관리

로깅 핵심 개념:

1. 로깅 드라이버:
   
   json-file (기본):
   ┌──────────────────────────────┐
   │ /var/lib/docker/containers/  │
   │ <container-id>/              │
   │   <container-id>-json.log    │
   │                              │
   │ 장점: 간단, 빠름                │
   │ 단점: 중앙화 없음, 용량 관리       │
   └──────────────────────────────┘
   
   syslog:
   ┌──────────────────────────────┐
   │ Container → syslog daemon    │
   │           → /var/log/syslog  │
   │                              │
   │ 장점: 표준 프로토콜              │
   │ 단점: 성능, 구조화 부족           │
   └──────────────────────────────┘
   
   fluentd:
   ┌──────────────────────────────┐
   │ Container → Fluentd          │
   │           → Elasticsearch    │
   │                              │
   │ 장점: 유연, 다양한 output        │
   │ 단점: 추가 컴포넌트 필요           │
   └──────────────────────────────┘

2. ELK Stack:
   
   ┌─────────────────────────────┐
   │ Kibana (Port 5601)          │
   │ - 검색 UI                    │
   │ - 대시보드                    │
   │ - 시각화                     │
   └──────────┬──────────────────┘
              │
   ┌──────────▼──────────────────┐
   │ Elasticsearch (Port 9200)   │
   │ - 인덱싱                      │
   │ - 검색 엔진                   │
   │ - 저장소                      │
   └──────────▲──────────────────┘
              │
   ┌──────────┴──────────────────┐
   │ Logstash / Fluentd          │
   │ - 수집                       │
   │ - 파싱                       │
   │ - 필터링                      │
   └──────────▲──────────────────┘
              │
   ┌──────────┴──────────────────┐
   │ Containers                  │
   └─────────────────────────────┘

3. 로그 레벨:
   
   ┌──────────────────────────────┐
   │ TRACE - 매우 상세 (개발)        │
   │ DEBUG - 디버깅 정보             │
   │ INFO  - 일반 정보              │
   │ WARN  - 경고 (주의 필요)        │
   │ ERROR - 에러 (복구 가능)        │
   │ FATAL - 치명적 (종료)           │
   └──────────────────────────────┘
   
   Production: INFO 이상
   Development: DEBUG 이상
   Troubleshooting: TRACE

4. 로그 보존 정책:
   
   Hot (빠른 검색):
   - 최근 7일
   - SSD 저장
   - 전체 인덱싱
   
   Warm (보관):
   - 8-30일
   - HDD 저장
   - 인덱스 압축
   
   Cold (아카이브):
   - 31-365일
   - Object Storage (S3)
   - 검색 느림
   
   Delete:
   - 365일 이상
   - 자동 삭제

실무 시나리오:

시나리오 1 - 에러 추적:
┌─────────────────────────────────────┐
│ 사용자 불만: "결제 실패"                 │
│ 시간: 2024-02-10 14:35               │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Kibana 검색:                         │
│ level:ERROR AND                     │
│ timestamp:[14:30 TO 14:40] AND      │
│ service:payment                     │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 결과 (5건):                           │
│ 14:35:23 - Database timeout         │
│ 14:35:24 - Retry failed             │
│ 14:35:25 - Transaction rolled back  │
│                                     │
│ 근본 원인: DB 연결 풀 부족               │
└─────────────────────────────────────┘

시나리오 2 - 보안 감사:
┌─────────────────────────────────────┐
│ 보안팀: "누가 admin 계정으로 로그인?"      │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Kibana 검색:                         │
│ action:login AND                    │
│ user:admin AND                      │
│ timestamp:[30d TO now]              │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 결과:                                │
│ 2024-01-15 09:00 - IP: 192.168.1.5  │
│ 2024-01-20 14:30 - IP: 10.0.0.100   │
│ 2024-02-05 11:20 - IP: 203.0.113.1  │
│                      ↑ 외부 IP!      │
│                                     │
│ 발견: 의심스러운 접근                    │
└─────────────────────────────────────┘

시나리오 3 - 성능 분석:
┌─────────────────────────────────────┐
│ "API 응답이 느려졌어요"                  │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Kibana Visualization:               │
│ API Response Time (7 days)          │
│   ▲                                 │
│ 2s│              ┌──                │
│   │         ┌────┘                  │
│500│    ┌────┘                       │
│ms │ ┌──┘                            │
│   └──────────────────────→          │
│        2/3  2/7  2/10               │
│                                     │
│ 패턴: 점진적 증가 (메모리 누수?)           │
└─────────────────────────────────────┘
```

---

## 🔧 실습 1: 로깅 드라이버 설정

### Step 1: json-file 드라이버 (기본)

```bash
# 기본 드라이버 확인
docker info | grep "Logging Driver"
# Logging Driver: json-file

# 로그 위치
docker inspect CONTAINER | grep LogPath
# "LogPath": "/var/lib/docker/containers/<id>/<id>-json.log"

# 로그 확인
docker logs CONTAINER

# 로그 파일 크기 제한 설정
docker run -d --name app \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp:latest

# 설정 확인
docker inspect app | grep -A 5 LogConfig

# 출력:
# "LogConfig": {
#     "Type": "json-file",
#     "Config": {
#         "max-file": "3",
#         "max-size": "10m"
#     }
# }
```

### Step 2: syslog 드라이버

```bash
# syslog 드라이버 사용
docker run -d --name app-syslog \
  --log-driver syslog \
  --log-opt syslog-address=udp://localhost:514 \
  --log-opt tag="{{.Name}}/{{.ID}}" \
  myapp:latest

# 호스트 syslog 확인
sudo tail -f /var/log/syslog | grep app-syslog

# 출력:
# Feb 10 10:00:01 hostname app-syslog[12345]: Application started
```

### Step 3: fluentd 드라이버

```bash
# Fluentd 실행 (먼저 필요)
docker run -d \
  --name fluentd \
  -p 24224:24224 \
  -v $(pwd)/fluent.conf:/fluentd/etc/fluent.conf \
  fluent/fluentd:latest

# fluentd 드라이버 사용
docker run -d --name app-fluentd \
  --log-driver fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag="docker.{{.Name}}" \
  myapp:latest

# docker logs 명령어는 작동 안 함 (중앙화됨)
docker logs app-fluentd
# Error: configured logging driver does not support reading
```

---

## 🔧 실습 2: ELK Stack 설치

### Step 1: Elasticsearch 설치

```bash
# Elasticsearch 실행
docker run -d \
  --name elasticsearch \
  -p 9200:9200 \
  -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  elasticsearch:8.11.0

# 상태 확인
curl http://localhost:9200

# 출력:
# {
#   "name" : "...",
#   "cluster_name" : "docker-cluster",
#   "version" : {
#     "number" : "8.11.0"
#   }
# }

# 인덱스 확인
curl http://localhost:9200/_cat/indices?v
```

### Step 2: Kibana 설치

```bash
# Kibana 실행
docker run -d \
  --name kibana \
  --link elasticsearch:elasticsearch \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  kibana:8.11.0

# 웹 UI 접속 (1-2분 대기)
# http://localhost:5601

# 상태 확인
curl http://localhost:5601/api/status

# 출력:
# {"status":{"overall":{"level":"available"}}}
```

### Step 3: Logstash 설치

```bash
# Logstash 설정 파일
cat > logstash.conf <<'EOF'
input {
  tcp {
    port => 5000
    codec => json
  }
}

filter {
  # 타임스탬프 파싱
  date {
    match => [ "timestamp", "ISO8601" ]
  }
  
  # 로그 레벨 추출
  grok {
    match => { "message" => "%{LOGLEVEL:level}" }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  
  stdout {
    codec => rubydebug
  }
}
EOF

# Logstash 실행
docker run -d \
  --name logstash \
  --link elasticsearch:elasticsearch \
  -p 5000:5000 \
  -v $(pwd)/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
  logstash:8.11.0

# 로그 전송 테스트
echo '{"message":"Test log","level":"INFO","timestamp":"2024-02-10T10:00:00Z"}' | \
  nc localhost 5000
```

---

## 🔧 실습 3: Fluentd 설정

### Step 1: Fluentd 설정 파일

```bash
# fluent.conf
cat > fluent.conf <<'EOF'
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

# 파싱 및 필터링
<filter docker.**>
  @type parser
  key_name log
  <parse>
    @type json
    time_key timestamp
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</filter>

# Elasticsearch로 전송
<match docker.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix docker
  include_tag_key true
  tag_key @log_name
  flush_interval 1s
</match>

# 디버그용 stdout
<match **>
  @type stdout
</match>
EOF

# Fluentd 실행
docker run -d \
  --name fluentd \
  --link elasticsearch:elasticsearch \
  -p 24224:24224 \
  -v $(pwd)/fluent.conf:/fluentd/etc/fluent.conf \
  fluent/fluentd:latest-debian
```

### Step 2: 애플리케이션 로깅

```bash
# Fluentd로 로그 전송하는 앱
docker run -d --name app \
  --log-driver=fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag="docker.app" \
  alpine sh -c '
    while true; do
      echo "{\"level\":\"INFO\",\"message\":\"Heartbeat\",\"timestamp\":\"$(date -Iseconds)\"}"
      sleep 10
    done
  '

# Kibana에서 확인
# http://localhost:5601
# Management → Stack Management → Index Patterns
# Create index pattern: docker-*
# Discover → 로그 확인
```

---

## 🔧 실습 4: Docker Compose 통합

### Step 1: ELK + Fluentd Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false
    ports:
      - 9200:9200
    volumes:
      - es-data:/usr/share/elasticsearch/data
    restart: always

  kibana:
    image: kibana:8.11.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch
    restart: always

  fluentd:
    build: ./fluentd
    container_name: fluentd
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    volumes:
      - ./fluent.conf:/fluentd/etc/fluent.conf
    depends_on:
      - elasticsearch
    restart: always

  app:
    image: alpine
    container_name: app
    command: sh -c 'while true; do echo "{\"level\":\"INFO\",\"message\":\"App running\"}"; sleep 5; done'
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.app
    depends_on:
      - fluentd

volumes:
  es-data:
```

### Step 2: Fluentd Dockerfile

```dockerfile
# fluentd/Dockerfile
FROM fluent/fluentd:v1.16-debian-1

USER root

# Elasticsearch plugin 설치
RUN gem install fluent-plugin-elasticsearch

USER fluent
```

### Step 3: 실행 및 확인

```bash
# 빌드 및 실행
docker-compose up -d

# 로그 확인
docker-compose logs -f fluentd

# Kibana에서 인덱스 패턴 생성
# http://localhost:5601
# Stack Management → Index Patterns
# Create: docker-*

# Discover에서 로그 확인
```

---

## 🔧 실습 5: 로그 분석 및 알림

### Step 1: Kibana 검색 쿼리

```
# 에러 로그 검색
level:ERROR

# 특정 시간대
level:ERROR AND timestamp:[now-1h TO now]

# 특정 서비스
level:ERROR AND service:payment

# 복합 조건
level:(ERROR OR WARN) AND service:api AND message:*timeout*

# 집계
- Aggregation: Date Histogram
- Field: @timestamp
- Interval: 1 hour

# 시각화
- Visualization: Line Chart
- Y-axis: Count
- X-axis: timestamp
```

### Step 2: Watcher (알림)

```json
# Elasticsearch Watcher (X-Pack 필요)
PUT _watcher/watch/error_alert
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["docker-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {
                  "match": {
                    "level": "ERROR"
                  }
                },
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total": {
        "gt": 10
      }
    }
  },
  "actions": {
    "slack_notification": {
      "webhook": {
        "url": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
        "body": "{\"text\":\"{{ ctx.payload.hits.total }} errors in last 5 minutes\"}"
      }
    }
  }
}
```

### Step 3: 로그 보존 정책

```bash
# Curator 설정 (로그 삭제 자동화)
cat > curator.yml <<'EOF'
---
client:
  hosts:
    - elasticsearch
  port: 9200

actions:
  1:
    action: delete_indices
    description: >-
      Delete indices older than 30 days
    options:
      ignore_empty_list: True
      disable_action: False
    filters:
      - filtertype: pattern
        kind: prefix
        value: docker-
      - filtertype: age
        source: name
        direction: older
        timestring: '%Y.%m.%d'
        unit: days
        unit_count: 30
EOF

# Curator 실행 (크론잡)
docker run --rm \
  --link elasticsearch:elasticsearch \
  -v $(pwd)/curator.yml:/curator.yml \
  bobrik/curator:latest \
  --config /curator.yml /curator.yml
```

---

## 💡 주요 명령어 정리

### Docker 로깅

```bash
# 로그 확인
docker logs CONTAINER
docker logs -f CONTAINER         # Follow
docker logs --tail 100 CONTAINER # 마지막 100줄
docker logs --since 10m CONTAINER # 최근 10분

# 로깅 드라이버 설정
docker run --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  IMAGE

# daemon.json (전역 설정)
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### Elasticsearch

```bash
# 인덱스 조회
curl http://localhost:9200/_cat/indices?v

# 검색
curl -X GET "localhost:9200/docker-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "level": "ERROR"
    }
  }
}
'

# 인덱스 삭제
curl -X DELETE "localhost:9200/docker-2024.01.01"
```

### Kibana

```
# Dev Tools에서 쿼리
GET docker-*/_search
{
  "query": {
    "match_all": {}
  }
}

# 대시보드 생성
Visualize → Create visualization → 차트 선택
```

---

## 🎓 연습 문제

### 문제 1: 로그 크기 제한

컨테이너 로그를 최대 50MB, 5개 파일로 제한하세요.

<details>
<summary>정답 보기</summary>

```bash
# 방법 1: 컨테이너별
docker run -d --name app \
  --log-driver json-file \
  --log-opt max-size=50m \
  --log-opt max-file=5 \
  myapp:latest

# 방법 2: 전역 설정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
EOF

sudo systemctl restart docker

# 확인
docker inspect app | grep -A 5 LogConfig
```

</details>

### 문제 2: ELK 검색 쿼리

최근 1시간 동안 "payment" 서비스의 ERROR 로그를 찾으세요.

<details>
<summary>정답 보기</summary>

```
Kibana Query:
level:ERROR AND service:payment AND @timestamp:[now-1h TO now]

또는 Elasticsearch API:
GET docker-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "level": "ERROR" } },
        { "match": { "service": "payment" } },
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h",
              "lte": "now"
            }
          }
        }
      ]
    }
  },
  "sort": [
    { "@timestamp": { "order": "desc" } }
  ]
}
```

</details>

### 문제 3: 로그 보존 자동화

30일 이상 된 로그 인덱스를 자동 삭제하는 크론잡을 만드세요.

<details>
<summary>정답 보기</summary>

```bash
# curator.yml
cat > curator.yml <<'EOF'
client:
  hosts: [elasticsearch]
  port: 9200

actions:
  1:
    action: delete_indices
    description: Delete old logs
    options:
      ignore_empty_list: True
    filters:
      - filtertype: pattern
        kind: prefix
        value: docker-
      - filtertype: age
        source: name
        direction: older
        timestring: '%Y.%m.%d'
        unit: days
        unit_count: 30
EOF

# 크론잡 (매일 새벽 2시)
0 2 * * * docker run --rm \
  --link elasticsearch:elasticsearch \
  -v /path/to/curator.yml:/curator.yml \
  bobrik/curator:latest \
  --config /curator.yml /curator.yml
```

</details>

---

## 📌 핵심 요약

### 로깅 드라이버 비교

| 드라이버 | 장점 | 단점 | 용도 |
|---------|------|------|------|
| **json-file** | 간단, 빠름 | 중앙화 없음 | 개발 |
| **syslog** | 표준 | 구조화 부족 | 레거시 |
| **fluentd** | 유연, 다양한 output | 복잡 | 프로덕션 |
| **awslogs** | AWS 통합 | Vendor lock-in | AWS 환경 |

### ELK Stack 구성

```
Kibana (5601)
  ↓ 검색/시각화
Elasticsearch (9200)
  ↓ 저장/인덱싱
Logstash/Fluentd
  ↓ 수집/파싱
Containers
```

### 로그 보존 전략

```
Hot (0-7일):
- SSD 저장
- 빠른 검색
- 전체 인덱스

Warm (8-30일):
- HDD 저장
- 압축
- Read-only

Cold (31-365일):
- S3/Object Storage
- 아카이브
- 느린 검색

Delete (365일+):
- 자동 삭제
- 규정 준수
```

### 로깅 Best Practices

- [ ] 구조화된 로그 (JSON)
- [ ] 타임스탬프 포함
- [ ] 로그 레벨 명시
- [ ] Request ID 추적
- [ ] 민감 정보 제외
- [ ] 로그 크기 제한
- [ ] 중앙화 (ELK)
- [ ] 보존 정책

---

## 📚 참고 자료

- [Docker Logging Drivers](https://docs.docker.com/config/containers/logging/)
- [Elasticsearch Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Fluentd Documentation](https://docs.fluentd.org/)
- [Kibana User Guide](https://www.elastic.co/guide/en/kibana/current/index.html)

---

## 🤔 생각해볼 문제

1. json-file 드라이버의 로그는 어디에 저장될까?
2. 중앙화 로깅이 왜 필요할까?
3. 로그를 영구 보존해야 할까?

> 💡 **답변**: 1) 위치: /var/lib/docker/containers/<container-id>/<container-id>-json.log, 구조: JSON Lines 형식, 각 줄이 하나의 로그, 문제점: 컨테이너 삭제 시 로그도 삭제, 디스크 Full 위험, 검색 어려움, 제한: max-size, max-file로 크기 제한, 기본값: 무제한 (위험), 권장: max-size=10m, max-file=3, 2) 필요성, 분산 시스템: 수십-수백 개 컨테이너, 각 로그 확인 불가능, 문제 추적: 여러 서비스 연관, 하나의 request가 여러 서비스 통과, Request ID로 전체 흐름 추적, 보안 감사: 누가, 언제, 무엇을, 규정 준수 (GDPR, PCI-DSS), 검색/분석: Kibana로 패턴 탐지, 알림 설정, 안정성: 컨테이너 삭제/재시작에도 로그 유지, 3) 규정에 따라, 금융: 7년 (법적 의무), 의료: 6년 (HIPAA), 일반: 30-90일, 전략: Hot (7일, 빠름) → Warm (30일, 느림) → Cold (365일, 아카이브) → Delete, 비용: 저장 비용 vs 필요성, S3 Glacier 사용 (저렴), 압축으로 90% 절감

---

<div align="center">

**[⬅️ 이전: Monitoring](./05-Monitoring.md)** | **[다음: Profiling ➡️](./07-Profiling.md)**

</div>
