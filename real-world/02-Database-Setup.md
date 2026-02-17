# 02. Database Setup - 데이터베이스 컨테이너화

## 🎯 이 챕터에서 배울 것

- **PostgreSQL 컨테이너화**: 완전한 설정
- **MySQL/MariaDB 설정**: Production-ready
- **MongoDB 구성**: NoSQL 데이터베이스
- **데이터 영속성**: Volume 관리
- **초기화**: Schema, Seed Data
- **백업 및 복구**: 자동화 전략

## 📌 왜 중요한가?

**"데이터베이스는 애플리케이션의 핵심이며, 올바른 컨테이너화가 필수입니다."**

```
Database in Docker:

Without Docker (Traditional):
┌─────────────────────────────────────────────────┐
│ Development Machine                             │
│                                                 │
│ PostgreSQL 설치 (로컬)                            │
│  - 버전 관리 어려움                                 │
│  - 포트 충돌 (5432)                               │
│  - 팀원마다 다른 버전                               │
│  - 정리 어려움                                     │
└─────────────────────────────────────────────────┘

With Docker:
┌─────────────────────────────────────────────────┐
│ docker-compose up                               │
│                                                 │
│  ┌──────────────────┐                           │
│  │   PostgreSQL     │                           │
│  │   in Container   │                           │
│  │                  │                           │
│  │  - 일관된 버전      │                           │
│  │  - 격리된 환경      │                           │
│  │  - 쉬운 정리       │                           │
│  │  - Volume (데이터  │                           │
│  │    영속성)         │                           │
│  └──────────────────┘                           │
└─────────────────────────────────────────────────┘

Data Persistence:
┌─────────────────────────────────────────────────┐
│ Without Volume:                                 │
│  Container 삭제 → 데이터 소실 ❌                    │
│                                                 │
│ With Volume:                                    │
│  Container 삭제 → 데이터 보존 ✅                    │
│                                                 │
│  ┌────────────┐                                 │
│  │ Container  │                                 │
│  │ (ephemeral)│                                 │
│  └─────┬──────┘                                 │
│        │                                        │
│        │ Mount                                  │
│        ▼                                        │
│  ┌────────────┐                                 │
│  │  Volume    │                                 │
│  │(persistent)│                                 │
│  └────────────┘                                 │
│                                                 │
│  컨테이너가 재시작되어도 데이터 유지                     │
└─────────────────────────────────────────────────┘

Database Architecture:
┌─────────────────────────────────────────────────┐
│ Multi-container Application                     │
│                                                 │
│  ┌──────────┐                                   │
│  │ Backend  │                                   │
│  │  API     │                                   │
│  └────┬─────┘                                   │
│       │                                         │
│       │ Connection                              │
│       ▼                                         │
│  ┌──────────┐       ┌──────────┐                │
│  │ Database │◄─────►│  Volume  │                │
│  │Container │       │  (Data)  │                │
│  └──────────┘       └──────────┘                │
│                                                 │
│  초기화:                                          │
│  ┌──────────┐                                   │
│  │ Init SQL │ → Database                        │
│  └──────────┘   (자동 실행)                       │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **일관성**: 모든 환경 동일한 DB 버전
- **격리**: 포트 충돌 없음
- **편리성**: docker-compose up 한 번
- **영속성**: Volume으로 데이터 보존

---

## 🔧 실습 1: PostgreSQL 완전 설정

### Step 1: 기본 PostgreSQL

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: always
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
    volumes:
      # 데이터 영속성
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

```bash
# 실행
docker-compose up -d

# 접속 확인
docker exec -it postgres psql -U myuser -d mydb

# SQL 실행
mydb=# \dt  -- 테이블 목록
mydb=# SELECT version();
mydb=# \q  -- 종료

# 외부에서 접속
psql -h localhost -U myuser -d mydb
```

### Step 2: 초기화 스크립트

```sql
-- init-scripts/01-schema.sql
-- 테이블 생성
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS posts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    content TEXT,
    published BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_published ON posts(published);

-- Trigger: updated_at 자동 업데이트
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

```sql
-- init-scripts/02-seed.sql
-- 초기 데이터
INSERT INTO users (username, email, password_hash) VALUES
    ('alice', 'alice@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5F8/'),
    ('bob', 'bob@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5F8/'),
    ('charlie', 'charlie@example.com', '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5F8/')
ON CONFLICT DO NOTHING;

INSERT INTO posts (user_id, title, content, published) VALUES
    (1, 'First Post', 'This is my first post!', true),
    (1, 'Second Post', 'Another post by Alice', true),
    (2, 'Bob''s Post', 'Hello from Bob', true),
    (3, 'Draft Post', 'This is a draft', false)
ON CONFLICT DO NOTHING;
```

```yaml
# docker-compose.yml (초기화 스크립트 추가)
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: always
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
    volumes:
      # 데이터
      - postgres_data:/var/lib/postgresql/data
      # 초기화 스크립트 (자동 실행)
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### Step 3: Backend 연동

```javascript
// backend/db.js
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST || 'postgres',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || 'mydb',
  user: process.env.DB_USER || 'myuser',
  password: process.env.DB_PASSWORD || 'mypassword',
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// 연결 테스트
pool.on('connect', () => {
  console.log('✅ Connected to PostgreSQL');
});

pool.on('error', (err) => {
  console.error('❌ PostgreSQL error:', err);
});

module.exports = { pool };
```

```javascript
// backend/server.js
const express = require('express');
const { pool } = require('./db');

const app = express();
app.use(express.json());

// Users API
app.get('/api/users', async (req, res) => {
  try {
    const result = await pool.query(
      'SELECT id, username, email, created_at FROM users ORDER BY id'
    );
    res.json(result.rows);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Database error' });
  }
});

app.get('/api/users/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const result = await pool.query(
      'SELECT id, username, email, created_at FROM users WHERE id = $1',
      [id]
    );
    
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Posts API
app.get('/api/posts', async (req, res) => {
  try {
    const result = await pool.query(`
      SELECT p.*, u.username 
      FROM posts p 
      JOIN users u ON p.user_id = u.id 
      WHERE p.published = true 
      ORDER BY p.created_at DESC
    `);
    res.json(result.rows);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Database error' });
  }
});

app.post('/api/posts', async (req, res) => {
  try {
    const { user_id, title, content } = req.body;
    
    const result = await pool.query(
      'INSERT INTO posts (user_id, title, content) VALUES ($1, $2, $3) RETURNING *',
      [user_id, title, content]
    );
    
    res.status(201).json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Database error' });
  }
});

app.listen(8080, () => {
  console.log('Server running on port 8080');
});
```

```json
// backend/package.json
{
  "name": "backend",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.3"
  }
}
```

### Step 4: 완전한 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_USER: ${DB_USER:-myuser}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-mypassword}
      POSTGRES_DB: ${DB_NAME:-mydb}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-myuser} -d ${DB_NAME:-mydb}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  # Backend API
  backend:
    build: ./backend
    container_name: backend
    restart: always
    ports:
      - "8080:8080"
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: ${DB_NAME:-mydb}
      DB_USER: ${DB_USER:-myuser}
      DB_PASSWORD: ${DB_PASSWORD:-mypassword}
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

```env
# .env
DB_USER=myuser
DB_PASSWORD=mypassword
DB_NAME=mydb
```

---

## 🔧 실습 2: MySQL/MariaDB 설정

### Step 1: MySQL Docker Compose

```yaml
# docker-compose.mysql.yml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: mydb
      MYSQL_USER: myuser
      MYSQL_PASSWORD: mypassword
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql-init:/docker-entrypoint-initdb.d
    command: --default-authentication-plugin=mysql_native_password
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-prootpassword"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mysql_data:
```

### Step 2: MySQL 초기화

```sql
-- mysql-init/01-schema.sql
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE INDEX idx_products_price ON products(price);
```

```sql
-- mysql-init/02-seed.sql
INSERT INTO users (username, email, password_hash) VALUES
    ('admin', 'admin@example.com', '$2b$12$hash...'),
    ('user1', 'user1@example.com', '$2b$12$hash...'),
    ('user2', 'user2@example.com', '$2b$12$hash...');

INSERT INTO products (name, description, price, stock) VALUES
    ('Product A', 'Description for Product A', 29.99, 100),
    ('Product B', 'Description for Product B', 49.99, 50),
    ('Product C', 'Description for Product C', 79.99, 25);
```

### Step 3: MariaDB 대안

```yaml
# docker-compose.mariadb.yml
version: '3.8'

services:
  mariadb:
    image: mariadb:10.11
    container_name: mariadb
    restart: always
    ports:
      - "3306:3306"
    environment:
      MARIADB_ROOT_PASSWORD: rootpassword
      MARIADB_DATABASE: mydb
      MARIADB_USER: myuser
      MARIADB_PASSWORD: mypassword
    volumes:
      - mariadb_data:/var/lib/mysql
      - ./mysql-init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  mariadb_data:
```

---

## 🔧 실습 3: MongoDB 설정

### Step 1: MongoDB Docker Compose

```yaml
# docker-compose.mongo.yml
version: '3.8'

services:
  mongodb:
    image: mongo:7
    container_name: mongodb
    restart: always
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: rootpassword
      MONGO_INITDB_DATABASE: mydb
    volumes:
      - mongodb_data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 10s
      timeout: 5s
      retries: 5

  # Mongo Express (GUI)
  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    restart: always
    ports:
      - "8081:8081"
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: rootpassword
      ME_CONFIG_MONGODB_URL: mongodb://root:rootpassword@mongodb:27017/
    depends_on:
      - mongodb

volumes:
  mongodb_data:
```

### Step 2: MongoDB 초기화

```javascript
// mongo-init/01-init.js
db = db.getSiblingDB('mydb');

// Users collection
db.createCollection('users');
db.users.createIndex({ "email": 1 }, { unique: true });
db.users.createIndex({ "username": 1 }, { unique: true });

db.users.insertMany([
  {
    username: "alice",
    email: "alice@example.com",
    profile: {
      firstName: "Alice",
      lastName: "Smith",
      age: 28
    },
    createdAt: new Date()
  },
  {
    username: "bob",
    email: "bob@example.com",
    profile: {
      firstName: "Bob",
      lastName: "Johnson",
      age: 32
    },
    createdAt: new Date()
  }
]);

// Products collection
db.createCollection('products');
db.products.createIndex({ "name": 1 });
db.products.createIndex({ "price": 1 });

db.products.insertMany([
  {
    name: "Laptop",
    description: "High-performance laptop",
    price: 1299.99,
    category: "Electronics",
    tags: ["computer", "laptop", "portable"],
    inStock: true,
    createdAt: new Date()
  },
  {
    name: "Mouse",
    description: "Wireless mouse",
    price: 29.99,
    category: "Accessories",
    tags: ["mouse", "wireless", "computer"],
    inStock: true,
    createdAt: new Date()
  }
]);

print('Database initialized successfully');
```

### Step 3: Node.js + MongoDB 연동

```javascript
// backend/mongodb.js
const { MongoClient } = require('mongodb');

const url = process.env.MONGO_URL || 'mongodb://root:rootpassword@mongodb:27017';
const dbName = process.env.MONGO_DB || 'mydb';

let client;
let db;

async function connectDB() {
  try {
    client = new MongoClient(url);
    await client.connect();
    db = client.db(dbName);
    console.log('✅ Connected to MongoDB');
    return db;
  } catch (err) {
    console.error('❌ MongoDB connection error:', err);
    process.exit(1);
  }
}

function getDB() {
  if (!db) {
    throw new Error('Database not initialized');
  }
  return db;
}

async function closeDB() {
  if (client) {
    await client.close();
    console.log('MongoDB connection closed');
  }
}

module.exports = { connectDB, getDB, closeDB };
```

```javascript
// backend/server.js (MongoDB)
const express = require('express');
const { connectDB, getDB } = require('./mongodb');

const app = express();
app.use(express.json());

// Users API
app.get('/api/users', async (req, res) => {
  try {
    const db = getDB();
    const users = await db.collection('users')
      .find({}, { projection: { password: 0 } })
      .toArray();
    res.json(users);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Database error' });
  }
});

app.post('/api/users', async (req, res) => {
  try {
    const db = getDB();
    const user = {
      ...req.body,
      createdAt: new Date()
    };
    
    const result = await db.collection('users').insertOne(user);
    res.status(201).json({ id: result.insertedId, ...user });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Products API
app.get('/api/products', async (req, res) => {
  try {
    const db = getDB();
    const products = await db.collection('products').find({}).toArray();
    res.json(products);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Database error' });
  }
});

// Start server
connectDB().then(() => {
  app.listen(8080, () => {
    console.log('Server running on port 8080');
  });
});
```

---

## 🔧 실습 4: Redis 캐시 추가

### Step 1: Redis 설정

```yaml
# docker-compose.yml (Redis 추가)
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    # ... (이전 설정)
  
  redis:
    image: redis:7-alpine
    container_name: redis
    restart: always
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --requirepass myredispassword
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - app-network
  
  backend:
    # ... (이전 설정)
    environment:
      # DB 설정
      DB_HOST: postgres
      # Redis 설정
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: myredispassword

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
```

### Step 2: Redis 캐싱 구현

```javascript
// backend/redis.js
const redis = require('redis');

const client = redis.createClient({
  socket: {
    host: process.env.REDIS_HOST || 'redis',
    port: process.env.REDIS_PORT || 6379
  },
  password: process.env.REDIS_PASSWORD || 'myredispassword'
});

client.on('error', (err) => console.error('Redis error:', err));
client.on('connect', () => console.log('✅ Connected to Redis'));

async function connectRedis() {
  await client.connect();
}

module.exports = { client, connectRedis };
```

```javascript
// backend/server.js (캐싱 추가)
const express = require('express');
const { pool } = require('./db');
const { client: redis, connectRedis } = require('./redis');

const app = express();
app.use(express.json());

// Cache middleware
async function cacheMiddleware(req, res, next) {
  const key = `cache:${req.originalUrl}`;
  
  try {
    const cached = await redis.get(key);
    if (cached) {
      console.log('📦 Cache hit:', key);
      return res.json(JSON.parse(cached));
    }
    
    // Cache miss - store result after response
    const originalJson = res.json.bind(res);
    res.json = function(data) {
      redis.setEx(key, 300, JSON.stringify(data)); // 5분 캐시
      return originalJson(data);
    };
    
    next();
  } catch (err) {
    console.error('Cache error:', err);
    next();
  }
}

// Users API with caching
app.get('/api/users', cacheMiddleware, async (req, res) => {
  const result = await pool.query('SELECT id, username, email FROM users');
  res.json(result.rows);
});

// Cache invalidation
app.post('/api/users', async (req, res) => {
  const { username, email } = req.body;
  
  const result = await pool.query(
    'INSERT INTO users (username, email, password_hash) VALUES ($1, $2, $3) RETURNING *',
    [username, email, 'hash']
  );
  
  // Invalidate cache
  await redis.del('cache:/api/users');
  
  res.status(201).json(result.rows[0]);
});

// Start
Promise.all([connectRedis()]).then(() => {
  app.listen(8080, () => console.log('Server running'));
});
```

---

## 🔧 실습 5: 백업 및 복구

### Step 1: PostgreSQL 백업

```bash
# backup.sh
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="./backups"
BACKUP_FILE="${BACKUP_DIR}/postgres_backup_${TIMESTAMP}.sql"

mkdir -p ${BACKUP_DIR}

docker exec postgres pg_dump -U myuser mydb > ${BACKUP_FILE}

echo "Backup created: ${BACKUP_FILE}"

# 오래된 백업 삭제 (7일 이상)
find ${BACKUP_DIR} -name "*.sql" -mtime +7 -delete
```

```bash
# restore.sh
#!/bin/bash
BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: ./restore.sh <backup_file>"
  exit 1
fi

docker exec -i postgres psql -U myuser -d mydb < ${BACKUP_FILE}

echo "Database restored from: ${BACKUP_FILE}"
```

### Step 2: 자동 백업 (Cron)

```yaml
# docker-compose.yml (백업 서비스 추가)
version: '3.8'

services:
  postgres:
    # ... (이전 설정)
  
  # 백업 서비스
  backup:
    image: postgres:15-alpine
    container_name: postgres-backup
    depends_on:
      - postgres
    volumes:
      - ./backups:/backups
      - ./backup-scripts:/scripts
    environment:
      PGPASSWORD: mypassword
    command: >
      sh -c "
      while true; do
        echo 'Running backup...'
        pg_dump -h postgres -U myuser mydb > /backups/backup_$(date +%Y%m%d_%H%M%S).sql
        echo 'Backup completed'
        sleep 86400
      done
      "
    networks:
      - app-network
```

---

## 💡 주요 패턴 정리

```
데이터베이스 컨테이너화 패턴:

1. Volume 사용 (필수)
   - 데이터 영속성
   - 컨테이너 재시작해도 보존

2. 초기화 스크립트
   - /docker-entrypoint-initdb.d/
   - Schema + Seed Data

3. Health Check
   - 서비스 준비 확인
   - depends_on: condition

4. 환경 변수
   - .env 파일 사용
   - 민감 정보 분리

5. 네트워크 격리
   - 내부 네트워크
   - 외부 접근 제한
```

---

## 📌 핵심 요약

```
Database Setup 핵심:
1. Volume으로 데이터 영속성
2. 초기화 스크립트 자동 실행
3. Health Check 설정
4. 백업 자동화
5. Redis 캐싱

Best Practices:
✅ Volume 사용 (필수)
✅ 초기화 스크립트
✅ .env로 민감 정보 관리
✅ Health Check
✅ 정기 백업
```

---

## 📚 참고 자료

- [PostgreSQL Docker Official Image](https://hub.docker.com/_/postgres)
- [MySQL Docker Official Image](https://hub.docker.com/_/mysql)
- [MongoDB Docker Official Image](https://hub.docker.com/_/mongo)
- [Docker Volumes](https://docs.docker.com/storage/volumes/)
- [Database Backup Best Practices](https://www.postgresql.org/docs/current/backup.html)

---

## 🤔 생각해볼 문제

1. 프로덕션 데이터베이스를 Docker로 운영해야 하는가?
2. Volume을 사용하지 않으면 어떻게 되는가?
3. Database 컨테이너가 재시작되면 데이터가 손실되는가?

> 💡 **답변**:
> 
> **1) 프로덕션 DB를 Docker로?**
> 
> ```
> 복잡한 문제, 상황에 따라 다름
> 
> ❌ 권장하지 않음 (대부분):
> - 대규모 프로덕션
> - 미션 크리티컬 데이터
> - 고성능 요구사항
> - 복잡한 복제 구성
> 
> 이유:
> 1. 복잡성
>    - 백업 관리
>    - 복제 설정
>    - 모니터링
>    - 디스크 I/O 성능
> 
> 2. 위험성
>    - Volume 손상
>    - 네트워크 오버헤드
>    - 리소스 경쟁
> 
> 3. 운영 부담
>    - 업그레이드 복잡
>    - HA 구성 어려움
>    - 스냅샷 관리
> 
> ✅ 대안 (권장):
> - Managed DB (RDS, Cloud SQL, Azure DB)
>   → AWS/GCP가 관리
>   → 자동 백업
>   → HA 내장
>   → 성능 최적화
> 
> - Dedicated DB Server
>   → 물리 서버에 직접 설치
>   → 최고 성능
>   → 안정성
> 
> ✅ Docker DB 사용 가능한 경우:
> - 개발/테스트 환경
> - 소규모 서비스 (< 1000 users)
> - 마이크로서비스 (각 서비스별 DB)
> - 일시적 데이터 (캐시)
> 
> 예외 (Docker OK):
> - Kubernetes StatefulSet
>   + 자동 복제
>   + 자동 백업
>   + 오케스트레이션
>   → 프로덕션 가능
> 
> - Redis (캐시)
>   → 데이터 손실 허용
>   → 재생성 가능
> 
> 결론:
> 개발: Docker ✅
> 프로덕션: Managed Service ✅
> 예산 없으면: Dedicated Server ✅
> Docker 프로덕션: StatefulSet만 ⚠️
> ```
> 
> **2) Volume 없으면?**
> 
> ```
> 데이터 손실 보장!
> 
> Volume 없는 컨테이너:
> docker run postgres  # ❌ Volume 없음
> 
> 문제:
> 1. 컨테이너 삭제 → 데이터 소실
>    docker rm postgres
>    → 모든 데이터베이스 사라짐
> 
> 2. 이미지 업데이트 → 데이터 소실
>    docker pull postgres:16
>    docker-compose up --build
>    → 새 컨테이너 = 빈 데이터베이스
> 
> 3. 호스트 재부팅 → 데이터 소실
>    (restart 정책 없으면)
> 
> 4. 컨테이너 재생성 → 데이터 소실
>    docker-compose down
>    docker-compose up
>    → 새 컨테이너 = 처음부터
> 
> 데이터 위치:
> # Volume 없음
> /var/lib/docker/overlay2/...
> → 컨테이너 레이어 (임시)
> 
> # Volume 있음
> /var/lib/docker/volumes/db_data/_data
> → 영구 저장소
> 
> Volume 사용:
> volumes:
>   - postgres_data:/var/lib/postgresql/data
> 
> volumes:
>   postgres_data:
> 
> 확인:
> docker volume ls
> docker volume inspect postgres_data
> 
> 결과:
> ✅ 컨테이너 재시작 → 데이터 유지
> ✅ 이미지 업데이트 → 데이터 유지
> ✅ docker-compose down → 데이터 유지
> 
> 삭제:
> docker-compose down -v  # ← -v로만 삭제
> 
> 결론:
> Volume 없음 = 데이터 손실 100%
> Volume 사용 = 데이터 영구 보존
> **절대** Volume 없이 DB 운영 금지!
> ```
> 
> **3) DB 컨테이너 재시작 시 데이터는?**
> 
> ```
> Volume 있으면 → 데이터 유지 ✅
> Volume 없으면 → 데이터 소실 ❌
> 
> 시나리오별:
> 
> 1. docker restart postgres
>    Volume O: 데이터 유지 ✅
>    Volume X: 데이터 유지 ✅
>    → 같은 컨테이너
> 
> 2. docker stop + docker start
>    Volume O: 데이터 유지 ✅
>    Volume X: 데이터 유지 ✅
>    → 같은 컨테이너
> 
> 3. docker rm + docker run
>    Volume O: 데이터 유지 ✅
>    Volume X: 데이터 소실 ❌
>    → 새 컨테이너
> 
> 4. docker-compose down/up
>    Volume O: 데이터 유지 ✅
>    Volume X: 데이터 소실 ❌
>    → 새 컨테이너
> 
> 5. docker-compose down -v
>    Volume O: 데이터 소실 ❌
>    Volume X: 데이터 소실 ❌
>    → Volume까지 삭제
> 
> 핵심:
> - restart/stop/start: 같은 컨테이너 → 유지
> - rm/down/up: 새 컨테이너 → Volume 필요
> 
> Volume 동작:
> 
> # 첫 실행
> docker-compose up
> → 컨테이너 생성
> → Volume 생성
> → Volume 마운트
> → 데이터 저장
> 
> # 중지
> docker-compose down
> → 컨테이너 삭제
> → Volume 유지 (중요!)
> 
> # 재실행
> docker-compose up
> → 새 컨테이너 생성
> → 기존 Volume 마운트
> → 데이터 복원 ✅
> 
> 확인 방법:
> # 데이터 삽입
> docker exec postgres psql -U user -c "INSERT INTO test VALUES (1)"
> 
> # 컨테이너 재생성
> docker-compose down
> docker-compose up
> 
> # 데이터 확인
> docker exec postgres psql -U user -c "SELECT * FROM test"
> → 1 (데이터 유지!)
> 
> 결론:
> Volume + restart = 안전 ✅
> Volume 없음 = 위험 ❌
> ```

---

<div align="center">

**[⬅️ 이전: Web Application](./01-Web-Application.md)** | **[다음: Reverse Proxy ➡️](./03-Reverse-Proxy.md)**

</div>
