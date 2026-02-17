# 04. Monitoring Stack - Prometheus + Grafana

## 🎯 이 챕터에서 배울 것

- **Prometheus 설정**: 메트릭 수집
- **Grafana 대시보드**: 시각화  
- **Exporters**: Node, cAdvisor, Application
- **Alerting**: AlertManager
- **실전 구성**: Production-ready 모니터링
- **커스텀 메트릭**: 애플리케이션 메트릭

## 📌 왜 중요한가?

**"모니터링 없이는 시스템 상태를 알 수 없고, 문제를 예방할 수 없습니다."**

```
Architecture:
┌─────────────────────────────────────────────────┐
│  ┌──────────┐   ┌──────────┐   ┌──────────┐     │
│  │ Backend  │   │Frontend  │   │Database  │     │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘     │
│       │              │              │           │
│       └──────────────┴──────────────┘           │
│                     │ (Metrics)                 │
│              ┌─────────────┐                    │
│              │ Prometheus  │                    │
│              └──────┬──────┘                    │
│                     │ (Query)                   │
│              ┌─────────────┐                    │
│              │  Grafana    │                    │
│              └─────────────┘                    │
└─────────────────────────────────────────────────┘
```

---

## 🔧 실습 1: 기본 모니터링 스택

### Docker Compose

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    privileged: true
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
```

### Prometheus 설정

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'backend'
    static_configs:
      - targets: ['backend:8080']
    metrics_path: '/metrics'
```

---

## 🔧 실습 2: 애플리케이션 메트릭

### Node.js Metrics

```javascript
// backend/metrics.js
const client = require('prom-client');

const register = new client.Registry();

// Default metrics
client.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5]
});

const httpRequestTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestTotal);

module.exports = { register, httpRequestDuration, httpRequestTotal };
```

```javascript
// backend/server.js
const express = require('express');
const { register, httpRequestDuration, httpRequestTotal } = require('./metrics');

const app = express();

// Metrics middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route ? req.route.path : req.path;
    
    httpRequestDuration.labels(req.method, route, res.statusCode).observe(duration);
    httpRequestTotal.labels(req.method, route, res.statusCode).inc();
  });
  
  next();
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(8080);
```

---

## 🔧 실습 3: AlertManager

### AlertManager 설정

```yaml
# alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'slack'

receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
```

### Alert Rules

```yaml
# prometheus/rules/alerts.yml
groups:
  - name: system
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU > 80%"

      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning

  - name: application
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100 > 5
        for: 5m
        labels:
          severity: critical
```

---

## 📌 핵심 요약

```
Monitoring 핵심:
1. Prometheus (수집)
2. Grafana (시각화)
3. Exporters (데이터)
4. AlertManager (알림)
5. Custom Metrics

Best Practices:
✅ 4 Golden Signals
✅ Alert 최소화
✅ 대시보드 자동화
✅ SLO 기반
```

---

## 📚 참고 자료

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Node Exporter](https://github.com/prometheus/node_exporter)
- [cAdvisor](https://github.com/google/cadvisor)
- [PromQL Tutorial](https://prometheus.io/docs/prometheus/latest/querying/basics/)

---

## 🤔 생각해볼 문제

1. 모든 메트릭을 수집해야 하는가?
2. Prometheus만으로 충분한가?
3. Alert는 얼마나 많이 설정해야 하는가?

> 💡 **답변**:
> 
> **1) 모든 메트릭 수집?**
> 
> ```
> NO! 필요한 것만 선택적으로
> 
> 문제:
> - 스토리지 폭발 (TB급)
> - 쿼리 느려짐
> - 비용 증가
> - 노이즈 증가
> 
> 4 Golden Signals (Google SRE):
> ✅ Latency (지연 시간)
> ✅ Traffic (트래픽)
> ✅ Errors (에러율)
> ✅ Saturation (포화도)
> 
> 우선순위:
> 
> P0 (필수):
> - CPU, Memory, Disk
> - Request Rate
> - Error Rate
> - Response Time (p95, p99)
> 
> P1 (중요):
> - Network I/O
> - Container Restarts
> - Database Connections
> - Queue Depth
> 
> P2 (선택):
> - 세부 비즈니스 메트릭
> - 사용자 행동
> - A/B 테스트 결과
> 
> 결론:
> 4 Golden Signals + 핵심 인프라
> = 80% 문제 발견 가능
> ```
> 
> **2) Prometheus만으로?**
> 
> ```
> Short-term: YES
> Long-term: NO
> 
> Prometheus 장점:
> ✅ 메트릭 수집
> ✅ 단기 저장 (15-30일)
> ✅ PromQL (강력)
> ✅ Alerting
> 
> Prometheus 한계:
> ❌ 장기 저장 (> 30일)
> ❌ 고가용성 (단일 노드)
> ❌ 수평 확장 어려움
> ❌ 로그 수집 없음
> ❌ 트레이싱 없음
> 
> 완전한 Observability:
> 
> 1. Metrics (Prometheus)
>    - 무엇이 일어나는가?
>    - CPU, Memory, Request Rate
> 
> 2. Logs (ELK/Loki)
>    - 왜 일어났는가?
>    - Error Stack Trace
>    - 상세 이벤트
> 
> 3. Traces (Jaeger/Zipkin)
>    - 어떻게 흘러갔는가?
>    - Request Flow
>    - Latency 병목
> 
> 확장 옵션:
> 
> Thanos:
> - Prometheus 장기 저장
> - 글로벌 쿼리
> - 다중 클러스터
> 
> Cortex:
> - Prometheus-as-a-Service
> - 멀티테넌시
> - 수평 확장
> 
> Managed:
> - Datadog
> - New Relic
> - Grafana Cloud
> 
> 결론:
> 시작: Prometheus
> 성장: Prometheus + ELK
> 대규모: Thanos + ELK + Jaeger
> ```
> 
> **3) Alert 개수?**
> 
> ```
> 적을수록 좋음!
> 
> Alert Fatigue:
> 너무 많음 → 무시함 → 실제 장애 놓침
> 
> 원칙:
> 
> 1. Actionable (실행 가능)
>    ✅ CPU > 80% for 5m → Scale up
>    ❌ CPU > 50% → 뭘 하지?
> 
> 2. High Impact (영향 큼)
>    ✅ Service Down
>    ❌ Disk 60% (여유 있음)
> 
> 3. Urgent (긴급)
>    ✅ Error Rate > 10%
>    ❌ Disk 80% (며칠 여유)
> 
> 4. User-Facing (사용자 영향)
>    ✅ API Response Time > 2s
>    ❌ Background Job Slow
> 
> Alert 계층:
> 
> P0 (즉시 대응):
> - Service Down
> - Error Rate > 10%
> - Latency p99 > 5s
> → PagerDuty (24/7)
> 
> P1 (업무 시간):
> - Disk > 85%
> - Memory > 90%
> - CPU > 80% for 30m
> → Slack
> 
> P2 (모니터링):
> - Disk > 70%
> - Slow Queries
> - Warning Logs
> → Email (일간 요약)
> 
> 숫자:
> 소규모: 5-10개 Alert
> 중규모: 10-20개
> 대규모: 20-50개
> 
> → 더 많으면 통합 필요
> 
> Best Practice:
> ✅ SLO 기반 Alert
> ✅ 증상 Alert (원인 X)
> ✅ Runbook 첨부
> ✅ 정기 리뷰 (불필요한 것 삭제)
> 
> 결론:
> 적을수록 좋음 (< 20개)
> 실행 가능 + 영향 큼 + 긴급
> ```


---

<div align="center">

**[⬅️ 이전: Reverse Proxy](./03-Reverse-Proxy.md)** | **[다음: Log Aggregation ➡️](./05-Log-Aggregation.md)**

</div>
