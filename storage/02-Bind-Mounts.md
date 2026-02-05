# 02. Bind Mounts - 바인드 마운트

## 🎯 이 챕터에서 배울 것

- **Bind Mount**의 개념과 동작 원리
- Volume vs Bind Mount **차이점**
- **개발 환경**에서의 활용 (Hot Reload)
- 실전 **사용 패턴**과 보안 고려사항

## 📌 왜 중요한가?

**"Bind Mount는 호스트와 컨테이너를 직접 연결합니다."**

```
Volume:
┌─────────────────────────────────┐
│ Container                       │
│ /app/data                       │
└────────┬────────────────────────┘
         │ 마운트
┌────────▼────────────────────────┐
│ Docker 관리 영역                  │
│ /var/lib/docker/volumes/...     │
│ ✅ Docker가 관리                  │
│ ✅ 이식성 높음                     │
└─────────────────────────────────┘

Bind Mount:
┌─────────────────────────────────┐
│ Container                       │
│ /app/src                        │
└────────┬────────────────────────┘
         │ 직접 마운트
┌────────▼────────────────────────┐
│ Host 파일시스템                    │
│ /home/user/project/src          │
│ ✅ 직접 접근                      │
│ ✅ 즉시 반영                      │
└─────────────────────────────────┘

차이:
- Volume: Docker 관리, 추상화
- Bind Mount: 호스트 경로 직접 노출
```

**실무 영향:**
- 개발: 코드 변경 즉시 반영 (Hot Reload)
- 설정: 호스트 설정 파일 주입
- 로그: 실시간 로그 확인
- 성능: 호스트 I/O 직접 사용

---

## 🔬 Deep Dive

### 1. Bind Mount 개념

#### 기본 구조

```
Bind Mount:
- 호스트의 파일/디렉토리를 컨테이너에 직접 마운트
- 호스트 경로 절대 경로로 지정
- 양방향 동기화 (호스트 ↔ 컨테이너)
- Docker가 관리하지 않음

구조:
┌───────────────────────────────────┐
│ Host                              │
│ /home/user/project/               │
│ ├── src/                          │
│ │   ├── index.js                  │
│ │   └── utils.js                  │
│ ├── config/                       │
│ │   └── app.yml                   │
│ └── logs/                         │
│     └── app.log                   │
└────┬──────────┬─────────┬─────────┘
     │          │         │
     │ 마운트   │ 마운트  │ 마운트
     ▼          ▼         ▼
┌───────────────────────────────────┐
│ Container                         │
│ /app/                             │
│ ├── src/         ← 호스트 src/      │
│ ├── config/      ← 호스트 config/   │
│ └── logs/        ← 호스트 logs/     │
└───────────────────────────────────┘

특징:
- 호스트에서 파일 수정 → 컨테이너에 즉시 반영
- 컨테이너에서 파일 생성 → 호스트에 즉시 나타남
- 양방향 실시간 동기화
```

#### 기본 사용법

```bash
# -v 구문 (레거시)
docker run -v /host/path:/container/path image

# --mount 구문 (권장)
docker run --mount type=bind,source=/host/path,target=/container/path image

# 예시
docker run -d --name web \
  --mount type=bind,source=$(pwd)/html,target=/usr/share/nginx/html \
  nginx:alpine

# 또는
docker run -d --name web \
  -v $(pwd)/html:/usr/share/nginx/html \
  nginx:alpine
```

---

### 2. Volume vs Bind Mount 비교

#### 상세 비교

```
┌──────────────────┬────────────────┬──────────────────┐
│ 특징              │ Volume         │ Bind Mount       │
├──────────────────┼────────────────┼──────────────────┤
│ 관리 주체          │ Docker         │ 사용자             │
├──────────────────┼────────────────┼──────────────────┤
│ 위치              │ Docker 관리 영역 │ 호스트 임의 위치     │
├──────────────────┼────────────────┼──────────────────┤
│ 경로 지정          │ 이름            │ 절대 경로           │
├──────────────────┼────────────────┼──────────────────┤
│ 이식성             │ 높음            │ 낮음              │
├──────────────────┼────────────────┼──────────────────┤
│ 백업              │ 쉬움            │ 수동              │
├──────────────────┼────────────────┼──────────────────┤
│ 초기 내용          │ 컨테이너 내용      │ 호스트 내용        │
├──────────────────┼────────────────┼──────────────────┤
│ 성능 (Linux)      │ 동일            │ 동일              │
├──────────────────┼────────────────┼──────────────────┤
│ 성능 (Mac/Win)    │ 빠름            │ 느림              │
├──────────────────┼────────────────┼──────────────────┤
│ 권한 문제          │ 적음            │ 많음              │
├──────────────────┼────────────────┼──────────────────┤
│ 사용 케이스         │ 프로덕션 데이터    │ 개발, 설정         │
└──────────────────┴────────────────┴──────────────────┘
```

#### 초기 내용 동작

```bash
# Volume: 컨테이너 내용 → 볼륨으로 복사
docker run -d --name test-vol \
  -v my-vol:/usr/share/nginx/html \
  nginx:alpine

# 볼륨에 nginx 기본 파일들이 복사됨
docker run --rm \
  -v my-vol:/data \
  alpine ls -la /data
# index.html, 50x.html 등 존재

docker rm -f test-vol
docker volume rm my-vol

# Bind Mount: 호스트 내용이 우선
mkdir empty-dir

docker run -d --name test-bind \
  -v $(pwd)/empty-dir:/usr/share/nginx/html \
  nginx:alpine

# 컨테이너에서 빈 디렉토리
docker exec test-bind ls /usr/share/nginx/html
# (아무것도 없음)

# nginx는 파일이 없어서 403 에러 발생
curl http://localhost:80
# 403 Forbidden

docker rm -f test-bind
rm -rf empty-dir
```

---

### 3. 개발 환경에서의 활용

#### Hot Reload (코드 변경 즉시 반영)

**Node.js 예시:**

```bash
# 프로젝트 구조
mkdir -p myapp
cd myapp

# package.json
cat > package.json << 'EOF'
{
  "name": "myapp",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.0"
  }
}
EOF

# index.js
cat > index.js << 'EOF'
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.json({ message: 'Hello World' });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

# Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install
CMD ["npm", "run", "dev"]
EOF

# 이미지 빌드
docker build -t myapp-dev .

# Bind Mount로 실행 (소스 코드)
docker run -d --name myapp \
  -v $(pwd):/app \
  -v /app/node_modules \
  -p 3000:3000 \
  myapp-dev

# 테스트
curl http://localhost:3000
# {"message":"Hello World"}

# 코드 수정 (호스트에서)
cat > index.js << 'EOF'
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.json({ message: 'Hello Docker!' });  // 변경!
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

# 즉시 반영 (nodemon이 재시작)
sleep 2
curl http://localhost:3000
# {"message":"Hello Docker!"}
# ✅ 즉시 반영됨!

# 정리
docker rm -f myapp
cd ..
rm -rf myapp
```

---

### 4. 읽기 전용 마운트

#### 보안 강화

```bash
# 읽기 전용 마운트 (:ro)
docker run -d --name web \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:alpine

# 또는 --mount
docker run -d --name web \
  --mount type=bind,source=$(pwd)/html,target=/usr/share/nginx/html,readonly \
  nginx:alpine

# 컨테이너에서 쓰기 시도
docker exec web sh -c \
  'echo "test" > /usr/share/nginx/html/test.txt'
# sh: can't create /usr/share/nginx/html/test.txt: Read-only file system
# ❌ 차단됨!

# 읽기는 가능
docker exec web cat /usr/share/nginx/html/index.html
# ✅ 성공
```

#### 사용 시나리오

```
읽기 전용 권장:
✅ 설정 파일 (config, secrets)
✅ 소스 코드 (프로덕션)
✅ 정적 파일 (HTML, CSS, JS)
✅ 인증서, 키

쓰기 필요:
⚠️ 로그 디렉토리
⚠️ 업로드 디렉토리
⚠️ 캐시 디렉토리
⚠️ 데이터베이스 파일
```

---

### 5. 권한 문제 해결

#### UID/GID 불일치

```bash
# 문제 상황
mkdir app-data

# 컨테이너 실행 (기본 root)
docker run -d --name app \
  -v $(pwd)/app-data:/data \
  alpine sh -c 'echo "test" > /data/file.txt && sleep 3600'

# 호스트에서 파일 확인
ls -la app-data/
# -rw-r--r-- 1 root root 5 Jan 15 10:00 file.txt
# root 소유!

# 호스트에서 수정 시도 (일반 사용자)
echo "update" > app-data/file.txt
# Permission denied
# ❌ 권한 없음!

docker rm -f app
rm -rf app-data
```

#### 해결 방법 1: USER 지시어

```dockerfile
# Dockerfile
FROM alpine

# 사용자 생성 (UID 1000)
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

USER appuser

CMD ["sh", "-c", "echo test > /data/file.txt && sleep 3600"]
```

```bash
docker build -t myapp .

mkdir app-data

docker run -d --name app \
  -v $(pwd)/app-data:/data \
  myapp

# 호스트에서 파일 확인
ls -la app-data/
# -rw-r--r-- 1 user user 5 Jan 15 10:00 file.txt
# 일반 사용자 소유!

docker rm -f app
rm -rf app-data
```

#### 해결 방법 2: --user 플래그

```bash
mkdir app-data

# 현재 사용자 UID/GID로 실행
docker run -d --name app \
  --user $(id -u):$(id -g) \
  -v $(pwd)/app-data:/data \
  alpine sh -c 'echo "test" > /data/file.txt && sleep 3600'

# 파일 확인
ls -la app-data/
# -rw-r--r-- 1 user user 5 Jan 15 10:00 file.txt
# ✅ 호스트 사용자 소유

docker rm -f app
rm -rf app-data
```

---

## 💻 실습

### 실습 1: 개발 환경 구성 (React)

#### 프로젝트 생성

```bash
# React 앱 생성
npx create-react-app react-docker-dev
cd react-docker-dev

# Dockerfile.dev
cat > Dockerfile.dev << 'EOF'
FROM node:18-alpine

WORKDIR /app

# package.json만 먼저 복사 (캐싱 최적화)
COPY package.json package-lock.json ./
RUN npm install

# 소스는 Bind Mount로
CMD ["npm", "start"]
EOF

# docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      # 소스 코드 (Bind Mount)
      - ./src:/app/src:ro
      - ./public:/app/public:ro
      
      # node_modules (Anonymous Volume - 우선순위)
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - CHOKIDAR_USEPOLLING=true  # Hot reload 안정화
    stdin_open: true
    tty: true
EOF

# 시작
docker-compose up -d

# 로그 확인
docker-compose logs -f

# 브라우저: http://localhost:3000
```

#### Hot Reload 테스트

```bash
# src/App.js 수정
cat > src/App.js << 'EOF'
import React from 'react';

function App() {
  return (
    <div>
      <h1>Docker Hot Reload Works!</h1>
      <p>Edit src/App.js and save to test.</p>
    </div>
  );
}

export default App;
EOF

# 브라우저 자동 새로고침!
# ✅ 즉시 반영됨!

# 정리
docker-compose down
cd ..
rm -rf react-docker-dev
```

---

### 실습 2: 설정 파일 주입

#### Nginx 설정 커스터마이징

```bash
# 프로젝트 구조
mkdir nginx-config
cd nginx-config

# 커스텀 nginx.conf
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    server {
        listen 80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        location /api/ {
            proxy_pass http://api:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF

# HTML 파일
mkdir html
cat > html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Custom Config</title>
</head>
<body>
    <h1>Nginx with Custom Config</h1>
    <p>Configuration loaded from host!</p>
</body>
</html>
EOF

# docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      # 설정 파일 (읽기 전용)
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      
      # HTML 파일
      - ./html:/usr/share/nginx/html:ro
      
      # 로그 (쓰기 가능)
      - ./logs:/var/log/nginx
    networks:
      - app-net

  api:
    image: nginx:alpine
    networks:
      - app-net

networks:
  app-net:
EOF

# 시작
mkdir logs
docker-compose up -d

# 테스트
curl http://localhost/
# Custom Config HTML

curl http://localhost/health
# healthy

# 로그 확인 (실시간)
tail -f logs/access.log

# 설정 수정 및 재로드
# nginx.conf 수정 후
docker-compose exec nginx nginx -s reload

# 정리
docker-compose down
cd ..
rm -rf nginx-config
```

---

### 실습 3: 데이터베이스 초기화 스크립트

#### PostgreSQL 초기화

```bash
# 프로젝트 구조
mkdir postgres-init
cd postgres-init

# 초기화 스크립트
mkdir init-scripts

cat > init-scripts/01-create-database.sql << 'EOF'
CREATE DATABASE myapp;
CREATE DATABASE test_db;
EOF

cat > init-scripts/02-create-schema.sql << 'EOF'
\c myapp

CREATE SCHEMA app;

CREATE TABLE app.users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO app.users (username, email) VALUES
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com');
EOF

cat > init-scripts/03-create-functions.sql << 'EOF'
\c myapp

CREATE OR REPLACE FUNCTION app.get_user_count()
RETURNS INTEGER AS $$
BEGIN
    RETURN (SELECT COUNT(*) FROM app.users);
END;
$$ LANGUAGE plpgsql;
EOF

# docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  db:
    image: postgres:14-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    volumes:
      # 초기화 스크립트 (읽기 전용)
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
      
      # 데이터 (Named Volume)
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres-data:
EOF

# 시작
docker-compose up -d

# 로그 확인 (초기화 과정)
docker-compose logs -f

# 대기 (초기화 완료)
sleep 10

# 테스트
docker-compose exec db psql -U postgres -d myapp -c \
  "SELECT * FROM app.users;"
# id | username | email
# ----+----------+--------------------
#  1 | alice    | alice@example.com
#  2 | bob      | bob@example.com

docker-compose exec db psql -U postgres -d myapp -c \
  "SELECT app.get_user_count();"
# get_user_count
# ----------------
#              2

# 정리
docker-compose down
cd ..
rm -rf postgres-init
```

---

### 실습 4: 로그 수집

#### 실시간 로그 모니터링

```bash
# 프로젝트 구조
mkdir log-collection
cd log-collection

# docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # 애플리케이션
  app:
    image: nginx:alpine
    volumes:
      - ./logs/app:/var/log/nginx
    ports:
      - "8080:80"

  # 로그 분석기 (간단한 예시)
  log-analyzer:
    image: alpine
    volumes:
      - ./logs/app:/logs:ro
    command: >
      sh -c "
        while true; do
          echo '=== Access Log Summary ==='
          if [ -f /logs/access.log ]; then
            echo 'Total requests:' 
            wc -l /logs/access.log | awk '{print \$1}'
            echo 'Status codes:'
            awk '{print \$9}' /logs/access.log | sort | uniq -c | sort -rn
          else
            echo 'No logs yet'
          fi
          sleep 10
        done
      "

volumes:
  app-logs:
EOF

# 시작
mkdir -p logs/app
docker-compose up -d

# 로그 생성 (트래픽)
for i in {1..50}; do
  curl -s http://localhost:8080/ > /dev/null
done

# 로그 분석기 출력 확인
docker-compose logs -f log-analyzer

# 출력:
# === Access Log Summary ===
# Total requests: 50
# Status codes:
#      50 200

# 호스트에서도 확인 가능
cat logs/app/access.log | tail -5

# 정리
docker-compose down
cd ..
rm -rf log-collection
```

---

## 🔥 실전 적용

### 시나리오 1: 풀스택 개발 환경

**구조:**

```
project/
├── frontend/
│   ├── src/
│   ├── public/
│   └── package.json
├── backend/
│   ├── src/
│   ├── config/
│   └── package.json
└── docker-compose.yml
```

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  # Frontend (React)
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    volumes:
      # 소스 코드 (Bind Mount - Hot Reload)
      - ./frontend/src:/app/src:ro
      - ./frontend/public:/app/public:ro
      
      # node_modules (Volume - 성능)
      - frontend-node-modules:/app/node_modules
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_API_URL=http://localhost:8080
      - CHOKIDAR_USEPOLLING=true
    stdin_open: true
    tty: true
    depends_on:
      - backend

  # Backend (Node.js/Express)
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    volumes:
      # 소스 코드 (Bind Mount - Hot Reload)
      - ./backend/src:/app/src:ro
      
      # 설정 파일 (Bind Mount - 읽기 전용)
      - ./backend/config:/app/config:ro
      
      # 로그 (Bind Mount - 쓰기)
      - ./logs/backend:/app/logs
      
      # node_modules (Volume)
      - backend-node-modules:/app/node_modules
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:dev@db:5432/myapp_dev
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  # Database (PostgreSQL)
  db:
    image: postgres:14-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: myapp_dev
    volumes:
      # 초기화 스크립트 (Bind Mount - 읽기 전용)
      - ./db/init:/docker-entrypoint-initdb.d:ro
      
      # 데이터 (Named Volume)
      - db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # Redis
  redis:
    image: redis:alpine
    volumes:
      # 데이터 (Named Volume)
      - redis-data:/data
    ports:
      - "6379:6379"

  # pgAdmin
  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    volumes:
      # 설정 (Bind Mount)
      - ./pgadmin/servers.json:/pgadmin4/servers.json:ro
    ports:
      - "5050:80"
    depends_on:
      - db

volumes:
  frontend-node-modules:
  backend-node-modules:
  db-data:
  redis-data:
```

**사용:**

```bash
# 초기 설정
mkdir -p logs/backend db/init

# 개발 시작
docker-compose up -d

# 로그 확인
docker-compose logs -f backend

# 코드 수정
# - frontend/src 수정 → 브라우저 자동 새로고침
# - backend/src 수정 → nodemon 자동 재시작

# DB 마이그레이션
docker-compose exec backend npm run migrate

# 정리 (데이터 유지)
docker-compose down

# 정리 (데이터 삭제)
docker-compose down -v
```

---

### 시나리오 2: 마이크로서비스 로컬 개발

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  # API Gateway
  gateway:
    build: ./gateway
    volumes:
      - ./gateway/src:/app/src:ro
      - ./gateway/config:/app/config:ro
      - ./logs/gateway:/app/logs
      - gateway-node-modules:/app/node_modules
    ports:
      - "3000:3000"
    environment:
      - AUTH_SERVICE_URL=http://auth:8080
      - USER_SERVICE_URL=http://user:8080
      - ORDER_SERVICE_URL=http://order:8080

  # Auth Service
  auth:
    build: ./services/auth
    volumes:
      - ./services/auth/src:/app/src:ro
      - ./services/auth/config:/app/config:ro
      - ./logs/auth:/app/logs
      - auth-node-modules:/app/node_modules
    environment:
      - DATABASE_URL=postgresql://postgres:dev@db:5432/auth_db
      - JWT_SECRET=dev-secret

  # User Service
  user:
    build: ./services/user
    volumes:
      - ./services/user/src:/app/src:ro
      - ./services/user/config:/app/config:ro
      - ./logs/user:/app/logs
      - user-node-modules:/app/node_modules
    environment:
      - DATABASE_URL=postgresql://postgres:dev@db:5432/user_db

  # Order Service
  order:
    build: ./services/order
    volumes:
      - ./services/order/src:/app/src:ro
      - ./services/order/config:/app/config:ro
      - ./logs/order:/app/logs
      - order-node-modules:/app/node_modules
    environment:
      - DATABASE_URL=postgresql://postgres:dev@db:5432/order_db

  # Shared Database
  db:
    image: postgres:14-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: dev
    volumes:
      - ./db/init:/docker-entrypoint-initdb.d:ro
      - db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # Shared Redis
  redis:
    image: redis:alpine
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"

volumes:
  gateway-node-modules:
  auth-node-modules:
  user-node-modules:
  order-node-modules:
  db-data:
  redis-data:
```

---

## ⚡ Bind Mount 체크리스트

### 개발 환경

```
□ Hot Reload 설정 (nodemon, webpack-dev-server)
□ node_modules 제외 (Volume 사용)
□ 소스 코드 읽기 전용 (:ro)
□ 로그 디렉토리 쓰기 가능
□ 설정 파일 버전 관리
```

### 보안

```
□ 민감 정보 제외 (.env 파일)
□ 읽기 전용 마운트 최대화
□ 호스트 루트 마운트 금지
□ 권한 최소화 (--user)
□ SELinux/AppArmor 설정
```

### 성능

```
□ 대용량 데이터는 Volume 사용
□ node_modules는 Volume
□ 빌드 아티팩트 제외
□ 불필요한 파일 .dockerignore
□ Linux에서 개발 (Mac/Windows 느림)
```

### 호환성

```
□ 절대 경로 피하기 ($(pwd) 사용)
□ Windows 경로 주의 (C:\ → /c/)
□ 권한 문제 해결 (UID/GID)
□ 플랫폼별 테스트
□ 문서화
```

---

## 🚫 안티패턴

### 1. 프로덕션에서 Bind Mount

```yaml
# ❌ 프로덕션에서 Bind Mount
services:
  app:
    image: myapp:prod
    volumes:
      - /opt/app/src:/app/src  # 위험!
# 호스트 의존성, 이식성 없음

# ✅ 이미지에 포함
# Dockerfile
FROM node:18-alpine
COPY . /app
CMD ["npm", "start"]
```

### 2. 호스트 루트 마운트

```bash
# ❌ 매우 위험
docker run -v /:/host alpine
# 호스트 전체 파일시스템 노출!

# ✅ 특정 디렉토리만
docker run -v /var/log:/logs:ro alpine
```

### 3. 권한 무시

```bash
# ❌ Root로 실행 (권한 문제)
docker run -v $(pwd)/data:/data alpine \
  sh -c 'echo test > /data/file'
# 호스트에 root 소유 파일 생성

# ✅ 사용자 지정
docker run --user $(id -u):$(id -g) \
  -v $(pwd)/data:/data alpine \
  sh -c 'echo test > /data/file'
```

### 4. 절대 경로 하드코딩

```yaml
# ❌ 절대 경로 하드코딩
services:
  app:
    volumes:
      - /home/user/project:/app
# 다른 환경에서 동작 안 함

# ✅ 상대 경로
services:
  app:
    volumes:
      - ./src:/app/src
```

---

## 🎓 핵심 정리

### 1. Bind Mount 특징

```
정의:
- 호스트 파일시스템을 직접 마운트
- 양방향 실시간 동기화
- Docker가 관리하지 않음

장점:
+ 즉시 반영 (Hot Reload)
+ 호스트 파일 직접 접근
+ 개발 편의성

단점:
- 호스트 의존적
- 이식성 낮음
- 보안 위험
```

### 2. Volume vs Bind Mount

```
Volume:
- 프로덕션 데이터
- 데이터베이스
- 백업 필요한 데이터
- 이식성 중요

Bind Mount:
- 개발 환경
- 설정 파일
- 로그 파일
- 소스 코드
```

### 3. 개발 패턴

```
Hot Reload:
- 소스: Bind Mount (즉시 반영)
- node_modules: Volume (성능)
- 빌드: Anonymous Volume (임시)

설정:
- 읽기 전용 마운트
- 버전 관리
- 환경별 분리
```

### 4. 핵심 명령어

```bash
# Bind Mount
docker run -v /host/path:/container/path image

# 읽기 전용
docker run -v /host/path:/container/path:ro image

# --mount (권장)
docker run --mount type=bind,source=/host/path,target=/container/path image

# 사용자 지정
docker run --user $(id -u):$(id -g) -v ... image
```

---

## 📚 참고 자료

- [Docker Bind Mounts](https://docs.docker.com/storage/bind-mounts/)
- [Use Bind Mounts](https://docs.docker.com/storage/bind-mounts/)
- [Storage Best Practices](https://docs.docker.com/develop/dev-best-practices/)

---

## 🤔 생각해볼 문제

1. Mac/Windows에서 Bind Mount가 느린 이유는?
2. Bind Mount로 node_modules를 마운트하면 안 되는 이유는?
3. 컨테이너 내에서 호스트 파일을 삭제하면?

> 💡 **답변**: 1) Docker Desktop은 VM을 사용 - 파일 시스템이 VM과 호스트 간 네트워크 공유(NFS/9p)로 동작, I/O가 가상화 레이어를 거침, Linux는 네이티브로 직접 접근하므로 빠름, 해결책은 Named Volume 사용 또는 Linux 개발 환경, 2) 호스트와 컨테이너 OS 차이 - Mac/Windows의 node_modules는 Linux 컨테이너와 호환 안 됨, 네이티브 모듈 (C++ 애드온) 문제, 성능 저하 (수천 개 작은 파일), 해결책은 node_modules를 Volume으로 분리, 3) 호스트 파일도 삭제됨! - Bind Mount는 양방향 동기화, 컨테이너 내 삭제 = 호스트 파일 삭제, 매우 위험하므로 중요 파일은 읽기 전용(:ro) 마운트 필수, 백업 전략 필요

---

<div align="center">

**[⬅️ 이전: Volume Types](./01-Volume-Types.md)** | **[다음: Tmpfs Mounts ➡️](./03-Tmpfs-Mounts.md)**

</div>
