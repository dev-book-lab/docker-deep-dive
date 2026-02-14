# 05. Init Containers - 초기화 컨테이너 패턴

## 🎯 이 챕터에서 배울 것

- **Init Container 개념**: 메인 컨테이너 시작 전 준비 작업
- **순차 실행**: 여러 Init Container의 실행 순서
- **사전 조건 검증**: 서비스 의존성 체크
- **설정 준비**: Config 다운로드 및 변환
- **권한 설정**: 파일 시스템 초기화
- **실전 구현**: 데이터베이스 마이그레이션, Git Clone 등

## 📌 왜 중요한가?

**"Init Container는 메인 애플리케이션이 시작되기 전에 필요한 모든 준비 작업을 수행합니다."**

```
Init Container의 핵심:

Without Init Container (문제):
┌─────────────────────────────────────────────────┐
│ Main Application Container                      │
│                                                 │
│  def main():                                    │
│      # 1. DB 연결 대기 (앱 코드에 포함)              │
│      while not db.is_ready():                   │
│          time.sleep(1)  # ← 앱 책임이 아님         │
│                                                 │
│      # 2. 설정 파일 다운로드 (앱 코드에 포함)           │
│      download_config()  # ← 앱 책임이 아님         │
│                                                 │
│      # 3. DB 마이그레이션 (앱 코드에 포함)            │
│      run_migrations()   # ← 앱 책임이 아님         │
│                                                 │
│      # 4. 실제 비즈니스 로직                        │
│      start_app()        # ← 앱의 실제 책임         │
└─────────────────────────────────────────────────┘

문제점:
❌ 앱 코드가 복잡해짐 (비즈니스 로직 + 인프라)
❌ 실패 시 전체 재시작 필요
❌ 권한 문제 (앱이 root 권한 필요)
❌ 재사용 불가 (모든 앱에 중복)

With Init Container:
┌─────────────────────────────────────────────────┐
│ Pod                                             │
│                                                 │
│  ┌────────────────────────────────────────┐     │
│  │ Init Container 1: wait-for-db          │     │
│  │                                        │     │
│  │ while not db.ping():                   │     │
│  │     sleep(1)                           │     │
│  │ ✅ DB Ready                            │     │
│  └────────────────────────────────────────┘     │
│         │                                       │
│         │ 완료 후 종료                             │
│         ▼                                       │
│  ┌────────────────────────────────────────┐     │
│  │ Init Container 2: download-config      │     │
│  │                                        │     │
│  │ curl config-server > /config/app.yaml  │     │
│  │ ✅ Config Downloaded                   │     │
│  └────────────────────────────────────────┘     │
│         │                                       │
│         │ 완료 후 종료                             │
│         ▼                                       │
│  ┌────────────────────────────────────────┐     │
│  │ Init Container 3: db-migration         │     │
│  │                                        │     │
│  │ alembic upgrade head                   │     │
│  │ ✅ Migrations Applied                  │     │
│  └────────────────────────────────────────┘     │
│         │                                       │
│         │ 모든 Init Container 완료                │
│         ▼                                       │
│  ┌────────────────────────────────────────┐     │
│  │ Main Application Container             │     │
│  │                                        │     │
│  │ def main():                            │     │
│  │     start_app()  # 바로 시작!            │     │
│  │                                        │     │
│  │ ✅ 모든 준비 완료 상태                     │     │
│  └────────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘

장점:
✅ 관심사 분리 (앱 로직 vs 초기화)
✅ 순차 실행 보장 (Init 1 → Init 2 → Init 3)
✅ 재사용 가능 (모든 Pod에서 사용)
✅ 실패 시 자동 재시도
✅ 권한 분리 (Init은 root, App은 일반 사용자)

Init Container vs Main Container:
┌──────────────────┬──────────────┬──────────────┐
│ 특성              │ Init         │ Main         │
├──────────────────┼──────────────┼──────────────┤
│ 실행 시점          │ Pod 시작 시    │ Init 완료 후   │
├──────────────────┼──────────────┼──────────────┤
│ 실행 방식          │ 순차 실행      │ 병렬 실행       │
├──────────────────┼──────────────┼──────────────┤
│ 생명주기           │ 완료 후 종료    │ 계속 실행      │
├──────────────────┼──────────────┼──────────────┤
│ 실패 시            │ Pod 재시작    │ Pod 재시작     │
├──────────────────┼──────────────┼──────────────┤
│ 용도              │ 준비 작업      │ 비즈니스 로직    │
└──────────────────┴──────────────┴──────────────┘

실행 순서:
1. Init Container 1 실행 → 완료
2. Init Container 2 실행 → 완료
3. Init Container 3 실행 → 완료
4. Main Container 시작 (병렬)
```

**실무 영향:**
- **안정적 시작**: 모든 의존성 준비 완료 후 시작
- **단순한 앱 코드**: 비즈니스 로직에만 집중
- **표준화**: 공통 초기화 로직 재사용
- **디버깅 용이**: Init 단계별 로그 확인

---

## 🔬 Deep Dive

### 1. Init Container 기본

#### 실행 순서

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  # Init Containers (순차 실행)
  initContainers:
  - name: init-1
    image: busybox
    command: ['sh', '-c', 'echo Init 1 && sleep 5']
  
  - name: init-2
    image: busybox
    command: ['sh', '-c', 'echo Init 2 && sleep 5']
  
  - name: init-3
    image: busybox
    command: ['sh', '-c', 'echo Init 3 && sleep 5']
  
  # Main Containers (병렬 실행)
  containers:
  - name: app
    image: myapp:latest

# 실행 흐름:
# 1. init-1 시작 → 완료 (5초)
# 2. init-2 시작 → 완료 (5초)
# 3. init-3 시작 → 완료 (5초)
# 4. app 시작 (총 15초 후)
```

#### 볼륨 공유

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  # 공유 볼륨
  volumes:
  - name: shared-data
    emptyDir: {}
  
  initContainers:
  # Init Container에서 파일 생성
  - name: init-data
    image: busybox
    command: ['sh', '-c', 'echo "Hello from Init" > /data/message.txt']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  
  containers:
  # Main Container에서 파일 읽기
  - name: app
    image: busybox
    command: ['sh', '-c', 'cat /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
      readOnly: true  # 읽기 전용
```

---

### 2. 주요 사용 사례

#### 서비스 의존성 대기

```yaml
# wait-for-services
initContainers:
- name: wait-for-db
  image: busybox
  command:
  - sh
  - -c
  - |
    until nc -z postgres-service 5432; do
      echo "Waiting for PostgreSQL..."
      sleep 2
    done
    echo "PostgreSQL is ready!"

- name: wait-for-redis
  image: busybox
  command:
  - sh
  - -c
  - |
    until nc -z redis-service 6379; do
      echo "Waiting for Redis..."
      sleep 2
    done
    echo "Redis is ready!"
```

#### 설정 파일 다운로드

```yaml
initContainers:
- name: download-config
  image: curlimages/curl
  command:
  - sh
  - -c
  - |
    curl -o /config/app.yaml \
      http://config-server/config/app.yaml
    echo "Config downloaded"
  volumeMounts:
  - name: config
    mountPath: /config
```

#### Git Repository Clone

```yaml
initContainers:
- name: git-clone
  image: alpine/git
  args:
  - clone
  - --single-branch
  - --branch=main
  - https://github.com/myorg/myrepo.git
  - /repo
  volumeMounts:
  - name: git-repo
    mountPath: /repo
```

---

## 🔧 실습 1: Database Migration Init Container

### Step 1: Migration Init Container

```yaml
# deployment-with-migration.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: migration-script
data:
  migrate.sh: |
    #!/bin/sh
    set -e
    
    echo "Waiting for database..."
    until nc -z postgres 5432; do
      sleep 1
    done
    echo "Database is ready!"
    
    echo "Running migrations..."
    python manage.py migrate
    echo "Migrations completed!"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      # Init Container: DB Migration
      initContainers:
      - name: db-migration
        image: myapp:latest
        command: ['/bin/sh', '/scripts/migrate.sh']
        env:
        - name: DATABASE_URL
          value: "postgresql://user:pass@postgres:5432/mydb"
        volumeMounts:
        - name: migration-script
          mountPath: /scripts
      
      # Main Container
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: DATABASE_URL
          value: "postgresql://user:pass@postgres:5432/mydb"
        ports:
        - containerPort: 8080
      
      volumes:
      - name: migration-script
        configMap:
          name: migration-script
          defaultMode: 0755
```

### Step 2: Docker Compose 버전

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

  # DB Migration (Init Container 역할)
  db-migration:
    build: .
    command: python manage.py migrate
    environment:
      DATABASE_URL: postgresql://user:pass@postgres:5432/mydb
    depends_on:
      - postgres
    networks:
      - app-network
    restart: "no"  # 한 번만 실행

  # Application
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgresql://user:pass@postgres:5432/mydb
    depends_on:
      db-migration:
        condition: service_completed_successfully
    networks:
      - app-network

volumes:
  postgres-data:

networks:
  app-network:
    driver: bridge
```

### Step 3: Migration 스크립트

```python
# manage.py
import os
import psycopg2
import time

DATABASE_URL = os.getenv('DATABASE_URL')

def wait_for_db():
    """데이터베이스 준비 대기"""
    print("Waiting for database...")
    max_retries = 30
    
    for i in range(max_retries):
        try:
            conn = psycopg2.connect(DATABASE_URL)
            conn.close()
            print("Database is ready!")
            return True
        except psycopg2.OperationalError:
            print(f"Attempt {i+1}/{max_retries}: Database not ready")
            time.sleep(1)
    
    raise Exception("Database not available")

def migrate():
    """데이터베이스 마이그레이션"""
    print("Running migrations...")
    
    conn = psycopg2.connect(DATABASE_URL)
    cur = conn.cursor()
    
    # Create tables
    cur.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            username VARCHAR(100) NOT NULL,
            email VARCHAR(100) NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    cur.execute("""
        CREATE TABLE IF NOT EXISTS posts (
            id SERIAL PRIMARY KEY,
            user_id INTEGER REFERENCES users(id),
            title VARCHAR(255) NOT NULL,
            content TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    conn.commit()
    cur.close()
    conn.close()
    
    print("Migrations completed!")

if __name__ == '__main__':
    wait_for_db()
    migrate()
```

---

## 🔧 실습 2: Config Download Init Container

### Step 1: Config Server

```python
# config-server/server.py
from flask import Flask, jsonify
import yaml

app = Flask(__name__)

@app.route('/config/<app_name>')
def get_config(app_name):
    """앱 설정 반환"""
    configs = {
        'myapp': {
            'database': {
                'host': 'postgres',
                'port': 5432,
                'name': 'mydb'
            },
            'redis': {
                'host': 'redis',
                'port': 6379
            },
            'features': {
                'feature_a': True,
                'feature_b': False
            }
        }
    }
    
    config = configs.get(app_name, {})
    return yaml.dump(config), 200, {'Content-Type': 'text/yaml'}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

### Step 2: Init Container with Config Download

```yaml
# deployment-with-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: download-config-script
data:
  download.sh: |
    #!/bin/sh
    set -e
    
    echo "Downloading configuration..."
    curl -o /config/app.yaml \
      http://config-server:8000/config/myapp
    
    echo "Configuration downloaded:"
    cat /config/app.yaml
    
    # Validate YAML
    python3 -c "import yaml; yaml.safe_load(open('/config/app.yaml'))"
    echo "Configuration is valid!"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      # Init Container: Download Config
      initContainers:
      - name: download-config
        image: python:3.9-alpine
        command: ['/bin/sh', '/scripts/download.sh']
        volumeMounts:
        - name: config
          mountPath: /config
        - name: download-script
          mountPath: /scripts
      
      # Main Container
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: config
          mountPath: /etc/app
          readOnly: true
        env:
        - name: CONFIG_PATH
          value: "/etc/app/app.yaml"
      
      volumes:
      - name: config
        emptyDir: {}
      - name: download-script
        configMap:
          name: download-config-script
          defaultMode: 0755
```

---

## 🔧 실습 3: Permission Setup Init Container

### Step 1: 권한 설정 Init Container

```yaml
# deployment-with-permissions.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      # Init Container: Setup Permissions
      initContainers:
      - name: setup-permissions
        image: busybox
        command:
        - sh
        - -c
        - |
          # 디렉토리 생성
          mkdir -p /data/uploads /data/cache /data/logs
          
          # 권한 설정 (앱이 쓸 수 있도록)
          chmod 777 /data/uploads
          chmod 777 /data/cache
          chmod 755 /data/logs
          
          # 파일 생성
          touch /data/logs/app.log
          chmod 666 /data/logs/app.log
          
          echo "Permissions set successfully"
          ls -la /data
        volumeMounts:
        - name: app-data
          mountPath: /data
        securityContext:
          runAsUser: 0  # root로 실행
      
      # Main Container
      containers:
      - name: app
        image: myapp:latest
        volumeMounts:
        - name: app-data
          mountPath: /app/data
        securityContext:
          runAsUser: 1000  # 일반 사용자로 실행
          runAsGroup: 1000
      
      volumes:
      - name: app-data
        persistentVolumeClaim:
          claimName: app-data-pvc
```

---

## 🔧 실습 4: Git Clone Init Container

### Step 1: Git Clone 및 빌드

```yaml
# deployment-with-git.yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
type: Opaque
stringData:
  username: myuser
  password: mytoken

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-site
spec:
  replicas: 1
  selector:
    matchLabels:
      app: static-site
  template:
    metadata:
      labels:
        app: static-site
    spec:
      # Init Container 1: Git Clone
      initContainers:
      - name: git-clone
        image: alpine/git
        command:
        - sh
        - -c
        - |
          git clone https://$GIT_USER:$GIT_PASS@github.com/myorg/mysite.git /repo
          cd /repo
          git log -1 --oneline
        env:
        - name: GIT_USER
          valueFrom:
            secretKeyRef:
              name: git-credentials
              key: username
        - name: GIT_PASS
          valueFrom:
            secretKeyRef:
              name: git-credentials
              key: password
        volumeMounts:
        - name: repo
          mountPath: /repo
      
      # Init Container 2: Build
      - name: build-site
        image: node:18-alpine
        command:
        - sh
        - -c
        - |
          cd /repo
          npm install
          npm run build
          echo "Build completed!"
          ls -la /repo/dist
        volumeMounts:
        - name: repo
          mountPath: /repo
      
      # Main Container: Nginx
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: repo
          mountPath: /usr/share/nginx/html
          subPath: dist  # dist 디렉토리만 마운트
      
      volumes:
      - name: repo
        emptyDir: {}
```

---

## 🔧 실습 5: 복합 Init Container (실전 예제)

### Step 1: 완전한 초기화 파이프라인

```yaml
# production-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: init-scripts
data:
  wait-for-services.sh: |
    #!/bin/sh
    echo "Waiting for dependencies..."
    
    # PostgreSQL
    until nc -z postgres 5432; do
      echo "  Waiting for PostgreSQL..."
      sleep 2
    done
    echo "  ✅ PostgreSQL ready"
    
    # Redis
    until nc -z redis 6379; do
      echo "  Waiting for Redis..."
      sleep 2
    done
    echo "  ✅ Redis ready"
    
    # Elasticsearch
    until curl -s http://elasticsearch:9200/_cluster/health > /dev/null; do
      echo "  Waiting for Elasticsearch..."
      sleep 2
    done
    echo "  ✅ Elasticsearch ready"
    
    echo "All services ready!"

  download-secrets.sh: |
    #!/bin/sh
    echo "Downloading secrets from Vault..."
    
    # Vault에서 시크릿 가져오기
    curl -H "X-Vault-Token: $VAULT_TOKEN" \
      http://vault:8200/v1/secret/data/myapp \
      | jq -r '.data.data' > /secrets/app-secrets.json
    
    echo "Secrets downloaded"

  run-migrations.sh: |
    #!/bin/sh
    echo "Running database migrations..."
    
    python manage.py migrate --noinput
    python manage.py collectstatic --noinput
    
    echo "Migrations completed"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      env: production
  template:
    metadata:
      labels:
        app: myapp
        env: production
    spec:
      # Init Containers (순차 실행)
      initContainers:
      # 1. 서비스 대기
      - name: wait-for-services
        image: busybox
        command: ['/bin/sh', '/scripts/wait-for-services.sh']
        volumeMounts:
        - name: init-scripts
          mountPath: /scripts
      
      # 2. 시크릿 다운로드
      - name: download-secrets
        image: curlimages/curl
        command: ['/bin/sh', '/scripts/download-secrets.sh']
        env:
        - name: VAULT_TOKEN
          valueFrom:
            secretKeyRef:
              name: vault-token
              key: token
        volumeMounts:
        - name: init-scripts
          mountPath: /scripts
        - name: secrets
          mountPath: /secrets
      
      # 3. DB 마이그레이션
      - name: db-migration
        image: myapp:latest
        command: ['/bin/sh', '/scripts/run-migrations.sh']
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        volumeMounts:
        - name: init-scripts
          mountPath: /scripts
      
      # 4. 설정 검증
      - name: validate-config
        image: myapp:latest
        command:
        - python
        - -c
        - |
          import json
          import sys
          
          # 시크릿 검증
          with open('/secrets/app-secrets.json') as f:
              secrets = json.load(f)
          
          required = ['api_key', 'db_password', 'redis_password']
          for key in required:
              if key not in secrets:
                  print(f"Missing required secret: {key}")
                  sys.exit(1)
          
          print("All secrets validated!")
        volumeMounts:
        - name: secrets
          mountPath: /secrets
      
      # Main Container
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        volumeMounts:
        - name: secrets
          mountPath: /app/secrets
          readOnly: true
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      
      volumes:
      - name: init-scripts
        configMap:
          name: init-scripts
          defaultMode: 0755
      - name: secrets
        emptyDir:
          medium: Memory  # tmpfs (메모리)
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ Init Container 용도   │ 예시                        │
├──────────────────────┼────────────────────────────┤
│ 서비스 대기             │ DB, Redis 준비 확인          │
├──────────────────────┼────────────────────────────┤
│ 설정 다운로드           │ Config Server에서 YAML 가져오기│
├──────────────────────┼────────────────────────────┤
│ DB 마이그레이션         │ Schema 업데이트               │
├──────────────────────┼────────────────────────────┤
│ Git Clone            │ 소스 코드 다운로드              │
├──────────────────────┼────────────────────────────┤
│ 권한 설정              │ 디렉토리 생성, chmod           │
├──────────────────────┼────────────────────────────┤
│ 데이터 초기화           │ Seed 데이터 삽입              │
└──────────────────────┴────────────────────────────┘
```

---

## 🎓 연습 문제

### 문제 1: Init Container가 실패하면 어떻게 되는가?

<details>
<summary>정답 보기</summary>

**Init Container 실패 시 동작:**

```yaml
# Init Container가 실패하면
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
  - name: init-fail
    image: busybox
    command: ['sh', '-c', 'exit 1']  # 실패!
  
  containers:
  - name: app
    image: myapp:latest

# 결과:
# 1. Init Container 재시도
# 2. BackOff 증가 (1s, 2s, 4s, 8s, ...)
# 3. Main Container는 시작 안 됨
# 4. Pod 상태: Init:CrashLoopBackOff
```

**확인:**
```bash
kubectl get pods
# NAME    READY   STATUS                  RESTARTS
# myapp   0/1     Init:CrashLoopBackOff   3

kubectl describe pod myapp
# Events:
#   Back-off restarting failed container init-fail

kubectl logs myapp -c init-fail
# (에러 로그 확인)
```

**해결:**
```yaml
# 1. Timeout 설정
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'timeout 60 sh -c "until nc -z db 5432; do sleep 1; done"']

# 2. Retry 로직
command:
- sh
- -c
- |
  for i in $(seq 1 30); do
    if nc -z db 5432; then
      exit 0
    fi
    sleep 2
  done
  exit 1
```

</details>

### 문제 2: Init Container와 Main Container 간 데이터 공유 방법은?

<details>
<summary>정답 보기</summary>

**방법 1: emptyDir (권장)**
```yaml
spec:
  volumes:
  - name: shared
    emptyDir: {}
  
  initContainers:
  - name: init
    volumeMounts:
    - name: shared
      mountPath: /init-data
    command: ['sh', '-c', 'echo "data" > /init-data/file.txt']
  
  containers:
  - name: app
    volumeMounts:
    - name: shared
      mountPath: /app-data
      readOnly: true  # 읽기 전용
```

**방법 2: ConfigMap/Secret (설정)**
```yaml
# Init에서 생성 불가, 사전 정의 필요
spec:
  initContainers:
  - name: init
    envFrom:
    - configMapRef:
        name: app-config
  
  containers:
  - name: app
    envFrom:
    - configMapRef:
        name: app-config
```

**방법 3: PersistentVolume (영구 저장)**
```yaml
spec:
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
  
  initContainers:
  - name: init
    volumeMounts:
    - name: data
      mountPath: /data
  
  containers:
  - name: app
    volumeMounts:
    - name: data
      mountPath: /data
```

</details>

### 문제 3: Init Container를 여러 개 사용할 때 주의사항은?

<details>
<summary>정답 보기</summary>

**주의사항:**

**1. 순차 실행 (직렬)**
```yaml
initContainers:
- name: init-1  # 1번 실행 (10초)
  command: ['sleep', '10']
- name: init-2  # 2번 실행 (10초)
  command: ['sleep', '10']
- name: init-3  # 3번 실행 (10초)
  command: ['sleep', '10']

# 총 시간: 30초 (병렬 아님!)
```

**2. 의존성 순서**
```yaml
# ❌ 잘못된 순서
initContainers:
- name: run-migration
  command: ['python', 'manage.py', 'migrate']
- name: wait-for-db
  command: ['wait-for', 'db:5432']

# ✅ 올바른 순서
initContainers:
- name: wait-for-db
  command: ['wait-for', 'db:5432']
- name: run-migration
  command: ['python', 'manage.py', 'migrate']
```

**3. 리소스 제한**
```yaml
initContainers:
- name: heavy-init
  resources:
    limits:
      memory: 512Mi  # Init도 리소스 제한 필요
      cpu: 500m
    requests:
      memory: 256Mi
      cpu: 100m
```

**4. 실패 전략**
```yaml
# 하나라도 실패하면 전체 실패
# 중요하지 않은 Init은 별도 처리
initContainers:
- name: critical-init
  command: ['must-succeed']

# Main에서 선택적 처리
containers:
- name: app
  command:
  - sh
  - -c
  - |
    if [ -f /data/optional.txt ]; then
      echo "Optional init completed"
    else
      echo "Optional init skipped"
    fi
    start-app
```

</details>

---

## 📌 핵심 요약

```
Init Container 핵심:
1. 메인 앱 시작 전 준비 작업
2. 순차 실행 (1 → 2 → 3)
3. 완료 후 종료
4. 실패 시 Pod 재시작
5. 볼륨 공유로 데이터 전달

Best Practices:
✅ 한 가지 작업만 수행 (단일 책임)
✅ 멱등성 (여러 번 실행해도 안전)
✅ Timeout 설정
✅ 리소스 제한
✅ 실패 로그 명확히
```

---

## 📚 참고 자료

- [Kubernetes Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Init Container Patterns](https://kubernetes.io/blog/2016/01/why-kubernetes-doesnt-use-libnetwork/)
- [Docker Compose depends_on](https://docs.docker.com/compose/compose-file/05-services/#depends_on)

---

## 🤔 생각해볼 문제

1. Init Container vs Readiness Probe - 서비스 준비를 확인하는 두 가지 방법의 차이는?
2. Init Container가 너무 많으면(10개 이상) 어떤 문제가 있는가?
3. Init Container에서 실패했을 때 Main Container로 에러를 전달하려면?

> 💡 **답변**:
> 
> **1) Init Container vs Readiness Probe:**
> 
> **Init Container (시작 전):**
> ```
> Pod 생성 → Init 1 → Init 2 → Main 시작
> 
> 용도:
> - 한 번만 실행 (준비 작업)
> - DB 마이그레이션
> - 설정 다운로드
> - 사전 조건 충족
> ```
> 
> **Readiness Probe (시작 후):**
> ```
> Main 시작 → Readiness Check (주기적)
> 
> 용도:
> - 계속 실행 (상태 확인)
> - 트래픽 수신 가능 여부
> - 일시적 장애 감지
> - Service에서 제외/포함
> ```
> 
> **함께 사용:**
> ```yaml
> initContainers:
> - name: migrate
>   # DB 준비 (한 번)
> 
> containers:
> - name: app
>   readinessProbe:
>     # 앱 준비 상태 (계속)
> ```
> 
> **2) Init Container 과다 사용 문제:**
> 
> ```
> 10개 Init Container:
> - 각 5초 → 총 50초 시작 시간
> - 복잡도 증가
> - 디버깅 어려움
> - 유지보수 부담
> 
> 해결:
> 1. 병합
>    10개 → 3개 (관련 작업 통합)
> 
> 2. Job으로 분리
>    일회성 작업 → Job
>    반복 작업 → CronJob
> 
> 3. 외부화
>    CI/CD에서 사전 처리
> ```
> 
> **3) Init에서 Main으로 에러 전달:**
> 
> ```yaml
> # 방법 1: Exit Code + 파일
> initContainers:
> - name: init
>   command:
>   - sh
>   - -c
>   - |
>     if some_check; then
>       echo "OK" > /status/result
>       exit 0
>     else
>       echo "ERROR: reason" > /status/result
>       exit 1  # Pod 재시작
>     fi
>   volumeMounts:
>   - name: status
>     mountPath: /status
> 
> containers:
> - name: app
>   command:
>   - sh
>   - -c
>   - |
>     if [ -f /status/result ]; then
>       result=$(cat /status/result)
>       echo "Init result: $result"
>     fi
>     start-app
>   volumeMounts:
>   - name: status
>     mountPath: /status
> ```

---

<div align="center">

**[⬅️ 이전: Adapter Pattern](./04-Adapter-Pattern.md)** | **[다음: Health Checks ➡️](./06-Health-Checks.md)**

</div>
