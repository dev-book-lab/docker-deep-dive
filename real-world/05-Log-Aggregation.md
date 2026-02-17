# 05. Log Aggregation - ELK/EFK 스택 구축

## 🎯 이 챕터에서 배울 것

- **Elasticsearch**: 로그 저장 및 검색
- **Logstash**: 로그 수집 및 변환
- **Kibana**: 로그 시각화
- **Fluentd**: 경량 로그 수집
- **Log 분석**: 실전 로그 분석 기법
- **실전 구성**: Production-ready 로깅

## 📌 왜 중요한가?

**"로그는 애플리케이션의 블랙박스이며, 중앙 집중식 로깅은 필수입니다."**

```
Log Aggregation의 필요성:

Without Centralized Logging:
┌─────────────────────────────────────────────────┐
│ 문제 발생                                         │
│   ↓                                             │
│ "에러가 났는데 어디서?"                              │
│   ↓                                             │
│ Container 1: docker logs container1             │
│ Container 2: docker logs container2             │
│ Container 3: docker logs container3             │
│   ↓                                             │
│ 수동으로 grep, 시간 맞추기 (30분)                     │
│   ↓                                             │
│ Container 재시작 → 로그 손실 ❌                     │
└─────────────────────────────────────────────────┘

With Centralized Logging (ELK):
┌─────────────────────────────────────────────────┐
│ 문제 발생                                         │
│   ↓                                             │
│ Kibana에서 검색: "error AND user_id:123"          │
│   ↓                                             │
│ 모든 컨테이너 로그 한눈에 (5초)                        │
│   ↓                                             │
│ 시간순 정렬, 필터링, 시각화                           │
│   ↓                                             │
│ Container 재시작해도 로그 보존 ✅                    │
└─────────────────────────────────────────────────┘

ELK Stack Architecture:
┌─────────────────────────────────────────────────┐
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │Container1│  │Container2│  │Container3│       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │             │             │             │
│       └─────────┬───┴─────────────┘             │
│                 ▼                               │
│         ┌───────────────┐                       │
│         │   Logstash    │ ← 로그 수집/변환         │
│         │   or Fluentd  │                       │
│         └───────┬───────┘                       │
│                 ▼                               │
│         ┌───────────────┐                       │
│         │ Elasticsearch │ ← 로그 저장/검색         │
│         └───────┬───────┘                       │
│                 ▼                               │
│         ┌───────────────┐                       │
│         │    Kibana     │ ← 로그 시각화            │
│         └───────────────┘                       │
│                                                 │
│  Features:                                      │
│  - 중앙 집중식 로그                                 │
│  - 실시간 검색                                     │
│  - 시각화 대시보드                                  │
│  - Alert 설정                                    │
│  - 장기 보관                                      │
└─────────────────────────────────────────────────┘

ELK vs EFK:
┌────────────────┬────────────┬────────────┐
│ Component      │ ELK        │ EFK        │
├────────────────┼────────────┼────────────┤
│ 수집            │ Logstash   │ Fluentd    │
├────────────────┼────────────┼────────────┤
│ 저장/검색        │ Elasticsearch (공통)     │
├────────────────┼────────────┼────────────┤
│ 시각화           │ Kibana (공통)            │
├────────────────┼────────────┼────────────┤
│ 리소스           │ 높음        │ 낮음        │
├────────────────┼────────────┼────────────┤
│ 복잡도           │ 높음        │ 낮음        │
├────────────────┼────────────┼────────────┤
│ 플러그인         │ 많음        │ 많음        │
└────────────────┴────────────┴────────────┘

Log Flow:
┌─────────────────────────────────────────────────┐
│ Application                                     │
│   ↓ (stdout/stderr)                             │
│ Docker Log Driver                               │
│   ↓ (json-file, fluentd, gelf)                  │
│ Logstash/Fluentd                                │
│   ↓ (parse, filter, enrich)                     │
│ Elasticsearch                                   │
│   ↓ (index, search)                             │
│ Kibana                                          │
│   ↓ (visualize, alert)                          │
│ User                                            │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **빠른 디버깅**: 분산 로그 한눈에
- **데이터 보존**: 재시작해도 유지
- **실시간 분석**: 패턴 발견
- **알림**: 에러 즉시 감지

---

## 🔧 실습 1: 기본 ELK Stack

### Step 1: Elasticsearch 설정

```yaml
# docker-compose.elk.yml
version: '3.8'

services:
  # Elasticsearch
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    restart: always
    environment:
      - node.name=elasticsearch
      - cluster.name=docker-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk

  # Logstash
  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    restart: always
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      - "LS_JAVA_OPTS=-Xms256m -Xmx256m"
    depends_on:
      - elasticsearch
    networks:
      - elk

  # Kibana
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    restart: always
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=kibana_system
      - ELASTICSEARCH_PASSWORD=kibana
    depends_on:
      - elasticsearch
    networks:
      - elk

volumes:
  elasticsearch_data:

networks:
  elk:
    driver: bridge
```

### Step 2: Logstash 설정

```yaml
# logstash/config/logstash.yml
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch:9200" ]
```

```ruby
# logstash/pipeline/logstash.conf
input {
  # TCP 입력 (JSON)
  tcp {
    port => 5000
    codec => json
  }

  # UDP 입력
  udp {
    port => 5000
    codec => json
  }

  # Beats 입력
  beats {
    port => 5044
  }
}

filter {
  # JSON 파싱
  if [message] =~ /^\{.*\}$/ {
    json {
      source => "message"
    }
  }

  # 타임스탬프 파싱
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
  }

  # User Agent 파싱
  if [user_agent] {
    useragent {
      source => "user_agent"
      target => "ua"
    }
  }

  # Grok 패턴 (Nginx 로그)
  if [type] == "nginx" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }

  # 필드 제거
  mutate {
    remove_field => [ "host", "port" ]
  }
}

output {
  # Elasticsearch로 전송
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }

  # 디버깅 (stdout)
  stdout {
    codec => rubydebug
  }
}
```

### Step 3: 실행 및 확인

```bash
# 실행
docker-compose -f docker-compose.elk.yml up -d

# 상태 확인
docker-compose -f docker-compose.elk.yml ps

# Elasticsearch 확인
curl http://localhost:9200
curl http://localhost:9200/_cat/indices?v

# Kibana 접속
# http://localhost:5601

# 테스트 로그 전송
echo '{"level":"info","message":"Test log","timestamp":"2024-01-15T10:00:00Z"}' | nc localhost 5000
```

---

## 🔧 실습 2: Application 로그 전송

### Step 1: Node.js Winston + Logstash

```bash
cd backend
npm install winston winston-logstash
```

```javascript
// backend/logger.js
const winston = require('winston');
const LogstashTransport = require('winston-logstash/lib/winston-logstash-latest');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'backend',
    environment: process.env.NODE_ENV || 'development'
  },
  transports: [
    // Console (개발)
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),

    // Logstash (프로덕션)
    new LogstashTransport({
      port: 5000,
      host: 'logstash',
      node_name: 'backend',
      max_connect_retries: -1
    })
  ]
});

module.exports = logger;
```

```javascript
// backend/server.js
const express = require('express');
const logger = require('./logger');

const app = express();

// Request logging middleware
app.use((req, res, next) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;

    logger.info('HTTP Request', {
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration: duration,
      user_agent: req.get('user-agent'),
      ip: req.ip
    });
  });

  next();
});

// API endpoints
app.get('/api/users', async (req, res) => {
  try {
    logger.info('Fetching users');
    const users = await db.query('SELECT * FROM users');
    res.json(users);
  } catch (err) {
    logger.error('Failed to fetch users', {
      error: err.message,
      stack: err.stack
    });
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.post('/api/users', async (req, res) => {
  try {
    const { username, email } = req.body;

    logger.info('Creating user', { username, email });

    const user = await db.query(
      'INSERT INTO users (username, email) VALUES ($1, $2) RETURNING *',
      [username, email]
    );

    logger.info('User created', { user_id: user.id });

    res.status(201).json(user);
  } catch (err) {
    logger.error('Failed to create user', {
      error: err.message,
      body: req.body
    });
    res.status(500).json({ error: 'Failed to create user' });
  }
});

// Uncaught exceptions
process.on('uncaughtException', (err) => {
  logger.error('Uncaught Exception', {
    error: err.message,
    stack: err.stack
  });
  process.exit(1);
});

app.listen(8080, () => {
  logger.info('Server started', { port: 8080 });
});
```

### Step 2: Docker Compose에 Backend 추가

```yaml
# docker-compose.elk.yml
services:
  # ... (ELK Stack)

  # Backend Application
  backend:
    build: ./backend
    container_name: backend
    restart: always
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=production
      - LOGSTASH_HOST=logstash
      - LOGSTASH_PORT=5000
    depends_on:
      - logstash
    networks:
      - elk
```

---

## 🔧 실습 3: Fluentd (EFK Stack)

### Step 1: Fluentd 설정

```yaml
# docker-compose.efk.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - efk

  # Fluentd
  fluentd:
    build: ./fluentd
    container_name: fluentd
    restart: always
    volumes:
      - ./fluentd/conf:/fluentd/etc
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    depends_on:
      - elasticsearch
    networks:
      - efk

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - efk

volumes:
  elasticsearch_data:

networks:
  efk:
```

```dockerfile
# fluentd/Dockerfile
FROM fluent/fluentd:v1.16-1

USER root

# Elasticsearch plugin 설치
RUN gem install fluent-plugin-elasticsearch

USER fluent
```

```xml
# fluentd/conf/fluent.conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

# Docker logs
<source>
  @type tail
  path /var/lib/docker/containers/*/*.log
  pos_file /fluentd/log/containers.log.pos
  tag docker.*
  format json
  time_key time
  time_format %Y-%m-%dT%H:%M:%S.%NZ
</source>

# Parser
<filter docker.**>
  @type parser
  key_name log
  <parse>
    @type json
    time_key timestamp
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</filter>

# Add hostname
<filter docker.**>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
  </record>
</filter>

# Elasticsearch output
<match docker.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix fluentd
  logstash_dateformat %Y.%m.%d
  include_tag_key true
  tag_key @log_name
  flush_interval 1s
  
  <buffer>
    @type file
    path /fluentd/log/buffer
    flush_mode interval
    retry_type exponential_backoff
    flush_interval 5s
    retry_forever
    retry_max_interval 30
  </buffer>
</match>

# Stdout (debugging)
<match **>
  @type stdout
</match>
```

### Step 2: Application에서 Fluentd 사용

```yaml
# docker-compose.efk.yml
services:
  backend:
    build: ./backend
    container_name: backend
    restart: always
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.backend
    ports:
      - "8080:8080"
    depends_on:
      - fluentd
    networks:
      - efk
```

---

## 🔧 실습 4: Kibana 대시보드 설정

### Step 1: Index Pattern 생성

```bash
# Kibana 접속
# http://localhost:5601

# 1. Management → Index Patterns
# 2. Create index pattern
# 3. Index pattern: logs-*
# 4. Time field: @timestamp
# 5. Create

# 2. Discover에서 로그 확인
# Discover → Select index pattern: logs-*
```

### Step 2: 검색 쿼리 (KQL)

```
# 기본 검색
level: "error"

# AND 조건
level: "error" AND service: "backend"

# OR 조건
level: "error" OR level: "warn"

# 범위
status_code >= 400 AND status_code < 500

# 존재 여부
user_id: *

# 정규 표현식
path: /api/users/*

# 시간 범위
@timestamp >= "2024-01-15" AND @timestamp <= "2024-01-16"

# 복합 쿼리
level: "error" AND service: "backend" AND NOT path: "/health"
```

### Step 3: Visualization 생성

```bash
# 1. Visualize → Create visualization
# 2. 차트 타입 선택:
#    - Line chart: 시간별 추이
#    - Bar chart: 비교
#    - Pie chart: 분포
#    - Metric: 단일 값

# 예: Error Rate Over Time
# - X-axis: Date Histogram (@timestamp)
# - Y-axis: Count
# - Filter: level: "error"

# 예: Top 10 Error Messages
# - Chart type: Bar
# - X-axis: Terms (message.keyword)
# - Y-axis: Count
# - Size: 10
```

### Step 4: Dashboard 생성

```bash
# 1. Dashboard → Create dashboard
# 2. Add panels (Visualizations)
# 3. Arrange and save

# 추천 Dashboard Panels:
# - Total Requests (Metric)
# - Requests Over Time (Line)
# - Error Rate (Line)
# - Top Endpoints (Bar)
# - Status Code Distribution (Pie)
# - Response Time (Line)
```

---

## 🔧 실습 5: Log Alerting

### Step 1: Watcher 설정 (Elasticsearch)

```json
// kibana/watcher/high-error-rate.json
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                },
                {
                  "term": {
                    "level": "error"
                  }
                }
              ]
            }
          },
          "aggs": {
            "error_count": {
              "value_count": {
                "field": "@timestamp"
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.aggregations.error_count.value": {
        "gt": 10
      }
    }
  },
  "actions": {
    "send_email": {
      "email": {
        "to": "admin@example.com",
        "subject": "High Error Rate Detected",
        "body": "Error count in last 5 minutes: {{ctx.payload.aggregations.error_count.value}}"
      }
    },
    "send_slack": {
      "webhook": {
        "scheme": "https",
        "host": "hooks.slack.com",
        "port": 443,
        "method": "post",
        "path": "/services/YOUR/WEBHOOK/URL",
        "params": {},
        "headers": {
          "Content-Type": "application/json"
        },
        "body": "{\"text\": \"⚠️ High Error Rate: {{ctx.payload.aggregations.error_count.value}} errors in 5 minutes\"}"
      }
    }
  }
}
```

### Step 2: Elastalert (대안)

```yaml
# elastalert/config.yaml
rules_folder: rules
run_every:
  minutes: 1
buffer_time:
  minutes: 15
es_host: elasticsearch
es_port: 9200
writeback_index: elastalert_status
alert_time_limit:
  days: 2
```

```yaml
# elastalert/rules/high_error_rate.yaml
name: High Error Rate
type: frequency
index: logs-*
num_events: 10
timeframe:
  minutes: 5

filter:
- term:
    level: "error"

alert:
- "email"
- "slack"

email:
- "admin@example.com"

slack_webhook_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
slack_username_override: "ElastAlert"
slack_emoji_override: ":warning:"
```

---

## 🔧 실습 6: 완전한 ELK Stack with Application

### Step 1: 전체 Docker Compose

```yaml
# docker-compose.production.yml
version: '3.8'

services:
  # Application
  backend:
    build: ./backend
    container_name: backend
    restart: always
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=production
      - LOGSTASH_HOST=logstash
      - LOGSTASH_PORT=5000
    depends_on:
      - postgres
      - logstash
    networks:
      - app
      - elk

  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app

  # ELK Stack
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    restart: always
    environment:
      - node.name=elasticsearch
      - cluster.name=docker-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    restart: always
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    ports:
      - "5000:5000/tcp"
      - "5000:5000/udp"
    environment:
      - "LS_JAVA_OPTS=-Xms512m -Xmx512m"
    depends_on:
      - elasticsearch
    networks:
      - elk

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    restart: always
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - elk

  # Nginx (Reverse Proxy)
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - backend
      - kibana
    networks:
      - app
      - elk

volumes:
  postgres_data:
  elasticsearch_data:

networks:
  app:
    driver: bridge
  elk:
    driver: bridge
```

### Step 2: Nginx 설정 (Kibana 프록시)

```nginx
# nginx/nginx.conf
server {
    listen 80;
    server_name localhost;

    # Application
    location / {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Kibana
    location /kibana/ {
        proxy_pass http://kibana:5601/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Basic Auth (선택)
        auth_basic "Kibana";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

---

## 💡 주요 패턴 정리

```
Log Levels:
- TRACE: 매우 상세한 디버깅
- DEBUG: 디버깅 정보
- INFO: 일반 정보 (기본)
- WARN: 경고
- ERROR: 에러
- FATAL: 치명적 에러

Structured Logging (JSON):
{
  "timestamp": "2024-01-15T10:00:00Z",
  "level": "INFO",
  "service": "backend",
  "message": "User created",
  "user_id": 123,
  "ip": "192.168.1.1",
  "duration": 45
}

Log Retention:
- Hot (검색): 7일
- Warm (보관): 30일
- Cold (아카이브): 1년+

Index Management:
- Daily: logs-2024.01.15
- Monthly: logs-2024.01
- ILM (Index Lifecycle Management)
```

---

## 📌 핵심 요약

```
Log Aggregation 핵심:
1. 중앙 집중식 로그
2. Structured Logging (JSON)
3. 실시간 검색 (Elasticsearch)
4. 시각화 (Kibana)
5. Alert 설정

ELK vs EFK:
- ELK: Logstash (무겁지만 강력)
- EFK: Fluentd (가볍고 유연)

Best Practices:
✅ JSON 로그 포맷
✅ 적절한 Log Level
✅ Context 정보 포함
✅ 민감 정보 제외
✅ Index Lifecycle 관리
```

---

## 📚 참고 자료

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Logstash Documentation](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Fluentd Documentation](https://docs.fluentd.org/)
- [Structured Logging Best Practices](https://www.datadoghq.com/blog/log-file-formats/)

---

## 🤔 생각해볼 문제

1. ELK Stack과 EFK Stack 중 어느 것을 선택해야 하는가?
2. 모든 로그를 저장해야 하는가?
3. Elasticsearch는 데이터베이스인가?

> 💡 **답변**:
> 
> **1) ELK vs EFK 선택:**
> 
> ```
> 둘 다 우수, 상황에 따라 선택
> 
> ELK (Elasticsearch + Logstash + Kibana):
> ✅ 강력한 데이터 변환 (Logstash)
> ✅ 복잡한 파싱
> ✅ 다양한 Input Plugin
> ✅ 성숙한 생태계
> 
> 단점:
> ❌ 무거움 (Java, JVM)
> ❌ 높은 메모리 (512MB+)
> ❌ 복잡한 설정
> 
> 사용 사례:
> - 복잡한 로그 파싱
> - 다양한 데이터 소스
> - 고급 변환 필요
> - 충분한 리소스
> 
> EFK (Elasticsearch + Fluentd + Kibana):
> ✅ 가벼움 (Ruby, C)
> ✅ 적은 메모리 (40MB)
> ✅ 간단한 설정
> ✅ 쿠버네티스 표준
> 
> 단점:
> ❌ 제한적 변환
> ❌ 적은 플러그인
> ❌ 러닝 커브 (다른 문법)
> 
> 사용 사례:
> - 쿠버네티스
> - 컨테이너 환경
> - 리소스 제한
> - 단순 로그 수집
> 
> 성능 비교:
> Logstash: 512MB RAM, 20K events/sec
> Fluentd:   40MB RAM, 15K events/sec
> 
> 실전 선택:
> 
> Kubernetes → Fluentd
> - CNCF 프로젝트
> - DaemonSet 최적화
> - 적은 오버헤드
> 
> 복잡한 파싱 → Logstash
> - Grok 패턴 (강력)
> - 풍부한 필터
> - 다양한 입력
> 
> Docker Swarm → Fluentd
> - Docker 로그 드라이버 네이티브
> - 가벼움
> 
> 결론:
> 쿠버네티스/간단 → Fluentd
> 복잡한 변환 → Logstash
> 성능 차이 크지 않음 (둘 다 우수)
> ```
> 
> **2) 모든 로그 저장?**
> 
> ```
> NO! 선택적으로
> 
> 문제:
> - 스토리지 폭발 (100GB/day)
> - 검색 느려짐
> - 비용 증가
> - 노이즈 많음
> 
> 로그 레벨 전략:
> 
> 프로덕션:
> ✅ ERROR (저장, 알림)
> ✅ WARN (저장)
> ⚠️ INFO (선택적)
> ❌ DEBUG (저장 안 함)
> ❌ TRACE (저장 안 함)
> 
> 개발:
> ✅ 모든 레벨
> 
> 샘플링:
> # 성공 요청 10%만 저장
> if status_code == 200:
>   if random() < 0.1:
>     log()
> 
> # 에러는 100% 저장
> if status_code >= 500:
>   log()
> 
> 필터링:
> # Health check 로그 제외
> filter {
>   if [path] == "/health" {
>     drop { }
>   }
> }
> 
> 보존 기간:
> - Hot (빠른 검색): 7일
> - Warm (보관): 30일
> - Cold (아카이브): 90일
> - 삭제: 90일 이상
> 
> Index Lifecycle:
> PUT _ilm/policy/logs_policy
> {
>   "policy": {
>     "phases": {
>       "hot": {
>         "actions": {}
>       },
>       "warm": {
>         "min_age": "7d",
>         "actions": {
>           "forcemerge": { "max_num_segments": 1 }
>         }
>       },
>       "cold": {
>         "min_age": "30d",
>         "actions": {
>           "freeze": {}
>         }
>       },
>       "delete": {
>         "min_age": "90d",
>         "actions": {
>           "delete": {}
>         }
>       }
>     }
>   }
> }
> 
> 비용 계산:
> 100GB/day × 30일 = 3TB
> AWS EBS: $300/month
> 
> 10GB/day × 30일 = 300GB
> AWS EBS: $30/month
> 
> → 10배 절감!
> 
> 결론:
> ERROR/WARN → 100% 저장
> INFO → 샘플링 (10-20%)
> DEBUG/TRACE → 개발만
> Health check → 제외
> 보존 기간 → 30-90일
> ```
> 
> **3) Elasticsearch는 DB?**
> 
> ```
> NO! Search Engine
> (하지만 유사점 있음)
> 
> Elasticsearch:
> ✅ Full-text Search (검색 엔진)
> ✅ 빠른 검색 (역인덱스)
> ✅ Analytics (집계)
> ✅ 시계열 데이터
> 
> ❌ 주 데이터베이스로 부적합
> ❌ ACID 트랜잭션 없음
> ❌ JOIN 제한적
> ❌ 데이터 정합성 약함
> ❌ 업데이트/삭제 비용 높음
> 
> vs 전통 DB:
> 
> PostgreSQL:
> - ACID 보장
> - Strong Consistency
> - 복잡한 쿼리 (JOIN)
> - 트랜잭션
> → Primary Database
> 
> Elasticsearch:
> - Eventually Consistent
> - Full-text Search
> - 로그/메트릭
> - 읽기 최적화
> → Search/Analytics
> 
> 올바른 사용:
> 
> ✅ 로그 검색
> ✅ 메트릭 저장
> ✅ 전문 검색
> ✅ 실시간 분석
> ✅ 시계열 데이터
> 
> ❌ 사용자 데이터 (DB 사용)
> ❌ 트랜잭션 (DB 사용)
> ❌ 금융 데이터 (DB 사용)
> ❌ 정합성 중요 (DB 사용)
> 
> 아키텍처:
> 
> App
>  ↓
> PostgreSQL (Primary)
>  ↓ (복제)
> Elasticsearch (Search)
> 
> → DB에 저장
> → ES에 인덱싱
> → ES에서 검색
> 
> 예:
> # 사용자 생성 → DB
> INSERT INTO users ...
> 
> # 검색용 인덱싱 → ES
> POST /users/_doc
> {
>   "name": "John",
>   "email": "john@example.com"
> }
> 
> # 검색 → ES
> GET /users/_search
> {
>   "query": {
>     "match": { "name": "John" }
>   }
> }
> 
> # 업데이트 → DB
> UPDATE users ...
> 
> 결론:
> Elasticsearch = Search Engine
> 로그/검색/분석용
> Primary DB로 부적합
> DB + ES 조합이 Best
> ```


---

<div align="center">

**[⬅️ 이전: Monitoring Stack](./04-Monitoring-Stack.md)** | **[다음: Backup System ➡️](./06-Backup-System.md)**

</div>
