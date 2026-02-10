# 05. Monitoring - 성능 모니터링

## 🎯 이 챕터에서 배울 것

- **Prometheus** - 메트릭 수집 및 저장
- **cAdvisor** - 컨테이너 메트릭 수집
- **Grafana** - 시각화 대시보드
- **Node Exporter** - 호스트 메트릭
- **알림 설정** - Alertmanager 통합

## 📌 왜 중요한가?

**"모니터링은 문제를 조기에 발견하고 성능을 최적화하는 핵심입니다."**

```
모니터링 없이 vs 모니터링 적용:

모니터링 없는 환경:
┌─────────────────────────────────────┐
│ 09:00 - 서비스 정상                    │
│ CPU: ??, Memory: ??                 │
│ 사용자: 정상                           │
└────────────┬────────────────────────┘
             │ (시간 경과 - 모름)
┌────────────▼────────────────────────┐
│ 15:00 - 사용자 불만 접수                │
│ "서비스가 느려요!"                      │
│ → 원인 파악 시작 (수동)                 │
│ → 로그 확인 (30분)                     │
│ → 리소스 확인 (추측)                    │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 16:00 - 원인 발견                     │
│ Memory: 90% (OOM 직전)               │
│ → 재시작 (5분 다운타임)                 │
│ → 근본 원인 미파악                      │
└─────────────────────────────────────┘

문제:
❌ 사후 대응 (Reactive)
❌ 긴 MTTD (Mean Time To Detect)
❌ 근본 원인 분석 어려움
❌ 비즈니스 영향 큼

모니터링 적용:
┌─────────────────────────────────────┐
│ 09:00 - 서비스 정상                    │
│ CPU: 20%, Memory: 40%               │
│ Dashboard: 모두 녹색                  │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 12:00 - Memory 증가 탐지              │
│ Memory: 60% → 70% → 80%             │
│ → Alert: Warning (Slack)            │
│ → 그래프: 선형 증가 패턴                 │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 12:05 - 자동 조치                     │
│ Playbook: Memory 80% 이상            │
│ → 컨테이너 재시작                       │
│ → 로그 수집                           │
│ → 근본 원인 분석 (메모리 누수)            │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 12:10 - 정상화                        │
│ Memory: 40% (재시작 완료)              │
│ 사용자 영향: 없음                       │
│ 근본 원인: 코드 수정 예정                 │
└─────────────────────────────────────┘

장점:
✅ 사전 대응 (Proactive)
✅ 짧은 MTTD (5분)
✅ 자동화된 대응
✅ 비즈니스 영향 최소화

모니터링 핵심 개념:

1. Metrics (메트릭):
   
   4가지 황금 신호 (Golden Signals):
   ┌──────────────────────────────┐
   │ 1. Latency (지연시간)          │
   │    P50, P95, P99             │
   │                              │
   │ 2. Traffic (트래픽)            │
   │    Requests/sec              │
   │                              │
   │ 3. Errors (에러율)             │
   │    Error rate %              │
   │                              │
   │ 4. Saturation (포화도)         │
   │    CPU, Memory 사용률          │
   └──────────────────────────────┘

2. 모니터링 스택:
   
   ┌──────────────────────────────┐
   │ Grafana (시각화)               │
   │ - Dashboard                  │
   │ - 알림 설정                    │
   └──────────┬───────────────────┘
              │
   ┌──────────▼───────────────────┐
   │ Prometheus (메트릭 저장)        │
   │ - Time series DB             │
   │ - PromQL                     │
   └──────────┬───────────────────┘
              │
   ┌──────────▼───────────────────┐
   │ Exporters (메트릭 수집)         │
   │ - cAdvisor (컨테이너)          │
   │ - Node Exporter (호스트)      │
   │ - App Exporter (앱)          │
   └──────────────────────────────┘

3. 알림 계층:
   
   ┌──────────────────────────────┐
   │ P1: Critical                 │
   │ - Service Down               │
   │ - PagerDuty (즉시)            │
   │ - 5분 내 대응                  │
   └──────────────────────────────┘
   
   ┌──────────────────────────────┐
   │ P2: Warning                  │
   │ - High CPU/Memory            │
   │ - Slack (15분 내)             │
   │ - 1시간 내 대응                 │
   └──────────────────────────────┘
   
   ┌──────────────────────────────┐
   │ P3: Info                     │
   │ - Deployment                 │
   │ - Email (일일)                │
   │ - 참고용                       │
   └──────────────────────────────┘

4. 대시보드 구성:
   
   Overview:
   - Service Health (UP/DOWN)
   - Request Rate
   - Error Rate
   - Latency (P95, P99)
   
   Resource:
   - CPU Usage (per container)
   - Memory Usage
   - Disk I/O
   - Network I/O
   
   Business:
   - Active Users
   - Transactions/sec
   - Revenue/hour
   - Conversion Rate

실무 시나리오:

시나리오 1 - 메모리 누수 탐지:
┌─────────────────────────────────────┐
│ Grafana Dashboard                   │
│                                     │
│ Memory Usage (7 days):              │
│   ▲                                 │
│   │              ┌──                │
│ 8G│         ┌────┘                  │
│   │    ┌────┘                       │
│ 4G│ ┌──┘                            │
│   └──────────────────────→          │
│        Time                         │
│                                     │
│ Alert: Memory > 80% (Day 6)         │
└─────────────────────────────────────┘

대응:
1. 알림 수신 (Slack)
2. 그래프 확인 → 선형 증가
3. 메모리 프로파일링
4. 코드 수정
5. 배포 후 확인

시나리오 2 - 레이턴시 스파이크:
┌─────────────────────────────────────┐
│ Latency P99 (1 hour):               │
│   ▲                                 │
│   │    ┌─┐                          │
│500│    │ │                          │
│ms │    │ │                          │
│   │────┘ └────────                  │
│ 0 └──────────────────────→          │
│        10:00  10:30                 │
│                                     │
│ Event: Deploy at 10:15              │
└─────────────────────────────────────┘

분석:
- 10:15 배포 → 레이턴시 증가
- 코드 문제 또는 DB 쿼리
- 롤백 후 정상화

시나리오 3 - CPU Throttling:
┌─────────────────────────────────────┐
│ CPU Usage vs Throttling:            │
│   ▲                                 │
│100│████████████████████             │
│ % │████████████████████             │
│   │                                 │
│   └──────────────────────→          │
│                                     │
│ Throttled: 45%                      │
│ → CPU limit 증가 필요                 │
└─────────────────────────────────────┘
```

---

## 🔧 실습 1: Prometheus 설치

### Step 1: Prometheus 실행

```bash
# Prometheus 설정 파일
cat > prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
EOF

# Prometheus 실행
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus:latest

# 웹 UI 접속
# http://localhost:9090

# Status → Targets에서 수집 대상 확인
```

### Step 2: cAdvisor 실행

```bash
# cAdvisor (컨테이너 메트릭)
docker run -d \
  --name cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  gcr.io/cadvisor/cadvisor:latest

# 웹 UI 접속
# http://localhost:8080

# 메트릭 확인
curl http://localhost:8080/metrics | grep container_cpu

# 출력:
# container_cpu_usage_seconds_total{container_label_com_docker_compose_service="web"...} 123.45
```

### Step 3: Node Exporter 실행

```bash
# Node Exporter (호스트 메트릭)
docker run -d \
  --name node-exporter \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host

# 메트릭 확인
curl http://localhost:9100/metrics | grep node_cpu

# 출력:
# node_cpu_seconds_total{cpu="0",mode="idle"} 123456.78
# node_cpu_seconds_total{cpu="0",mode="system"} 1234.56
```

---

## 🔧 실습 2: PromQL 쿼리

### Step 1: 기본 쿼리

```promql
# CPU 사용률 (전체)
rate(container_cpu_usage_seconds_total[5m])

# 메모리 사용량
container_memory_usage_bytes

# 메모리 사용률 (%)
(container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100

# 네트워크 수신 속도
rate(container_network_receive_bytes_total[5m])

# 디스크 I/O
rate(container_fs_writes_bytes_total[5m])
```

### Step 2: 컨테이너별 쿼리

```promql
# 특정 컨테이너 CPU
rate(container_cpu_usage_seconds_total{name="web"}[5m])

# 서비스별 평균 메모리
avg(container_memory_usage_bytes) by (container_label_com_docker_compose_service)

# 최대 CPU 사용 컨테이너
topk(5, rate(container_cpu_usage_seconds_total[5m]))

# CPU Throttling
rate(container_cpu_cfs_throttled_seconds_total[5m])
```

### Step 3: 집계 및 계산

```promql
# 전체 컨테이너 CPU 합계
sum(rate(container_cpu_usage_seconds_total[5m]))

# 평균 메모리 사용률
avg(container_memory_usage_bytes / container_spec_memory_limit_bytes)

# P95 레이턴시 (애플리케이션 메트릭)
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m])
)

# 에러율
sum(rate(http_requests_total{status=~"5.."}[5m])) / 
sum(rate(http_requests_total[5m]))
```

---

## 🔧 실습 3: Grafana 대시보드

### Step 1: Grafana 설치

```bash
# Grafana 실행
docker run -d \
  --name grafana \
  -p 3000:3000 \
  grafana/grafana:latest

# 웹 UI 접속
# http://localhost:3000
# 기본 로그인: admin / admin

# Data Source 추가
# Configuration → Data Sources → Add data source
# Prometheus
# URL: http://prometheus:9090
```

### Step 2: 대시보드 생성

```bash
# Dashboard → New Dashboard → Add Panel

# Panel 1: CPU Usage
# Query:
rate(container_cpu_usage_seconds_total{name!=""}[5m])

# Legend: {{name}}
# Y-axis: CPU Cores
# Visualization: Time series

# Panel 2: Memory Usage
# Query:
container_memory_usage_bytes{name!=""}

# Legend: {{name}}
# Y-axis: Bytes (Auto)
# Visualization: Time series

# Panel 3: Memory Usage %
# Query:
(container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100

# Legend: {{name}}
# Y-axis: Percent (0-100)
# Visualization: Gauge

# Panel 4: Network I/O
# Query (Receive):
rate(container_network_receive_bytes_total{name!=""}[5m])

# Query (Transmit):
rate(container_network_transmit_bytes_total{name!=""}[5m])

# Legend: {{name}} - {{interface}}
# Y-axis: Bytes/sec
```

### Step 3: 사전 제작 대시보드 import

```bash
# Docker 모니터링 대시보드
# Grafana UI → Dashboards → Import
# Dashboard ID: 893 (cAdvisor)
# https://grafana.com/grafana/dashboards/893

# 또는 JSON 다운로드
curl -o docker-dashboard.json \
  https://grafana.com/api/dashboards/893/revisions/latest/download

# Import JSON
# Grafana UI → Dashboards → Import → Upload JSON file
```

---

## 🔧 실습 4: 알림 설정

### Step 1: Alertmanager 설치

```bash
# Alertmanager 설정
cat > alertmanager.yml <<'EOF'
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'slack'

receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
EOF

# Alertmanager 실행
docker run -d \
  --name alertmanager \
  -p 9093:9093 \
  -v $(pwd)/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  prom/alertmanager:latest
```

### Step 2: Prometheus 알림 규칙

```bash
# Alert 규칙 파일
cat > alert.rules.yml <<'EOF'
groups:
  - name: container_alerts
    interval: 30s
    rules:
      # High CPU Alert
      - alert: HighCPUUsage
        expr: rate(container_cpu_usage_seconds_total{name!=""}[5m]) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.name }}"
          description: "Container {{ $labels.name }} CPU usage is {{ $value | humanize }}%"

      # High Memory Alert
      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.name }}"
          description: "Container {{ $labels.name }} memory usage is {{ $value | humanize }}%"

      # Container Down
      - alert: ContainerDown
        expr: up{job="cadvisor"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "cAdvisor is down"
          description: "cAdvisor has been down for more than 1 minute"

      # High Disk I/O
      - alert: HighDiskIO
        expr: rate(container_fs_writes_bytes_total[5m]) > 100000000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High disk I/O on {{ $labels.name }}"
          description: "Container {{ $labels.name }} disk write is {{ $value | humanize }}B/s"

      # OOM Killed
      - alert: OOMKilled
        expr: increase(container_memory_failcnt[5m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Container {{ $labels.name }} OOM killed"
          description: "Container has been OOM killed {{ $value }} times in the last 5 minutes"
EOF

# Prometheus 설정 업데이트
cat > prometheus.yml <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'alert.rules.yml'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
EOF

# Prometheus 재시작
docker restart prometheus
```

### Step 3: 알림 테스트

```bash
# CPU 부하 생성 (알림 트리거)
docker run -d --name cpu-stress \
  --cpus=0.5 \
  alpine sh -c 'while true; do :; done'

# 5분 후 Prometheus Alerts 확인
# http://localhost:9090/alerts

# Alertmanager 확인
# http://localhost:9093

# Slack 알림 확인
# #alerts 채널에 메시지 수신

# 정리
docker rm -f cpu-stress
```

---

## 🔧 실습 5: Docker Compose 통합

### Step 1: 전체 스택 구성

```yaml
# docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert.rules.yml:/etc/prometheus/alert.rules.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: always

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - 9093:9093
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    restart: always

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - 3000:3000
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: always

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - 8080:8080
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    devices:
      - /dev/kmsg
    restart: always

  node-exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node-exporter
    command:
      - '--path.rootfs=/host'
    network_mode: host
    pid: host
    volumes:
      - '/:/host:ro,rslave'
    restart: always

volumes:
  prometheus-data:
  grafana-data:
```

### Step 2: 실행 및 확인

```bash
# 전체 스택 시작
docker-compose up -d

# 상태 확인
docker-compose ps

# 출력:
# NAME           STATUS   PORTS
# prometheus     Up       0.0.0.0:9090->9090/tcp
# alertmanager   Up       0.0.0.0:9093->9093/tcp
# grafana        Up       0.0.0.0:3000->3000/tcp
# cadvisor       Up       0.0.0.0:8080->8080/tcp
# node-exporter  Up       host network

# 로그 확인
docker-compose logs -f prometheus

# 접속 테스트
curl http://localhost:9090/-/healthy
# Prometheus is Healthy.

curl http://localhost:3000/api/health
# {"database":"ok","version":"..."}
```

---

## 💡 주요 명령어 정리

### Prometheus

```bash
# 설치
docker run -d -p 9090:9090 prom/prometheus

# 쿼리 (PromQL)
rate(container_cpu_usage_seconds_total[5m])
container_memory_usage_bytes

# API 호출
curl 'http://localhost:9090/api/v1/query?query=up'
```

### Grafana

```bash
# 설치
docker run -d -p 3000:3000 grafana/grafana

# Data Source 추가 (CLI)
curl -X POST -H "Content-Type: application/json" \
  -d '{"name":"Prometheus","type":"prometheus","url":"http://prometheus:9090"}' \
  http://admin:admin@localhost:3000/api/datasources
```

### cAdvisor

```bash
# 설치
docker run -d -p 8080:8080 \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  gcr.io/cadvisor/cadvisor

# 메트릭 확인
curl http://localhost:8080/metrics
```

---

## 🎓 연습 문제

### 문제 1: CPU 알림 규칙

CPU 사용률이 90% 이상 10분 지속 시 알림을 보내는 규칙을 작성하세요.

<details>
<summary>정답 보기</summary>

```yaml
# alert.rules.yml
groups:
  - name: cpu_alerts
    interval: 30s
    rules:
      - alert: HighCPUUsage90
        expr: rate(container_cpu_usage_seconds_total{name!=""}[5m]) > 0.9
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Critical CPU usage on {{ $labels.name }}"
          description: "Container {{ $labels.name }} CPU has been above 90% for 10 minutes (current: {{ $value | humanizePercentage }})"
```

</details>

### 문제 2: 메모리 사용량 대시보드

컨테이너별 메모리 사용률(%)을 Gauge로 표시하는 Grafana 패널을 만드세요.

<details>
<summary>정답 보기</summary>

```
Panel Settings:
- Visualization: Gauge
- Query:
  (container_memory_usage_bytes{name!=""} / 
   container_spec_memory_limit_bytes) * 100
- Legend: {{name}}
- Thresholds:
  - Green: 0-70
  - Yellow: 70-85
  - Red: 85-100
- Unit: Percent (0-100)
```

</details>

### 문제 3: Disk I/O 모니터링

1시간 동안의 Disk Write 총량을 계산하는 PromQL을 작성하세요.

<details>
<summary>정답 보기</summary>

```promql
# 1시간 동안의 총 Write 바이트
increase(container_fs_writes_bytes_total[1h])

# 컨테이너별 합계
sum(increase(container_fs_writes_bytes_total[1h])) by (name)

# Human readable
# Panel에서 Unit을 "bytes" 또는 "decbytes"로 설정
```

</details>

---

## 📌 핵심 요약

### 모니터링 스택

```
┌─────────────────────────────┐
│ Grafana (Port 3000)         │
│ - Dashboard                 │
│ - Visualization             │
└──────────┬──────────────────┘
           │
┌──────────▼──────────────────┐
│ Prometheus (Port 9090)      │
│ - Metrics Storage           │
│ - PromQL                    │
│ - Alerting                  │
└──────────┬──────────────────┘
           │
┌──────────▼──────────────────┐
│ Exporters                   │
│ - cAdvisor (8080)           │
│ - Node Exporter (9100)      │
│ - App Exporter (custom)     │
└─────────────────────────────┘
```

### 주요 메트릭

| 카테고리 | 메트릭 | 쿼리 |
|---------|--------|------|
| **CPU** | 사용률 | `rate(container_cpu_usage_seconds_total[5m])` |
| **Memory** | 사용량 | `container_memory_usage_bytes` |
| **Memory** | 사용률 | `(usage / limit) * 100` |
| **Network** | RX | `rate(container_network_receive_bytes_total[5m])` |
| **Disk** | Write | `rate(container_fs_writes_bytes_total[5m])` |

### 알림 레벨

```yaml
Critical (P1):
  - Service Down
  - OOM Killed
  - Response: 즉시
  - Channel: PagerDuty

Warning (P2):
  - High CPU/Memory (80%+)
  - Response: 1시간 내
  - Channel: Slack

Info (P3):
  - Deployment
  - Scale Event
  - Response: 참고
  - Channel: Email
```

### 대시보드 구성

```
Overview:
- Service Status
- Request Rate
- Error Rate
- P95/P99 Latency

Resources:
- CPU per Container
- Memory per Container
- Disk I/O
- Network I/O

Business:
- Active Users
- Transaction Rate
- Revenue
```

---

## 📚 참고 자료

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)
- [cAdvisor](https://github.com/google/cadvisor)
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)

---

## 🤔 생각해볼 문제

1. Prometheus는 왜 Pull 방식을 사용할까?
2. 메트릭 수집 간격을 어떻게 정해야 할까?
3. 알림의 반복 전송은 왜 필요할까?

> 💡 **답변**: 1) Pull vs Push 비교, Pull (Prometheus): 중앙에서 주기적으로 가져옴, 장점: Service Discovery 가능, 타겟 상태 파악 (Up/Down), 중앙에서 제어, 스크랩 실패 감지, 단점: 방화벽 통과 어려움, 네트워크 연결 필요, Push (StatsD 등): 타겟에서 중앙으로 전송, 장점: 방화벽 친화적, 일시적 작업 측정 가능, 단점: 타겟 상태 모름, 중앙 과부하 가능, Prometheus 선택 이유: 타겟 Health 체크, Scrape 실패 = Alert, 일관된 수집 간격, 2) 수집 간격 결정 요소, 리소스 변화 속도, CPU: 빠름 → 15s, Disk: 느림 → 60s, 데이터 정밀도, 고정밀: 5-10s, 일반: 15-30s, 저장 공간, 짧을수록 더 많은 저장, 15s → 1년에 2M 포인트, Query 성능, 데이터 많을수록 느림, 권장: 기본 15s, CPU/메모리: 15s, Disk/네트워크: 30s, 비즈니스: 60s, 3) 반복 전송 필요성, 알림 놓침 방지, Slack 확인 안 함, 첫 알림 무시, 상황 악화 알림, 문제 지속 중, 해결 안 됨, 교대 근무, 다음 담당자에게, 패턴: repeat_interval: 12h (기본), Critical: 1h, Warning: 12h, Info: 무한 (한 번만)

---

<div align="center">

**[⬅️ 이전: I/O Performance](./04-IO-Performance.md)** | **[다음: Logging ➡️](./06-Logging.md)**

</div>
