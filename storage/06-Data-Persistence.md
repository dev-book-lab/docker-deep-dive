# 06. Data Persistence - 데이터 영속성

## 🎯 이 챕터에서 배울 것

- 컨테이너 **데이터 영속성** 전략
- **Stateful** vs **Stateless** 애플리케이션
- 데이터베이스 **백업/복구** 패턴
- **마이그레이션**과 **업그레이드** 전략

## 📌 왜 중요한가?

**"컨테이너는 임시적이지만, 데이터는 영구적이어야 합니다."**

```
컨테이너 생명주기:
┌──────────────────────────────────────┐
│ Container Lifecycle                  │
│ ┌──────┐  ┌──────┐  ┌──────┐         │
│ │Create│→ │ Run  │→ │ Stop │→[Delete]│
│ └──────┘  └──────┘  └──────┘         │
│   ↓         ↓         ↓              │
│ Ephemeral Ephemeral Ephemeral        │
└──────────────────────────────────────┘

데이터 영속성 문제:
┌──────────────────────────────────────┐
│ Without Volumes:                     │
│ Container → [Data] → Delete → 💥     │
│                      Data Lost!      │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│ With Volumes:                        │
│ Container → [Volume] ← Data          │
│     ↓                    ↑           │
│   Delete              Persist        │
│                          ↓           │
│ New Container → [Same Volume] ✅     │
└──────────────────────────────────────┘

실무 시나리오:
1. 데이터베이스 (PostgreSQL, MySQL)
   - 데이터 파일
   - 트랜잭션 로그
   - 백업

2. 파일 스토리지 (업로드, 미디어)
   - 사용자 업로드
   - 생성된 컨텐츠
   - 캐시 파일

3. 설정 파일
   - 애플리케이션 config
   - 인증서
   - 라이센스

4. 로그
   - 애플리케이션 로그
   - 액세스 로그
   - 감사 로그
```

**실무 영향:**
- 데이터 손실 방지
- 컨테이너 재시작/업그레이드 시 연속성
- 백업/복구 전략
- 마이그레이션 용이성

---

## 🔬 Deep Dive

### 1. Stateful vs Stateless

#### 개념 비교

```
Stateless Application:
┌──────────────────────────────────────┐
│ Request → [Container] → Response     │
│                                      │
│ - 상태 저장 안 함                       │
│ - 각 요청 독립적                        │
│ - 수평 확장 쉬움                        │
│ - 컨테이너 교체 가능                     │
└──────────────────────────────────────┘

Examples:
- REST API 서버
- 웹 프론트엔드
- 프록시 (Nginx, HAProxy)
- 함수 (Serverless)

Stateful Application:
┌──────────────────────────────────────┐
│ Request → [Container + State] → Resp │
│                  ↕                   │
│              [Volume]                │
│                                      │
│ - 상태 유지 필요                        │
│ - 데이터 영속성 중요                     │
│ - 확장 복잡                            │
│ - 신중한 관리 필요                       │
└──────────────────────────────────────┘

Examples:
- 데이터베이스 (PostgreSQL, MySQL)
- 메시지 큐 (RabbitMQ, Kafka)
- 세션 스토어 (Redis with persistence)
- 파일 스토리지
```

#### Stateless 예시

```bash
# Stateless: 웹 서버 (재시작해도 문제 없음)
docker run -d --name web1 nginx:alpine
docker exec web1 sh -c 'echo "Test" > /usr/share/nginx/html/test.html'
curl localhost/test.html  # Test

# 컨테이너 재시작
docker restart web1
curl localhost/test.html  # 404 (데이터 소실)

# 여러 인스턴스 가능 (상태 없으므로)
docker run -d --name web2 nginx:alpine
docker run -d --name web3 nginx:alpine
# 모두 동일하게 동작

docker rm -f web1 web2 web3
```

#### Stateful 예시

```bash
# Stateful: 데이터베이스 (볼륨 필수)
docker volume create postgres-data

docker run -d --name postgres \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# 데이터 삽입
docker exec -it postgres psql -U postgres -c \
  "CREATE TABLE users (id SERIAL, name VARCHAR(50));"
docker exec -it postgres psql -U postgres -c \
  "INSERT INTO users (name) VALUES ('Alice'), ('Bob');"

# 확인
docker exec -it postgres psql -U postgres -c \
  "SELECT * FROM users;"
#  id | name  
# ----+-------
#   1 | Alice
#   2 | Bob

# 컨테이너 삭제 후 재생성
docker rm -f postgres

docker run -d --name postgres-new \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# 데이터 여전히 존재!
docker exec -it postgres-new psql -U postgres -c \
  "SELECT * FROM users;"
#  id | name  
# ----+-------
#   1 | Alice
#   2 | Bob
# ✅ 영속성 보장!

docker rm -f postgres-new
docker volume rm postgres-data
```

---

### 2. 데이터베이스 영속성 패턴

#### 패턴 1: 단일 볼륨

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres-data:
```

```bash
docker-compose up -d

# 데이터 삽입
docker-compose exec postgres psql -U postgres -c \
  "CREATE DATABASE myapp;"

# 재시작
docker-compose restart

# 데이터 유지 확인
docker-compose exec postgres psql -U postgres -c "\l"
# ✅ myapp 존재

docker-compose down
# 볼륨은 유지됨 (down -v 하지 않으면)
```

#### 패턴 2: 분리된 볼륨 (데이터 + 로그)

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      # 데이터 파일
      - mysql-data:/var/lib/mysql
      
      # 로그 파일 (선택적)
      - mysql-logs:/var/log/mysql
    ports:
      - "3306:3306"

volumes:
  mysql-data:
    driver: local
  
  mysql-logs:
    driver: local
```

#### 패턴 3: 설정 파일 분리

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      # 데이터
      - postgres-data:/var/lib/postgresql/data
      
      # 커스텀 설정 (bind mount)
      - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
      
      # 초기화 스크립트
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    command: postgres -c config_file=/etc/postgresql/postgresql.conf

volumes:
  postgres-data:
```

```bash
# postgresql.conf
cat > postgresql.conf << EOF
max_connections = 200
shared_buffers = 256MB
work_mem = 16MB
EOF

# init.sql
cat > init.sql << EOF
CREATE DATABASE myapp;
\c myapp
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
EOF

docker-compose up -d

# 자동으로 초기화 스크립트 실행됨
docker-compose exec postgres psql -U postgres -d myapp -c "\dt"
# ✅ users 테이블 존재

docker-compose down
```

---

### 3. 백업 전략

#### 전략 1: Volume 백업

```bash
# PostgreSQL 볼륨 백업
docker volume create postgres-data

# 컨테이너 실행
docker run -d --name postgres \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# 테스트 데이터
docker exec postgres psql -U postgres -c \
  "CREATE TABLE test (id SERIAL, data TEXT);"
docker exec postgres psql -U postgres -c \
  "INSERT INTO test (data) VALUES ('Important data');"

# 방법 1: tar로 볼륨 백업
docker run --rm \
  -v postgres-data:/source:ro \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/postgres-backup-$(date +%Y%m%d).tar.gz -C /source .

ls -lh postgres-backup-*.tar.gz
# -rw-r--r-- 1 root root 1.2M Jan 15 10:00 postgres-backup-20240115.tar.gz

# 방법 2: rsync로 백업 (증분)
docker run --rm \
  -v postgres-data:/source:ro \
  -v $(pwd)/backup:/backup \
  instrumentisto/rsync-ssh \
  rsync -av /source/ /backup/

# 복원 (새 볼륨으로)
docker volume create postgres-data-restored

docker run --rm \
  -v postgres-data-restored:/target \
  -v $(pwd):/backup \
  alpine \
  tar xzf /backup/postgres-backup-20240115.tar.gz -C /target

# 복원된 볼륨으로 컨테이너 시작
docker run -d --name postgres-restored \
  -v postgres-data-restored:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# 데이터 확인
docker exec postgres-restored psql -U postgres -c \
  "SELECT * FROM test;"
#  id |      data       
# ----+-----------------
#   1 | Important data
# ✅ 복원 성공!

# 정리
docker rm -f postgres postgres-restored
docker volume rm postgres-data postgres-data-restored
rm -f postgres-backup-*.tar.gz
```

#### 전략 2: Database Dump

```bash
# PostgreSQL dump
docker volume create postgres-data

docker run -d --name postgres \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# 테스트 데이터
docker exec postgres psql -U postgres << EOF
CREATE DATABASE myapp;
\c myapp
CREATE TABLE users (id SERIAL, name VARCHAR(50));
INSERT INTO users (name) VALUES ('Alice'), ('Bob'), ('Charlie');
EOF

# SQL dump 백업
docker exec postgres pg_dump -U postgres myapp > myapp-backup.sql

cat myapp-backup.sql
# --
# -- PostgreSQL database dump
# --
# CREATE TABLE users ...
# COPY users (id, name) FROM stdin;
# 1	Alice
# 2	Bob
# 3	Charlie

# 복원 테스트 (새 데이터베이스)
docker exec postgres psql -U postgres -c "CREATE DATABASE myapp_restored;"

docker exec -i postgres psql -U postgres myapp_restored < myapp-backup.sql

# 확인
docker exec postgres psql -U postgres myapp_restored -c "SELECT * FROM users;"
#  id |  name   
# ----+---------
#   1 | Alice
#   2 | Bob
#   3 | Charlie
# ✅ 복원 성공!

# 정리
docker rm -f postgres
docker volume rm postgres-data
rm myapp-backup.sql
```

#### 전략 3: 자동 백업 (Cron)

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data

  # 백업 서비스
  backup:
    image: postgres:14-alpine
    depends_on:
      - postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data:ro
      - ./backups:/backups
    environment:
      POSTGRES_PASSWORD: secret
    entrypoint: >
      sh -c "
        while true; do
          echo 'Starting backup...'
          PGPASSWORD=secret pg_dump -h postgres -U postgres postgres > /backups/backup-$$(date +%Y%m%d-%H%M%S).sql
          echo 'Backup completed'
          
          # 7일 이상 오래된 백업 삭제
          find /backups -name 'backup-*.sql' -mtime +7 -delete
          
          # 24시간 대기
          sleep 86400
        done
      "

volumes:
  postgres-data:
```

```bash
docker-compose up -d

# 백업 로그 확인
docker-compose logs -f backup
# Starting backup...
# Backup completed

# 백업 파일 확인
ls -lh backups/
# -rw-r--r-- 1 root root 1.2K Jan 15 02:00 backup-20240115-020000.sql

docker-compose down
```

---

### 4. 마이그레이션 패턴

#### 패턴 1: 버전 업그레이드 (PostgreSQL)

```bash
# PostgreSQL 13 시작
docker volume create pg13-data

docker run -d --name postgres13 \
  -v pg13-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:13-alpine

# 데이터 생성
docker exec postgres13 psql -U postgres << EOF
CREATE DATABASE myapp;
\c myapp
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  price NUMERIC(10,2)
);
INSERT INTO products (name, price) VALUES
  ('Laptop', 999.99),
  ('Mouse', 29.99),
  ('Keyboard', 79.99);
EOF

# 데이터 백업 (pg_dump)
docker exec postgres13 pg_dump -U postgres myapp > myapp-pg13.sql

# PostgreSQL 13 중지
docker stop postgres13

# PostgreSQL 14로 업그레이드
docker volume create pg14-data

docker run -d --name postgres14 \
  -v pg14-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# 데이터베이스 생성 및 복원
docker exec postgres14 psql -U postgres -c "CREATE DATABASE myapp;"
docker exec -i postgres14 psql -U postgres myapp < myapp-pg13.sql

# 확인
docker exec postgres14 psql -U postgres myapp -c "SELECT * FROM products;"
#  id |   name   | price  
# ----+----------+--------
#   1 | Laptop   | 999.99
#   2 | Mouse    |  29.99
#   3 | Keyboard |  79.99
# ✅ 마이그레이션 성공!

# 정리
docker rm -f postgres13 postgres14
docker volume rm pg13-data pg14-data
rm myapp-pg13.sql
```

#### 패턴 2: 다른 데이터베이스로 마이그레이션 (MySQL → PostgreSQL)

```bash
# MySQL 시작
docker volume create mysql-data

docker run -d --name mysql \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=myapp \
  mysql:8

# MySQL 데이터
docker exec mysql mysql -proot myapp << EOF
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100)
);
INSERT INTO users (name, email) VALUES
  ('Alice', 'alice@example.com'),
  ('Bob', 'bob@example.com');
EOF

# 데이터 추출 (CSV)
docker exec mysql mysql -proot myapp -e \
  "SELECT * FROM users INTO OUTFILE '/tmp/users.csv' 
   FIELDS TERMINATED BY ',' 
   ENCLOSED BY '\"' 
   LINES TERMINATED BY '\n';"

docker cp mysql:/tmp/users.csv .

cat users.csv
# "1","Alice","alice@example.com"
# "2","Bob","bob@example.com"

# PostgreSQL 시작
docker volume create postgres-data

docker run -d --name postgres \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# PostgreSQL 테이블 생성
docker exec postgres psql -U postgres << EOF
CREATE DATABASE myapp;
\c myapp
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100)
);
EOF

# CSV 데이터 임포트
docker cp users.csv postgres:/tmp/
docker exec postgres psql -U postgres myapp -c \
  "COPY users(id, name, email) FROM '/tmp/users.csv' CSV;"

# 확인
docker exec postgres psql -U postgres myapp -c "SELECT * FROM users;"
#  id | name  |       email        
# ----+-------+--------------------
#   1 | Alice | alice@example.com
#   2 | Bob   | bob@example.com
# ✅ 마이그레이션 성공!

# 정리
docker rm -f mysql postgres
docker volume rm mysql-data postgres-data
rm users.csv
```

---

## 💻 실습

### 실습 1: Stateful 애플리케이션 (WordPress)

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: mysql:8
    volumes:
      - db-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    restart: unless-stopped

  wordpress:
    image: wordpress:latest
    depends_on:
      - db
    ports:
      - "8080:80"
    volumes:
      - wordpress-data:/var/www/html
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    restart: unless-stopped

volumes:
  db-data:
  wordpress-data:
```

```bash
# 시작
docker-compose up -d

# WordPress 설정 (브라우저에서 http://localhost:8080)
# 또는 CLI로 확인
sleep 10
curl -I http://localhost:8080
# HTTP/1.1 200 OK

# 데이터 확인
docker-compose exec db mysql -uwpuser -pwppass wordpress -e "SHOW TABLES;"

# 컨테이너 재생성
docker-compose down
docker-compose up -d

# 데이터 여전히 존재
curl -I http://localhost:8080
# HTTP/1.1 200 OK
# ✅ 설정 유지됨!

# 정리
docker-compose down -v
```

---

### 실습 2: 데이터베이스 백업 자동화

```bash
# 백업 디렉토리 생성
mkdir -p backups

# docker-compose.yml
cat > docker-compose-backup.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  backup:
    image: postgres:14-alpine
    depends_on:
      - postgres
    volumes:
      - ./backups:/backups
    environment:
      PGPASSWORD: secret
    entrypoint: |
      sh -c "
        echo 'Waiting for PostgreSQL to be ready...'
        until pg_isready -h postgres -U postgres; do
          sleep 1
        done
        
        echo 'Starting backup loop...'
        while true; do
          BACKUP_FILE=/backups/backup-$$(date +%Y%m%d-%H%M%S).sql
          echo \"Creating backup: \$$BACKUP_FILE\"
          
          pg_dump -h postgres -U postgres myapp > \$$BACKUP_FILE
          
          if [ -f \$$BACKUP_FILE ]; then
            echo \"Backup successful: \$$BACKUP_FILE\"
            gzip \$$BACKUP_FILE
            echo \"Compressed: \$$BACKUP_FILE.gz\"
          else
            echo \"Backup failed!\"
          fi
          
          # 30일 이상 오래된 백업 삭제
          find /backups -name 'backup-*.sql.gz' -mtime +30 -delete
          echo \"Old backups cleaned up\"
          
          # 6시간 대기
          echo \"Waiting 6 hours until next backup...\"
          sleep 21600
        done
      "

volumes:
  postgres-data:
EOF

# 시작
docker-compose -f docker-compose-backup.yml up -d

# 테스트 데이터 생성
docker-compose -f docker-compose-backup.yml exec postgres psql -U postgres myapp << EOF
CREATE TABLE logs (
  id SERIAL PRIMARY KEY,
  message TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);
INSERT INTO logs (message) VALUES ('Test log 1'), ('Test log 2');
EOF

# 백업 로그 확인
docker-compose -f docker-compose-backup.yml logs backup
# Creating backup: /backups/backup-20240115-100000.sql
# Backup successful
# Compressed: /backups/backup-20240115-100000.sql.gz

# 백업 파일 확인
ls -lh backups/
# -rw-r--r-- 1 root root 512 Jan 15 10:00 backup-20240115-100000.sql.gz

# 복원 테스트
gunzip backups/backup-20240115-100000.sql.gz

docker-compose -f docker-compose-backup.yml exec postgres \
  psql -U postgres -c "DROP DATABASE myapp;"
docker-compose -f docker-compose-backup.yml exec postgres \
  psql -U postgres -c "CREATE DATABASE myapp;"

docker cp backups/backup-20240115-100000.sql \
  $(docker-compose -f docker-compose-backup.yml ps -q postgres):/tmp/

docker-compose -f docker-compose-backup.yml exec postgres \
  psql -U postgres myapp < /tmp/backup-20240115-100000.sql

# 확인
docker-compose -f docker-compose-backup.yml exec postgres \
  psql -U postgres myapp -c "SELECT * FROM logs;"
#  id |  message   |       created_at       
# ----+------------+------------------------
#   1 | Test log 1 | 2024-01-15 10:00:00
#   2 | Test log 2 | 2024-01-15 10:00:00
# ✅ 복원 성공!

# 정리
docker-compose -f docker-compose-backup.yml down -v
rm -rf backups
rm docker-compose-backup.yml
```

---

### 실습 3: 블루-그린 데이터베이스 마이그레이션

```yaml
# docker-compose-blue-green.yml
version: '3.8'

services:
  # Blue (기존)
  db-blue:
    image: postgres:13-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - db-blue-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # Green (새 버전)
  db-green:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - db-green-data:/var/lib/postgresql/data
    ports:
      - "5433:5432"

volumes:
  db-blue-data:
  db-green-data:
```

```bash
# Blue 시작
docker-compose -f docker-compose-blue-green.yml up -d db-blue

# Blue에 데이터 생성
docker-compose -f docker-compose-blue-green.yml exec db-blue \
  psql -U postgres myapp << EOF
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  price NUMERIC(10,2),
  updated_at TIMESTAMP DEFAULT NOW()
);
INSERT INTO products (name, price) VALUES
  ('Product A', 100.00),
  ('Product B', 200.00);
EOF

# Blue 백업
docker-compose -f docker-compose-blue-green.yml exec db-blue \
  pg_dump -U postgres myapp > blue-backup.sql

# Green 시작
docker-compose -f docker-compose-blue-green.yml up -d db-green

# Green으로 데이터 복원
docker cp blue-backup.sql \
  $(docker-compose -f docker-compose-blue-green.yml ps -q db-green):/tmp/

docker-compose -f docker-compose-blue-green.yml exec db-green \
  psql -U postgres myapp < /tmp/blue-backup.sql

# Green 확인
docker-compose -f docker-compose-blue-green.yml exec db-green \
  psql -U postgres myapp -c "SELECT * FROM products;"
#  id |   name    | price  |       updated_at       
# ----+-----------+--------+------------------------
#   1 | Product A | 100.00 | 2024-01-15 10:00:00
#   2 | Product B | 200.00 | 2024-01-15 10:00:00

# 애플리케이션 스위치 (포트 변경)
# Blue: 5432 → Green: 5433
# 설정 변경 후 테스트

# Blue 중지 (문제 없으면)
docker-compose -f docker-compose-blue-green.yml stop db-blue

# 롤백 필요 시 Blue 재시작
# docker-compose -f docker-compose-blue-green.yml start db-blue

# 정리
docker-compose -f docker-compose-blue-green.yml down -v
rm blue-backup.sql docker-compose-blue-green.yml
```

---

## 🔥 실전 적용

### 시나리오 1: 프로덕션 PostgreSQL 클러스터

**요구사항:**
- 고가용성
- 자동 백업
- Point-in-Time Recovery
- 모니터링

```yaml
# docker-compose-prod.yml
version: '3.8'

services:
  # Primary DB
  postgres-primary:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_INITDB_ARGS: "-E UTF8 --locale=C"
    volumes:
      # 데이터
      - postgres-primary-data:/var/lib/postgresql/data
      
      # WAL 아카이브 (PITR)
      - postgres-wal:/var/lib/postgresql/wal_archive
      
      # 설정
      - ./postgresql-primary.conf:/etc/postgresql/postgresql.conf:ro
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Replica DB (Read-only)
  postgres-replica:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-replica-data:/var/lib/postgresql/data
      - ./postgresql-replica.conf:/etc/postgresql/postgresql.conf:ro
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    ports:
      - "5433:5432"
    depends_on:
      - postgres-primary

  # 백업 서비스
  backup:
    image: postgres:14-alpine
    depends_on:
      - postgres-primary
    volumes:
      - postgres-primary-data:/var/lib/postgresql/data:ro
      - postgres-wal:/var/lib/postgresql/wal_archive:ro
      - ./backups:/backups
    environment:
      PGPASSWORD: ${DB_PASSWORD}
    entrypoint: |
      sh -c "
        while true; do
          # Full backup (daily)
          BACKUP_FILE=/backups/full-$$(date +%Y%m%d).tar.gz
          echo \"Creating full backup: \$$BACKUP_FILE\"
          tar czf \$$BACKUP_FILE -C /var/lib/postgresql data wal_archive
          
          # 30일 이상 오래된 백업 삭제
          find /backups -name 'full-*.tar.gz' -mtime +30 -delete
          
          # 24시간 대기
          sleep 86400
        done
      "

  # 모니터링 (Prometheus exporter)
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://postgres:${DB_PASSWORD}@postgres-primary:5432/${DB_NAME}?sslmode=disable"
    ports:
      - "9187:9187"
    depends_on:
      - postgres-primary

volumes:
  postgres-primary-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ssd/postgres-primary

  postgres-replica-data:
    driver: local

  postgres-wal:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ssd/postgres-wal
```

```bash
# postgresql-primary.conf
cat > postgresql-primary.conf << EOF
# Performance
max_connections = 200
shared_buffers = 2GB
work_mem = 16MB
maintenance_work_mem = 512MB
effective_cache_size = 6GB

# WAL (Write-Ahead Log)
wal_level = replica
max_wal_senders = 3
wal_keep_size = 1GB
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/wal_archive/%f && cp %p /var/lib/postgresql/wal_archive/%f'

# Replication
hot_standby = on
EOF

# .env
cat > .env << EOF
DB_PASSWORD=super_secret_password
DB_NAME=myapp
EOF

# 시작
docker-compose -f docker-compose-prod.yml up -d

# 헬스체크 확인
docker-compose -f docker-compose-prod.yml ps

# 백업 확인
ls -lh backups/

# 정리는 생략 (프로덕션 시뮬레이션)
```

---

### 시나리오 2: 마이크로서비스 데이터 관리

**요구사항:**
- 서비스별 독립 데이터베이스
- 공유 캐시 레이어
- 중앙 로깅

```yaml
# docker-compose-microservices.yml
version: '3.8'

services:
  # User Service DB
  user-db:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: users
      POSTGRES_PASSWORD: secret
    volumes:
      - user-db-data:/var/lib/postgresql/data

  user-service:
    image: myapp/user-service
    depends_on:
      - user-db
      - redis
    environment:
      DB_HOST: user-db
      REDIS_HOST: redis

  # Order Service DB
  order-db:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_PASSWORD: secret
    volumes:
      - order-db-data:/var/lib/postgresql/data

  order-service:
    image: myapp/order-service
    depends_on:
      - order-db
      - redis
    environment:
      DB_HOST: order-db
      REDIS_HOST: redis

  # Product Service DB (MongoDB)
  product-db:
    image: mongo:6
    volumes:
      - product-db-data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: secret

  product-service:
    image: myapp/product-service
    depends_on:
      - product-db
      - redis
    environment:
      MONGO_HOST: product-db
      REDIS_HOST: redis

  # Shared Cache
  redis:
    image: redis:alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data

  # Central Logging
  elasticsearch:
    image: elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  logstash:
    image: logstash:8.5.0
    depends_on:
      - elasticsearch
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro

volumes:
  user-db-data:
  order-db-data:
  product-db-data:
  redis-data:
  elasticsearch-data:
```

---

### 시나리오 3: 재해 복구 (Disaster Recovery)

**전략:**

```yaml
# docker-compose-dr.yml
version: '3.8'

services:
  # Primary Site
  postgres-primary:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-primary-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # DR Site Sync
  dr-sync:
    image: postgres:14-alpine
    depends_on:
      - postgres-primary
    volumes:
      - postgres-primary-data:/var/lib/postgresql/data:ro
      - nfs-dr:/dr-backup
    entrypoint: |
      sh -c "
        while true; do
          echo 'Syncing to DR site...'
          
          # Incremental backup to NFS (DR site)
          rsync -avz --delete \
            /var/lib/postgresql/data/ \
            /dr-backup/postgres-latest/
          
          echo 'DR sync completed'
          
          # Every 1 hour
          sleep 3600
        done
      "

volumes:
  postgres-primary-data:
  
  nfs-dr:
    driver: local
    driver_opts:
      type: nfs
      o: addr=dr-site.example.com,rw
      device: ":/dr-backup"
```

**복구 절차:**

```bash
# DR Site에서 복구
docker volume create postgres-recovered

docker run --rm \
  -v nfs-dr:/source:ro \
  -v postgres-recovered:/target \
  alpine \
  cp -a /source/postgres-latest/. /target/

docker run -d --name postgres-recovered \
  -v postgres-recovered:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:14-alpine

# 데이터 확인
# RTO (Recovery Time Objective): ~5분
# RPO (Recovery Point Objective): ~1시간
```

---

## 🚫 안티패턴

### 1. 볼륨 없이 stateful 앱 실행

```yaml
# ❌ 데이터베이스 볼륨 없음
services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
    # volumes 없음!

# 컨테이너 재시작 시 데이터 소실!

# ✅ 올바른 방법
services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

### 2. Named Volume을 down -v로 삭제

```bash
# ❌ 프로덕션 데이터 삭제
docker-compose down -v
# 모든 볼륨 삭제됨!

# ✅ 볼륨 유지
docker-compose down
# 컨테이너만 삭제

# 명시적 볼륨 삭제 (확인 후)
docker volume ls
docker volume rm <volume-name>
```

### 3. 백업 없이 마이그레이션

```bash
# ❌ 백업 없이 업그레이드
docker-compose down
# 버전 변경
docker-compose up -d
# 실패 시 복구 불가!

# ✅ 백업 후 마이그레이션
docker-compose exec db pg_dump ... > backup.sql
docker-compose down
# 버전 변경
docker-compose up -d
# 문제 있으면 복원
```

### 4. Bind Mount로 데이터베이스 데이터

```yaml
# ❌ Bind mount (권한/성능 문제)
services:
  postgres:
    image: postgres:14-alpine
    volumes:
      - ./postgres-data:/var/lib/postgresql/data

# ✅ Named volume
services:
  postgres:
    image: postgres:14-alpine
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

---

## 🎓 핵심 정리

### 1. 영속성 전략

```
Stateless:
- 볼륨 불필요
- 수평 확장 쉬움
- 컨테이너 교체 가능

Stateful:
- 볼륨 필수
- 신중한 관리
- 백업/복구 계획
```

### 2. 백업 전략

```
Volume Backup:
- tar로 전체 볼륨 백업
- 빠르고 간단
- 파일 레벨

Database Dump:
- SQL dump
- 버전 간 호환성
- 선택적 복원

Continuous Backup:
- WAL archiving
- Point-in-Time Recovery
- 최소 데이터 손실
```

### 3. 마이그레이션

```
동일 DB 버전 업:
1. Dump
2. 새 버전 시작
3. Restore
4. 검증

다른 DB로:
1. 데이터 추출 (CSV)
2. 스키마 변환
3. 데이터 로드
4. 검증

블루-그린:
1. Green 준비
2. 데이터 동기화
3. 스위치
4. Blue 종료
```

### 4. 핵심 명령어

```bash
# 볼륨 백업
docker run --rm -v <vol>:/source -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz -C /source .

# DB dump
docker exec <container> pg_dump ... > backup.sql

# 복원
docker exec -i <container> psql ... < backup.sql

# 볼륨 목록
docker volume ls

# 볼륨 삭제 (주의!)
docker volume rm <name>
```

---

## 📚 참고 자료

- [Docker Volumes](https://docs.docker.com/storage/volumes/)
- [PostgreSQL Backup](https://www.postgresql.org/docs/current/backup.html)
- [MySQL Backup](https://dev.mysql.com/doc/refman/8.0/en/backup-and-recovery.html)

---

## 🤔 생각해볼 문제

1. PITR (Point-in-Time Recovery)가 왜 필요한가?
2. 블루-그린 vs 롤링 배포, 데이터베이스에서는?
3. 컨테이너 재시작 vs 재생성, 데이터 영속성 차이는?

> 💡 **답변**: 1) 특정 시점으로 복구 필요 - 예: 실수로 데이터 삭제 (오후 3시), Full backup은 오전 2시 것뿐, PITR: WAL (Write-Ahead Log) 아카이브 재생, 오후 2시 59분 59초로 복구 가능, 구현: archive_mode=on, WAL 보관, pg_basebackup + WAL replay, RTO 짧음, RPO 거의 0 (WAL까지), 2) 블루-그린: 데이터베이스는 위험 - 두 DB 동시 실행 시 데이터 불일치, 스키마 변경 복잡, 스토리지 2배, 권장: 롤링 배포 - 애플리케이션만, DB는 마이그레이션 스크립트, 또는 Read replica 활용, 3) 재시작 (restart): 컨테이너 프로세스만 재시작, 파일시스템 유지 (컨테이너 레이어 유지), 데이터 보존 (볼륨 + 컨테이너 레이어), 재생성 (rm + run): 컨테이너 완전 삭제 후 새로 생성, 컨테이너 레이어 소실, 볼륨만 유지, 따라서 볼륨 없으면 데이터 소실!

---

<div align="center">

**[⬅️ 이전: Storage Drivers](./05-Storage-Drivers.md)** | **[다음: Storage Best Practices ➡️](./07-Storage-Best-Practices.md)**

</div>
