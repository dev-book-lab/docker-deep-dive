# 03. Docker Swarm - 클러스터 구성

## 🎯 이 챕터에서 배울 것

- **Docker Swarm**이란 무엇인가
- **클러스터 구성** (Manager/Worker 노드)
- **Swarm 초기화**와 노드 관리
- **기본 개념**과 아키텍처

## 📌 왜 중요한가?

**"Docker Swarm은 여러 Docker 호스트를 하나의 클러스터로 관리하는 네이티브 오케스트레이션 도구입니다."**

```
단일 호스트 vs Swarm 클러스터:

단일 호스트 (docker-compose):
┌─────────────────────────────────────┐
│ Single Docker Host                  │
│ ┌─────────┐  ┌─────────┐            │
│ │   App   │  │   DB    │            │
│ └─────────┘  └─────────┘            │
└─────────────────────────────────────┘
❌ 단일 실패점 (SPOF)
❌ 확장성 제한
❌ 고가용성 없음
❌ 로드 밸런싱 수동

Swarm 클러스터:
┌─────────────────────────────────────┐
│ Manager Node 1 (Leader)             │
│ ┌─────────┐  ┌─────────┐            │
│ │   App   │  │   DB    │            │
│ └─────────┘  └─────────┘            │
└────────────┬────────────────────────┘
             │
    ┌────────┼──────────┐
    │        │          │
┌───▼────┐ ┌─▼──────┐ ┌─▼──────┐
│Manager │ │Worker  │ │Worker  │
│Node 2  │ │Node 1  │ │Node 2  │
│┌──────┐│ │┌──────┐│ │┌──────┐│
││ App  ││ ││ App  ││ ││ App  ││
│└──────┘│ │└──────┘│ │└──────┘│
└────────┘ └────────┘ └────────┘

✅ 고가용성 (HA)
✅ 자동 스케일링
✅ 장애 복구
✅ 내장 로드 밸런싱
✅ 롤링 업데이트
✅ Secret/Config 관리

Swarm의 핵심 가치:

1. 고가용성 (High Availability):
   - 여러 Manager (Raft consensus)
   - 자동 리더 선출
   - 노드 장애 시 자동 복구

2. 확장성 (Scalability):
   - 서비스 레플리카 증가
   - 노드 추가로 용량 확장
   - 자동 로드 밸런싱

3. 간편성 (Simplicity):
   - Docker 내장
   - 별도 설치 불필요
   - docker-compose 호환

4. 보안 (Security):
   - TLS 자동 생성
   - 암호화 통신
   - Secret 관리

실무 적용:
- 소규모~중규모 프로덕션
- 마이크로서비스 배포
- 고가용성 웹 서비스
- CI/CD 파이프라인
```

**실무 영향:**
- 무중단 배포
- 자동 장애 복구
- 수평 확장
- 보안 강화

---

## 🔬 Deep Dive

### 1. Swarm 아키텍처

#### 핵심 개념

```
Swarm 클러스터 구조:

┌────────────────────────────────────────┐
│ Manager Nodes (관리 평면)                │
│ ┌─────────┐┌───────────┐ ┌────────────┐│
│ │Manager 1││Manager 2  │ │Manager 3   ││
│ │(Leader) ││(Reachable)│ │(Reachable) ││
│ └────┬────┘└────┬──────┘ └────┬───────┘│
│      │          │             │        │
│      └──────────┼─────────────┘        │
│            Raft Consensus              │
└────────────────┬───────────────────────┘
                 │
        ┌────────┼──────────┐
        │        │          │
┌───────▼──┐ ┌───▼─────┐ ┌──▼──────┐
│Worker 1  │ │Worker 2 │ │Worker 3 │
│(데이터평면) │ │(데이터평면)│ │(데이터평면)│
└──────────┘ └─────────┘ └─────────┘

핵심 구성 요소:

1. Node (노드):
   - Manager: 클러스터 관리, 스케줄링
   - Worker: 컨테이너 실행

2. Service (서비스):
   - Replicated: N개 복제본
   - Global: 모든 노드에 1개

3. Task (태스크):
   - 서비스의 단일 인스턴스
   - 실제 컨테이너

4. Stack (스택):
   - 여러 서비스 그룹
   - docker-compose.yml 사용
```

#### Manager vs Worker

```
Manager Node:
┌─────────────────────────────────────┐
│ 역할:                                │
│ - 클러스터 상태 관리                    │
│ - 서비스 스케줄링                       │
│ - API 엔드포인트                       │
│ - Raft 합의 알고리즘                    │
│                                     │
│ 특징:                                │
│ - 홀수 개 권장 (3, 5, 7)               │
│ - 쿼럼 유지 필요 (과반수)                │
│ - 리더 자동 선출                       │
│ - Worker 역할도 가능                   │
└─────────────────────────────────────┘

Worker Node:
┌─────────────────────────────────────┐
│ 역할:                                │
│ - 태스크(컨테이너) 실행                  │
│ - Manager 명령 수신                   │
│                                     │
│ 특징:                                │
│ - 개수 제한 없음                       │
│ - 순수 실행 노드                       │
│ - Manager로 승격 가능                  │
└─────────────────────────────────────┘

권장 구성:
- 개발: Manager 1개
- 테스트: Manager 1개 + Worker 2개
- 스테이징: Manager 3개 + Worker N개
- 프로덕션: Manager 3~5개 + Worker N개
```

#### Raft Consensus

```
Raft 합의 알고리즘:

┌─────────┐  ┌─────────┐  ┌─────────┐
│Manager 1│  │Manager 2│  │Manager 3│
│(Leader) │  │         │  │         │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     │ 1. Propose │            │
     │──────────→ │            │
     │            │ 2. Vote    │
     │            │──────────→ │
     │            │            │
     │ 3. Commit  │            │
     │←─────────────────────── │
     │            │            │

특징:
- 쿼럼 기반 (과반수 동의)
- 리더 선출
- 로그 복제
- 장애 허용

쿼럼 계산:
- 1개: 쿼럼 1, 장애 허용 0
- 3개: 쿼럼 2, 장애 허용 1
- 5개: 쿼럼 3, 장애 허용 2
- 7개: 쿼럼 4, 장애 허용 3

과반수 유지 중요!
```

---

### 2. Swarm 초기화

#### 단일 노드 Swarm

```bash
# Swarm 초기화
docker swarm init

# 출력:
# Swarm initialized: current node (abc123...) is now a manager.
# 
# To add a worker to this swarm, run the following command:
#     docker swarm join --token SWMTKN-1-... 192.168.1.100:2377
# 
# To add a manager to this swarm, run 'docker swarm join-token manager'
# and follow the instructions.

# 노드 확인
docker node ls
# ID              HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc123... *     node1      Ready   Active        Leader

# Swarm 정보
docker info | grep -A 10 Swarm
# Swarm: active
#  NodeID: abc123...
#  Is Manager: true
#  ClusterID: def456...
#  Managers: 1
#  Nodes: 1
```

#### 멀티 노드 Swarm

```bash
# ========================================
# Node 1 (Manager) - 192.168.1.101
# ========================================

# Swarm 초기화 (advertise-addr 명시)
docker swarm init --advertise-addr 192.168.1.101

# Join 토큰 확인
docker swarm join-token worker
# docker swarm join --token SWMTKN-1-xxxxx 192.168.1.101:2377

docker swarm join-token manager
# docker swarm join --token SWMTKN-1-yyyyy 192.168.1.101:2377

# ========================================
# Node 2 (Worker) - 192.168.1.102
# ========================================

# Worker로 Join
docker swarm join \
  --token SWMTKN-1-xxxxx \
  192.168.1.101:2377

# 출력:
# This node joined a swarm as a worker.

# ========================================
# Node 3 (Worker) - 192.168.1.103
# ========================================

docker swarm join \
  --token SWMTKN-1-xxxxx \
  192.168.1.101:2377

# ========================================
# Node 1 (Manager) - 노드 확인
# ========================================

docker node ls
# ID              HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc123... *     node1      Ready   Active        Leader
# def456...       node2      Ready   Active        
# ghi789...       node3      Ready   Active
```

#### 네트워크 요구사항

```
Swarm 포트:

Control Plane:
- TCP 2377: 클러스터 관리 (Manager 전용)

Data Plane:
- TCP/UDP 7946: 노드 간 통신 (Gossip)
- UDP 4789: Overlay 네트워크 (VXLAN)

방화벽 설정 (iptables):
```

```bash
# Manager Node
sudo iptables -A INPUT -p tcp --dport 2377 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 7946 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 7946 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4789 -j ACCEPT

# Worker Node
sudo iptables -A INPUT -p tcp --dport 7946 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 7946 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4789 -j ACCEPT

# 영구 저장
sudo iptables-save > /etc/iptables/rules.v4
```

---

### 3. 노드 관리

#### 노드 조회

```bash
# 노드 목록
docker node ls
# ID              HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc123... *     manager1   Ready   Active        Leader
# def456...       manager2   Ready   Active        Reachable
# ghi789...       manager3   Ready   Active        Reachable
# jkl012...       worker1    Ready   Active        
# mno345...       worker2    Ready   Active

# 특정 노드 상세 정보
docker node inspect manager1

# JSON 형식으로
docker node inspect manager1 --pretty

# 특정 필드만
docker node inspect manager1 --format '{{.Status.State}}'
# ready

docker node inspect manager1 --format '{{.ManagerStatus.Leader}}'
# true
```

#### 노드 레이블

```bash
# 레이블 추가
docker node update \
  --label-add region=us-east \
  --label-add zone=us-east-1a \
  worker1

# 레이블 확인
docker node inspect worker1 --format '{{.Spec.Labels}}'
# map[region:us-east zone:us-east-1a]

# 여러 레이블
docker node update \
  --label-add type=ssd \
  --label-add env=prod \
  worker1

# 레이블 제거
docker node update --label-rm zone worker1
```

#### 노드 가용성 (Availability)

```bash
# Active (기본): 태스크 스케줄 가능
docker node update --availability active worker1

# Pause: 새 태스크 스케줄 안 됨, 기존은 유지
docker node update --availability pause worker1

# Drain: 모든 태스크 다른 노드로 이동
docker node update --availability drain worker1

# 확인
docker node ls
# ID         HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
# abc123...  worker1   Ready   Drain
```

#### 노드 승격/강등

```bash
# Worker → Manager 승격
docker node promote worker1

# 확인
docker node ls
# ID         HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
# abc123...  worker1   Ready   Active        Reachable

# Manager → Worker 강등
docker node demote worker1

# 또는 직접 역할 변경
docker node update --role manager worker2
docker node update --role worker worker2
```

#### 노드 제거

```bash
# Worker 노드에서 (자체 제거)
docker swarm leave

# Manager에서 확인
docker node ls
# ID         HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
# abc123...  worker1   Down    Active

# 노드 삭제 (Manager에서)
docker node rm worker1

# 강제 삭제
docker node rm --force worker1

# Manager 노드 제거 (신중!)
# 1. 먼저 강등
docker node demote manager2

# 2. 노드에서 나가기
docker swarm leave

# 3. 다른 Manager에서 삭제
docker node rm manager2
```

---

### 4. Swarm 관리

#### Swarm 정보

```bash
# Swarm 상태
docker info | grep -A 20 Swarm
# Swarm: active
#  NodeID: abc123...
#  Is Manager: true
#  ClusterID: def456...
#  Managers: 3
#  Nodes: 5
#  Default Address Pool: 10.0.0.0/8
#  SubnetSize: 24
#  Data Path Port: 4789
#  Orchestration:
#   Task History Retention Limit: 5
#  Raft:
#   Snapshot Interval: 10000
#   Number of Old Snapshots to Retain: 0
#   Heartbeat Tick: 1
#   Election Tick: 10
#  Dispatcher:
#   Heartbeat Period: 5 seconds
#  CA Configuration:
#   Expiry Duration: 3 months
#   Force Rotate: 0

# Join 토큰 갱신
docker swarm join-token --rotate worker
docker swarm join-token --rotate manager

# CA 인증서 갱신
docker swarm ca --rotate
```

#### 자동 잠금 (Auto-lock)

```bash
# Swarm 잠금 활성화
docker swarm update --autolock=true

# 출력:
# Swarm updated.
# To unlock a swarm manager after it restarts, run:
#   docker swarm unlock
# and provide the following key:
#   SWMKEY-1-...
# 
# Please remember to store this key in a password manager.

# Manager 재시작 후
docker swarm unlock
# Please enter unlock key: SWMKEY-1-...

# 잠금 키 확인
docker swarm unlock-key

# 잠금 비활성화
docker swarm update --autolock=false
```

---

## 💻 실습

### 실습 1: 단일 노드 Swarm

```bash
# Swarm 초기화
docker swarm init

# 노드 확인
docker node ls
# ID              HOSTNAME        STATUS  AVAILABILITY  MANAGER STATUS
# abc123... *     docker-desktop  Ready   Active        Leader

# 간단한 서비스 생성
docker service create \
  --name web \
  --replicas 3 \
  --publish 8080:80 \
  nginx:alpine

# 서비스 확인
docker service ls
# ID         NAME  MODE         REPLICAS  IMAGE
# xyz789...  web   replicated   3/3       nginx:alpine

# 서비스 상세
docker service ps web
# ID         NAME    IMAGE          NODE            DESIRED STATE  CURRENT STATE
# abc...     web.1   nginx:alpine   docker-desktop  Running        Running
# def...     web.2   nginx:alpine   docker-desktop  Running        Running
# ghi...     web.3   nginx:alpine   docker-desktop  Running        Running

# 테스트
curl http://localhost:8080
# (Nginx 기본 페이지)

# 정리
docker service rm web
docker swarm leave --force
```

---

### 실습 2: 멀티 노드 Swarm (Docker-in-Docker)

```yaml
# docker-compose.yml
version: '3.8'

services:
  manager1:
    image: docker:dind
    privileged: true
    hostname: manager1
    networks:
      - swarm-net
    environment:
      DOCKER_TLS_CERTDIR: ""
    command: dockerd --host=tcp://0.0.0.0:2375

  worker1:
    image: docker:dind
    privileged: true
    hostname: worker1
    networks:
      - swarm-net
    environment:
      DOCKER_TLS_CERTDIR: ""
    command: dockerd --host=tcp://0.0.0.0:2375

  worker2:
    image: docker:dind
    privileged: true
    hostname: worker2
    networks:
      - swarm-net
    environment:
      DOCKER_TLS_CERTDIR: ""
    command: dockerd --host=tcp://0.0.0.0:2375

networks:
  swarm-net:
    driver: bridge
```

```bash
# 클러스터 시작
docker-compose up -d

# Manager 컨테이너 접속
docker-compose exec manager1 sh

# Swarm 초기화
docker swarm init --advertise-addr eth0

# Join 토큰 확인
docker swarm join-token worker -q
# SWMTKN-1-...

# 토큰 저장
TOKEN=$(docker swarm join-token worker -q)
MANAGER_IP=$(docker network inspect swarm-net \
  --format '{{range .Containers}}{{if eq .Name "manager1"}}{{.IPv4Address}}{{end}}{{end}}' \
  | cut -d'/' -f1)

# 종료
exit

# Worker1 접속 및 Join
docker-compose exec worker1 sh
docker swarm join --token $TOKEN $MANAGER_IP:2377
exit

# Worker2 접속 및 Join
docker-compose exec worker2 sh
docker swarm join --token $TOKEN $MANAGER_IP:2377
exit

# Manager에서 노드 확인
docker-compose exec manager1 docker node ls
# ID              HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc123... *     manager1   Ready   Active        Leader
# def456...       worker1    Ready   Active        
# ghi789...       worker2    Ready   Active

# 정리
docker-compose down
```

---

### 실습 3: 노드 레이블과 제약 조건

```bash
# Swarm 초기화 (단일 노드로 테스트)
docker swarm init

# 자기 노드에 레이블 추가
NODE_ID=$(docker node ls -q)
docker node update --label-add type=ssd $NODE_ID
docker node update --label-add env=prod $NODE_ID
docker node update --label-add region=us-east $NODE_ID

# 레이블 확인
docker node inspect $NODE_ID --format '{{.Spec.Labels}}'
# map[env:prod region:us-east type:ssd]

# 제약 조건으로 서비스 생성
docker service create \
  --name db \
  --constraint 'node.labels.type==ssd' \
  --constraint 'node.labels.env==prod' \
  postgres:14-alpine

# 서비스 확인
docker service ps db

# 정리
docker service rm db
docker swarm leave --force
```

---

## 🔥 실전 시나리오

### 시나리오 1: 3 Manager + 2 Worker 클러스터

```
클러스터 구성:
┌─────────────────────────────────────┐
│ Manager Nodes (고가용성)              │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│ │Manager 1│ │Manager 2│ │Manager 3│ │
│ │(Leader) │ │         │ │         │ │
│ └─────────┘ └─────────┘ └─────────┘ │
└─────────────────────────────────────┘
              │
    ┌─────────┴─────────┐
    │                   │
┌───▼─────┐       ┌─────▼───┐
│Worker 1 │       │Worker 2 │
│(DB 전용) │       │(Web전용) │
└─────────┘       └─────────┘

구성 단계:
```

```bash
# ========================================
# Manager 1 (192.168.1.101)
# ========================================
docker swarm init --advertise-addr 192.168.1.101

# Manager 토큰
MANAGER_TOKEN=$(docker swarm join-token manager -q)
echo $MANAGER_TOKEN

# ========================================
# Manager 2 (192.168.1.102)
# ========================================
docker swarm join \
  --token $MANAGER_TOKEN \
  192.168.1.101:2377

# ========================================
# Manager 3 (192.168.1.103)
# ========================================
docker swarm join \
  --token $MANAGER_TOKEN \
  192.168.1.101:2377

# ========================================
# Worker 1 (192.168.1.104)
# ========================================
WORKER_TOKEN=$(docker swarm join-token worker -q)

docker swarm join \
  --token $WORKER_TOKEN \
  192.168.1.101:2377

# 레이블 추가 (Manager 1에서)
docker node update --label-add type=database worker1

# ========================================
# Worker 2 (192.168.1.105)
# ========================================
docker swarm join \
  --token $WORKER_TOKEN \
  192.168.1.101:2377

# 레이블 추가
docker node update --label-add type=web worker2

# ========================================
# 클러스터 확인 (Manager 1)
# ========================================
docker node ls
# ID         HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc...  *  manager1   Ready   Active        Leader
# def...     manager2   Ready   Active        Reachable
# ghi...     manager3   Ready   Active        Reachable
# jkl...     worker1    Ready   Active        
# mno...     worker2    Ready   Active
```

---

### 시나리오 2: Swarm 유지보수

```bash
# ========================================
# 노드 유지보수 절차
# ========================================

# 1. Worker 노드 Drain (태스크 이동)
docker node update --availability drain worker1

# 2. 태스크 이동 확인
docker node ps worker1
# (모든 태스크가 Shutdown)

docker service ps web
# (다른 노드로 재배치)

# 3. 유지보수 작업 (worker1에서)
# - 시스템 업데이트
# - 하드웨어 점검
# - Docker 업그레이드

# 4. 유지보수 완료 후 Active
docker node update --availability active worker1

# 5. 태스크가 자동으로 재분배됨 (다음 스케일 시)

# ========================================
# Manager 노드 교체
# ========================================

# 1. 새 Manager 추가 (Manager4)
docker swarm join-token manager
# 새 노드에서 Join

# 2. 기존 Manager 확인
docker node ls
# 4 Managers

# 3. 교체할 Manager 강등
docker node demote manager2

# 4. Drain
docker node update --availability drain manager2

# 5. 노드 제거
# manager2에서:
docker swarm leave

# Leader에서:
docker node rm manager2

# 6. 확인
docker node ls
# 3 Managers (정상)
```

---

## 🚫 안티패턴

### 1. Manager 노드 짝수 개

```bash
# ❌ 2개 Manager
# - 쿼럼: 2
# - 장애 허용: 0
# - 1대 장애 시 클러스터 마비

# ✅ 3개 Manager
# - 쿼럼: 2
# - 장애 허용: 1
# - 1대 장애 시 정상 운영
```

### 2. 모든 Manager가 Worker 역할

```bash
# ❌ Manager가 워크로드 실행
# - 관리 평면 부하
# - 성능 저하

# ✅ Manager 전용 또는 최소 워크로드
docker node update --availability drain manager1
docker node update --availability drain manager2
docker node update --availability drain manager3
```

### 3. 자동 잠금 없음

```bash
# ❌ 보안 취약
# - Manager 재시작 시 자동 Join
# - Raft 로그 평문

# ✅ Auto-lock 활성화
docker swarm update --autolock=true
# 재시작 시 키 입력 필요
```

### 4. 네트워크 방화벽 미설정

```bash
# ❌ 포트 열려있음
# - 2377 외부 노출
# - 보안 위험

# ✅ 방화벽 설정
sudo ufw allow from 192.168.1.0/24 to any port 2377
sudo ufw allow from 192.168.1.0/24 to any port 7946
sudo ufw allow from 192.168.1.0/24 to any port 4789
```

---

## 🎓 핵심 정리

### 1. Swarm 핵심 개념

```
Node:
- Manager: 관리 + 실행
- Worker: 실행 전용

Raft Consensus:
- 홀수 Manager
- 쿼럼 유지
- 자동 리더 선출
```

### 2. 초기화

```bash
# Manager
docker swarm init

# Worker Join
docker swarm join --token <TOKEN> <IP>:2377

# 노드 확인
docker node ls
```

### 3. 노드 관리

```bash
# 레이블
docker node update --label-add <K>=<V> <NODE>

# 가용성
docker node update --availability <drain|pause|active>

# 승격/강등
docker node promote <NODE>
docker node demote <NODE>
```

### 4. Best Practices

```
✅ Manager 홀수 개 (3, 5, 7)
✅ Manager는 관리 전용
✅ Auto-lock 활성화
✅ 레이블 활용
✅ 정기 백업
✅ 방화벽 설정
```

---

<div align="center">

**[⬅️ 이전: Compose Advanced](./02-Compose-Advanced.md)** | **[다음: Swarm Services ➡️](./04-Swarm-Services.md)**

</div>
