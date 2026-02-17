# 06. Backup System - 자동 백업 시스템

## 🎯 이 챕터에서 배울 것

- **Database 백업**: PostgreSQL, MySQL, MongoDB
- **Volume 백업**: Docker Volume 백업
- **자동화**: Cron, Backup Scheduler
- **복구 전략**: Point-in-time Recovery
- **원격 백업**: S3, Cloud Storage
- **실전 구성**: Production-ready 백업

## 📌 왜 중요한가?

**"백업이 없는 프로덕션은 시한폭탄과 같습니다."**

```
Backup의 중요성:

Without Backup:
┌─────────────────────────────────────────────────┐
│ 사고 발생 (서버 장애, 실수로 삭제, 랜섬웨어)      │
│   ↓                                             │
│ 데이터 손실 💥                                   │
│   ↓                                             │
│ 복구 불가능 ❌                                   │
│   ↓                                             │
│ 비즈니스 중단                                    │
│ 고객 신뢰 상실                                   │
│ 법적 문제                                        │
└─────────────────────────────────────────────────┘

With Backup:
┌─────────────────────────────────────────────────┐
│ 사고 발생                                        │
│   ↓                                             │
│ 백업에서 복구 (1시간 전 데이터)                  │
│   ↓                                             │
│ 서비스 재개 ✅                                   │
│   ↓                                             │
│ 최소한의 데이터 손실                             │
│ 비즈니스 연속성 유지                             │
└─────────────────────────────────────────────────┘

Backup Strategy (3-2-1 Rule):
┌─────────────────────────────────────────────────┐
│ 3-2-1 Backup Rule                               │
│                                                 │
│ 3: 데이터를 3개 복사본으로 유지                  │
│    - Original (운영)                            │
│    - Local Backup (로컬 서버)                   │
│    - Remote Backup (클라우드)                   │
│                                                 │
│ 2: 2개의 다른 미디어에 저장                      │
│    - Local Disk                                 │
│    - Cloud Storage (S3, GCS)                    │
│                                                 │
│ 1: 1개는 오프사이트 (원격지)                     │
│    - 다른 지역의 데이터센터                      │
│    - 화재, 침수 등 물리적 재해 대비              │
└─────────────────────────────────────────────────┘

Backup Architecture:
┌─────────────────────────────────────────────────┐
│                                                 │
│  ┌──────────────┐                               │
│  │  Production  │                               │
│  │   Database   │                               │
│  └──────┬───────┘                               │
│         │                                       │
│         │ (Daily Backup)                        │
│         ▼                                       │
│  ┌──────────────┐                               │
│  │    Local     │                               │
│  │   Backup     │ ← 빠른 복구 (RPO: 1일)        │
│  └──────┬───────┘                               │
│         │                                       │
│         │ (Sync to Cloud)                       │
│         ▼                                       │
│  ┌──────────────┐                               │
│  │  S3/Cloud    │                               │
│  │   Backup     │ ← 장기 보관, 재해 복구         │
│  └──────────────┘                               │
│                                                 │
│  Retention Policy:                              │
│  - Daily: 7일                                   │
│  - Weekly: 4주                                  │
│  - Monthly: 12개월                              │
└─────────────────────────────────────────────────┘

Key Concepts:
┌─────────────────────────────────────────────────┐
│ RPO (Recovery Point Objective)                  │
│  - 손실 가능한 데이터의 최대 시간                │
│  - 예: RPO 1시간 → 최대 1시간 데이터 손실       │
│                                                 │
│ RTO (Recovery Time Objective)                   │
│  - 복구에 걸리는 최대 시간                       │
│  - 예: RTO 4시간 → 4시간 내 서비스 재개         │
│                                                 │
│ Full Backup:                                    │
│  - 모든 데이터 백업                              │
│  - 시간 오래 걸림                                │
│  - 복구 빠름                                     │
│                                                 │
│ Incremental Backup:                             │
│  - 변경된 데이터만 백업                          │
│  - 시간 짧음                                     │
│  - 복구 느림 (Full + 모든 Incremental)          │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **재해 복구**: 데이터 손실 방지
- **규정 준수**: 법적 요구사항 충족
- **비즈니스 연속성**: 서비스 중단 최소화
- **안심**: 언제든 복구 가능

---

## 🔧 실습 1: PostgreSQL 백업

### Step 1: 기본 백업 스크립트

```bash
#!/bin/bash
# backup-postgres.sh

# 설정
DB_HOST="postgres"
DB_PORT="5432"
DB_NAME="mydb"
DB_USER="myuser"
DB_PASSWORD="mypassword"
BACKUP_DIR="/backups/postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/postgres_${DB_NAME}_${TIMESTAMP}.sql"

# 디렉토리 생성
mkdir -p ${BACKUP_DIR}

# 백업 실행
echo "Starting backup: ${BACKUP_FILE}"
PGPASSWORD=${DB_PASSWORD} pg_dump \
  -h ${DB_HOST} \
  -p ${DB_PORT} \
  -U ${DB_USER} \
  -d ${DB_NAME} \
  -F c \
  -b \
  -v \
  -f ${BACKUP_FILE}

# 압축
gzip ${BACKUP_FILE}
BACKUP_FILE="${BACKUP_FILE}.gz"

# 결과 확인
if [ $? -eq 0 ]; then
  SIZE=$(du -h ${BACKUP_FILE} | cut -f1)
  echo "✅ Backup completed: ${BACKUP_FILE} (${SIZE})"
else
  echo "❌ Backup failed"
  exit 1
fi

# 오래된 백업 삭제 (7일 이상)
find ${BACKUP_DIR} -name "*.sql.gz" -mtime +7 -delete
echo "🗑️  Cleaned up old backups (>7 days)"

# 백업 검증
gunzip -t ${BACKUP_FILE}
if [ $? -eq 0 ]; then
  echo "✅ Backup file is valid"
else
  echo "❌ Backup file is corrupted"
  exit 1
fi
```

### Step 2: Docker Compose with Backup

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app

  # Backup Container
  postgres-backup:
    image: postgres:15-alpine
    container_name: postgres-backup
    restart: always
    volumes:
      - ./backup-scripts:/scripts
      - ./backups:/backups
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=mydb
      - DB_USER=myuser
      - DB_PASSWORD=mypassword
    command: >
      sh -c "
      while true; do
        echo 'Running backup at' \$(date)
        /scripts/backup-postgres.sh
        echo 'Next backup in 24 hours'
        sleep 86400
      done
      "
    depends_on:
      - postgres
    networks:
      - app

volumes:
  postgres_data:

networks:
  app:
    driver: bridge
```

### Step 3: 복구 스크립트

```bash
#!/bin/bash
# restore-postgres.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: $0 <backup_file.sql.gz>"
  exit 1
fi

if [ ! -f "$BACKUP_FILE" ]; then
  echo "Error: Backup file not found: $BACKUP_FILE"
  exit 1
fi

# 압축 해제
gunzip -c ${BACKUP_FILE} > /tmp/restore.sql

# 복구 전 확인
echo "⚠️  WARNING: This will restore the database"
echo "Backup file: ${BACKUP_FILE}"
echo "Database: ${DB_NAME}"
read -p "Continue? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
  echo "Aborted"
  exit 0
fi

# 기존 연결 종료
PGPASSWORD=${DB_PASSWORD} psql \
  -h ${DB_HOST} \
  -U ${DB_USER} \
  -d postgres \
  -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='${DB_NAME}';"

# 데이터베이스 삭제 및 재생성
PGPASSWORD=${DB_PASSWORD} psql \
  -h ${DB_HOST} \
  -U ${DB_USER} \
  -d postgres \
  -c "DROP DATABASE IF EXISTS ${DB_NAME};"

PGPASSWORD=${DB_PASSWORD} psql \
  -h ${DB_HOST} \
  -U ${DB_USER} \
  -d postgres \
  -c "CREATE DATABASE ${DB_NAME};"

# 복구
PGPASSWORD=${DB_PASSWORD} pg_restore \
  -h ${DB_HOST} \
  -p ${DB_PORT} \
  -U ${DB_USER} \
  -d ${DB_NAME} \
  -v \
  /tmp/restore.sql

if [ $? -eq 0 ]; then
  echo "✅ Restore completed"
else
  echo "❌ Restore failed"
  exit 1
fi

# 임시 파일 삭제
rm /tmp/restore.sql
```

---

## 🔧 실습 2: MySQL 백업

### Step 1: MySQL 백업 스크립트

```bash
#!/bin/bash
# backup-mysql.sh

# 설정
DB_HOST="mysql"
DB_PORT="3306"
DB_NAME="mydb"
DB_USER="myuser"
DB_PASSWORD="mypassword"
BACKUP_DIR="/backups/mysql"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/mysql_${DB_NAME}_${TIMESTAMP}.sql"

mkdir -p ${BACKUP_DIR}

# 백업 실행
echo "Starting MySQL backup: ${BACKUP_FILE}"
mysqldump \
  -h ${DB_HOST} \
  -P ${DB_PORT} \
  -u ${DB_USER} \
  -p${DB_PASSWORD} \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  ${DB_NAME} > ${BACKUP_FILE}

# 압축
gzip ${BACKUP_FILE}
BACKUP_FILE="${BACKUP_FILE}.gz"

if [ $? -eq 0 ]; then
  SIZE=$(du -h ${BACKUP_FILE} | cut -f1)
  echo "✅ MySQL backup completed: ${BACKUP_FILE} (${SIZE})"
else
  echo "❌ MySQL backup failed"
  exit 1
fi

# 오래된 백업 삭제
find ${BACKUP_DIR} -name "*.sql.gz" -mtime +7 -delete
```

### Step 2: MySQL 복구

```bash
#!/bin/bash
# restore-mysql.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: $0 <backup_file.sql.gz>"
  exit 1
fi

# 압축 해제 및 복구
gunzip -c ${BACKUP_FILE} | mysql \
  -h ${DB_HOST} \
  -P ${DB_PORT} \
  -u ${DB_USER} \
  -p${DB_PASSWORD} \
  ${DB_NAME}

if [ $? -eq 0 ]; then
  echo "✅ MySQL restore completed"
else
  echo "❌ MySQL restore failed"
  exit 1
fi
```

---

## 🔧 실습 3: MongoDB 백업

### Step 1: MongoDB 백업

```bash
#!/bin/bash
# backup-mongodb.sh

# 설정
MONGO_HOST="mongodb"
MONGO_PORT="27017"
MONGO_USER="root"
MONGO_PASSWORD="rootpassword"
MONGO_DB="mydb"
BACKUP_DIR="/backups/mongodb"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="${BACKUP_DIR}/mongodb_${TIMESTAMP}"

mkdir -p ${BACKUP_DIR}

# 백업 실행
echo "Starting MongoDB backup: ${BACKUP_PATH}"
mongodump \
  --host ${MONGO_HOST} \
  --port ${MONGO_PORT} \
  --username ${MONGO_USER} \
  --password ${MONGO_PASSWORD} \
  --authenticationDatabase admin \
  --db ${MONGO_DB} \
  --out ${BACKUP_PATH}

# 압축
tar -czf ${BACKUP_PATH}.tar.gz -C ${BACKUP_DIR} $(basename ${BACKUP_PATH})
rm -rf ${BACKUP_PATH}

if [ $? -eq 0 ]; then
  SIZE=$(du -h ${BACKUP_PATH}.tar.gz | cut -f1)
  echo "✅ MongoDB backup completed: ${BACKUP_PATH}.tar.gz (${SIZE})"
else
  echo "❌ MongoDB backup failed"
  exit 1
fi

# 오래된 백업 삭제
find ${BACKUP_DIR} -name "*.tar.gz" -mtime +7 -delete
```

### Step 2: MongoDB 복구

```bash
#!/bin/bash
# restore-mongodb.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: $0 <backup_file.tar.gz>"
  exit 1
fi

# 압축 해제
TEMP_DIR="/tmp/mongodb_restore"
mkdir -p ${TEMP_DIR}
tar -xzf ${BACKUP_FILE} -C ${TEMP_DIR}

# 복구
mongorestore \
  --host ${MONGO_HOST} \
  --port ${MONGO_PORT} \
  --username ${MONGO_USER} \
  --password ${MONGO_PASSWORD} \
  --authenticationDatabase admin \
  --db ${MONGO_DB} \
  --drop \
  ${TEMP_DIR}/$(ls ${TEMP_DIR})/${MONGO_DB}

if [ $? -eq 0 ]; then
  echo "✅ MongoDB restore completed"
else
  echo "❌ MongoDB restore failed"
  exit 1
fi

# 정리
rm -rf ${TEMP_DIR}
```

---

## 🔧 실습 4: Docker Volume 백업

### Step 1: Volume 백업 스크립트

```bash
#!/bin/bash
# backup-volumes.sh

VOLUME_NAME=$1
BACKUP_DIR="/backups/volumes"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/${VOLUME_NAME}_${TIMESTAMP}.tar.gz"

if [ -z "$VOLUME_NAME" ]; then
  echo "Usage: $0 <volume_name>"
  exit 1
fi

mkdir -p ${BACKUP_DIR}

# Volume 백업 (임시 컨테이너 사용)
echo "Backing up volume: ${VOLUME_NAME}"
docker run --rm \
  -v ${VOLUME_NAME}:/data \
  -v ${BACKUP_DIR}:/backup \
  alpine \
  tar -czf /backup/$(basename ${BACKUP_FILE}) -C /data .

if [ $? -eq 0 ]; then
  SIZE=$(du -h ${BACKUP_FILE} | cut -f1)
  echo "✅ Volume backup completed: ${BACKUP_FILE} (${SIZE})"
else
  echo "❌ Volume backup failed"
  exit 1
fi

# 오래된 백업 삭제
find ${BACKUP_DIR} -name "${VOLUME_NAME}_*.tar.gz" -mtime +7 -delete
```

### Step 2: Volume 복구

```bash
#!/bin/bash
# restore-volumes.sh

VOLUME_NAME=$1
BACKUP_FILE=$2

if [ -z "$VOLUME_NAME" ] || [ -z "$BACKUP_FILE" ]; then
  echo "Usage: $0 <volume_name> <backup_file>"
  exit 1
fi

# Volume 생성 (없으면)
docker volume create ${VOLUME_NAME}

# Volume 복구
echo "Restoring volume: ${VOLUME_NAME}"
docker run --rm \
  -v ${VOLUME_NAME}:/data \
  -v $(dirname ${BACKUP_FILE}):/backup \
  alpine \
  sh -c "cd /data && tar -xzf /backup/$(basename ${BACKUP_FILE})"

if [ $? -eq 0 ]; then
  echo "✅ Volume restore completed"
else
  echo "❌ Volume restore failed"
  exit 1
fi
```

---

## 🔧 실습 5: S3로 원격 백업

### Step 1: S3 업로드 스크립트

```bash
#!/bin/bash
# upload-to-s3.sh

AWS_ACCESS_KEY_ID="your-access-key"
AWS_SECRET_ACCESS_KEY="your-secret-key"
S3_BUCKET="my-backup-bucket"
S3_PREFIX="docker-backups"
LOCAL_BACKUP_DIR="/backups"

# AWS CLI 설치 확인
if ! command -v aws &> /dev/null; then
  echo "Installing AWS CLI..."
  pip install awscli
fi

# S3 동기화
echo "Syncing backups to S3..."
aws s3 sync \
  ${LOCAL_BACKUP_DIR} \
  s3://${S3_BUCKET}/${S3_PREFIX} \
  --storage-class STANDARD_IA \
  --delete

if [ $? -eq 0 ]; then
  echo "✅ S3 sync completed"
else
  echo "❌ S3 sync failed"
  exit 1
fi

# Lifecycle Policy (선택)
# S3에서 자동으로:
# - 30일 후 Glacier로 이동
# - 90일 후 삭제
```

### Step 2: Docker Compose with S3 Backup

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    # ... (이전 설정)

  # Local Backup
  postgres-backup:
    image: postgres:15-alpine
    volumes:
      - ./backup-scripts:/scripts
      - ./backups:/backups
    command: >
      sh -c "
      while true; do
        /scripts/backup-postgres.sh
        sleep 86400
      done
      "

  # S3 Sync
  s3-sync:
    image: amazon/aws-cli
    container_name: s3-sync
    restart: always
    volumes:
      - ./backups:/backups
    environment:
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_DEFAULT_REGION=us-east-1
    command: >
      sh -c "
      while true; do
        echo 'Syncing to S3...'
        aws s3 sync /backups s3://my-backup-bucket/docker-backups --delete
        echo 'Next sync in 6 hours'
        sleep 21600
      done
      "
```

---

## 🔧 실습 6: 자동화 백업 시스템 (Cron)

### Step 1: Cron 기반 백업

```bash
# crontab -e
# 매일 새벽 2시에 백업
0 2 * * * /path/to/backup-postgres.sh >> /var/log/backup.log 2>&1

# 매주 일요일 새벽 3시에 Full 백업
0 3 * * 0 /path/to/backup-full.sh >> /var/log/backup-full.log 2>&1

# 매 6시간마다 S3 동기화
0 */6 * * * /path/to/upload-to-s3.sh >> /var/log/s3-sync.log 2>&1
```

### Step 2: Backup 모니터링

```bash
#!/bin/bash
# check-backup.sh

BACKUP_DIR="/backups/postgres"
MAX_AGE_HOURS=24

# 최근 백업 확인
LATEST_BACKUP=$(ls -t ${BACKUP_DIR}/*.sql.gz 2>/dev/null | head -1)

if [ -z "$LATEST_BACKUP" ]; then
  echo "❌ No backups found"
  exit 1
fi

# 백업 나이 확인
BACKUP_AGE=$(( ($(date +%s) - $(stat -f%m "$LATEST_BACKUP")) / 3600 ))

if [ $BACKUP_AGE -gt $MAX_AGE_HOURS ]; then
  echo "⚠️  Backup is old: ${BACKUP_AGE} hours"
  # Alert (Slack, Email)
  curl -X POST https://hooks.slack.com/services/YOUR/WEBHOOK \
    -d "{\"text\":\"⚠️ Backup is old: ${BACKUP_AGE} hours\"}"
  exit 1
else
  echo "✅ Backup is recent: ${BACKUP_AGE} hours old"
fi

# 백업 크기 확인
BACKUP_SIZE=$(du -h "$LATEST_BACKUP" | cut -f1)
echo "Backup size: ${BACKUP_SIZE}"
```

---

## 🔧 실습 7: 완전한 백업 시스템

### Step 1: 전체 Docker Compose

```yaml
# docker-compose.backup.yml
version: '3.8'

services:
  # Application
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app

  backend:
    build: ./backend
    container_name: backend
    restart: always
    depends_on:
      - postgres
    networks:
      - app

  # Backup Services
  backup-postgres:
    image: postgres:15-alpine
    container_name: backup-postgres
    restart: always
    volumes:
      - ./backup-scripts:/scripts
      - ./backups/postgres:/backups/postgres
    environment:
      - DB_HOST=postgres
      - DB_NAME=mydb
      - DB_USER=myuser
      - DB_PASSWORD=mypassword
    command: >
      sh -c "
      while true; do
        echo '[Backup] Starting at' \$(date)
        /scripts/backup-postgres.sh
        /scripts/backup-volumes.sh postgres_data
        echo '[Backup] Completed'
        sleep 86400
      done
      "
    depends_on:
      - postgres
    networks:
      - app

  backup-s3-sync:
    image: amazon/aws-cli
    container_name: backup-s3-sync
    restart: always
    volumes:
      - ./backups:/backups
    environment:
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_DEFAULT_REGION=us-east-1
    command: >
      sh -c "
      while true; do
        echo '[S3] Syncing backups at' \$(date)
        aws s3 sync /backups s3://my-backup-bucket/backups \
          --storage-class STANDARD_IA \
          --delete
        echo '[S3] Next sync in 6 hours'
        sleep 21600
      done
      "

  backup-monitor:
    image: alpine
    container_name: backup-monitor
    restart: always
    volumes:
      - ./backup-scripts:/scripts
      - ./backups:/backups
    command: >
      sh -c "
      apk add --no-cache curl
      while true; do
        echo '[Monitor] Checking backups at' \$(date)
        /scripts/check-backup.sh
        sleep 3600
      done
      "

volumes:
  postgres_data:

networks:
  app:
    driver: bridge
```

### Step 2: 백업 복구 테스트

```bash
# test-restore.sh
#!/bin/bash

echo "=== Backup & Restore Test ==="

# 1. 테스트 데이터 생성
echo "Creating test data..."
docker exec postgres psql -U myuser -d mydb -c "CREATE TABLE test (id SERIAL, value TEXT);"
docker exec postgres psql -U myuser -d mydb -c "INSERT INTO test (value) VALUES ('test1'), ('test2'), ('test3');"

# 2. 백업 실행
echo "Running backup..."
./backup-scripts/backup-postgres.sh

# 3. 데이터 삭제
echo "Deleting test data..."
docker exec postgres psql -U myuser -d mydb -c "DROP TABLE test;"

# 4. 백업 복구
echo "Restoring from backup..."
LATEST_BACKUP=$(ls -t backups/postgres/*.sql.gz | head -1)
./backup-scripts/restore-postgres.sh ${LATEST_BACKUP}

# 5. 검증
echo "Verifying restore..."
RESULT=$(docker exec postgres psql -U myuser -d mydb -t -c "SELECT COUNT(*) FROM test;")

if [ "$RESULT" -eq 3 ]; then
  echo "✅ Restore test passed"
else
  echo "❌ Restore test failed"
  exit 1
fi

# 6. 정리
docker exec postgres psql -U myuser -d mydb -c "DROP TABLE test;"
echo "✅ Test completed"
```

---

## 💡 백업 전략 정리

```
Backup Schedule:
┌─────────────────┬──────────┬────────────┐
│ 백업 타입        │ 주기      │ 보존 기간   │
├─────────────────┼──────────┼────────────┤
│ Full Backup     │ 매주      │ 4주        │
├─────────────────┼──────────┼────────────┤
│ Daily Backup    │ 매일      │ 7일        │
├─────────────────┼──────────┼────────────┤
│ Hourly Snapshot │ 매시간    │ 24시간     │
├─────────────────┼──────────┼────────────┤
│ Cloud Sync      │ 6시간마다 │ 90일       │
└─────────────────┴──────────┴────────────┘

Backup Checklist:
✅ 자동화 (Cron, Scheduler)
✅ 압축 (gzip)
✅ 암호화 (GPG, 선택)
✅ 원격 저장 (S3, GCS)
✅ 정기 복구 테스트
✅ 모니터링 및 알림
✅ 문서화 (복구 절차)

Recovery Procedures:
1. 백업 파일 확인
2. 서비스 중지 (선택)
3. 백업 복구
4. 검증
5. 서비스 재시작
```

---

## 📌 핵심 요약

```
Backup System 핵심:
1. 3-2-1 Rule (3 복사본, 2 미디어, 1 오프사이트)
2. 자동화 (Cron, Scheduler)
3. 원격 백업 (S3, Cloud)
4. 정기 테스트 (복구 가능한지 확인)
5. 모니터링 및 알림

Best Practices:
✅ 일일 백업 (최소)
✅ 압축 및 암호화
✅ 오래된 백업 자동 삭제
✅ S3 Glacier (장기 보관)
✅ 정기 복구 테스트 (월 1회)
✅ 문서화된 복구 절차
```

---

<div align="center">

**[⬅️ 이전: Log Aggregation](./05-Log-Aggregation.md)** | **[다음: Multi-Tier App ➡️](./07-Multi-Tier-App.md)**

</div>
