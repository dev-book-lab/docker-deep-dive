# 04. Swarm Services - 서비스 배포

## 🎯 이 챕터에서 배울 것

- **Service** 개념과 생성
- **Replicated** vs **Global** 모드
- **배치 전략**과 **제약 조건**
- **서비스 관리**와 업데이트

## 📌 왜 중요한가?

**"Swarm Service는 컨테이너를 선언적으로 관리하는 핵심 추상화입니다."**

```
docker run vs docker service:

docker run (단일 컨테이너):
┌─────────────────────────────────────┐
│ $ docker run -d nginx                │
│ ┌─────────┐                         │
│ │Container│                         │
│ └─────────┘                         │
└─────────────────────────────────────┘
❌ 수동 관리
❌ 장애 시 재시작 안 함
❌ 로드 밸런싱 없음
❌ 확장 어려움

docker service (클러스터 서비스):
┌─────────────────────────────────────┐
│ $ docker service create \            │
│     --replicas 3 nginx               │
│                                     │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐│
│ │Task 1   │ │Task 2   │ │Task 3   ││
│ │(node1)  │ │(node2)  │ │(node3)  ││
│ └─────────┘ └─────────┘ └─────────┘│
└─────────────────────────────────────┘
✅ 자동 복구
✅ 로드 밸런싱
✅ 확장 간편
✅ 롤링 업데이트

Service의 핵심 가치:

1. 선언적 상태:
   - 원하는 상태 선언
   - Swarm이 자동 유지
   - 장애 시 자동 복구

2. 고가용성:
   - 여러 레플리카
   - 노드 분산 배치
   - 헬스체크 기반 재시작

3. 로드 밸런싱:
   - Ingress 네트워크
   - 라운드 로빈
   - 자동 서비스 디스커버리

4. 확장성:
   - 레플리카 조정
   - 자동 재배치
   - 리소스 관리

실무 시나리오:

Web Application:
┌─────────────────────────────────────┐
│ Nginx (replicas: 3)                 │
│ ┌─────┐ ┌─────┐ ┌─────┐           │
│ │  1  │ │  2  │ │  3  │           │
│ └─────┘ └─────┘ └─────┘           │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│ API Server (replicas: 5)            │
│ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐    │
│ │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │    │
│ └───┘ └───┘ └───┘ └───┘ └───┘    │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│ Database (replicas: 1)              │
│ ┌─────────┐                         │
│ │Primary  │                         │
│ └─────────┘                         │
└─────────────────────────────────────┘

Monitoring (Global):
┌─────────────────────────────────────┐
│ Node Exporter (모든 노드)           │
│ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐    │
│ │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │    │
│ └───┘ └───┘ └───┘ └───┘ └───┘    │
└─────────────────────────────────────┘
```

**실무 영향:**
- 무중단 서비스
- 자동 장애 복구
- 트래픽 분산
- 간편한 확장

---

## 🔬 Deep Dive

### 1. Service 기본

#### Service 생성

```bash
# 기본 서비스 생성
docker service create \
  --name web \
  nginx:alpine

# 서비스 확인
docker service ls
# ID         NAME  MODE         REPLICAS  IMAGE
# xyz123...  web   replicated   1/1       nginx:alpine

# 태스크 확인
docker service ps web
# ID         NAME   IMAGE          NODE     DESIRED STATE  CURRENT STATE
# abc...     web.1  nginx:alpine   node1    Running        Running 10s ago

# 로그 확인
docker service logs web

# 서비스 삭제
docker service rm web
```

#### 레플리카 서비스

```bash
# 3개 레플리카
docker service create \
  --name web \
  --replicas 3 \
  --publish 8080:80 \
  nginx:alpine

# 확인
docker service ps web
# ID      NAME   IMAGE          NODE     DESIRED STATE  CURRENT STATE
# abc...  web.1  nginx:alpine   node1    Running        Running
# def...  web.2  nginx:alpine   node2    Running        Running  
# ghi...  web.3  nginx:alpine   node3    Running        Running

# 스케일 변경
docker service scale web=5

# 확인
docker service ps web
# 5개 태스크 실행 중

# 스케일 다운
docker service scale web=2

# 정리
docker service rm web
```

#### Global 서비스

```bash
# 모든 노드에 1개씩
docker service create \
  --name monitor \
  --mode global \
  --mount type=bind,source=/,target=/rootfs,readonly=true \
  --mount type=bind,source=/var/run,target=/var/run,readonly=true \
  --mount type=bind,source=/sys,target=/sys,readonly=true \
  prom/node-exporter

# 확인 (노드 수만큼 태스크)
docker service ps monitor
# ID      NAME              IMAGE                  NODE     STATE
# abc...  monitor.node1     prom/node-exporter     node1    Running
# def...  monitor.node2     prom/node-exporter     node2    Running
# ghi...  monitor.node3     prom/node-exporter     node3    Running

# Global 서비스는 scale 불가
docker service scale monitor=5
# Error: service mode must be replicated

# 정리
docker service rm monitor
```

---

### 2. Service 옵션

#### 포트 퍼블리싱

```bash
# Ingress 모드 (기본) - 로드 밸런싱
docker service create \
  --name web \
  --replicas 3 \
  --publish 8080:80 \
  nginx:alpine

# 또는 명시적
docker service create \
  --name web \
  --replicas 3 \
  --publish published=8080,target=80,mode=ingress \
  nginx:alpine

# Host 모드 - 직접 바인딩
docker service create \
  --name web \
  --publish published=8080,target=80,mode=host \
  nginx:alpine
# 주의: 같은 노드에 여러 레플리카 불가

# 여러 포트
docker service create \
  --name app \
  --publish 8080:80 \
  --publish 8443:443 \
  nginx:alpine
```

**Ingress vs Host 모드:**

```
Ingress (기본):
┌────────────────────────────────────┐
│ 클라이언트                         │
└────────┬───────────────────────────┘
         │ :8080
    ┌────▼────┐
    │ Ingress │ (어느 노드든 접근 가능)
    │ Network │
    └────┬────┘
         │ 로드 밸런싱
  ┌──────┼──────┐
  │      │      │
┌─▼─┐  ┌─▼─┐  ┌─▼─┐
│T.1│  │T.2│  │T.3│
└───┘  └───┘  └───┘

Host:
┌────────────────────────────────────┐
│ 클라이언트                         │
└────────┬───────────────────────────┘
         │ node1:8080 (직접 지정)
    ┌────▼────┐
    │ node1   │
    │ ┌─────┐ │
    │ │ T.1 │ │
    │ └─────┘ │
    └─────────┘
```

#### 환경 변수

```bash
# 단일 환경 변수
docker service create \
  --name db \
  --env POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# 여러 환경 변수
docker service create \
  --name app \
  --env NODE_ENV=production \
  --env DEBUG=false \
  --env PORT=3000 \
  node:18-alpine

# 파일에서 로드
cat > app.env << EOF
NODE_ENV=production
DEBUG=false
API_KEY=abc123
EOF

docker service create \
  --name app \
  --env-file app.env \
  node:18-alpine

rm app.env
```

#### 볼륨 마운트

```bash
# Named volume
docker service create \
  --name db \
  --mount type=volume,source=db-data,target=/var/lib/postgresql/data \
  postgres:14-alpine

# Bind mount
docker service create \
  --name web \
  --mount type=bind,source=/var/www,target=/usr/share/nginx/html,readonly \
  nginx:alpine

# Tmpfs
docker service create \
  --name cache \
  --mount type=tmpfs,target=/tmp,tmpfs-size=100m \
  redis:alpine

# 여러 마운트
docker service create \
  --name app \
  --mount type=volume,source=app-data,target=/app/data \
  --mount type=bind,source=./config,target=/app/config,readonly \
  --mount type=tmpfs,target=/tmp \
  myapp:latest
```

#### 네트워크

```bash
# 커스텀 네트워크 생성
docker network create --driver overlay my-network

# 서비스에 연결
docker service create \
  --name web \
  --network my-network \
  nginx:alpine

# 여러 네트워크
docker service create \
  --name api \
  --network frontend \
  --network backend \
  node:18-alpine

# 네트워크 별칭
docker service create \
  --name db \
  --network backend \
  --network-alias database \
  --network-alias db-master \
  postgres:14-alpine
```

---

### 3. 배치 전략

#### 제약 조건 (Constraints)

```bash
# Node ID
docker service create \
  --name web \
  --constraint 'node.id==abc123...' \
  nginx:alpine

# Node 역할
docker service create \
  --name monitor \
  --constraint 'node.role==manager' \
  prometheus:latest

# Node 호스트명
docker service create \
  --name db \
  --constraint 'node.hostname==db-server' \
  postgres:14-alpine

# Node 레이블
docker service create \
  --name cache \
  --constraint 'node.labels.type==ssd' \
  --constraint 'node.labels.env==prod' \
  redis:alpine

# 여러 조건 (AND)
docker service create \
  --name app \
  --constraint 'node.role==worker' \
  --constraint 'node.labels.region==us-east' \
  myapp:latest

# 부정 조건 (!=)
docker service create \
  --name web \
  --constraint 'node.role!=manager' \
  nginx:alpine
```

#### 배치 선호도 (Preferences)

```bash
# 레이블 기반 분산
docker service create \
  --name web \
  --replicas 6 \
  --placement-pref 'spread=node.labels.datacenter' \
  nginx:alpine

# zone별 분산
docker service create \
  --name api \
  --replicas 9 \
  --placement-pref 'spread=node.labels.zone' \
  node:18-alpine

# 여러 선호도
docker service create \
  --name app \
  --replicas 12 \
  --placement-pref 'spread=node.labels.datacenter' \
  --placement-pref 'spread=node.labels.rack' \
  myapp:latest
```

**Spread 동작:**

```
6 replicas, 3 datacenters:

datacenter=dc1: 2 tasks
datacenter=dc2: 2 tasks
datacenter=dc3: 2 tasks

균등 분산!
```

#### 리소스 제약

```bash
# CPU/메모리 제한
docker service create \
  --name web \
  --limit-cpu 0.5 \
  --limit-memory 512M \
  nginx:alpine

# 예약 (최소 보장)
docker service create \
  --name db \
  --reserve-cpu 1.0 \
  --reserve-memory 2G \
  --limit-cpu 2.0 \
  --limit-memory 4G \
  postgres:14-alpine

# 둘 다
docker service create \
  --name app \
  --reserve-cpu 0.5 \
  --reserve-memory 512M \
  --limit-cpu 1.0 \
  --limit-memory 1G \
  myapp:latest
```

---

### 4. 서비스 관리

#### 서비스 조회

```bash
# 서비스 목록
docker service ls

# 상세 정보
docker service inspect web

# JSON Pretty
docker service inspect web --pretty

# 특정 필드
docker service inspect web --format '{{.Spec.Mode}}'
# replicated

docker service inspect web --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'
# nginx:alpine

# 태스크 목록
docker service ps web

# 모든 태스크 (종료된 것 포함)
docker service ps web --filter desired-state=shutdown

# 특정 노드의 태스크
docker service ps web --filter node=worker1

# 로그
docker service logs web

# 실시간 로그
docker service logs -f web

# 타임스탬프 포함
docker service logs -t web

# 마지막 100줄
docker service logs --tail 100 web
```

#### 서비스 업데이트

```bash
# 이미지 업데이트
docker service update \
  --image nginx:1.23-alpine \
  web

# 레플리카 변경
docker service update \
  --replicas 5 \
  web

# 환경 변수 추가
docker service update \
  --env-add NODE_ENV=production \
  app

# 환경 변수 제거
docker service update \
  --env-rm DEBUG \
  app

# 포트 추가
docker service update \
  --publish-add 8443:443 \
  web

# 포트 제거
docker service update \
  --publish-rm 8080 \
  web

# 네트워크 추가
docker service update \
  --network-add backend \
  app

# 레이블 추가
docker service update \
  --label-add version=2.0 \
  app

# 제약 조건 변경
docker service update \
  --constraint-add 'node.labels.type==ssd' \
  db

# 여러 옵션 동시
docker service update \
  --image myapp:2.0 \
  --replicas 10 \
  --env-add VERSION=2.0 \
  app
```

#### 롤백

```bash
# 이전 버전으로 롤백
docker service rollback web

# 업데이트 + 자동 롤백 설정
docker service update \
  --image nginx:1.23-alpine \
  --update-failure-action rollback \
  --update-monitor 10s \
  web
```

#### 서비스 재시작

```bash
# Force update (재시작)
docker service update --force web

# 특정 태스크 재시작 (태스크 ID로)
# 1. 태스크 ID 확인
docker service ps web --format '{{.ID}}'

# 2. 태스크 제거 (자동 재생성)
docker service update \
  --force \
  --replicas 3 \
  web
```

---

## 💻 실습

### 실습 1: 기본 서비스 배포

```bash
# Swarm 초기화
docker swarm init

# 웹 서비스 생성 (3 replicas)
docker service create \
  --name web \
  --replicas 3 \
  --publish 8080:80 \
  nginx:alpine

# 확인
docker service ls
docker service ps web

# 테스트
curl http://localhost:8080

# 스케일 업
docker service scale web=5

# 확인
docker service ps web

# 로그
docker service logs -f web

# 정리
docker service rm web
docker swarm leave --force
```

---

### 실습 2: 배치 제약 조건

```bash
# Swarm 초기화
docker swarm init

# 노드 레이블 추가
NODE_ID=$(docker node ls -q)
docker node update --label-add type=ssd $NODE_ID
docker node update --label-add env=prod $NODE_ID
docker node update --label-add zone=us-east-1a $NODE_ID

# 제약 조건으로 서비스 생성
docker service create \
  --name db \
  --replicas 1 \
  --constraint 'node.labels.type==ssd' \
  --constraint 'node.labels.env==prod' \
  --reserve-memory 2G \
  --limit-memory 4G \
  postgres:14-alpine

# 확인
docker service ps db

# Global 서비스 (모든 노드)
docker service create \
  --name monitor \
  --mode global \
  --constraint 'node.labels.env==prod' \
  prom/node-exporter

# 확인
docker service ps monitor

# 정리
docker service rm db monitor
docker swarm leave --force
```

---

### 실습 3: 서비스 업데이트

```bash
# Swarm 초기화
docker swarm init

# 서비스 생성 (버전 1.21)
docker service create \
  --name web \
  --replicas 3 \
  --publish 8080:80 \
  --update-delay 10s \
  --update-parallelism 1 \
  nginx:1.21-alpine

# 확인
docker service ps web

# 이미지 업데이트 (1.22)
docker service update \
  --image nginx:1.22-alpine \
  web

# 업데이트 진행 확인
docker service ps web
# 하나씩 순차적으로 업데이트됨 (10초 간격)

# 롤백
docker service rollback web

# 확인
docker service ps web
# 1.21로 복구

# 정리
docker service rm web
docker swarm leave --force
```

---

## 🔥 실전 시나리오

### 시나리오 1: 3-Tier 애플리케이션

```bash
# Swarm 초기화
docker swarm init

# Overlay 네트워크 생성
docker network create --driver overlay frontend
docker network create --driver overlay backend

# Database (1 replica, backend 전용)
docker service create \
  --name db \
  --replicas 1 \
  --network backend \
  --constraint 'node.labels.type==database' \
  --reserve-memory 2G \
  --limit-memory 4G \
  --mount type=volume,source=db-data,target=/var/lib/postgresql/data \
  --env POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# API Server (5 replicas, frontend + backend)
docker service create \
  --name api \
  --replicas 5 \
  --network frontend \
  --network backend \
  --constraint 'node.role==worker' \
  --reserve-cpu 0.5 \
  --limit-cpu 1.0 \
  --env DB_HOST=db \
  --env DB_PASSWORD=secret \
  node:18-alpine

# Nginx (3 replicas, frontend, Ingress)
docker service create \
  --name web \
  --replicas 3 \
  --network frontend \
  --publish 80:80 \
  --constraint 'node.role==worker' \
  nginx:alpine

# 확인
docker service ls
docker service ps db
docker service ps api
docker service ps web

# 정리
docker service rm web api db
docker network rm frontend backend
docker volume rm db-data
docker swarm leave --force
```

---

### 시나리오 2: 모니터링 스택

```bash
# Swarm 초기화
docker swarm init

# Monitoring 네트워크
docker network create --driver overlay monitoring

# Node Exporter (Global - 모든 노드)
docker service create \
  --name node-exporter \
  --mode global \
  --network monitoring \
  --mount type=bind,source=/proc,target=/host/proc,readonly \
  --mount type=bind,source=/sys,target=/host/sys,readonly \
  --mount type=bind,source=/,target=/rootfs,readonly \
  prom/node-exporter \
    --path.procfs=/host/proc \
    --path.sysfs=/host/sys \
    --collector.filesystem.mount-points-exclude="^/(sys|proc|dev|host|etc)($|/)"

# Prometheus (1 replica)
docker service create \
  --name prometheus \
  --replicas 1 \
  --network monitoring \
  --publish 9090:9090 \
  --constraint 'node.role==manager' \
  --mount type=volume,source=prometheus-data,target=/prometheus \
  prom/prometheus

# Grafana (1 replica)
docker service create \
  --name grafana \
  --replicas 1 \
  --network monitoring \
  --publish 3000:3000 \
  --constraint 'node.role==manager' \
  --mount type=volume,source=grafana-data,target=/var/lib/grafana \
  --env GF_SECURITY_ADMIN_PASSWORD=admin \
  grafana/grafana

# 확인
docker service ls

# Prometheus: http://localhost:9090
# Grafana: http://localhost:3000 (admin/admin)

# 정리
docker service rm node-exporter prometheus grafana
docker network rm monitoring
docker volume rm prometheus-data grafana-data
docker swarm leave --force
```

---

## 🚫 안티패턴

### 1. 리소스 제한 없음

```bash
# ❌ 리소스 무제한
docker service create \
  --name app \
  --replicas 10 \
  myapp

# ✅ 리소스 제한
docker service create \
  --name app \
  --replicas 10 \
  --reserve-cpu 0.25 \
  --limit-cpu 0.5 \
  --reserve-memory 256M \
  --limit-memory 512M \
  myapp
```

### 2. 업데이트 설정 없음

```bash
# ❌ 한 번에 모두 업데이트
docker service update --image myapp:2.0 app
# 모든 레플리카 동시 중단!

# ✅ 롤링 업데이트
docker service create \
  --name app \
  --replicas 6 \
  --update-parallelism 2 \
  --update-delay 10s \
  --update-failure-action rollback \
  myapp:1.0
```

### 3. 헬스체크 없음

```bash
# ❌ 헬스체크 없음
docker service create \
  --name web \
  nginx

# ✅ 헬스체크 포함
docker service create \
  --name web \
  --health-cmd "curl -f http://localhost/ || exit 1" \
  --health-interval 30s \
  --health-timeout 10s \
  --health-retries 3 \
  nginx
```

### 4. 네트워크 격리 없음

```bash
# ❌ 모든 서비스 같은 네트워크
docker service create --name web nginx
docker service create --name api node
docker service create --name db postgres

# ✅ 네트워크 격리
docker network create --driver overlay frontend
docker network create --driver overlay backend

docker service create --name web --network frontend nginx
docker service create --name api --network frontend --network backend node
docker service create --name db --network backend postgres
```

---

## 🎓 핵심 정리

### 1. Service 모드

```
Replicated:
- 지정한 개수만큼
- 스케일 가능
- 로드 밸런싱

Global:
- 모든 노드에 1개
- 스케일 불가
- 모니터링, 로깅 에이전트
```

### 2. 주요 명령어

```bash
# 생성
docker service create --name <n> --replicas <N> <image>

# 확인
docker service ls
docker service ps <n>

# 업데이트
docker service update --image <img> <n>
docker service scale <n>=<N>

# 롤백
docker service rollback <n>

# 삭제
docker service rm <n>
```

### 3. 배치 전략

```
Constraints:
- node.role
- node.hostname
- node.labels.*

Preferences:
- spread=node.labels.*

Resources:
- --reserve-cpu/memory
- --limit-cpu/memory
```

### 4. Best Practices

```
✅ 헬스체크 설정
✅ 롤링 업데이트 설정
✅ 리소스 제한
✅ 네트워크 격리
✅ 레이블 활용
✅ 로그 중앙화
```

---

<div align="center">

**[⬅️ 이전: Docker Swarm](./03-Docker-Swarm.md)** | **[다음: Swarm Networking ➡️](./05-Swarm-Networking.md)**

</div>
