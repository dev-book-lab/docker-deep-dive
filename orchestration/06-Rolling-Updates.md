# 06. Rolling Updates - 롤링 업데이트

## 🎯 이 챕터에서 배울 것

- **무중단 배포** (Zero-Downtime Deployment)
- **롤링 업데이트** 전략
- **롤백** 메커니즘
- **헬스체크**와 업데이트 제어

## 📌 왜 중요한가?

**"롤링 업데이트는 서비스 중단 없이 애플리케이션을 업데이트하는 핵심 기능입니다."**

```
전통적 배포 vs 롤링 업데이트:

전통적 배포 (Downtime):
┌─────────────────────────────────────┐
│ Step 1: 모든 인스턴스 중지              │
│ ┌───┐ ┌───┐ ┌───┐                   │
│ │ X │ │ X │ │ X │ (v1.0 종료)        │
│ └───┘ └───┘ └───┘                   │
└─────────────────────────────────────┘
        ⏱️ Downtime!
┌─────────────────────────────────────┐
│ Step 2: 새 버전 시작                   │
│ ┌───┐ ┌───┐ ┌───┐                   │
│ │ ✓ │ │ ✓ │ │ ✓ │ (v2.0 시작)        │
│ └───┘ └───┘ └───┘                   │
└─────────────────────────────────────┘
❌ 서비스 중단
❌ 사용자 불편
❌ 롤백 어려움

롤링 업데이트 (Zero-Downtime):
┌──────────────────────────────────────┐
│ Step 1: 1개씩 순차 업데이트              │
│ ┌───┐ ┌───┐ ┌───┐                   │
│ │ ✓ │ │OLD│ │OLD│                   │
│ │v2 │ │v1 │ │v1 │                   │
│ └───┘ └───┘ └───┘                   │
└─────────────────────────────────────┘
        ⏱️ 10초 대기
┌─────────────────────────────────────┐
│ Step 2: 다음 1개 업데이트               │
│ ┌───┐ ┌───┐ ┌───┐                   │
│ │ ✓ │ │ ✓ │ │OLD│                   │
│ │v2 │ │v2 │ │v1 │                   │
│ └───┘ └───┘ └───┘                   │
└─────────────────────────────────────┘
        ⏱️ 10초 대기
┌─────────────────────────────────────┐
│ Step 3: 마지막 1개 업데이트              │
│ ┌───┐ ┌───┐ ┌───┐                   │
│ │ ✓ │ │ ✓ │ │ ✓ │                   │
│ │v2 │ │v2 │ │v2 │                   │
│ └───┘ └───┘ └───┘                   │
└─────────────────────────────────────┘
✅ 무중단 서비스
✅ 점진적 배포
✅ 쉬운 롤백
✅ 모니터링 가능

롤링 업데이트의 핵심 가치:

1. 무중단 서비스:
   - 항상 일부 인스턴스 가동
   - 사용자 영향 최소화
   - 비즈니스 연속성

2. 위험 최소화:
   - 점진적 배포
   - 문제 조기 발견
   - 영향 범위 제한

3. 빠른 롤백:
   - 이전 버전 유지
   - 즉시 복구 가능
   - 자동 롤백 지원

4. 모니터링:
   - 단계별 확인
   - 헬스체크 통합
   - 실패 감지

실무 시나리오:

Production Deployment:
┌─────────────────────────────────────┐
│ 09:00 - 배포 시작                     │
│ v1.0 (6개) → v2.0 (1개씩)            │
│                                     │
│ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ │
│ │v2 │ │v1 │ │v1 │ │v1 │ │v1 │ │v1 │ │
│ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ │
│ ⏱️ 1분 대기 + 모니터링                  │
└─────────────────────────────────────┘
        ⬇️
┌─────────────────────────────────────┐
│ 09:06 - 전체 완료                     │
│ v2.0 (6개)                          │
│                                     │
│ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ │
│ │v2 │ │v2 │ │v2 │ │v2 │ │v2 │ │v2 │ │
│ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ │
│ ✅ 무중단 배포 완료                     │
└─────────────────────────────────────┘

문제 발생 시:
┌─────────────────────────────────────┐
│ 09:02 - v2.0 에러 감지!               │
│                                     │
│ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ │
│ │v2 │ │v2 │ │v1 │ │v1 │ │v1 │ │v1 │ │
│ │❌ │ │❌ │ └───┘ └───┘ └───┘ └───┘ │
│ └───┘ └───┘  (영향 제한)              │
└─────────────────────────────────────┘
        ⬇️ 자동 롤백
┌─────────────────────────────────────┐
│ 09:03 - 롤백 완료                     │
│                                     │
│ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ │
│ │v1 │ │v1 │ │v1 │ │v1 │ │v1 │ │v1 │ │
│ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ │
│ ✅ 안전하게 복구                       │
└─────────────────────────────────────┘
```

**실무 영향:**
- 24/7 서비스 가능
- 위험 관리
- 고객 만족도
- DevOps 효율성

---

## 🔬 Deep Dive

### 1. 롤링 업데이트 기본

#### 기본 동작

```bash
# Swarm 초기화
docker swarm init

# 서비스 생성 (v1.0)
docker service create \
  --name web \
  --replicas 6 \
  --publish 8080:80 \
  nginx:1.21-alpine

# 현재 상태 확인
docker service ps web

# 이미지 업데이트 (v1.22)
docker service update \
  --image nginx:1.22-alpine \
  web

# 업데이트 진행 확인 (실시간)
watch docker service ps web
# 하나씩 순차적으로 업데이트됨

# 완료 후 확인
docker service ps web
# 모두 1.22-alpine

# 정리
docker service rm web
```

#### 업데이트 설정

```bash
# 상세 설정으로 서비스 생성
docker service create \
  --name web \
  --replicas 6 \
  --publish 8080:80 \
  --update-parallelism 2 \      # 동시 업데이트 개수
  --update-delay 10s \            # 업데이트 간 대기
  --update-failure-action pause \ # 실패 시 일시 정지
  --update-monitor 5s \           # 모니터링 기간
  --update-max-failure-ratio 0.2 \ # 실패 허용 비율 (20%)
  nginx:1.21-alpine

# 업데이트 실행
docker service update --image nginx:1.22-alpine web

# 진행 상황:
# [===>        ] 2/6
# - 2개 동시 업데이트
# - 완료 후 10초 대기
# - 다음 2개 업데이트
```

---

### 2. 업데이트 전략

#### Parallelism (동시 업데이트 수)

```bash
# 1개씩 (가장 안전)
docker service create \
  --name web \
  --replicas 10 \
  --update-parallelism 1 \
  --update-delay 30s \
  nginx:alpine
# 업데이트 시간: 10 * 30s = 5분

# 2개씩 (균형)
docker service update \
  --update-parallelism 2 \
  --update-delay 15s \
  web
# 업데이트 시간: 5 * 15s = 1분 15초

# 5개씩 (빠름, 위험)
docker service update \
  --update-parallelism 5 \
  --update-delay 5s \
  web
# 업데이트 시간: 2 * 5s = 10초

# 전체 동시 (가장 빠름, 가장 위험)
docker service update \
  --update-parallelism 10 \
  web
# 업데이트 시간: 즉시
# ⚠️ Downtime 발생 가능!
```

#### Update Delay (대기 시간)

```bash
# 짧은 대기 (빠른 배포)
docker service create \
  --name api \
  --replicas 20 \
  --update-parallelism 5 \
  --update-delay 5s \
  node:18-alpine
# 5개씩, 5초 간격
# 총 시간: 4 * 5s = 20초

# 긴 대기 (안전한 배포)
docker service update \
  --update-parallelism 2 \
  --update-delay 60s \
  api
# 2개씩, 1분 간격
# 총 시간: 10 * 60s = 10분
# 충분한 모니터링 시간
```

#### Failure Action (실패 처리)

```bash
# Pause (일시 정지, 기본)
docker service create \
  --name web \
  --replicas 6 \
  --update-parallelism 2 \
  --update-failure-action pause \
  nginx:alpine
# 실패 시: 업데이트 중지, 수동 개입 필요

# Continue (계속 진행)
docker service update \
  --update-failure-action continue \
  web
# 실패 시: 나머지 계속 업데이트
# ⚠️ 혼합 버전 상태 가능

# Rollback (자동 롤백)
docker service update \
  --update-failure-action rollback \
  web
# 실패 시: 자동으로 이전 버전 복구
# ✅ 가장 안전
```

---

### 3. 헬스체크와 모니터링

#### 헬스체크 통합

```bash
# 헬스체크 포함 서비스
docker service create \
  --name web \
  --replicas 6 \
  --publish 8080:80 \
  --update-parallelism 2 \
  --update-delay 10s \
  --update-monitor 30s \         # 30초 동안 모니터링
  --update-failure-action rollback \
  --health-cmd "curl -f http://localhost/ || exit 1" \
  --health-interval 10s \
  --health-timeout 5s \
  --health-retries 3 \
  --health-start-period 30s \
  nginx:alpine

# 헬스체크 통과해야 다음 업데이트 진행
# 실패 시 자동 롤백
```

#### Start First (시작 우선)

```bash
# Start First 전략
docker service create \
  --name web \
  --replicas 6 \
  --update-order start-first \
  nginx:alpine

# 동작:
# 1. 새 태스크 시작
# 2. 헬스체크 통과 확인
# 3. 구 태스크 종료
# ✅ 항상 N개 이상 가동

# Stop First (기본)
docker service update \
  --update-order stop-first \
  web

# 동작:
# 1. 구 태스크 종료
# 2. 새 태스크 시작
# ⚠️ 일시적으로 N-1개
```

---

### 4. 롤백

#### 수동 롤백

```bash
# 서비스 생성 (v1.0)
docker service create \
  --name web \
  --replicas 6 \
  --publish 8080:80 \
  nginx:1.21-alpine

# 업데이트 (v1.22)
docker service update --image nginx:1.22-alpine web

# 문제 발견! 롤백
docker service rollback web

# 이전 버전(1.21)으로 복구됨
docker service ps web
```

#### 자동 롤백

```bash
# 자동 롤백 설정
docker service create \
  --name web \
  --replicas 6 \
  --publish 8080:80 \
  --update-parallelism 2 \
  --update-delay 10s \
  --update-failure-action rollback \
  --update-monitor 20s \
  --update-max-failure-ratio 0.3 \
  --health-cmd "curl -f http://localhost/ || exit 1" \
  --health-interval 10s \
  nginx:1.21-alpine

# 잘못된 이미지로 업데이트
docker service update --image nginx:invalid-tag web

# 자동으로 롤백됨!
# 1. 헬스체크 실패 감지
# 2. 실패율 30% 초과
# 3. 자동 롤백 시작
# 4. 1.21로 복구

docker service ps web
# 실패한 태스크들: Shutdown
# 롤백된 태스크들: Running (1.21)
```

#### 롤백 설정

```bash
# 롤백 전략 설정
docker service create \
  --name web \
  --replicas 6 \
  --update-parallelism 2 \
  --update-delay 10s \
  --update-failure-action rollback \
  --rollback-parallelism 3 \      # 롤백 시 동시 3개
  --rollback-delay 5s \            # 롤백 간 5초
  --rollback-monitor 10s \         # 롤백 모니터링
  --rollback-max-failure-ratio 0 \ # 롤백은 실패 허용 안 함
  nginx:alpine

# 롤백은 업데이트보다 빠르게!
```

---

## 💻 실습

### 실습 1: 기본 롤링 업데이트

```bash
# Swarm 초기화
docker swarm init

# 서비스 생성 (nginx 1.21)
docker service create \
  --name web \
  --replicas 6 \
  --publish 8080:80 \
  --update-parallelism 2 \
  --update-delay 10s \
  nginx:1.21-alpine

# 현재 버전 확인
docker service ps web

# 업데이트 시작 (nginx 1.22)
docker service update --image nginx:1.22-alpine web

# 실시간 모니터링
watch -n 1 'docker service ps web | head -10'
# 2개씩 업데이트되는 것 확인

# 완료 확인
docker service ps web | grep Running
# 모두 1.22-alpine

# 정리
docker service rm web
docker swarm leave --force
```

---

### 실습 2: 헬스체크와 자동 롤백

```bash
# Swarm 초기화
docker swarm init

# 헬스체크 포함 서비스
docker service create \
  --name web \
  --replicas 4 \
  --publish 8080:80 \
  --update-parallelism 1 \
  --update-delay 15s \
  --update-monitor 20s \
  --update-failure-action rollback \
  --update-max-failure-ratio 0.25 \
  --health-cmd "curl -f http://localhost/ || exit 1" \
  --health-interval 10s \
  --health-timeout 5s \
  --health-retries 2 \
  nginx:1.21-alpine

# 정상 업데이트 테스트 (1.22)
docker service update --image nginx:1.22-alpine web

# 확인
docker service ps web
# ✅ 모두 1.22로 업데이트됨

# 잘못된 업데이트 (invalid)
docker service update --image nginx:invalid-tag web

# 실시간 확인
watch -n 1 'docker service ps web'
# 1. 새 태스크 시작 시도
# 2. 이미지 pull 실패
# 3. 자동 롤백 시작
# 4. 1.22로 복구

# 최종 상태
docker service ps web | grep Running
# 모두 1.22 (롤백 완료)

# 정리
docker service rm web
docker swarm leave --force
```

---

### 실습 3: Blue-Green 스타일 업데이트

```bash
# Swarm 초기화
docker swarm init

# Blue (현재 버전)
docker service create \
  --name web-blue \
  --replicas 3 \
  --label color=blue \
  nginx:1.21-alpine

# Green (새 버전) 준비
docker service create \
  --name web-green \
  --replicas 3 \
  --label color=green \
  nginx:1.22-alpine

# Green 테스트
docker service ps web-green
# 모두 정상 Running

# 트래픽 전환 (서비스 교체)
docker service update \
  --publish-add 8080:80 \
  web-green

docker service update \
  --publish-rm 8080 \
  web-blue

# Green으로 완전 전환됨

# 문제 없으면 Blue 제거
docker service rm web-blue

# 또는 롤백이 필요하면
docker service update --publish-rm 8080 web-green
docker service update --publish-add 8080:80 web-blue

# 정리
docker service rm web-blue web-green
docker swarm leave --force
```

---

## 🔥 실전 시나리오

### 시나리오 1: 프로덕션 배포 전략

```bash
# Swarm 초기화
docker swarm init

# 프로덕션 서비스 생성
docker service create \
  --name api \
  --replicas 10 \
  --publish 80:3000 \
  \
  # 업데이트 설정 (보수적)
  --update-parallelism 2 \        # 2개씩
  --update-delay 60s \             # 1분 대기
  --update-order start-first \     # 새 태스크 먼저
  --update-monitor 30s \           # 30초 모니터링
  --update-failure-action rollback \
  --update-max-failure-ratio 0.1 \ # 10% 실패 허용
  \
  # 롤백 설정 (빠르게)
  --rollback-parallelism 5 \       # 5개씩
  --rollback-delay 10s \           # 10초 간격
  --rollback-monitor 15s \
  --rollback-max-failure-ratio 0 \
  \
  # 헬스체크
  --health-cmd "curl -f http://localhost:3000/health || exit 1" \
  --health-interval 15s \
  --health-timeout 10s \
  --health-retries 3 \
  --health-start-period 60s \
  \
  myapp:1.0

# 배포 시작
docker service update --image myapp:2.0 api

# 모니터링
watch -n 2 'echo "=== Service Status ===" && \
             docker service ps api --filter desired-state=running && \
             echo && echo "=== Update Progress ===" && \
             docker service inspect api --format "{{.UpdateStatus}}"'

# 업데이트 진행:
# 09:00 - 시작 (2개)
# 09:01 - 다음 2개
# 09:02 - 다음 2개
# ...
# 09:05 - 완료 (총 5분)

# 정리
docker service rm api
```

---

### 시나리오 2: 카나리 배포

```yaml
# stack.yml
version: '3.8'

services:
  # Stable (90%)
  web-stable:
    image: nginx:1.21-alpine
    deploy:
      replicas: 9
      labels:
        version: stable
        traffic: "90%"
      update_config:
        parallelism: 2
        delay: 30s

  # Canary (10%)
  web-canary:
    image: nginx:1.22-alpine
    deploy:
      replicas: 1
      labels:
        version: canary
        traffic: "10%"

  # Load Balancer
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    configs:
      - source: nginx-lb-config
        target: /etc/nginx/nginx.conf
    depends_on:
      - web-stable
      - web-canary

configs:
  nginx-lb-config:
    file: ./nginx-lb.conf
```

```nginx
# nginx-lb.conf
events {
    worker_connections 1024;
}

http {
    upstream backend {
        # 90% to stable
        server web-stable:80 weight=9;
        
        # 10% to canary
        server web-canary:80 weight=1;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
        }
    }
}
```

```bash
# 배포
docker stack deploy -c stack.yml canary-test

# 트래픽 확인
for i in {1..100}; do
  curl -s http://localhost/ | grep Server
done | sort | uniq -c
# ~90% stable
# ~10% canary

# 카나리 정상이면 점진적 증가
docker service scale canary-test_web-canary=3
docker service scale canary-test_web-stable=7
# 이제 30% vs 70%

# 최종적으로 전체 전환
docker service scale canary-test_web-canary=10
docker service scale canary-test_web-stable=0

# 정리
docker stack rm canary-test
rm nginx-lb.conf
```

---

### 시나리오 3: 데이터베이스 업그레이드

```bash
# Swarm 초기화
docker swarm init

# 네트워크 생성
docker network create --driver overlay db-net

# PostgreSQL 13 시작
docker service create \
  --name db \
  --network db-net \
  --replicas 1 \
  --mount type=volume,source=db-data,target=/var/lib/postgresql/data \
  --env POSTGRES_PASSWORD=secret \
  postgres:13-alpine

# 애플리케이션
docker service create \
  --name app \
  --network db-net \
  --replicas 5 \
  --env DB_HOST=db \
  --env DB_PASSWORD=secret \
  node:18-alpine

# 데이터베이스 업그레이드 (주의!)
# 1. 백업
docker exec $(docker ps -q -f name=db) \
  pg_dumpall -U postgres > backup.sql

# 2. Read-Only 모드 (선택)
docker service update \
  --env-add POSTGRES_READ_ONLY=1 \
  db

# 3. 새 버전으로 업데이트
docker service update \
  --image postgres:14-alpine \
  --update-delay 60s \
  db

# 4. 마이그레이션 확인
docker exec $(docker ps -q -f name=db) \
  psql -U postgres -c "SELECT version();"

# 5. Read-Write 복원
docker service update \
  --env-rm POSTGRES_READ_ONLY \
  db

# 정리
docker service rm app db
docker network rm db-net
docker volume rm db-data
rm backup.sql
```

---

## 🚫 안티패턴

### 1. 업데이트 설정 없음

```bash
# ❌ 기본 설정 (한 번에 모두)
docker service create --name web --replicas 10 nginx
docker service update --image nginx:new web
# 모든 레플리카 동시 업데이트!

# ✅ 롤링 업데이트 설정
docker service create \
  --name web \
  --replicas 10 \
  --update-parallelism 2 \
  --update-delay 30s \
  --update-failure-action rollback \
  nginx
```

### 2. 헬스체크 없음

```bash
# ❌ 헬스체크 없이 업데이트
docker service update --image myapp:buggy web
# 잘못된 버전 배포!

# ✅ 헬스체크 포함
docker service create \
  --health-cmd "curl -f http://localhost/health" \
  --health-interval 10s \
  --update-monitor 30s \
  --update-failure-action rollback \
  myapp
```

### 3. 롤백 준비 없음

```bash
# ❌ 롤백 불가
docker service update --image app:2.0 web
# 문제 발생 시 수동 복구 필요

# ✅ 자동 롤백
docker service update \
  --update-failure-action rollback \
  --rollback-parallelism 5 \
  --image app:2.0 \
  web
```

### 4. 모니터링 없음

```bash
# ❌ 업데이트 후 방치
docker service update --image app:2.0 web
# 문제를 나중에 발견

# ✅ 실시간 모니터링
docker service update --image app:2.0 web &
watch -n 1 'docker service ps web && docker service inspect web --format "{{.UpdateStatus}}"'
```

---

## 🎓 핵심 정리

### 1. 업데이트 전략

```
Parallelism:
- 1: 가장 안전, 느림
- N/2: 균형
- N: 가장 빠름, 위험

Delay:
- 짧음: 빠른 배포
- 길음: 안전한 배포

Failure Action:
- pause: 수동 개입
- continue: 계속 진행
- rollback: 자동 복구 (권장)
```

### 2. 주요 명령어

```bash
# 업데이트
docker service update --image <img> <svc>

# 롤백
docker service rollback <svc>

# 설정
--update-parallelism <N>
--update-delay <time>
--update-failure-action <action>
--update-monitor <time>
--update-max-failure-ratio <ratio>
```

### 3. 헬스체크

```bash
--health-cmd <cmd>
--health-interval <time>
--health-timeout <time>
--health-retries <N>
--health-start-period <time>
```

### 4. Best Practices

```
✅ 헬스체크 필수
✅ 자동 롤백 설정
✅ 점진적 배포 (2-3개씩)
✅ 충분한 모니터링 시간
✅ start-first 전략
✅ 실시간 모니터링
```

---

<div align="center">

**[⬅️ 이전: Swarm Networking](./05-Swarm-Networking.md)** | **[다음: High Availability ➡️](./07-High-Availability.md)**

</div>
