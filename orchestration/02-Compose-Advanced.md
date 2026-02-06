# 02. Compose Advanced - Compose 고급 기능

## 🎯 이 챕터에서 배울 것

- **extends**와 **재사용** 패턴
- **profiles**로 환경 분리
- **여러 Compose 파일** 병합
- **고급 설정**과 최적화

## 📌 왜 중요한가?

**"고급 Compose 기능으로 복잡한 환경을 효율적으로 관리할 수 있습니다."**

```
기본 Compose의 한계:

단일 파일 방식:
┌──────────────────────────────────────┐
│ docker-compose.yml                   │
│                                      │
│ - 개발 설정                            │
│ - 스테이징 설정                         │
│ - 프로덕션 설정                         │
│ - 모든 서비스                           │
└──────────────────────────────────────┘
❌ 파일이 너무 커짐
❌ 환경별 분리 어려움
❌ 중복 설정 많음
❌ 유지보수 힘듦

고급 기능 활용:
┌──────────────────────────────────────┐
│ docker-compose.yml (base)            │
│ - 공통 설정                            │
└──────────────────────────────────────┘
┌──────────────────────────────────────┐
│ docker-compose.dev.yml               │
│ - 개발 전용 (디버깅, hot reload)         │
└──────────────────────────────────────┘
┌──────────────────────────────────────┐
│ docker-compose.prod.yml              │
│ - 프로덕션 전용 (최적화, 보안)             │
└──────────────────────────────────────┘
✅ 파일 분리
✅ 환경별 명확
✅ 재사용 쉬움
✅ 유지보수 용이

실무 시나리오:

1. 환경 분리:
   - 개발: 디버그 모드, 로컬 DB
   - 스테이징: 프로덕션 유사, 테스트 데이터
   - 프로덕션: 최적화, 보안 강화

2. 기능 분리:
   - 기본: 필수 서비스
   - 모니터링: Prometheus, Grafana
   - 로깅: ELK Stack
   - 백업: 정기 백업 서비스

3. 팀별 분리:
   - 프론트엔드팀: frontend.yml
   - 백엔드팀: backend.yml
   - 인프라팀: infra.yml
```

**실무 영향:**
- 환경별 배포 자동화
- 코드 중복 제거
- 팀 협업 향상
- 설정 관리 간소화

---

## 🔬 Deep Dive

### 1. Extends (확장)

#### 기본 개념

```yaml
# common.yml (공통 설정)
version: '3.8'

services:
  base-app:
    image: node:18-alpine
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    extends:
      file: common.yml
      service: base-app
    # base-app 설정 상속
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
```

**주의: extends는 Compose v3에서 deprecated!**
대신 **YAML anchors** 또는 **여러 파일 병합** 사용

#### YAML Anchors (권장)

```yaml
# docker-compose.yml
version: '3.8'

# 공통 설정 정의
x-common-variables: &common-env
  NODE_ENV: production
  LOG_LEVEL: info

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

x-restart-policy: &restart-policy
  restart: unless-stopped

services:
  web:
    <<: *restart-policy  # 재시작 정책 상속
    image: nginx:alpine
    environment:
      <<: *common-env    # 환경 변수 상속
      SERVICE: web
    logging: *default-logging

  api:
    <<: *restart-policy
    image: node:18-alpine
    environment:
      <<: *common-env
      SERVICE: api
    logging: *default-logging
  
  worker:
    <<: *restart-policy
    image: node:18-alpine
    environment:
      <<: *common-env
      SERVICE: worker
    logging: *default-logging
```

---

### 2. Multiple Compose Files (파일 병합)

#### 기본 사용법

```yaml
# docker-compose.yml (기본)
version: '3.8'

services:
  web:
    image: nginx:alpine
    volumes:
      - ./html:/usr/share/nginx/html

  db:
    image: postgres:14-alpine
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

```yaml
# docker-compose.override.yml (자동 병합)
version: '3.8'

services:
  web:
    # 개발 환경: 포트 추가
    ports:
      - "8080:80"
  
  db:
    # 개발 환경: 포트 노출
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: dev_password
```

```bash
# 자동 병합 (override 파일)
docker-compose up -d
# docker-compose.yml + docker-compose.override.yml

# 결과:
# - web: 8080 포트 열림
# - db: 5432 포트 열림, dev_password
```

#### 명시적 파일 지정

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  web:
    # 프로덕션: 포트 80
    ports:
      - "80:80"
  
  db:
    # 프로덕션: 포트 노출 안 함
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}  # .env에서 로드
```

```bash
# 명시적 파일 지정 (-f)
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 결과:
# - web: 80 포트
# - db: 환경 변수에서 비밀번호 로드
```

#### 병합 규칙

```yaml
# base.yml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
    environment:
      ENV: base
      DEBUG: "false"

# override.yml
services:
  web:
    ports:
      - "9090:80"  # 기존 8080 대체
    environment:
      DEBUG: "true"  # 기존 값 덮어쓰기
      NEW_VAR: value # 새 변수 추가

# 병합 결과:
# ports:
#   - "9090:80"      # override가 우선
# environment:
#   ENV: base        # base 유지
#   DEBUG: "true"    # override로 변경
#   NEW_VAR: value   # 추가
```

---

### 3. Profiles (환경 프로파일)

#### 기본 개념

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 항상 실행
  web:
    image: nginx:alpine
    ports:
      - "80:80"

  db:
    image: postgres:14-alpine
    volumes:
      - db-data:/var/lib/postgresql/data

  # 프로파일: debug
  debug-tool:
    image: nicolaka/netshoot
    profiles:
      - debug
    command: sleep infinity

  # 프로파일: monitoring
  prometheus:
    image: prom/prometheus
    profiles:
      - monitoring
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    profiles:
      - monitoring
    ports:
      - "3000:3000"

  # 프로파일: backup
  backup:
    image: alpine
    profiles:
      - backup
    volumes:
      - db-data:/source:ro
      - ./backups:/backups
    command: sh -c 'tar czf /backups/backup-$(date +%Y%m%d).tar.gz -C /source .'

volumes:
  db-data:
```

```bash
# 기본 실행 (web, db만)
docker-compose up -d

# 프로파일 활성화: debug
docker-compose --profile debug up -d
# web, db, debug-tool 실행

# 프로파일 활성화: monitoring
docker-compose --profile monitoring up -d
# web, db, prometheus, grafana 실행

# 여러 프로파일
docker-compose --profile debug --profile monitoring up -d
# 모두 실행

# 환경 변수로 지정
export COMPOSE_PROFILES=monitoring,debug
docker-compose up -d
```

#### 환경별 프로파일

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    image: myapp:latest
    
  # 개발 환경
  dev-db:
    image: postgres:14-alpine
    profiles:
      - dev
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: dev

  # 프로덕션 환경
  prod-db:
    image: postgres:14-alpine
    profiles:
      - prod
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - prod-db-data:/var/lib/postgresql/data

  # 테스트 환경
  test-db:
    image: postgres:14-alpine
    profiles:
      - test
    tmpfs:
      - /var/lib/postgresql/data

volumes:
  prod-db-data:
```

```bash
# 개발
docker-compose --profile dev up -d

# 프로덕션
docker-compose --profile prod up -d

# 테스트
docker-compose --profile test up -d
```

---

### 4. 환경별 구성

#### 패턴 1: 파일 분리

```yaml
# docker-compose.yml (공통)
version: '3.8'

services:
  web:
    build: .
    volumes:
      - ./app:/app

  db:
    image: postgres:14-alpine
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  web:
    command: npm run dev  # 개발 모드
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      DEBUG: "true"
    volumes:
      - ./app:/app:delegated  # hot reload
  
  db:
    ports:
      - "5432:5432"  # 로컬 접근
    environment:
      POSTGRES_PASSWORD: dev_password
```

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  web:
    command: npm start  # 프로덕션 모드
    ports:
      - "80:3000"
    environment:
      NODE_ENV: production
      DEBUG: "false"
    restart: always
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
  
  db:
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    restart: always
    deploy:
      resources:
        limits:
          memory: 1G
```

```bash
# 개발
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d

# 프로덕션
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Makefile로 간소화
cat > Makefile << 'EOF'
dev:
	docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d

prod:
	docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

down:
	docker-compose down
EOF

# 사용
make dev
make prod
```

#### 패턴 2: 환경 변수 파일

```bash
# .env.dev
NODE_ENV=development
DB_HOST=localhost
DB_PASSWORD=dev_password
DEBUG=true
LOG_LEVEL=debug

# .env.prod
NODE_ENV=production
DB_HOST=prod-db.internal
DB_PASSWORD=super_secret_prod_password
DEBUG=false
LOG_LEVEL=warning
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    image: myapp:latest
    environment:
      NODE_ENV: ${NODE_ENV}
      DB_HOST: ${DB_HOST}
      DB_PASSWORD: ${DB_PASSWORD}
      DEBUG: ${DEBUG}
      LOG_LEVEL: ${LOG_LEVEL}
```

```bash
# 개발
docker-compose --env-file .env.dev up -d

# 프로덕션
docker-compose --env-file .env.prod up -d
```

---

### 5. 고급 패턴

#### 패턴 1: 서비스 템플릿

```yaml
version: '3.8'

# 공통 서비스 템플릿
x-service-template: &service-template
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
  healthcheck:
    interval: 30s
    timeout: 10s
    retries: 3

x-node-service: &node-service
  <<: *service-template
  image: node:18-alpine
  environment: &node-env
    NODE_ENV: production
    LOG_LEVEL: info

services:
  api:
    <<: *node-service
    container_name: api
    environment:
      <<: *node-env
      SERVICE: api
    ports:
      - "3001:3000"
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]

  worker:
    <<: *node-service
    container_name: worker
    environment:
      <<: *node-env
      SERVICE: worker
    command: node worker.js

  scheduler:
    <<: *node-service
    container_name: scheduler
    environment:
      <<: *node-env
      SERVICE: scheduler
    command: node scheduler.js
```

#### 패턴 2: 조건부 서비스

```yaml
version: '3.8'

services:
  app:
    image: myapp
    environment:
      ENABLE_CACHE: ${ENABLE_CACHE:-false}

  # 캐시 (조건부)
  redis:
    image: redis:alpine
    profiles:
      - cache
    # ENABLE_CACHE=true일 때만 실행

  # 메시지 큐 (조건부)
  rabbitmq:
    image: rabbitmq:3-management-alpine
    profiles:
      - messaging
```

```bash
# 캐시 활성화
export ENABLE_CACHE=true
docker-compose --profile cache up -d

# 메시징 활성화
docker-compose --profile messaging up -d

# 둘 다 활성화
docker-compose --profile cache --profile messaging up -d
```

#### 패턴 3: 다중 환경 전환

```bash
#!/bin/bash
# deploy.sh

ENVIRONMENT=$1

case $ENVIRONMENT in
  dev)
    docker-compose \
      -f docker-compose.yml \
      -f docker-compose.dev.yml \
      --env-file .env.dev \
      up -d
    ;;
  
  staging)
    docker-compose \
      -f docker-compose.yml \
      -f docker-compose.staging.yml \
      --env-file .env.staging \
      --profile monitoring \
      up -d
    ;;
  
  prod)
    docker-compose \
      -f docker-compose.yml \
      -f docker-compose.prod.yml \
      --env-file .env.prod \
      --profile monitoring \
      --profile backup \
      up -d
    ;;
  
  *)
    echo "Usage: $0 {dev|staging|prod}"
    exit 1
    ;;
esac
```

```bash
chmod +x deploy.sh

# 사용
./deploy.sh dev
./deploy.sh staging
./deploy.sh prod
```

---

## 💻 실습

### 실습 1: Anchors 활용

```yaml
# docker-compose.yml
version: '3.8'

# 공통 설정 정의
x-common: &common
  restart: unless-stopped
  logging:
    driver: json-file
    options:
      max-size: "10m"

x-healthcheck: &healthcheck
  interval: 30s
  timeout: 10s
  retries: 3

services:
  web:
    <<: *common
    image: nginx:alpine
    ports:
      - "80:80"
    healthcheck:
      <<: *healthcheck
      test: ["CMD", "wget", "-q", "--spider", "http://localhost"]

  api:
    <<: *common
    image: node:18-alpine
    ports:
      - "3000:3000"
    healthcheck:
      <<: *healthcheck
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]

  db:
    <<: *common
    image: postgres:14-alpine
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      <<: *healthcheck
      test: ["CMD-SHELL", "pg_isready"]

volumes:
  db-data:
```

```bash
# 시작
docker-compose up -d

# 설정 확인
docker-compose config

# 정리
docker-compose down -v
```

---

### 실습 2: 환경별 파일

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    volumes:
      - ./src:/app/src
```

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    command: npm run dev
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      DEBUG: "*"
```

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    command: npm start
    ports:
      - "80:3000"
    environment:
      NODE_ENV: production
    restart: always
```

```dockerfile
# Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

```bash
# 개발
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up --build

# 프로덕션
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# 정리
docker-compose down
rm Dockerfile
```

---

### 실습 3: Profiles 활용

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 필수 서비스
  web:
    image: nginx:alpine
    ports:
      - "80:80"

  # 모니터링 (선택)
  prometheus:
    image: prom/prometheus
    profiles: [monitoring]
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    profiles: [monitoring]
    ports:
      - "3000:3000"

  # 디버그 (선택)
  debug:
    image: nicolaka/netshoot
    profiles: [debug]
    command: sleep infinity

  # 테스트 (선택)
  test:
    image: alpine
    profiles: [test]
    command: echo "Running tests..."
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

```bash
# 기본 (web만)
docker-compose up -d

# 모니터링 포함
docker-compose --profile monitoring up -d

# 디버그 포함
docker-compose --profile debug up -d

# 여러 프로파일
docker-compose --profile monitoring --profile debug up -d

# 환경 변수로
export COMPOSE_PROFILES=monitoring,debug
docker-compose up -d

# 정리
docker-compose down -v
rm prometheus.yml
```

---

## 🔥 실전 시나리오

### 시나리오 1: 완전한 환경 분리

```yaml
# docker-compose.yml (베이스)
version: '3.8'

x-logging: &logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  app:
    build:
      context: .
      target: ${BUILD_TARGET:-production}
    logging: *logging
    volumes:
      - app-data:/app/data

  db:
    image: postgres:14-alpine
    logging: *logging
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  app-data:
  db-data:
```

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    build:
      target: development
    command: npm run dev
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      DEBUG: "app:*"
    volumes:
      - ./src:/app/src:cached  # hot reload
  
  db:
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: dev_pass
  
  # 개발 도구
  adminer:
    image: adminer
    ports:
      - "8080:8080"
    environment:
      ADMINER_DEFAULT_SERVER: db
```

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build:
      target: production
    command: npm start
    ports:
      - "80:3000"
    environment:
      NODE_ENV: production
    restart: always
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
  
  db:
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    restart: always
    deploy:
      resources:
        limits:
          memory: 2G
  
  # 프로덕션 모니터링
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    restart: always

volumes:
  prometheus-data:
```

```dockerfile
# Dockerfile (Multi-stage)
# Development
FROM node:18-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

# Production
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
RUN npm run build
CMD ["npm", "start"]
```

```bash
# 개발
docker-compose \
  -f docker-compose.yml \
  -f docker-compose.dev.yml \
  up --build

# 프로덕션
docker-compose \
  -f docker-compose.yml \
  -f docker-compose.prod.yml \
  up -d --build
```

---

### 시나리오 2: 기능별 프로파일

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 코어 서비스
  app:
    image: myapp:latest
    ports:
      - "3000:3000"

  db:
    image: postgres:14-alpine
    volumes:
      - db-data:/var/lib/postgresql/data

  # 캐시 레이어
  redis:
    image: redis:alpine
    profiles: [cache]
    volumes:
      - redis-data:/data

  # 메시지 큐
  rabbitmq:
    image: rabbitmq:3-management-alpine
    profiles: [messaging]
    ports:
      - "5672:5672"
      - "15672:15672"

  # 모니터링
  prometheus:
    image: prom/prometheus
    profiles: [monitoring]
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    profiles: [monitoring]
    ports:
      - "3000:3000"

  # 로깅
  elasticsearch:
    image: elasticsearch:8.5.0
    profiles: [logging]
    environment:
      - discovery.type=single-node

  logstash:
    image: logstash:8.5.0
    profiles: [logging]

  kibana:
    image: kibana:8.5.0
    profiles: [logging]
    ports:
      - "5601:5601"

  # 백업
  backup:
    image: alpine
    profiles: [backup]
    volumes:
      - db-data:/source:ro
      - ./backups:/backups
    command: sh -c 'tar czf /backups/backup-$(date +%Y%m%d).tar.gz -C /source .'

volumes:
  db-data:
  redis-data:
```

```bash
# 기본 (app + db만)
docker-compose up -d

# 캐시 추가
docker-compose --profile cache up -d

# 풀 스택 (모니터링 + 로깅)
docker-compose \
  --profile cache \
  --profile messaging \
  --profile monitoring \
  --profile logging \
  up -d

# 백업 실행
docker-compose --profile backup up
```

---

## 🚫 안티패턴

### 1. 하드코딩된 환경 설정

```yaml
# ❌ 하드코딩
services:
  app:
    environment:
      NODE_ENV: production  # 개발/프로덕션 구분 안 됨

# ✅ 환경 변수
services:
  app:
    environment:
      NODE_ENV: ${NODE_ENV}
```

### 2. override 파일 과다 사용

```yaml
# ❌ override에 너무 많은 설정
# docker-compose.override.yml
services:
  app:
    # 100줄의 설정...

# ✅ 명시적 파일 분리
# docker-compose.dev.yml
# docker-compose.prod.yml
```

### 3. 프로파일 남용

```yaml
# ❌ 모든 서비스에 프로파일
services:
  app:
    profiles: [main]
  db:
    profiles: [main]
# 복잡도만 증가

# ✅ 선택적 서비스만
services:
  app:  # 프로파일 없음 (기본)
  db:   # 프로파일 없음
  debug-tool:
    profiles: [debug]  # 선택적
```

---

## 🎓 핵심 정리

### 1. 재사용 패턴

```
YAML Anchors:
- x-common: &common
- <<: *common
- 같은 파일 내 재사용

Extends (deprecated):
- file: common.yml
- service: base
- 여러 파일 재사용 (v2)
```

### 2. 파일 병합

```bash
# 기본 + override (자동)
docker-compose up

# 명시적 파일
docker-compose \
  -f base.yml \
  -f dev.yml \
  up
```

### 3. Profiles

```yaml
services:
  monitoring:
    profiles: [monitor]

# 실행
docker-compose --profile monitor up
```

### 4. Best Practices

```
✅ 베이스 파일 분리
✅ 환경별 override
✅ Anchors로 중복 제거
✅ Profiles로 선택적 실행
✅ 환경 변수 활용
✅ Makefile/스크립트 자동화
```

---

<div align="center">

**[⬅️ 이전: Docker Compose](./01-Docker-Compose.md)** | **[다음: Docker Swarm ➡️](./03-Docker-Swarm.md)**

</div>
