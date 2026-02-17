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
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   │
│  │ Backend  │   │Frontend  │   │Database  │   │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   │
│       │              │              │          │
│       └──────────────┴──────────────┘          │
│                     │ (Metrics)                │
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

<div align="center">

**[⬅️ 이전: Reverse Proxy](./03-Reverse-Proxy.md)** | **[다음: Log Aggregation ➡️](./05-Log-Aggregation.md)**

</div>
