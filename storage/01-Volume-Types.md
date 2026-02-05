# 01. Volume Types - 볼륨 타입

## 🎯 이 챕터에서 배울 것

- Docker **볼륨의 필요성**과 컨테이너 데이터 문제
- **Named Volume**과 **Anonymous Volume** 비교
- 볼륨 **생명주기** 관리
- 실전 **사용 패턴**과 베스트 프랙티스

## 📌 왜 중요한가?

**"컨테이너는 일시적(ephemeral)입니다. 데이터는 영속적이어야 합니다."**

```
볼륨 없음:
┌─────────────────────────────────┐
│ Container                       │
│ ┌─────────────────────────────┐ │
│ │ /var/lib/mysql/data         │ │
│ │ - 데이터베이스 파일             │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘
         │ 컨테이너 삭제
         ▼
    💥 데이터 소실!

볼륨 사용:
┌─────────────────────────────────┐
│ Container                       │
│ ┌─────────────────────────────┐ │
│ │ /var/lib/mysql/data         │ │
│ └──────────┬──────────────────┘ │
└────────────┼────────────────────┘
             │ 마운트
┌────────────▼────────────────────┐
│ Volume (Host)                   │
│ /var/lib/docker/volumes/db-data │
│ ✅ 컨테이너 독립적                  │
│ ✅ 영속적 저장                     │
└─────────────────────────────────┘

컨테이너 삭제 후에도 데이터 유지!
```

**실무 영향:**
- 데이터 손실 방지: 컨테이너 재시작/삭제 시에도 데이터 보존
- 데이터 공유: 여러 컨테이너 간 데이터 공유
- 백업/복구: 볼륨 단위로 백업 가능
- 성능: 최적화된 I/O 성능

---

## 🔬 Deep Dive

### 1. 컨테이너 파일시스템의 문제

#### 컨테이너 레이어 구조

```
컨테이너 파일시스템:
┌─────────────────────────────────┐
│ Container Layer (R/W)           │ ← 읽기/쓰기 가능
├─────────────────────────────────┤
│ Image Layer 3 (R/O)             │ ↓
├─────────────────────────────────┤
│ Image Layer 2 (R/O)             │ ↓ 읽기 전용
├─────────────────────────────────┤
│ Image Layer 1 (R/O)             │ ↓
└─────────────────────────────────┘

문제점:
1. Container Layer는 일시적
   - 컨테이너 삭제 시 소실
   - 재시작 시에도 새 레이어 생성

2. Copy-on-Write 오버헤드
   - 이미지 레이어 수정 시 복사 필요
   - I/O 성능 저하

3. 데이터 공유 불가
   - 컨테이너 간 데이터 공유 어려움
```

#### 데이터 소실 시나리오

```bash
# 데이터베이스 컨테이너 실행 (볼륨 없음)
docker run -d --name mysql-no-volume \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8

# 데이터 생성
docker exec mysql-no-volume mysql -psecret -e \
  "CREATE DATABASE myapp; USE myapp; CREATE TABLE users (id INT, name VARCHAR(50)); INSERT INTO users VALUES (1, 'Alice');"

# 데이터 확인
docker exec mysql-no-volume mysql -psecret -e \
  "SELECT * FROM myapp.users;"
# +------+-------+
# | id   | name  |
# +------+-------+
# |    1 | Alice |
# +------+-------+

# 컨테이너 삭제
docker rm -f mysql-no-volume

# 재시작
docker run -d --name mysql-no-volume \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8

# 데이터 조회 시도
docker exec mysql-no-volume mysql -psecret -e \
  "SELECT * FROM myapp.users;"
# ERROR 1049 (42000): Unknown database 'myapp'

# 💥 데이터 소실!
```

---

### 2. Named Volume

#### 개념

```
Named Volume:
- Docker가 관리하는 영속적 저장소
- 명시적 이름 부여
- 여러 컨테이너 간 공유 가능
- 독립적 생명주기

위치:
/var/lib/docker/volumes/<volume-name>/_data

특징:
✅ 명시적 관리
✅ 재사용 가능
✅ 백업 용이
✅ 컨테이너 독립적
```

#### 기본 사용

```bash
# 1. 볼륨 생성
docker volume create my-data

# 볼륨 확인
docker volume ls
# DRIVER    VOLUME NAME
# local     my-data

# 볼륨 상세 정보
docker volume inspect my-data
# [
#     {
#         "CreatedAt": "2024-01-15T10:30:00Z",
#         "Driver": "local",
#         "Labels": {},
#         "Mountpoint": "/var/lib/docker/volumes/my-data/_data",
#         "Name": "my-data",
#         "Options": {},
#         "Scope": "local"
#     }
# ]

# 2. 컨테이너에 마운트
docker run -d \
  --name app \
  -v my-data:/app/data \
  nginx:alpine

# 또는 --mount 구문
docker run -d \
  --name app \
  --mount type=volume,source=my-data,target=/app/data \
  nginx:alpine

# 3. 데이터 생성
docker exec app sh -c 'echo "Hello" > /app/data/test.txt'

# 4. 컨테이너 삭제
docker rm -f app

# 5. 새 컨테이너에 같은 볼륨 마운트
docker run -d \
  --name app2 \
  -v my-data:/app/data \
  nginx:alpine

# 6. 데이터 확인
docker exec app2 cat /app/data/test.txt
# Hello
# ✅ 데이터 유지됨!

# 정리
docker rm -f app2
docker volume rm my-data
```

#### 실제 사용 예시 - MySQL

```bash
# Named Volume 생성
docker volume create mysql-data

# MySQL 컨테이너
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8

# 데이터 생성
docker exec mysql mysql -psecret -e \
  "CREATE DATABASE myapp; USE myapp; CREATE TABLE users (id INT, name VARCHAR(50)); INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob');"

# 컨테이너 삭제
docker rm -f mysql

# 새 컨테이너로 재시작
docker run -d \
  --name mysql-new \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8

# 대기 (MySQL 초기화)
sleep 10

# 데이터 확인
docker exec mysql-new mysql -psecret -e \
  "SELECT * FROM myapp.users;"
# +------+-------+
# | id   | name  |
# +------+-------+
# |    1 | Alice |
# |    2 | Bob   |
# +------+-------+
# ✅ 데이터 유지!

# 정리
docker rm -f mysql-new
docker volume rm mysql-data
```

---

### 3. Anonymous Volume

#### 개념

```
Anonymous Volume:
- 이름 없는 임시 볼륨
- 컨테이너 생성 시 자동 생성
- 임의의 이름 할당 (해시값)
- 일회성 사용

위치:
/var/lib/docker/volumes/<hash>/_data

특징:
✅ 자동 생성
❌ 재사용 어려움
❌ 관리 어려움
⚠️ 컨테이너 삭제 시 고아 볼륨 생성 가능
```

#### 기본 사용

```bash
# 1. Anonymous Volume (이름 없음)
docker run -d \
  --name app \
  -v /app/data \
  nginx:alpine

# 볼륨 확인
docker volume ls
# DRIVER    VOLUME NAME
# local     abc123def456...  ← 해시값

# 볼륨 상세 정보
docker inspect app | grep -A 10 Mounts
# "Mounts": [
#     {
#         "Type": "volume",
#         "Name": "abc123def456...",
#         "Source": "/var/lib/docker/volumes/abc123def456.../_data",
#         "Destination": "/app/data",
#         ...
#     }
# ]

# 2. 데이터 생성
docker exec app sh -c 'echo "Hello" > /app/data/test.txt'

# 3. 컨테이너 삭제
docker rm -f app

# 4. 볼륨 확인
docker volume ls
# DRIVER    VOLUME NAME
# local     abc123def456...  ← 여전히 존재 (고아 볼륨)

# 5. 수동 삭제 필요
docker volume rm abc123def456...

# 또는 컨테이너 삭제 시 자동 제거 (--rm)
docker run -d --rm \
  --name app \
  -v /app/data \
  nginx:alpine

docker stop app
# 컨테이너와 볼륨 모두 삭제됨
```

#### Dockerfile VOLUME 지시어

```dockerfile
# Dockerfile
FROM nginx:alpine

# Anonymous Volume 선언
VOLUME /usr/share/nginx/html
VOLUME /var/log/nginx
```

```bash
# 이미지 빌드
docker build -t my-nginx .

# 컨테이너 실행 (VOLUME 자동 생성)
docker run -d --name web my-nginx

# 볼륨 확인
docker volume ls
# DRIVER    VOLUME NAME
# local     xyz789abc123...  ← /usr/share/nginx/html
# local     def456ghi789...  ← /var/log/nginx

# 자동 생성된 Anonymous Volume

# 정리
docker rm -f web
docker volume prune  # 고아 볼륨 정리
```

---

### 4. Named vs Anonymous 비교

#### 생명주기

```
Named Volume:
1. 명시적 생성: docker volume create
2. 컨테이너 마운트: -v name:/path
3. 컨테이너 삭제 후에도 유지
4. 명시적 삭제: docker volume rm

Anonymous Volume:
1. 암묵적 생성: -v /path 또는 VOLUME
2. 임의 이름 할당
3. 컨테이너 삭제 후 고아 볼륨
4. prune으로 일괄 정리
```

#### 사용 시나리오

```
Named Volume (권장):
✅ 데이터베이스 데이터
✅ 애플리케이션 상태
✅ 사용자 업로드 파일
✅ 로그 (장기 보관)
✅ 개발 환경 공유 데이터

Anonymous Volume:
⚠️ 임시 캐시
⚠️ 테스트 데이터
⚠️ 빌드 아티팩트
⚠️ 일회성 작업
```

#### 비교표

```
┌─────────────────┬──────────────┬────────────────┐
│ 특징             │ Named Volume │ Anonymous Vol  │
├─────────────────┼──────────────┼────────────────┤
│ 생성 방법         │ 명시적         │ 자동            │
├─────────────────┼──────────────┼────────────────┤
│ 이름             │ 사용자 지정     │ 해시값           │
├─────────────────┼──────────────┼────────────────┤
│ 재사용            │ 쉬움          │ 어려움          │
├─────────────────┼──────────────┼────────────────┤
│ 관리             │ 용이          │ 복잡            │
├─────────────────┼──────────────┼────────────────┤
│ 공유             │ 가능          │ 어려움           │
├─────────────────┼──────────────┼────────────────┤
│ 백업             │ 쉬움          │ 식별 어려움       │
├─────────────────┼──────────────┼────────────────┤
│ 권장 사용         │ 프로덕션       │ 임시/테스트       │
└─────────────────┴──────────────┴────────────────┘
```

---

## 💻 실습

### 실습 1: Named Volume 생명주기

#### 볼륨 생성 및 관리

```bash
# 1. 볼륨 생성
docker volume create app-data

# 레이블 추가
docker volume create --label env=production \
  --label team=backend \
  prod-data

# 2. 볼륨 목록
docker volume ls

# 레이블 필터
docker volume ls --filter label=env=production

# 3. 볼륨 상세 정보
docker volume inspect app-data

# 4. 여러 컨테이너에서 공유
docker run -d --name app1 \
  -v app-data:/data \
  alpine sleep 3600

docker run -d --name app2 \
  -v app-data:/data \
  alpine sleep 3600

# app1에서 파일 생성
docker exec app1 sh -c 'echo "Shared data" > /data/shared.txt'

# app2에서 확인
docker exec app2 cat /data/shared.txt
# Shared data
# ✅ 공유됨!

# 5. 컨테이너 삭제
docker rm -f app1 app2

# 6. 볼륨 여전히 존재
docker volume ls | grep app-data
# app-data

# 7. 볼륨 삭제
docker volume rm app-data
```

---

### 실습 2: Anonymous Volume 문제점

#### 고아 볼륨 생성

```bash
# 초기 상태
docker volume ls
# (비어있음)

# Anonymous Volume으로 여러 컨테이너 실행
for i in {1..5}; do
  docker run -d --name test$i \
    -v /data \
    alpine sleep 3600
done

# 볼륨 확인
docker volume ls
# DRIVER    VOLUME NAME
# local     abc123...
# local     def456...
# local     ghi789...
# local     jkl012...
# local     mno345...
# 5개의 Anonymous Volume 생성!

# 컨테이너 삭제
docker rm -f test1 test2 test3 test4 test5

# 볼륨 여전히 존재 (고아 볼륨)
docker volume ls
# 5개 모두 남아있음!

# 사용 중인지 확인
for vol in $(docker volume ls -q); do
  echo "Volume: $vol"
  docker ps -a --filter volume=$vol
done
# 모두 사용 안 함

# 정리 (dangling volumes)
docker volume prune
# WARNING! This will remove all local volumes not used by at least one container.
# Are you sure you want to continue? [y/N] y

# 모든 고아 볼륨 삭제됨
```

---

### 실습 3: 올바른 패턴 - Named Volume

#### Docker Compose 사용

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Web 애플리케이션
  web:
    image: nginx:alpine
    volumes:
      - web-content:/usr/share/nginx/html
      - web-logs:/var/log/nginx
    ports:
      - "80:80"

  # 데이터베이스
  db:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - db-data:/var/lib/postgresql/data
      - db-backups:/backups

  # Redis 캐시
  cache:
    image: redis:alpine
    volumes:
      - redis-data:/data

volumes:
  web-content:
    driver: local
    labels:
      com.example.description: "Web content files"
      com.example.team: "frontend"
  
  web-logs:
    driver: local
    labels:
      com.example.description: "Web server logs"
  
  db-data:
    driver: local
    labels:
      com.example.description: "Database files"
      com.example.team: "backend"
  
  db-backups:
    driver: local
    labels:
      com.example.description: "Database backups"
  
  redis-data:
    driver: local
    labels:
      com.example.description: "Redis persistence"
```

```bash
# 서비스 시작
docker-compose up -d

# 볼륨 확인
docker volume ls
# DRIVER    VOLUME NAME
# local     myapp_web-content
# local     myapp_web-logs
# local     myapp_db-data
# local     myapp_db-backups
# local     myapp_redis-data

# 데이터 생성
docker-compose exec web sh -c \
  'echo "<h1>Hello</h1>" > /usr/share/nginx/html/index.html'

docker-compose exec db psql -U postgres -c \
  "CREATE DATABASE myapp;"

# 서비스 재시작 (데이터 유지)
docker-compose down
docker-compose up -d

# 데이터 확인
curl http://localhost
# <h1>Hello</h1>

docker-compose exec db psql -U postgres -c "\l" | grep myapp
# myapp

# ✅ 모두 유지됨!

# 정리 (볼륨 제외)
docker-compose down

# 볼륨 포함 정리
docker-compose down -v
```

---

### 실습 4: 볼륨 검사 및 디버깅

#### 볼륨 내용 확인

```bash
# Named Volume 생성
docker volume create debug-vol

# 데이터 추가
docker run --rm \
  -v debug-vol:/data \
  alpine sh -c 'echo "Test data" > /data/test.txt'

# 방법 1: 임시 컨테이너로 확인
docker run --rm \
  -v debug-vol:/data \
  alpine ls -la /data

docker run --rm \
  -v debug-vol:/data \
  alpine cat /data/test.txt
# Test data

# 방법 2: 호스트에서 직접 확인 (권한 필요)
sudo ls -la /var/lib/docker/volumes/debug-vol/_data/
sudo cat /var/lib/docker/volumes/debug-vol/_data/test.txt

# 방법 3: 볼륨 복사 (백업)
docker run --rm \
  -v debug-vol:/source \
  -v $(pwd)/backup:/backup \
  alpine sh -c 'tar czf /backup/debug-vol.tar.gz -C /source .'

# 확인
tar tzf backup/debug-vol.tar.gz

# 정리
docker volume rm debug-vol
rm -rf backup
```

---

## 🔥 실전 적용

### 시나리오 1: 마이크로서비스 스택

**구조:**

```
┌─────────────────────────────────────┐
│ Frontend (React)                    │
│ - Nginx                             │
│ - Build artifacts → named volume    │
└─────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│ API Gateway (Node.js)               │
│ - Logs → named volume               │
└─────────────────────────────────────┘
                 ↓
┌─────────────┬───────────────────────┐
│ Services    │ Services              │
│ - Auth      │ - Payment             │
│ - User      │ - Order               │
└─────────────┴───────────────────────┘
                 ↓
┌─────────────────────────────────────┐
│ Data Layer                          │
│ - PostgreSQL → named volume         │
│ - Redis → named volume              │
│ - Elasticsearch → named volume      │
└─────────────────────────────────────┘
```

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  # Frontend
  frontend:
    image: myapp/frontend:latest
    volumes:
      - frontend-build:/usr/share/nginx/html:ro
      - frontend-logs:/var/log/nginx
    ports:
      - "80:80"
    networks:
      - frontend-net

  # API Gateway
  gateway:
    image: myapp/gateway:latest
    volumes:
      - gateway-logs:/app/logs
    environment:
      - NODE_ENV=production
    networks:
      - frontend-net
      - backend-net

  # Auth Service
  auth:
    image: myapp/auth:latest
    volumes:
      - auth-data:/app/data
    networks:
      - backend-net
      - data-net

  # Payment Service
  payment:
    image: myapp/payment:latest
    volumes:
      - payment-logs:/app/logs
      - payment-receipts:/app/receipts
    networks:
      - backend-net
      - data-net

  # PostgreSQL
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - postgres-backups:/backups
    networks:
      - data-net

  # Redis
  redis:
    image: redis:alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - data-net

  # Elasticsearch
  elasticsearch:
    image: elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
    volumes:
      - es-data:/usr/share/elasticsearch/data
    networks:
      - data-net

volumes:
  # Frontend
  frontend-build:
    driver: local
    labels:
      com.example.tier: frontend
      com.example.backup: "false"
  
  frontend-logs:
    driver: local
    labels:
      com.example.tier: frontend
      com.example.backup: "true"

  # Gateway
  gateway-logs:
    driver: local
    labels:
      com.example.tier: gateway
      com.example.backup: "true"

  # Services
  auth-data:
    driver: local
    labels:
      com.example.tier: service
      com.example.backup: "true"
  
  payment-logs:
    driver: local
    labels:
      com.example.tier: service
      com.example.backup: "true"
  
  payment-receipts:
    driver: local
    labels:
      com.example.tier: service
      com.example.backup: "true"
      com.example.retention: "7years"

  # Data
  postgres-data:
    driver: local
    labels:
      com.example.tier: data
      com.example.backup: "true"
      com.example.critical: "true"
  
  postgres-backups:
    driver: local
    labels:
      com.example.tier: data
      com.example.backup: "true"
  
  redis-data:
    driver: local
    labels:
      com.example.tier: data
      com.example.backup: "true"
  
  es-data:
    driver: local
    labels:
      com.example.tier: data
      com.example.backup: "true"

networks:
  frontend-net:
  backend-net:
  data-net:
    internal: true
```

---

### 시나리오 2: 개발 환경

**요구사항:**
- 코드 변경 즉시 반영
- 데이터베이스 데이터 유지
- 로그 확인 용이
- 빠른 재시작

**docker-compose.dev.yml:**

```yaml
version: '3.8'

services:
  # 개발 서버 (Hot Reload)
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      # 소스 코드 (Bind Mount - 즉시 반영)
      - ./src:/app/src:ro
      - ./public:/app/public:ro
      
      # Node modules (Named Volume - 성능)
      - node-modules:/app/node_modules
      
      # 빌드 출력 (Anonymous - 임시)
      - /app/.next
      
      # 로그 (Named Volume - 디버깅)
      - dev-logs:/app/logs
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:dev@db:5432/myapp_dev
    command: npm run dev
    depends_on:
      - db

  # 데이터베이스
  db:
    image: postgres:14-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: myapp_dev
    volumes:
      # 데이터 (Named Volume - 유지)
      - dev-db-data:/var/lib/postgresql/data
      
      # 초기화 스크립트 (Bind Mount)
      - ./db/init:/docker-entrypoint-initdb.d:ro
    ports:
      - "5432:5432"

  # Redis
  redis:
    image: redis:alpine
    volumes:
      - dev-redis-data:/data
    ports:
      - "6379:6379"

volumes:
  node-modules:
    driver: local
  dev-logs:
    driver: local
  dev-db-data:
    driver: local
  dev-redis-data:
    driver: local
```

```bash
# 개발 환경 시작
docker-compose -f docker-compose.dev.yml up -d

# 코드 수정 → 즉시 반영 (Hot Reload)
echo "console.log('Updated');" >> src/index.js

# 로그 확인
docker-compose -f docker-compose.dev.yml logs -f app

# 데이터베이스 확인
docker-compose -f docker-compose.dev.yml exec db \
  psql -U postgres -d myapp_dev

# 정리 (데이터 유지)
docker-compose -f docker-compose.dev.yml down

# 재시작 (데이터 그대로)
docker-compose -f docker-compose.dev.yml up -d
```

---

## ⚡ 볼륨 관리 체크리스트

### 볼륨 생성

```
□ 목적에 맞는 타입 선택 (Named/Anonymous)
□ 의미 있는 이름 지정
□ 레이블 추가 (환경, 팀, 중요도)
□ 드라이버 선택 (local/NFS/etc)
□ 옵션 설정 (크기, 성능)
```

### 볼륨 사용

```
□ 읽기 전용 마운트 (보안)
□ 적절한 마운트 포인트
□ 권한 설정 (소유자, 그룹)
□ 여러 컨테이너 공유 시 동시성 고려
□ 성능 요구사항 확인
```

### 볼륨 정리

```
□ 사용하지 않는 볼륨 식별
□ 정기적인 prune 실행
□ 중요 데이터 백업 먼저
□ dangling 볼륨 제거
□ 로그 확인
```

### 모니터링

```
□ 볼륨 사용량 추적
□ 증가율 모니터링
□ I/O 성능 측정
□ 에러 로그 확인
□ 백업 상태 체크
```

---

## 🚫 안티패턴

### 1. Anonymous Volume 남발

```yaml
# ❌ Anonymous Volume (관리 어려움)
services:
  app:
    image: myapp
    volumes:
      - /app/data
      - /app/logs
      - /app/cache

# ✅ Named Volume (관리 용이)
services:
  app:
    image: myapp
    volumes:
      - app-data:/app/data
      - app-logs:/app/logs
      - app-cache:/app/cache

volumes:
  app-data:
  app-logs:
  app-cache:
```

### 2. 볼륨 미사용 (데이터 손실)

```bash
# ❌ 볼륨 없이 데이터베이스
docker run -d postgres
# 컨테이너 삭제 시 데이터 소실!

# ✅ Named Volume 사용
docker volume create postgres-data
docker run -d \
  -v postgres-data:/var/lib/postgresql/data \
  postgres
```

### 3. 고아 볼륨 방치

```bash
# ❌ 정리하지 않음
docker volume ls
# 수십 개의 Anonymous Volume...

# ✅ 정기적으로 정리
docker volume prune

# 또는 사용하지 않는 볼륨 전체 제거
docker volume prune -a
```

### 4. 잘못된 마운트 포인트

```bash
# ❌ 전체 루트 마운트
docker run -v my-vol:/ nginx
# 위험! 컨테이너 파일시스템 덮어씀

# ✅ 특정 디렉토리만
docker run -v my-vol:/app/data nginx
```

---

## 🎓 핵심 정리

### 1. 볼륨의 필요성

```
문제:
- 컨테이너는 일시적 (Ephemeral)
- 데이터는 영속적이어야 함 (Persistent)
- 컨테이너 레이어는 성능 저하

해결:
- Docker Volume 사용
- 컨테이너 독립적 저장소
- 최적화된 I/O 성능
```

### 2. Named vs Anonymous

```
Named Volume:
+ 명시적 이름
+ 재사용 가능
+ 관리 용이
+ 권장 방식

Anonymous Volume:
+ 자동 생성
- 해시 이름
- 재사용 어려움
- 임시 사용만
```

### 3. 생명주기

```
생성:
- docker volume create (Named)
- 자동 생성 (Anonymous)

사용:
- -v name:/path
- --mount type=volume

삭제:
- docker volume rm (Named)
- docker volume prune (Dangling)
```

### 4. 핵심 명령어

```bash
# 볼륨 생성
docker volume create <name>

# 볼륨 목록
docker volume ls

# 볼륨 상세
docker volume inspect <name>

# 볼륨 삭제
docker volume rm <name>

# 미사용 볼륨 정리
docker volume prune

# 컨테이너에 마운트
docker run -v <name>:<path> <image>
```

---

## 📚 참고 자료

- [Docker Volumes](https://docs.docker.com/storage/volumes/)
- [Manage Docker Volumes](https://docs.docker.com/engine/reference/commandline/volume/)
- [Storage Best Practices](https://docs.docker.com/develop/dev-best-practices/)

---

## 🤔 생각해볼 문제

1. Anonymous Volume은 언제 사용해야 할까?
2. 같은 볼륨을 여러 컨테이너가 동시에 쓸 때 문제는?
3. Named Volume의 실제 저장 위치는 어디일까?

> 💡 **답변**: 1) Anonymous Volume은 매우 제한적으로만 사용 - Dockerfile에서 VOLUME 지시어로 기본 마운트 포인트 선언, 임시 테스트나 일회성 작업, 캐시나 빌드 아티팩트 같은 휘발성 데이터, 대부분의 경우 Named Volume 사용 권장, 2) 동시 쓰기 시 데이터 손상 위험 - 파일 잠금 메커니즘 필요, 데이터베이스는 하나의 인스턴스만, 읽기 전용 마운트 (:ro) 고려, 애플리케이션 레벨 동기화 필요, NFS 같은 공유 스토리지 드라이버 사용, 3) /var/lib/docker/volumes/<volume-name>/_data - Linux 기준, Docker Desktop(Mac/Windows)은 VM 내부, root 권한 필요, docker volume inspect로 정확한 경로 확인, 직접 접근보다 docker run --rm -v 사용 권장

---

<div align="center">

**[⬅️ 이전 섹션: Networking](../networking/09-Network-Security.md)** | **[다음: Bind Mounts ➡️](./02-Bind-Mounts.md)**

</div>
