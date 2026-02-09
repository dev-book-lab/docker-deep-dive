# 01. Docker Compose - Docker Compose 기초

## 🎯 이 챕터에서 배울 것

- **Docker Compose**란 무엇인가
- **YAML 작성법**과 기본 문법
- **멀티 컨테이너 앱** 정의와 실행
- **기본 명령어**와 라이프사이클

## 📌 왜 중요한가?

**"Docker Compose는 멀티 컨테이너 애플리케이션을 정의하고 실행하는 표준 도구입니다."**

```
Docker CLI vs Docker Compose:

Docker CLI (명령형):
┌──────────────────────────────────────┐
│ $ docker network create mynet        │
│ $ docker volume create db-data       │
│ $ docker run -d --name db \          │
│     --network mynet \                │
│     -v db-data:/var/lib/postgresql \ │
│     -e POSTGRES_PASSWORD=secret \    │
│     postgres:14                      │
│ $ docker run -d --name web \         │
│     --network mynet \                │
│     -p 8080:80 \                     │
│     -e DB_HOST=db \                  │
│     nginx                            │
└──────────────────────────────────────┘
- 여러 명령어 실행
- 재현하기 어려움
- 관리 복잡
- 휴먼 에러 발생

Docker Compose (선언형):
┌──────────────────────────────────────┐
│ # docker-compose.yml                 │
│ version: '3.8'                       │
│ services:                            │
│   db:                                │
│     image: postgres:14               │
│     volumes:                         │
│       - db-data:/var/lib/postgresql  │
│     environment:                     │
│       POSTGRES_PASSWORD: secret      │
│   web:                               │
│     image: nginx                     │
│     ports:                           │
│       - "8080:80"                    │
│     environment:                     │
│       DB_HOST: db                    │
│ volumes:                             │
│   db-data:                           │
└──────────────────────────────────────┘
$ docker-compose up -d

✅ 한 명령어로 실행
✅ 재현 가능 (IaC)
✅ 버전 관리
✅ 팀 공유

Compose의 핵심 가치:

1. Infrastructure as Code:
   - YAML로 인프라 정의
   - Git으로 버전 관리
   - 코드 리뷰 가능

2. 개발-프로덕션 일관성:
   - 동일한 환경
   - 설정 공유
   - "내 컴퓨터에선 되는데" 해결

3. 빠른 반복:
   - 코드 변경 → docker-compose up
   - 즉시 재빌드/재시작
   - 빠른 피드백

4. 팀 협업:
   - 표준화된 개발 환경
   - 온보딩 간소화
   - 문서 역할
```

**실무 영향:**
- 로컬 개발 환경 표준화
- CI/CD 통합
- 테스트 환경 구성
- 소규모 프로덕션 배포

---

## 🔬 Deep Dive

### 1. Docker Compose 개념

#### 아키텍처

```
Compose 실행 흐름:

┌─────────────────────────────────────┐
│ docker-compose.yml                  │
│ (선언적 정의)                          │
└────────────┬────────────────────────┘
             │ 파싱
┌────────────▼────────────────────────┐
│ Docker Compose CLI                  │
│ - YAML 검증                          │
│ - 리소스 생성 계획                      │
│ - 의존성 순서 결정                      │
└────────────┬────────────────────────┘
             │ API 호출
┌────────────▼────────────────────────┐
│ Docker Engine                       │
│ ┌─────────────────────────────────┐ │
│ │ Containers (서비스)               │ │
│ ├─────────────────────────────────┤ │
│ │ Networks (네트워크)               │ │
│ ├─────────────────────────────────┤ │
│ │ Volumes (볼륨)                   │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘

Compose의 구성 요소:

1. Project (프로젝트):
   - docker-compose.yml이 있는 디렉토리
   - 기본 이름: 디렉토리명
   - -p 옵션으로 변경 가능

2. Service (서비스):
   - 컨테이너 정의
   - 확장 가능 (replicas)
   - 이미지 or 빌드

3. Network (네트워크):
   - 서비스 간 통신
   - 자동 DNS
   - 격리

4. Volume (볼륨):
   - 데이터 영속성
   - 서비스 간 공유
   - Named / Bind mount
```

#### 버전 히스토리

```
Compose 파일 버전:

version: '1' (deprecated)
- 2015년
- 기본 기능만
- 사용 금지

version: '2' / '2.x' (legacy)
- 2016년
- depends_on 추가
- volumes, networks 섹션
- 레거시

version: '3' / '3.x' (현재 표준)
- 2017년+
- Swarm 호환
- 주요 버전:
  - 3.0: 기본
  - 3.4: configs, secrets
  - 3.7: init, isolation
  - 3.8: 최신 (권장)

Compose Spec (차세대)
- 2020년+
- 버전 번호 없음
- Kubernetes 호환
- Docker Desktop 최신 버전

권장: version: '3.8'
```

---

### 2. YAML 기초

#### 문법 규칙

```yaml
# YAML 기본 문법

# 1. 주석
# 이것은 주석입니다
# 들여쓰기는 스페이스 2칸 (탭 아님!)

# 2. Key-Value (문자열)
name: myapp
version: 1.0.0

# 3. 숫자
port: 8080
replicas: 3

# 4. 불린
debug: true
production: false

# 5. 문자열 (따옴표 선택)
message1: Hello World
message2: "Hello World"
message3: 'Hello World'

# 6. 리스트 (배열)
ports:
  - 80
  - 443
  - 8080

# 또는 한 줄로
ports: [80, 443, 8080]

# 7. 딕셔너리 (맵)
database:
  host: localhost
  port: 5432
  name: mydb

# 8. 중첩 구조
services:
  web:
    image: nginx
    ports:
      - "80:80"
  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD: secret

# 9. 멀티라인 문자열
# | : 줄바꿈 유지
script: |
  #!/bin/bash
  echo "Line 1"
  echo "Line 2"
  echo "Line 3"

# > : 줄바꿈을 공백으로
description: >
  This is a very long
  description that spans
  multiple lines.

# 10. Anchor & Alias (재사용)
defaults: &defaults
  restart: always
  logging:
    driver: json-file
    options:
      max-size: "10m"

services:
  web:
    <<: *defaults  # defaults 내용 복사
    image: nginx

  db:
    <<: *defaults
    image: postgres

# 11. 환경 변수 참조
environment:
  DB_HOST: ${DB_HOST}
  DB_PORT: ${DB_PORT:-5432}  # 기본값 5432
```

---

### 3. Compose 파일 구조

#### 전체 구조

```yaml
# docker-compose.yml
version: '3.8'

# 서비스 정의 (필수)
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
  
  db:
    image: postgres:14
    volumes:
      - db-data:/var/lib/postgresql/data

# 네트워크 정의 (선택)
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true

# 볼륨 정의 (선택)
volumes:
  db-data:
  app-logs:

# 설정 파일 (선택, Swarm)
configs:
  my-config:
    file: ./config.txt

# 비밀 정보 (선택, Swarm)
secrets:
  db-password:
    file: ./password.txt
```

#### 최소 예제

```yaml
# 가장 간단한 Compose 파일
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
```

---

### 4. Service 정의

#### 기본 속성

```yaml
version: '3.8'

services:
  myapp:
    # --- 이미지 관련 ---
    
    # 방법 1: 이미지 사용
    image: nginx:alpine
    
    # 방법 2: Dockerfile 빌드
    # build: .
    # build:
    #   context: ./app
    #   dockerfile: Dockerfile.prod
    #   args:
    #     VERSION: "1.0"
    
    # --- 컨테이너 설정 ---
    
    # 컨테이너 이름 (고정)
    container_name: my-container
    
    # 호스트명
    hostname: myapp
    
    # --- 재시작 정책 ---
    restart: unless-stopped
    # - no: 재시작 안 함 (기본)
    # - always: 항상 재시작
    # - on-failure: 실패 시만
    # - unless-stopped: 명시적 중지 전까지
    
    # --- 네트워크 ---
    
    # 포트 매핑
    ports:
      - "8080:80"          # HOST:CONTAINER
      - "443:443"
      - "127.0.0.1:3000:3000"  # IP 지정
    
    # expose (컨테이너 간만)
    expose:
      - "3000"
    
    # 네트워크
    networks:
      - frontend
      - backend
    
    # --- 환경 변수 ---
    
    environment:
      NODE_ENV: production
      DEBUG: "false"
      API_KEY: ${API_KEY}  # 호스트 환경변수
    
    # 또는 파일로
    env_file:
      - .env
      - .env.production
    
    # --- 볼륨 ---
    
    volumes:
      # Named volume
      - app-data:/app/data
      
      # Bind mount
      - ./config:/etc/app:ro
      
      # Anonymous volume
      - /var/log
    
    # --- 의존성 ---
    
    depends_on:
      - db
      - redis
    
    # 조건부 의존성 (healthcheck)
    # depends_on:
    #   db:
    #     condition: service_healthy
    
    # --- 실행 설정 ---
    
    # 명령어 오버라이드
    command: python manage.py runserver 0.0.0.0:8000
    
    # 엔트리포인트 오버라이드
    entrypoint: /app/entrypoint.sh
    
    # 작업 디렉토리
    working_dir: /app
    
    # 사용자
    user: "1000:1000"
    
    # --- 헬스체크 ---
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    
    # --- 리소스 제한 ---
    
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    
    # --- 로깅 ---
    
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    
    # --- 기타 ---
    
    # 표준 입력 유지
    stdin_open: true
    
    # TTY 할당
    tty: true
    
    # 레이블
    labels:
      com.example.version: "1.0"
      com.example.environment: "production"

networks:
  frontend:
  backend:

volumes:
  app-data:
```

---

### 5. 기본 명령어

#### 라이프사이클

```bash
# --- 시작 ---

# 빌드 + 생성 + 시작
docker-compose up

# 백그라운드 실행
docker-compose up -d

# 특정 서비스만
docker-compose up web

# 항상 재빌드
docker-compose up --build

# 컨테이너 강제 재생성
docker-compose up --force-recreate

# 스케일 (확장)
docker-compose up -d --scale web=3

# --- 중지 ---

# 중지 (컨테이너 유지)
docker-compose stop

# 중지 (컨테이너 삭제)
docker-compose down

# 볼륨도 삭제 (주의!)
docker-compose down -v

# 이미지도 삭제
docker-compose down --rmi all

# --- 재시작 ---

# 재시작
docker-compose restart

# 특정 서비스만
docker-compose restart web

# --- 시작/중지 ---

# 시작 (이미 생성된 컨테이너)
docker-compose start

# 중지
docker-compose stop
```

#### 관리 명령어

```bash
# --- 상태 확인 ---

# 서비스 목록
docker-compose ps

# 모든 컨테이너 (중지된 것 포함)
docker-compose ps -a

# 특정 서비스
docker-compose ps web

# --- 로그 ---

# 모든 서비스 로그
docker-compose logs

# 실시간 로그 (tail -f)
docker-compose logs -f

# 특정 서비스
docker-compose logs web

# 마지막 N줄
docker-compose logs --tail=100 web

# 타임스탬프 표시
docker-compose logs -t

# --- 실행 ---

# 실행 중인 컨테이너에서 명령 실행
docker-compose exec web sh

# 새 컨테이너로 실행
docker-compose run web bash

# 실행 후 삭제
docker-compose run --rm web python manage.py migrate

# 포트 매핑 없이
docker-compose run --no-deps web bash

# --- 빌드 ---

# 모든 서비스 빌드
docker-compose build

# 캐시 없이
docker-compose build --no-cache

# 특정 서비스
docker-compose build web

# 병렬 빌드
docker-compose build --parallel

# --- 설정 ---

# YAML 검증 및 최종 설정 출력
docker-compose config

# 서비스 목록만
docker-compose config --services

# 볼륨 목록
docker-compose config --volumes

# --- 기타 ---

# 최상위 디렉토리
docker-compose top

# 이벤트 모니터링
docker-compose events

# 포트 확인
docker-compose port web 80

# 일시 정지/재개
docker-compose pause
docker-compose unpause

# 컨테이너 강제 종료
docker-compose kill
```

---

## 💻 실습

### 실습 1: Hello Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  hello:
    image: alpine
    command: echo "Hello from Docker Compose!"
```

```bash
# 실행
docker-compose up

# 출력:
# Creating network "myapp_default" with the default driver
# Creating myapp_hello_1 ... done
# Attaching to myapp_hello_1
# hello_1  | Hello from Docker Compose!
# myapp_hello_1 exited with code 0

# 정리
docker-compose down
```

---

### 실습 2: 웹 서버

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
```

```bash
# HTML 파일 생성
mkdir html
cat > html/index.html << EOF
<!DOCTYPE html>
<html>
<head><title>Docker Compose</title></head>
<body>
  <h1>Hello from Nginx!</h1>
  <p>Running with Docker Compose</p>
</body>
</html>
EOF

# 시작
docker-compose up -d

# 확인
curl http://localhost:8080
# <!DOCTYPE html>
# <html>
# ...

# 로그
docker-compose logs -f web

# 정리
docker-compose down
rm -rf html
```

---

### 실습 3: 웹 + 데이터베이스

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5

  web:
    image: nginx:alpine
    ports:
      - "80:80"
    depends_on:
      db:
        condition: service_healthy

volumes:
  db-data:
```

```bash
# 시작
docker-compose up -d

# 로그 확인
docker-compose logs -f

# DB 상태 확인
docker-compose ps
#     Name                  State           Ports
# --------------------------------------------------------
# myapp_db_1      Up (healthy)   0.0.0.0:5432->5432/tcp
# myapp_web_1     Up             0.0.0.0:80->80/tcp

# DB 접속
docker-compose exec db psql -U user -d myapp

# myapp=# \l
# myapp=# CREATE TABLE users (id SERIAL, name VARCHAR(50));
# myapp=# INSERT INTO users (name) VALUES ('Alice'), ('Bob');
# myapp=# SELECT * FROM users;
#  id | name  
# ----+-------
#   1 | Alice
#   2 | Bob
# myapp=# \q

# 재시작 (데이터 유지)
docker-compose restart

# 데이터 확인
docker-compose exec db psql -U user -d myapp -c "SELECT * FROM users;"
#  id | name  
# ----+-------
#   1 | Alice
#   2 | Bob
# ✅ 데이터 유지됨!

# 정리 (볼륨 유지)
docker-compose down

# 정리 (볼륨 삭제)
docker-compose down -v
```

---

### 실습 4: 빌드 + 실행

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
```

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

RUN npm init -y && npm install express

COPY <<EOF server.js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Express!',
    env: process.env.NODE_ENV
  });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

CMD ["node", "server.js"]
```

```bash
# 빌드 + 시작
docker-compose up -d --build

# 로그
docker-compose logs -f app

# 테스트
curl http://localhost:3000
# {"message":"Hello from Express!","env":"development"}

# 코드 수정 후 재빌드
docker-compose up -d --build

# 정리
docker-compose down
rm Dockerfile
```

---

## 🔥 실전 예제

### 예제 1: WordPress

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
    restart: always

  wordpress:
    image: wordpress:latest
    depends_on:
      - db
    ports:
      - "8080:80"
    volumes:
      - wp-data:/var/www/html
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    restart: always

volumes:
  db-data:
  wp-data:
```

```bash
# 시작
docker-compose up -d

# 로그
docker-compose logs -f

# WordPress 접속: http://localhost:8080

# 정리
docker-compose down -v
```

---

### 예제 2: MERN 스택

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongo:
    image: mongo:6
    volumes:
      - mongo-data:/data/db
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: secret

  backend:
    build: ./backend
    ports:
      - "3001:3001"
    environment:
      MONGO_URI: mongodb://admin:secret@mongo:27017/myapp?authSource=admin
    depends_on:
      - mongo

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      REACT_APP_API_URL: http://localhost:3001
    depends_on:
      - backend

volumes:
  mongo-data:
```

---

## 🚫 안티패턴

### 1. 하드코딩된 비밀번호

```yaml
# ❌ 나쁜 예
services:
  db:
    environment:
      POSTGRES_PASSWORD: my_secret_password_123
# Git에 커밋됨!

# ✅ 좋은 예
services:
  db:
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
# .env 파일 사용 (.gitignore에 추가)
```

### 2. 컨테이너 이름 충돌

```yaml
# ❌ 고정 이름
services:
  web:
    container_name: web
# 여러 프로젝트 충돌

# ✅ 자동 생성 (기본)
services:
  web:
    image: nginx
# 프로젝트명_서비스명_번호
```

### 3. 볼륨 경로 오류

```yaml
# ❌ 절대 경로
services:
  web:
    volumes:
      - /home/user/data:/data
# 이식성 없음

# ✅ 상대 경로
services:
  web:
    volumes:
      - ./data:/data
```

---

## 🎓 핵심 정리

### 1. Compose 핵심

```
- 멀티 컨테이너 정의 (YAML)
- Infrastructure as Code
- 선언적 구성
- 버전 관리
```

### 2. 파일 구조

```yaml
version: '3.8'

services:  # 필수
  # 컨테이너 정의

networks:  # 선택
  # 네트워크

volumes:  # 선택
  # 볼륨
```

### 3. 주요 명령어

```bash
# 시작
docker-compose up -d

# 중지 & 삭제
docker-compose down

# 로그
docker-compose logs -f

# 상태
docker-compose ps

# 실행
docker-compose exec <service> <cmd>
```

### 4. Best Practices

```
✅ version 3.8 사용
✅ 환경 변수 활용
✅ Named volume
✅ depends_on + healthcheck
✅ .env 파일 (.gitignore)
✅ 상대 경로
```

---

## 📚 참고 자료

- [Docker Compose Overview](https://docs.docker.com/compose/)
- [Compose file version 3 reference](https://docs.docker.com/compose/compose-file/compose-file-v3/)
- [Docker Compose CLI reference](https://docs.docker.com/compose/reference/)
- [Compose file specification](https://github.com/compose-spec/compose-spec/blob/master/spec.md)

---

## 🤔 생각해볼 문제

1. docker-compose.yml의 `version`은 왜 필요한가? 생략하면 어떻게 될까?
2. `depends_on`만으로 서비스 시작 순서를 완벽하게 제어할 수 있을까?
3. Named volume과 Bind mount, 어떤 상황에서 각각 사용해야 할까?

> 💡 **답변**: 1) `version`은 Compose 파일 스키마 버전 지정 - 생략 시 최신 Compose Spec 사용 (버전 번호 없음), 하지만 명시적 버전 지정(3.8)이 호환성 측면에서 안전, 구버전 Compose는 새 기능 사용 시 에러, 2) 불가능 - `depends_on`은 단순 시작 순서만 보장 (컨테이너 생성), 서비스 준비 여부는 보장 안 함, DB가 완전히 준비되기 전에 앱이 시작될 수 있음, 해결: `healthcheck` + `condition: service_healthy` 또는 애플리케이션에서 retry 로직, 3) Named volume: 영속성 필요(DB 데이터, 업로드 파일), Docker가 관리, 백업 용이, 성능 좋음, Bind mount: 개발 환경(hot reload), 설정 파일(읽기 전용), 호스트와 직접 공유 필요, 프로덕션에서는 Named volume 권장

---

<div align="center">

**[다음: Compose Advanced ➡️](./02-Compose-Advanced.md)**

</div>
