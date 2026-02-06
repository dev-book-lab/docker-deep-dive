# 07. Storage Best Practices - 스토리지 모범 사례

## 🎯 이 챕터에서 배울 것

- Docker 스토리지 **설계 원칙**
- **성능 최적화** 기법
- **보안** 모범 사례
- **모니터링**과 **트러블슈팅**

## 📌 왜 중요한가?

**"올바른 스토리지 전략은 애플리케이션 성능, 안정성, 보안을 결정합니다."**

```
스토리지 설계 결정이 미치는 영향:

성능:
┌─────────────────────────────────────┐
│ 잘못된 선택:                           │
│ - Bind mount + 느린 디스크             │
│ - 부적절한 Storage Driver             │
│ - 과도한 레이어                        │
│ → IOPS 저하, 레이턴시 증가              │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ 올바른 선택:                           │
│ - Named volume + SSD                │
│ - overlay2 (Linux)                  │
│ - 최소 레이어 이미지                    │
│ → 최적 성능                           │
└─────────────────────────────────────┘

안정성:
❌ 백업 없음 → 데이터 손실 위험
❌ 단일 볼륨 → 단일 실패점
❌ 모니터링 없음 → 디스크 가득 참

✅ 정기 백업 → 복구 가능
✅ 복제/고가용성 → 장애 대응
✅ 알림 설정 → 사전 대응

보안:
❌ 권한 관리 없음 → 데이터 유출
❌ 암호화 없음 → 민감 정보 노출
❌ 감사 로그 없음 → 추적 불가

✅ 최소 권한 원칙 → 접근 제어
✅ 저장 데이터 암호화 → 보호
✅ 감사 로그 → 컴플라이언스

비용:
잘못된 관리 → 불필요한 디스크 사용
올바른 정리 → 비용 절감
```

**실무 원칙:**
- 데이터 중요도에 따른 전략
- 워크로드별 최적화
- 자동화된 관리
- 지속적 모니터링

---

## 🔬 Deep Dive

### 1. 설계 원칙

#### 원칙 1: 데이터 분류

```
데이터 분류 체계:

Critical (중요):
- 데이터베이스 데이터
- 사용자 업로드
- 트랜잭션 로그
→ 전략:
  - Named volume
  - 고성능 스토리지 (SSD/NVMe)
  - 복제/백업
  - 암호화

Important (중요도 중):
- 애플리케이션 로그
- 캐시 데이터
- 세션 정보
→ 전략:
  - Named volume
  - 일반 스토리지
  - 주기적 정리

Temporary (임시):
- 빌드 캐시
- 임시 파일
- 테스트 데이터
→ 전략:
  - Tmpfs
  - 자동 정리
  - 백업 불필요

실제 적용:
```

```yaml
version: '3.8'

services:
  app:
    image: myapp
    volumes:
      # Critical: 데이터베이스
      - type: volume
        source: db-data
        target: /var/lib/postgresql/data
        volume:
          nocopy: true
      
      # Important: 로그
      - type: volume
        source: app-logs
        target: /var/log/app
      
      # Temporary: 캐시
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1G

volumes:
  # Critical: SSD, 백업, 복제
  db-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ssd/db-data
  
  # Important: 일반 디스크
  app-logs:
    driver: local
```

#### 원칙 2: 분리의 원칙 (Separation of Concerns)

```
계층별 분리:

Application Layer:
- 이미지 레이어
- Storage Driver 관리
- 읽기 전용

Data Layer:
- Named volumes
- 읽기/쓰기
- 영속성

Configuration Layer:
- Bind mounts
- 읽기 전용 권장
- 버전 관리

예시:
```

```yaml
version: '3.8'

services:
  web:
    image: nginx:alpine
    volumes:
      # Configuration (bind mount, read-only)
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
      
      # Data (named volume, read-write)
      - static-files:/usr/share/nginx/html
      - uploads:/var/www/uploads
      
      # Logs (named volume)
      - nginx-logs:/var/log/nginx

volumes:
  static-files:
  uploads:
  nginx-logs:
```

#### 원칙 3: 최소 권한 (Least Privilege)

```bash
# ❌ 모든 것에 root 권한
docker run -v /:/host ubuntu

# ✅ 필요한 것만, 읽기 전용
docker run -v /etc/hostname:/etc/hostname:ro ubuntu

# ✅ 특정 디렉토리만
docker run -v my-data:/data ubuntu

# ✅ 비특권 사용자
docker run --user 1000:1000 -v my-data:/data ubuntu
```

---

### 2. 성능 최적화

#### 최적화 1: Storage Driver 선택

```bash
# 현재 드라이버 확인
docker info | grep "Storage Driver"
# Storage Driver: overlay2

# Linux에서 최적 설정
cat > /etc/docker/daemon.json << EOF
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

sudo systemctl restart docker
```

**드라이버별 성능:**

```
┌────────────┬──────────┬──────────┬────────────┐
│ 워크로드     │ 권장      │ 대안      │ 피하기       │
├────────────┼──────────┼──────────┼────────────┤
│ 일반 앱      │ overlay2 │ btrfs    │devicemapper│
├────────────┼──────────┼──────────┼────────────┤
│ 대용량 쓰기   │ zfs      │ btrfs    │ overlay2   │
├────────────┼──────────┼──────────┼────────────┤
│ 많은 레이어   │ overlay2 │ zfs      │devicemapper│
├────────────┼──────────┼──────────┼────────────┤
│ 고IOPS DB   │ 로컬볼륨   │ zfs      │ NFS        │
└────────────┴──────────┴──────────┴────────────┘
```

#### 최적화 2: 볼륨 유형 선택

```yaml
version: '3.8'

services:
  # 고성능 DB
  database:
    image: postgres:14
    volumes:
      # Named volume on SSD
      - type: volume
        source: db-data
        target: /var/lib/postgresql/data
        volume:
          nocopy: true  # 복사 오버헤드 제거

  # 캐시
  redis:
    image: redis:alpine
    volumes:
      # Tmpfs (메모리)
      - type: tmpfs
        target: /data
        tmpfs:
          size: 2G

  # 정적 파일
  web:
    image: nginx:alpine
    volumes:
      # Bind mount (개발)
      - type: bind
        source: ./html
        target: /usr/share/nginx/html
        read_only: true

volumes:
  db-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/nvme/postgres  # 고성능 디스크
```

**성능 특성:**

```
┌──────────────┬─────────────┬─────────────┬──────────┐
│ 유형          │ 읽기         │ 쓰기         │ 용도      │
├──────────────┼─────────────┼─────────────┼──────────┤
│ Tmpfs        │ ⭐⭐⭐⭐⭐ │ ⭐⭐⭐⭐⭐ │ 캐시      │
├──────────────┼─────────────┼─────────────┼──────────┤
│ Volume(NVMe) │ ⭐⭐⭐⭐   │ ⭐⭐⭐⭐    │ DB       │
├──────────────┼─────────────┼─────────────┼──────────┤
│ Volume(SSD)  │ ⭐⭐⭐⭐   │ ⭐⭐⭐      │ 일반      │
├──────────────┼─────────────┼─────────────┼──────────┤
│ Bind mount   │ ⭐⭐⭐     │ ⭐⭐⭐      │ 개발      │
├──────────────┼─────────────┼─────────────┼──────────┤
│ NFS          │ ⭐⭐        │ ⭐⭐       │ 공유      │
└──────────────┴─────────────┴─────────────┴──────────┘
```

#### 최적화 3: 이미지 레이어 최소화

```dockerfile
# ❌ 많은 레이어 (느림)
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y nginx
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get clean
# 6 레이어

# ✅ 레이어 최소화 (빠름)
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y \
        nginx \
        curl \
        vim && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# 2 레이어

# ✅✅ Multi-stage로 더 최적화
FROM ubuntu:22.04 AS builder
RUN apt-get update && \
    apt-get install -y build-essential
COPY ../../Downloads /src
RUN cd /src && make

FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y nginx && \
    apt-get clean
COPY --from=builder /src/app /app
# 최종 이미지: 3 레이어, 크기 작음
```

#### 최적화 4: 디스크 I/O 모니터링

```bash
# iostat으로 디스크 성능 확인
sudo apt-get install -y sysstat
iostat -x 1

# Docker 볼륨별 I/O
docker stats --format "table {{.Container}}\t{{.BlockIO}}"

# 특정 컨테이너 상세
docker stats <container> --no-stream

# 실시간 블록 I/O
docker run --rm -it \
  -v /var/run/docker.sock:/var/run/docker.sock \
  nicolaka/netshoot \
  iotop
```

---

### 3. 보안 모범 사례

#### 보안 1: 볼륨 권한 관리

```yaml
version: '3.8'

services:
  app:
    image: myapp
    user: "1000:1000"  # 비특권 사용자
    volumes:
      # 읽기 전용 설정
      - config:/etc/app:ro
      
      # 쓰기 필요한 것만
      - data:/var/lib/app
    security_opt:
      - no-new-privileges:true
    read_only: true  # 루트 파일시스템 읽기 전용
    tmpfs:
      - /tmp  # 임시 파일용

volumes:
  config:
  data:
```

```bash
# 볼륨 권한 확인
docker volume inspect data --format '{{.Mountpoint}}'
# /var/lib/docker/volumes/data/_data

sudo ls -la /var/lib/docker/volumes/data/_data
# drwxr-xr-x 2 1000 1000 4096 Jan 15 10:00 .

# 권한 수정 (필요시)
sudo chown -R 1000:1000 /var/lib/docker/volumes/data/_data
sudo chmod 750 /var/lib/docker/volumes/data/_data
```

#### 보안 2: 데이터 암호화

```bash
# LUKS로 암호화된 볼륨 생성
sudo apt-get install -y cryptsetup

# 1. 암호화 디바이스 생성
sudo cryptsetup luksFormat /dev/sdb
# 패스워드 설정

# 2. 열기
sudo cryptsetup luksOpen /dev/sdb encrypted-docker

# 3. 파일시스템 생성
sudo mkfs.ext4 /dev/mapper/encrypted-docker

# 4. 마운트
sudo mkdir /mnt/encrypted-docker
sudo mount /dev/mapper/encrypted-docker /mnt/encrypted-docker

# 5. Docker 볼륨 사용
docker volume create \
  --driver local \
  --opt type=none \
  --opt o=bind \
  --opt device=/mnt/encrypted-docker \
  encrypted-vol

# 6. 컨테이너에서 사용
docker run -d --name secure-db \
  -v encrypted-vol:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# 데이터는 디스크에 암호화되어 저장됨
```

#### 보안 3: Secrets 관리

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:14
    secrets:
      - db_password
      - db_root_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
    volumes:
      - db-data:/var/lib/postgresql/data

secrets:
  db_password:
    file: ./secrets/db_password.txt
  db_root_password:
    file: ./secrets/db_root_password.txt

volumes:
  db-data:
```

```bash
# Secrets 파일 생성
mkdir secrets
echo "my_secure_password" > secrets/db_password.txt
echo "root_password" > secrets/db_root_password.txt
chmod 600 secrets/*.txt

# Swarm mode에서는 더 안전
docker swarm init
echo "my_secure_password" | docker secret create db_password -

# 사용
docker service create \
  --name postgres \
  --secret db_password \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  postgres:14
```

#### 보안 4: 감사 로깅

```yaml
version: '3.8'

services:
  app:
    image: myapp
    volumes:
      - data:/var/lib/app
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "security,audit"
        env: "os,customer"

  # 중앙 로깅
  fluentd:
    image: fluent/fluentd:latest
    volumes:
      - ./fluentd.conf:/fluentd/etc/fluent.conf:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    ports:
      - "24224:24224"

volumes:
  data:
```

---

### 4. 모니터링과 알림

#### 모니터링 설정

```yaml
# docker-compose-monitoring.yml
version: '3.8'

services:
  # Prometheus (메트릭 수집)
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    ports:
      - "9090:9090"

  # cAdvisor (컨테이너 메트릭)
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8080:8080"
    privileged: true

  # Node Exporter (시스템 메트릭)
  node-exporter:
    image: prom/node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"

  # Grafana (시각화)
  grafana:
    image: grafana/grafana
    depends_on:
      - prometheus
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana-dashboards:/etc/grafana/provisioning/dashboards:ro
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3000:3000"

  # Alertmanager (알림)
  alertmanager:
    image: prom/alertmanager
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager-data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    ports:
      - "9093:9093"

volumes:
  prometheus-data:
  grafana-data:
  alertmanager-data:
```

```yaml
# prometheus.yml
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

# 알림 규칙
rule_files:
  - '/etc/prometheus/alerts.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

```yaml
# alerts.yml (Prometheus 알림 규칙)
groups:
  - name: storage_alerts
    interval: 30s
    rules:
      # 디스크 사용률 80% 이상
      - alert: HighDiskUsage
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High disk usage on {{ $labels.instance }}"
          description: "Disk usage is above 80% (current: {{ $value }}%)"

      # 디스크 사용률 90% 이상
      - alert: CriticalDiskUsage
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical disk usage on {{ $labels.instance }}"
          description: "Disk usage is above 90% (current: {{ $value }}%)"

      # 높은 I/O 대기
      - alert: HighIOWait
        expr: rate(node_disk_io_time_seconds_total[5m]) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High I/O wait on {{ $labels.instance }}"
          description: "I/O wait is high (current: {{ $value }})"

      # 볼륨 크기 급증
      - alert: VolumeGrowthRate
        expr: rate(docker_volume_size_bytes[1h]) > 1073741824  # 1GB/hour
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Rapid volume growth"
          description: "Volume {{ $labels.volume }} is growing rapidly"
```

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'critical'
    - match:
        severity: warning
      receiver: 'warning'

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://webhook-receiver:5000/alerts'

  - name: 'critical'
    email_configs:
      - to: 'ops-critical@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alertmanager'
        auth_password: 'password'
        headers:
          Subject: '[CRITICAL] {{ .GroupLabels.alertname }}'
    
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts-critical'
        title: 'Critical Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'warning'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts-warning'
        title: 'Warning: {{ .GroupLabels.alertname }}'
```

---

### 5. 유지보수와 정리

#### 자동 정리 스크립트

```bash
#!/bin/bash
# docker-cleanup.sh

echo "=== Docker Storage Cleanup ==="

# 1. 사용하지 않는 컨테이너
echo "Removing stopped containers..."
docker container prune -f

# 2. 사용하지 않는 이미지
echo "Removing dangling images..."
docker image prune -f

# 3. 사용하지 않는 볼륨
echo "Removing unused volumes..."
docker volume prune -f

# 4. 사용하지 않는 네트워크
echo "Removing unused networks..."
docker network prune -f

# 5. 빌드 캐시 (7일 이상)
echo "Removing old build cache..."
docker builder prune -a --filter "until=168h" -f

# 6. 전체 정리 (주의!)
# docker system prune -a --volumes -f

# 7. 디스크 사용량 출력
echo ""
echo "=== Current Disk Usage ==="
docker system df -v

# 8. 볼륨별 크기
echo ""
echo "=== Volume Sizes ==="
docker volume ls -q | while read vol; do
  size=$(docker run --rm -v $vol:/data alpine du -sh /data | cut -f1)
  echo "$vol: $size"
done
```

```bash
chmod +x docker-cleanup.sh

# Cron으로 매일 새벽 3시 실행
crontab -e
# 0 3 * * * /path/to/docker-cleanup.sh >> /var/log/docker-cleanup.log 2>&1
```

#### 볼륨 백업 자동화

```bash
#!/bin/bash
# backup-volumes.sh

BACKUP_DIR="/backups/docker-volumes"
RETENTION_DAYS=30

mkdir -p $BACKUP_DIR

# 백업할 볼륨 목록
VOLUMES=(
  "postgres-data"
  "mysql-data"
  "uploads"
)

for vol in "${VOLUMES[@]}"; do
  echo "Backing up volume: $vol"
  
  BACKUP_FILE="$BACKUP_DIR/${vol}-$(date +%Y%m%d-%H%M%S).tar.gz"
  
  docker run --rm \
    -v $vol:/source:ro \
    -v $BACKUP_DIR:/backup \
    alpine \
    tar czf /backup/$(basename $BACKUP_FILE) -C /source .
  
  if [ $? -eq 0 ]; then
    echo "✓ Backup successful: $BACKUP_FILE"
  else
    echo "✗ Backup failed for $vol"
  fi
done

# 오래된 백업 삭제
echo "Cleaning up old backups (older than $RETENTION_DAYS days)..."
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed!"
```

```bash
chmod +x backup-volumes.sh

# Cron: 매일 새벽 2시
# 0 2 * * * /path/to/backup-volumes.sh >> /var/log/volume-backup.log 2>&1
```

---

## 💻 실습

### 실습 1: 종합 모니터링 구축

```bash
# 디렉토리 구조
mkdir -p monitoring/{prometheus,grafana-dashboards,alertmanager}
cd monitoring

# prometheus.yml
cat > prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
  
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

rule_files:
  - '/etc/prometheus/alerts.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
EOF

# alerts.yml
cat > prometheus/alerts.yml << 'EOF'
groups:
  - name: storage
    rules:
      - alert: HighDiskUsage
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk usage high"
          description: "Less than 20% free"
EOF

# alertmanager.yml
cat > alertmanager/alertmanager.yml << 'EOF'
global:
  resolve_timeout: 5m

route:
  receiver: 'default'

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://localhost:5000/webhook'
EOF

# docker-compose.yml (이전 예제 사용)
# ...

docker-compose up -d

# Grafana 접속: http://localhost:3000
# - Username: admin
# - Password: admin

# Prometheus 데이터소스 추가
# URL: http://prometheus:9090

# 대시보드 import: 893 (Docker 대시보드)

# 정리
docker-compose down -v
cd ..
rm -rf monitoring
```

---

### 실습 2: 성능 벤치마크

```bash
# 벤치마크 스크립트
cat > storage-benchmark.sh << 'EOF'
#!/bin/bash

echo "=== Storage Performance Benchmark ==="

# 1. Tmpfs (메모리)
echo "1. Tmpfs (Memory):"
docker run --rm \
  --tmpfs /benchmark:rw,size=1G \
  alpine \
  sh -c 'dd if=/dev/zero of=/benchmark/test bs=1M count=1000 2>&1 | grep copied'

# 2. Named Volume
echo "2. Named Volume:"
docker volume create bench-vol
docker run --rm \
  -v bench-vol:/benchmark \
  alpine \
  sh -c 'dd if=/dev/zero of=/benchmark/test bs=1M count=1000 2>&1 | grep copied'
docker volume rm bench-vol

# 3. Bind Mount
echo "3. Bind Mount:"
mkdir /tmp/bench-bind
docker run --rm \
  -v /tmp/bench-bind:/benchmark \
  alpine \
  sh -c 'dd if=/dev/zero of=/benchmark/test bs=1M count=1000 2>&1 | grep copied'
rm -rf /tmp/bench-bind

# 4. 읽기 성능 (Volume)
echo "4. Read Performance (Volume):"
docker volume create bench-read
docker run --rm \
  -v bench-read:/benchmark \
  alpine \
  sh -c 'dd if=/dev/zero of=/benchmark/test bs=1M count=1000 && \
         dd if=/benchmark/test of=/dev/null bs=1M 2>&1 | grep copied'
docker volume rm bench-read

# 5. IOPS 테스트 (fio)
echo "5. IOPS Test:"
docker volume create bench-iops
docker run --rm \
  -v bench-iops:/benchmark \
  ljishen/fio \
  fio --name=randwrite --ioengine=libaio --iodepth=16 \
      --rw=randwrite --bs=4k --direct=1 --size=1G \
      --numjobs=4 --runtime=60 --group_reporting \
      --directory=/benchmark
docker volume rm bench-iops

echo "Benchmark completed!"
EOF

chmod +x storage-benchmark.sh
./storage-benchmark.sh

# 결과 예시:
# 1. Tmpfs: 2000 MB/s (매우 빠름)
# 2. Volume: 800 MB/s (SSD)
# 3. Bind: 600 MB/s
# 4. Read: 1200 MB/s
# 5. IOPS: 15000 (4K random write)

rm storage-benchmark.sh
```

---

### 실습 3: 보안 강화 실습

```yaml
# docker-compose-secure.yml
version: '3.8'

services:
  secure-app:
    image: nginx:alpine
    user: "101:101"  # nginx 사용자
    read_only: true   # 루트 FS 읽기 전용
    security_opt:
      - no-new-privileges:true
      - seccomp=unconfined
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - SETUID
      - SETGID
    volumes:
      # 설정 (읽기 전용)
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      
      # 데이터 (쓰기)
      - secure-data:/data
      
      # 임시 파일 (tmpfs)
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100M
          mode: 1777
      
      - type: tmpfs
        target: /var/run
        tmpfs:
          size: 10M
    environment:
      - NGINX_USER=nginx
    labels:
      - "security.level=high"
      - "compliance=pci-dss"

volumes:
  secure-data:
    driver: local
    driver_opts:
      type: none
      o: bind,mode=0750
      device: /mnt/secure/data
```

```bash
# 보안 디렉토리 준비
sudo mkdir -p /mnt/secure/data
sudo chown 101:101 /mnt/secure/data
sudo chmod 750 /mnt/secure/data

# nginx.conf
cat > nginx.conf << 'EOF'
user nginx;
worker_processes auto;
error_log /dev/stderr warn;
pid /tmp/nginx.pid;

events {
    worker_connections 1024;
}

http {
    client_body_temp_path /tmp/client_body;
    proxy_temp_path /tmp/proxy;
    fastcgi_temp_path /tmp/fastcgi;
    uwsgi_temp_path /tmp/uwsgi;
    scgi_temp_path /tmp/scgi;

    server {
        listen 8080;
        root /data;
        
        location / {
            autoindex on;
        }
    }
}
EOF

docker-compose -f docker-compose-secure.yml up -d

# 보안 검증
docker exec $(docker-compose -f docker-compose-secure.yml ps -q secure-app) id
# uid=101(nginx) gid=101(nginx) groups=101(nginx)

docker exec $(docker-compose -f docker-compose-secure.yml ps -q secure-app) \
  sh -c 'touch /test' || echo "✓ Root FS is read-only"

# 정리
docker-compose -f docker-compose-secure.yml down -v
rm nginx.conf docker-compose-secure.yml
sudo rm -rf /mnt/secure
```

---

## 🔥 실전 체크리스트

### 📋 프로덕션 배포 체크리스트

```
□ 설계
  □ 데이터 분류 (Critical/Important/Temporary)
  □ 볼륨 유형 선택 (Named/Bind/Tmpfs)
  □ Storage Driver 확인 (overlay2 권장)
  □ 디스크 타입 (SSD/NVMe for DB)

□ 성능
  □ IOPS 요구사항 확인
  □ 레이어 최소화
  □ 캐시 전략 (Tmpfs)
  □ 벤치마크 실행

□ 보안
  □ 최소 권한 원칙
  □ 읽기 전용 설정
  □ 민감 데이터 암호화
  □ Secrets 관리
  □ 감사 로깅

□ 고가용성
  □ 복제 설정
  □ 백업 자동화
  □ PITR 구성 (DB)
  □ DR 계획

□ 모니터링
  □ 디스크 사용률 알림
  □ I/O 성능 모니터링
  □ 볼륨 크기 추적
  □ 로그 수집

□ 유지보수
  □ 정리 스크립트
  □ 백업 검증
  □ 문서화
  □ 복구 테스트
```

---

## 🚫 안티패턴

### 1. 무분별한 Bind Mount

```yaml
# ❌ 프로덕션에서 bind mount 남용
services:
  app:
    volumes:
      - /var/data:/app/data      # 권한 문제
      - /home/user/logs:/logs    # 이식성 없음
      - ./config:/etc/app        # 보안 위험

# ✅ Named volume 사용
services:
  app:
    volumes:
      - app-data:/app/data
      - app-logs:/logs
      - type: bind
        source: ./config
        target: /etc/app
        read_only: true          # 읽기 전용

volumes:
  app-data:
  app-logs:
```

### 2. 모니터링 없음

```yaml
# ❌ 모니터링 없이 운영
services:
  db:
    image: postgres
    volumes:
      - db-data:/var/lib/postgresql/data
# 디스크 가득 차도 모름!

# ✅ 모니터링 포함
services:
  db:
    image: postgres
    volumes:
      - db-data:/var/lib/postgresql/data
    labels:
      - "prometheus.io/scrape=true"
  
  exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "..."
```

### 3. 백업 없음

```yaml
# ❌ 중요 데이터 백업 없음
volumes:
  critical-data:
# 데이터 손실 시?

# ✅ 자동 백업
services:
  backup:
    image: alpine
    volumes:
      - critical-data:/source:ro
      - ./backups:/backups
    entrypoint: |
      sh -c "while true; do
        tar czf /backups/backup-$(date +%Y%m%d).tar.gz -C /source .
        sleep 86400
      done"
```

### 4. 정리 안 함

```bash
# ❌ 계속 쌓임
docker system df
# Images:  50GB
# Volumes: 100GB
# ...

# ✅ 정기 정리
docker system prune -a --volumes --filter "until=168h" -f
# 또는 자동화 스크립트
```

---

## 🎓 핵심 정리

### 1. 설계 원칙

```
데이터 분류:
- Critical → SSD + 백업 + 복제
- Important → 백업
- Temporary → Tmpfs

분리:
- Application (이미지)
- Data (볼륨)
- Config (bind mount)

최소 권한:
- 비특권 사용자
- 읽기 전용
- 필요한 것만
```

### 2. 성능 최적화

```
Storage Driver:
- Linux: overlay2
- 대용량 쓰기: zfs/btrfs

볼륨 유형:
- DB: Named (SSD/NVMe)
- 캐시: Tmpfs
- 공유: NFS (주의)

이미지:
- 레이어 최소화
- Multi-stage
- .dockerignore
```

### 3. 보안

```
권한:
- user 지정
- read_only
- no-new-privileges

암호화:
- LUKS (저장)
- TLS (전송)
- Secrets

감사:
- 로깅
- 모니터링
- 알림
```

### 4. 운영

```
모니터링:
- 디스크 사용률
- IOPS
- 볼륨 크기

백업:
- 자동화
- 검증
- 복원 테스트

정리:
- 스케줄
- 정책
- 문서화
```

---

## 📚 참고 자료

- [Docker Storage Best Practices](https://docs.docker.com/storage/)
- [Production Best Practices](https://docs.docker.com/engine/swarm/swarm-tutorial/)
- [Security Best Practices](https://docs.docker.com/engine/security/)

---

## 🤔 생각해볼 문제

1. Tmpfs는 언제 사용하고 언제 피해야 하는가?
2. 프로덕션에서 bind mount를 사용하는 경우는?
3. 디스크 사용률 80% 알림, 적절한가?

> 💡 **답변**: 1) Tmpfs 사용: 캐시 (Redis without persistence, 세션 스토어), 임시 파일 (빌드, /tmp), 빠른 읽기/쓰기 필요, 재시작 시 초기화 OK, 피해야: 영속성 필요, 메모리 부족, 큰 데이터 (메모리 압박), 2) 프로덕션 bind mount: 설정 파일 (read-only!), 예: nginx.conf, SSL 인증서 (갱신 필요), 로그 (호스트에서 직접 접근), Secrets (읽기 전용), 하지만 데이터는 Named volume!, 3) 80%는 늦을 수 있음 - 권장: Warning 70%, Critical 85%, 이유: 로그/데이터 급증 시 90%+ 금방, 정리 시간 필요, 성능 저하 (80%+ 시), 디스크 타입별 다름 (SSD vs HDD), 모니터링 + 증가율 추적 중요

---

<div align="center">

**[⬅️ 이전: Data Persistence](./06-Data-Persistence.md)** | **[홈으로 🏠](../README.md)**

</div>
