# 07. High Availability - 고가용성

## 🎯 이 챕터에서 배울 것

- **고가용성(HA)** 아키텍처 설계
- **장애 복구** 메커니즘
- **데이터 복제**와 백업
- **분산 시스템** 패턴

## 📌 왜 중요한가?

**"고가용성은 시스템이 장애 상황에서도 지속적으로 서비스를 제공하는 능력입니다."**

```
단일 장애점 vs 고가용성:

단일 장애점 (SPOF - Single Point of Failure):
┌─────────────────────────────────────┐
│ Single Server                       │
│ ┌─────────┐                         │
│ │   App   │                         │
│ └────┬────┘                         │
│      │                              │
│ ┌────▼────┐                         │
│ │   DB    │                         │
│ └─────────┘                         │
└─────────────────────────────────────┘
❌ 서버 장애 = 서비스 중단
❌ 복구 시간 필요
❌ 데이터 손실 위험
❌ 사용자 영향 큼

고가용성 아키텍처:
┌───────────────────────────────────┐
│ Load Balancer (HA)                │
│ ┌────────┐ ┌─────────┐            │
│ │  LB 1  │ │  LB 2   │            │
│ └───┬────┘ └────┬────┘            │
└─────┼───────────┼─────────────────┘
      │           │
  ┌───┴───┬───────┴─┬───────┐
  │       │         │       │
┌─▼───┐ ┌─▼───┐ ┌───▼─┐ ┌───▼─┐
│App 1│ │App 2│ │App 3│ │App 4│
└──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘
   │       │       │       │
   └───────┼───────┼───────┘
           │       │
      ┌────▼───────▼────┐
      │ Database Cluster│
      │ ┌────┐  ┌────┐  │
      │ │ M  │←→│ R  │  │
      │ └────┘  └────┘  │
      │ Primary Replica │
      └─────────────────┘

✅ 노드 장애 시 자동 복구
✅ 무중단 서비스
✅ 데이터 복제/백업
✅ 사용자 영향 최소화

고가용성의 핵심 가치:

1. 가용성 목표 (SLA):
   - 99.9% (Three Nines): 연 8.76시간 다운타임
   - 99.99% (Four Nines): 연 52.6분 다운타임
   - 99.999% (Five Nines): 연 5.26분 다운타임

2. 장애 허용 (Fault Tolerance):
   - 노드 장애
   - 네트워크 장애
   - 디스크 장애
   - 데이터센터 장애

3. 자동 복구 (Self-Healing):
   - 장애 감지
   - 자동 재시작
   - 트래픽 재분배
   - 데이터 복구

4. 무중단 운영:
   - 롤링 업데이트
   - 유지보수
   - 확장/축소

실무 시나리오:

E-Commerce 플랫폼:
┌─────────────────────────────────────┐
│ Region 1 (Primary)                  │
│ ┌───────────────────────────────┐   │
│ │ Manager Nodes (3)             │   │
│ │ ┌────┐ ┌────┐ ┌────┐          │   │
│ │ │ M1 │ │ M2 │ │ M3 │          │   │
│ │ └────┘ └────┘ └────┘          │   │
│ │ (Raft Consensus)              │   │
│ └───────────────────────────────┘   │
│ ┌───────────────────────────────┐   │
│ │ Worker Nodes (10)             │   │
│ │ - Web (3 replicas)            │   │
│ │ - API (5 replicas)            │   │
│ │ - Workers (5 replicas)        │   │
│ └───────────────────────────────┘   │
│ ┌───────────────────────────────┐   │
│ │ Database Cluster              │   │
│ │ - Primary (1)                 │   │
│ │ - Replicas (2)                │   │
│ │ - Auto Failover               │   │
│ └───────────────────────────────┘   │
└─────────────────────────────────────┘
              ↕ Replication
┌─────────────────────────────────────┐
│ Region 2 (DR Standby)               │
│ - Passive Replica                   │
│ - 15분 RPO                          │
│ - 30분 RTO                          │
└─────────────────────────────────────┘

장애 시나리오:
- Worker 1대 장애 → 자동 복구 (30초)
- Manager 1대 장애 → 쿼럼 유지, 정상 운영
- Database Primary 장애 → Replica 승격 (1분)
- 전체 Region 장애 → DR 전환 (30분)
```

**실무 영향:**
- SLA 달성
- 고객 신뢰
- 매출 손실 방지
- 브랜드 가치

---

## 🔬 Deep Dive

### 1. Manager 노드 고가용성

#### Raft Consensus

```
Raft 합의 알고리즘:

3 Manager Cluster:
┌─────────┐  ┌─────────┐  ┌─────────┐
│Manager 1│  │Manager 2│  │Manager 3│
│(Leader) │  │         │  │         │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
              Heartbeat

장애 시나리오:

1. Manager 1 장애 (Leader):
┌─────────┐  ┌─────────┐  ┌─────────┐
│Manager 1│  │Manager 2│  │Manager 3│
│   ❌    │  │         │  │         │
└─────────┘  └────┬────┘  └────┬────┘
                  │            │
              Election (10초)
                  │            │
┌─────────┐  ┌────▼────┐  ┌─────────┐
│Manager 1│  │Manager 2│  │Manager 3│
│   ❌    │  │(Leader) │  │         │
└─────────┘  └─────────┘  └─────────┘
✅ 새 리더 선출, 정상 운영

2. Manager 2대 동시 장애:
┌─────────┐  ┌─────────┐  ┌─────────┐
│Manager 1│  │Manager 2│  │Manager 3│
│   ❌    │  │   ❌    │  │         │
└─────────┘  └─────────┘  └────┬────┘
                               │
                          쿼럼 상실
❌ 클러스터 정지
⚠️ Read-Only 모드

권장 구성:
- 1 Manager: 장애 허용 0
- 3 Managers: 장애 허용 1 (권장)
- 5 Managers: 장애 허용 2 (대규모)
- 7 Managers: 장애 허용 3 (초대규모)
```

#### Manager 고가용성 구성

```bash
# 3 Manager 클러스터 구성

# Node 1 (Manager 1)
docker swarm init --advertise-addr 192.168.1.101

# Manager Join Token
MANAGER_TOKEN=$(docker swarm join-token manager -q)

# Node 2 (Manager 2)
docker swarm join \
  --token $MANAGER_TOKEN \
  --advertise-addr 192.168.1.102 \
  192.168.1.101:2377

# Node 3 (Manager 3)
docker swarm join \
  --token $MANAGER_TOKEN \
  --advertise-addr 192.168.1.103 \
  192.168.1.101:2377

# 확인
docker node ls
# ID         HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc... *   manager1   Ready   Active        Leader
# def...     manager2   Ready   Active        Reachable
# ghi...     manager3   Ready   Active        Reachable

# Manager 전용 (워크로드 제거)
docker node update --availability drain manager1
docker node update --availability drain manager2
docker node update --availability drain manager3
```

#### Manager 장애 복구 시뮬레이션

```bash
# 리더 노드 강제 중지
docker node inspect manager1 --format '{{.ManagerStatus.Leader}}'
# true

# manager1 중지 (시뮬레이션)
# SSH to manager1
sudo systemctl stop docker

# 다른 Manager에서 확인
docker node ls
# manager1: Down
# manager2: Leader (새로 선출됨!)
# manager3: Reachable

# 서비스는 정상 운영
docker service ls
# ✅ 모든 서비스 정상

# manager1 복구
# SSH to manager1
sudo systemctl start docker

# 확인
docker node ls
# manager1: Ready, Reachable (복구됨)
# manager2: Ready, Leader
# manager3: Ready, Reachable
```

---

### 2. 애플리케이션 고가용성

#### 레플리카 분산

```bash
# Swarm 초기화 (3 Worker)
docker swarm init

# 노드 레이블 (가용 영역)
docker node update --label-add zone=zone-a worker1
docker node update --label-add zone=zone-b worker2
docker node update --label-add zone=zone-c worker3

# Zone별 분산 배치
docker service create \
  --name web \
  --replicas 9 \
  --placement-pref spread=node.labels.zone \
  --publish 8080:80 \
  nginx:alpine

# 확인
docker service ps web --format '{{.Name}}\t{{.Node}}'
# web.1  worker1 (zone-a)
# web.2  worker2 (zone-b)
# web.3  worker3 (zone-c)
# web.4  worker1 (zone-a)
# web.5  worker2 (zone-b)
# web.6  worker3 (zone-c)
# web.7  worker1 (zone-a)
# web.8  worker2 (zone-b)
# web.9  worker3 (zone-c)
# ✅ 균등 분산!

# Worker 1 장애 시
# → web.1, web.4, web.7 자동 재배치
# → worker2, worker3로 이동
```

#### 자동 복구

```bash
# 서비스 생성
docker service create \
  --name app \
  --replicas 5 \
  --restart-condition on-failure \
  --restart-max-attempts 3 \
  --restart-delay 10s \
  myapp:latest

# 컨테이너 강제 종료 (장애 시뮬레이션)
CONTAINER_ID=$(docker ps -q -f name=app | head -1)
docker kill $CONTAINER_ID

# 자동 재시작 확인
docker service ps app
# ID      NAME      IMAGE         DESIRED  CURRENT
# abc...  app.1     myapp:latest  Running  Running (10s ago)
# def...   \_ app.1 myapp:latest  Shutdown Failed (killed)
# ✅ 자동 재시작됨!
```

#### Anti-Affinity (분산 배치)

```bash
# 같은 노드에 여러 레플리카 방지
docker service create \
  --name db \
  --replicas 3 \
  --constraint 'node.labels.type==database' \
  --placement-max-replicas-per-node 1 \
  postgres:14-alpine

# 확인
docker service ps db
# db.1  node1
# db.2  node2
# db.3  node3
# ✅ 각 노드에 1개씩만!

# node1 장애 시
# → db.1만 영향
# → db.2, db.3은 정상 운영
```

---

### 3. 데이터베이스 고가용성

#### Primary-Replica 구조

```yaml
# stack.yml
version: '3.8'

services:
  # Primary Database
  db-primary:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - db-primary-data:/var/lib/postgresql/data
    networks:
      - db-network
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.db-role==primary

  # Replica 1
  db-replica1:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
      PRIMARY_HOST: db-primary
    volumes:
      - db-replica1-data:/var/lib/postgresql/data
    networks:
      - db-network
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.db-role==replica

  # Replica 2
  db-replica2:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
      PRIMARY_HOST: db-primary
    volumes:
      - db-replica2-data:/var/lib/postgresql/data
    networks:
      - db-network
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.db-role==replica

  # PgPool (Connection Pooling + Load Balancing)
  pgpool:
    image: pgpool/pgpool:latest
    environment:
      POSTGRES_PASSWORD: secret
      PGPOOL_BACKEND_NODES: "0:db-primary:5432,1:db-replica1:5432,2:db-replica2:5432"
      PGPOOL_ENABLE_LOAD_BALANCING: "yes"
    networks:
      - db-network
    ports:
      - "5432:5432"
    deploy:
      replicas: 2

networks:
  db-network:
    driver: overlay

volumes:
  db-primary-data:
  db-replica1-data:
  db-replica2-data:
```

#### 장애 복구 시나리오

```bash
# Primary 장애 시:
# 1. Replica 승격 (Manual or Auto)
docker service update \
  --label-add role=primary \
  db-replica1

# 2. 다른 Replica를 새 Primary에 연결
docker service update \
  --env-add PRIMARY_HOST=db-replica1 \
  db-replica2

# 3. 새 Replica 추가
docker service create \
  --name db-replica3 \
  --env PRIMARY_HOST=db-replica1 \
  postgres:14-alpine
```

---

### 4. 백업과 복구

#### 자동 백업

```yaml
# backup.yml
version: '3.8'

services:
  # Backup Service
  backup:
    image: postgres:14-alpine
    networks:
      - db-network
    volumes:
      - db-primary-data:/var/lib/postgresql/data:ro
      - ./backups:/backups
    environment:
      POSTGRES_PASSWORD: secret
    entrypoint: >
      sh -c "
        while true; do
          echo 'Starting backup at $(date)'
          
          # Full backup
          BACKUP_FILE=/backups/full-backup-$(date +%Y%m%d-%H%M%S).sql
          PGPASSWORD=secret pg_dumpall -h db-primary -U postgres > \$$BACKUP_FILE
          
          # Compress
          gzip \$$BACKUP_FILE
          
          # Retention (30 days)
          find /backups -name '*.sql.gz' -mtime +30 -delete
          
          echo 'Backup completed'
          
          # Daily at 2 AM
          sleep 86400
        done
      "
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.backup==true

networks:
  db-network:
    external: true

volumes:
  db-primary-data:
    external: true
```

#### 재해 복구 (DR)

```yaml
# dr-sync.yml
version: '3.8'

services:
  # DR Sync (Continuous Replication)
  dr-sync:
    image: alpine
    volumes:
      - db-primary-data:/source:ro
      - nfs-dr:/dr-site
    entrypoint: >
      sh -c "
        apk add --no-cache rsync
        
        while true; do
          echo 'Syncing to DR site...'
          
          rsync -avz --delete \
            /source/ \
            /dr-site/postgresql/
          
          echo 'Sync completed at $(date)'
          
          # Every 15 minutes
          sleep 900
        done
      "
    deploy:
      replicas: 1

volumes:
  db-primary-data:
    external: true
  
  nfs-dr:
    driver: local
    driver_opts:
      type: nfs
      o: addr=dr-site.example.com,rw
      device: ":/dr-backup"
```

---

## 💻 실습

### 실습 1: Manager 고가용성

```bash
# Swarm 초기화
docker swarm init

# 자신을 3개로 확장 (시뮬레이션)
# 실제로는 3개 노드 필요

# 현재 Manager 상태
docker node ls

# 리더 확인
docker node inspect $(docker node ls -q) \
  --format '{{.ManagerStatus.Leader}}'
# true

# Auto-lock 활성화 (보안)
docker swarm update --autolock=true
# 키 저장 필수!

# Manager 재시작 시뮬레이션
sudo systemctl restart docker

# Unlock 필요
docker swarm unlock
# 키 입력

# 정리
docker swarm leave --force
```

---

### 실습 2: 애플리케이션 자동 복구

```bash
# Swarm 초기화
docker swarm init

# 서비스 생성 (자동 복구 설정)
docker service create \
  --name web \
  --replicas 5 \
  --publish 8080:80 \
  --restart-condition on-failure \
  --restart-max-attempts 5 \
  --restart-delay 10s \
  --health-cmd "wget -q --spider http://localhost || exit 1" \
  --health-interval 10s \
  --health-timeout 5s \
  --health-retries 3 \
  nginx:alpine

# 정상 동작 확인
docker service ps web

# 장애 시뮬레이션 (컨테이너 강제 종료)
CONTAINER_ID=$(docker ps -q -f name=web | head -1)
docker kill $CONTAINER_ID

# 자동 복구 확인
watch -n 1 'docker service ps web | head -10'
# 새 태스크 자동 생성됨!

# 여러 번 장애 발생 (5회 초과)
for i in {1..6}; do
  CONTAINER_ID=$(docker ps -q -f name=web | head -1)
  docker kill $CONTAINER_ID
  sleep 5
done

# 확인
docker service ps web
# 5회 재시작 후 중지됨

# 정리
docker service rm web
docker swarm leave --force
```

---

### 실습 3: 분산 배치

```bash
# Swarm 초기화
docker swarm init

# 노드에 레이블 (Zone 시뮬레이션)
NODE_ID=$(docker node ls -q)
docker node update --label-add zone=zone-a $NODE_ID

# 서비스 생성 (Zone별 분산)
docker service create \
  --name app \
  --replicas 6 \
  --placement-pref spread=node.labels.zone \
  nginx:alpine

# 확인
docker service ps app

# Anti-Affinity (노드당 최대 2개)
docker service create \
  --name db \
  --replicas 4 \
  --placement-max-replicas-per-node 2 \
  postgres:14-alpine

# 확인
docker service ps db

# 정리
docker service rm app db
docker swarm leave --force
```

---

## 🔥 실전 시나리오

### 시나리오 1: 완전한 HA 스택

```yaml
# ha-stack.yml
version: '3.8'

services:
  # Nginx (Load Balancer)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    networks:
      - frontend
    configs:
      - source: nginx-config
        target: /etc/nginx/nginx.conf
    deploy:
      replicas: 3
      placement:
        max_replicas_per_node: 1
        preferences:
          - spread: node.labels.zone
      update_config:
        parallelism: 1
        delay: 30s
        order: start-first
      restart_policy:
        condition: on-failure
        max_attempts: 3

  # Application
  app:
    image: myapp:latest
    networks:
      - frontend
      - backend
    environment:
      DB_HOST: pgpool
      REDIS_HOST: redis-sentinel
    deploy:
      replicas: 10
      placement:
        preferences:
          - spread: node.labels.zone
      update_config:
        parallelism: 2
        delay: 30s
        failure_action: rollback
        order: start-first
      restart_policy:
        condition: on-failure
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  # PostgreSQL Primary
  postgres-primary:
    image: postgres:14-alpine
    networks:
      - backend
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - postgres-primary:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.db-role==primary
      restart_policy:
        condition: on-failure

  # PostgreSQL Replica
  postgres-replica:
    image: postgres:14-alpine
    networks:
      - backend
    environment:
      POSTGRES_PASSWORD: secret
      PRIMARY_HOST: postgres-primary
    volumes:
      - postgres-replica:/var/lib/postgresql/data
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.labels.db-role==replica
        max_replicas_per_node: 1

  # PgPool (Connection Pool + LB)
  pgpool:
    image: pgpool/pgpool:latest
    networks:
      - backend
    environment:
      POSTGRES_PASSWORD: secret
      PGPOOL_BACKEND_NODES: "0:postgres-primary:5432,1:postgres-replica:5432"
      PGPOOL_ENABLE_LOAD_BALANCING: "yes"
    deploy:
      replicas: 2
      placement:
        max_replicas_per_node: 1

  # Redis Master
  redis-master:
    image: redis:alpine
    networks:
      - backend
    command: redis-server --appendonly yes
    volumes:
      - redis-master:/data
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.cache==primary

  # Redis Replica
  redis-replica:
    image: redis:alpine
    networks:
      - backend
    command: redis-server --appendonly yes --slaveof redis-master 6379
    volumes:
      - redis-replica:/data
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.labels.cache==replica
        max_replicas_per_node: 1

  # Redis Sentinel (HA)
  redis-sentinel:
    image: redis:alpine
    networks:
      - backend
    command: redis-sentinel /etc/redis/sentinel.conf
    configs:
      - source: sentinel-config
        target: /etc/redis/sentinel.conf
    deploy:
      replicas: 3
      placement:
        max_replicas_per_node: 1

  # Backup Service
  backup:
    image: postgres:14-alpine
    networks:
      - backend
    volumes:
      - postgres-primary:/source:ro
      - ./backups:/backups
    environment:
      PGPASSWORD: secret
    entrypoint: >
      sh -c "
        while true; do
          pg_dumpall -h postgres-primary -U postgres | gzip > /backups/backup-$(date +%Y%m%d).sql.gz
          find /backups -mtime +7 -delete
          sleep 86400
        done
      "
    deploy:
      replicas: 1

networks:
  frontend:
    driver: overlay
  backend:
    driver: overlay
    internal: true

volumes:
  postgres-primary:
  postgres-replica:
  redis-master:
  redis-replica:

configs:
  nginx-config:
    file: ./nginx.conf
  sentinel-config:
    file: ./sentinel.conf
```

---

## 🚫 안티패턴

### 1. 단일 Manager

```bash
# ❌ 1개 Manager (SPOF)
docker swarm init
# Manager 장애 = 클러스터 마비

# ✅ 3개 Manager (HA)
# manager1, manager2, manager3
# 1대 장애 허용
```

### 2. 레플리카 1개

```bash
# ❌ 단일 레플리카
docker service create --replicas 1 myapp
# 노드 장애 = 서비스 중단

# ✅ 다중 레플리카
docker service create --replicas 3 myapp
# 노드 장애 시 나머지가 처리
```

### 3. 백업 없음

```bash
# ❌ 백업 전략 없음
# 데이터 손실 시 복구 불가

# ✅ 자동 백업
# - Daily full backup
# - Hourly incremental
# - 30일 retention
# - DR site sync
```

---

## 🎓 핵심 정리

### 1. HA 구성 요소

```
Manager:
- 홀수 개 (3, 5, 7)
- Raft 합의
- 쿼럼 유지

Application:
- 다중 레플리카
- Zone 분산
- 자동 복구

Database:
- Primary-Replica
- Auto Failover
- 백업/복구
```

### 2. 가용성 계산

```
SLA 목표:
- 99.9%: 연 8.76시간
- 99.99%: 연 52.6분
- 99.999%: 연 5.26분

달성 방법:
- 중복성 (Redundancy)
- 장애 허용 (Fault Tolerance)
- 빠른 복구 (MTTR 최소화)
```

### 3. 핵심 명령어

```bash
# Manager
docker node promote <node>
docker node demote <node>

# 배치
--placement-pref spread=node.labels.<key>
--placement-max-replicas-per-node <N>

# 재시작
--restart-condition on-failure
--restart-max-attempts <N>
```

### 4. Best Practices

```
✅ 3+ Manager 노드
✅ Zone별 분산 배치
✅ 자동 복구 설정
✅ 헬스체크 필수
✅ 정기 백업
✅ DR 계획
✅ 모니터링/알림
```

---

<div align="center">

**[⬅️ 이전: Rolling Updates](./06-Rolling-Updates.md)** | **[홈으로 🏠](../README.md)**

</div>
