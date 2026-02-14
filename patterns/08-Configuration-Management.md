# 08. Configuration Management - 설정 관리 베스트 프랙티스

## 🎯 이 챕터에서 배울 것

- **설정 관리 전략**: 환경별 설정 분리
- **ConfigMap & Secret**: Kubernetes 네이티브 설정
- **동적 설정**: Hot Reload without Restart
- **외부 설정 서버**: Spring Cloud Config, Consul
- **시크릿 관리**: Vault, Sealed Secrets
- **실전 구현**: 12-Factor App 원칙

## 📌 왜 중요한가?

**"설정 관리는 코드 변경 없이 환경별 동작을 제어하는 핵심입니다."**

```
Configuration Management의 핵심:

Without Proper Config Management (하드코딩):
┌─────────────────────────────────────────────────┐
│ Application Code                                │
│                                                 │
│  # ❌ 하드코딩된 설정                               │
│  DATABASE_URL = "postgres://prod-db:5432/app"   │
│  API_KEY = "sk_prod_abc123xyz"                  │
│  DEBUG = False                                  │
│  LOG_LEVEL = "ERROR"                            │
│                                                 │
│  def connect_database():                        │
│      conn = psycopg2.connect(DATABASE_URL)      │
└─────────────────────────────────────────────────┘

문제점:
❌ 환경별 코드 수정 필요 (dev, staging, prod)
❌ 시크릿이 코드에 노출
❌ Git에 민감 정보 커밋
❌ 재배포 없이 설정 변경 불가
❌ 설정 변경 추적 어려움

With Configuration Management:
┌─────────────────────────────────────────────────┐
│ Application Code                                │
│                                                 │
│  # ✅ 환경 변수에서 읽기                            │
│  import os                                      │
│                                                 │
│  DATABASE_URL = os.getenv('DATABASE_URL')       │
│  API_KEY = os.getenv('API_KEY')                 │
│  DEBUG = os.getenv('DEBUG', 'false') == 'true'  │
│  LOG_LEVEL = os.getenv('LOG_LEVEL', 'INFO')     │
│                                                 │
│  def connect_database():                        │
│      conn = psycopg2.connect(DATABASE_URL)      │
└─────────────────────────────────────────────────┘
         ▲                                         
         │ 환경별 주입                              
         │                                         
┌────────┴─────────────────────────────────────────┐
│ Configuration Sources                            │
│                                                  │
│  Development:                                    │
│  - .env 파일                                      │
│  - docker-compose.yml                            │
│                                                  │
│  Staging:                                        │
│  - ConfigMap (일반 설정)                           │
│  - Secret (민감 정보)                              │
│                                                  │
│  Production:                                     │
│  - External Config Server (Consul, Vault)        │
│  - Encrypted Secrets                             │
│  - Hot Reload 지원                                │
└──────────────────────────────────────────────────┘

장점:
✅ 환경별 동일한 코드
✅ 시크릿 암호화
✅ 설정 중앙 관리
✅ 재배포 없이 변경
✅ 변경 이력 추적
✅ 권한 관리

12-Factor App 원칙 #3:
"Store config in the environment"

설정 레이어:
┌─────────────────────────────────────────────────┐
│ 1. Default (코드 내)                              │
│    ├─ 개발용 기본값                                │
│    └─ Fallback                                  │
│                                                 │
│ 2. Environment Variables                        │
│    ├─ OS 환경 변수                                │
│    └─ Container ENV                             │
│                                                 │
│ 3. ConfigMap (Kubernetes)                       │
│    ├─ 파일로 마운트                                │
│    └─ 환경 변수로 주입                              │
│                                                 │
│ 4. Secret (Kubernetes)                          │
│    ├─ Base64 인코딩                               │
│    └─ RBAC 보호                                  │
│                                                 │
│ 5. External Config Server                       │
│    ├─ Consul, Spring Cloud Config               │
│    ├─ 동적 갱신 (Hot Reload)                      │
│    └─ 버전 관리                                   │
│                                                 │
│ 6. Secret Management (Vault)                    │
│    ├─ 암호화 저장                                  │
│    ├─ 동적 시크릿 생성                              │
│    └─ Audit Log                                 │
└─────────────────────────────────────────────────┘

우선순위 (나중 것이 덮어씀):
Default < Env < ConfigMap < Secret < External Config

예시:
LOG_LEVEL = "INFO"  (Default)
LOG_LEVEL = "DEBUG" (ConfigMap) → DEBUG 사용
LOG_LEVEL = "ERROR" (Secret)    → ERROR 사용
```

**실무 영향:**
- **유연성**: 환경별 다른 설정 (dev, prod)
- **보안**: 시크릿 암호화 및 권한 관리
- **운영 효율**: 재배포 없이 설정 변경
- **규정 준수**: 설정 변경 감사 추적

---

## 🔬 Deep Dive

### 1. 설정 유형

#### 환경별 설정

```yaml
# Development
DATABASE_URL: postgresql://localhost:5432/dev_db
DEBUG: true
LOG_LEVEL: DEBUG
CACHE_TTL: 60

# Staging
DATABASE_URL: postgresql://staging-db:5432/app
DEBUG: true
LOG_LEVEL: INFO
CACHE_TTL: 300

# Production
DATABASE_URL: postgresql://prod-db:5432/app
DEBUG: false
LOG_LEVEL: ERROR
CACHE_TTL: 3600
```

#### 시크릿 vs 일반 설정

```yaml
# ConfigMap (일반 설정)
- Database 호스트명
- 로그 레벨
- Feature Flags
- 공개 API 엔드포인트

# Secret (민감 정보)
- Database 비밀번호
- API Keys
- TLS 인증서
- OAuth Client Secret
```

---

### 2. ConfigMap & Secret

#### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # Key-Value
  database.host: "postgres"
  database.port: "5432"
  log.level: "INFO"
  
  # 파일 전체
  app.yaml: |
    server:
      port: 8080
      timeout: 30
    features:
      feature_a: true
      feature_b: false

---
# 사용 방법 1: 환경 변수
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.host

---
# 사용 방법 2: 파일 마운트
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config
      mountPath: /etc/app
  volumes:
  - name: config
    configMap:
      name: app-config
```

#### Secret

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # Base64 인코딩
  database.password: cGFzc3dvcmQxMjM=  # "password123"
  api.key: c2tfc2VjcmV0X2tleQ==         # "sk_secret_key"

---
# 생성
kubectl create secret generic app-secret \
  --from-literal=database.password=password123 \
  --from-literal=api.key=sk_secret_key

# 또는 파일에서
kubectl create secret generic tls-cert \
  --from-file=tls.crt=./cert.crt \
  --from-file=tls.key=./cert.key

---
# 사용
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: database.password
```

---

## 🔧 실습 1: 기본 ConfigMap & Secret

### Step 1: ConfigMap 생성

```yaml
# app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  # 간단한 설정
  APP_NAME: "MyApp"
  LOG_LEVEL: "INFO"
  CACHE_ENABLED: "true"
  MAX_CONNECTIONS: "100"
  
  # YAML 파일
  config.yaml: |
    server:
      port: 8080
      host: 0.0.0.0
    
    database:
      host: postgres
      port: 5432
      name: mydb
      pool_size: 10
    
    redis:
      host: redis
      port: 6379
      db: 0
    
    features:
      feature_flags:
        new_ui: true
        beta_feature: false
  
  # 설정 파일
  app.properties: |
    server.timeout=30
    server.max_requests=1000
    logging.format=json
```

### Step 2: Secret 생성

```yaml
# app-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:  # stringData는 자동으로 Base64 인코딩
  DB_PASSWORD: "MySecurePassword123!"
  API_KEY: "sk_live_abc123xyz789"
  JWT_SECRET: "super-secret-jwt-key-do-not-share"
  REDIS_PASSWORD: "redis-pass-123"
```

### Step 3: 애플리케이션에서 사용

```yaml
# deployment-with-config.yaml
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
      containers:
      - name: app
        image: myapp:latest
        
        # 환경 변수로 주입
        env:
        # ConfigMap에서
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_NAME
        
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        
        # Secret에서
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
        
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: API_KEY
        
        # 파일로 마운트
        volumeMounts:
        - name: config-volume
          mountPath: /etc/app
          readOnly: true
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
      
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: secret-volume
        secret:
          secretName: app-secret
```

### Step 4: 애플리케이션 코드

```python
# app/config.py
import os
import yaml

class Config:
    """설정 로드"""
    
    def __init__(self):
        # 환경 변수
        self.app_name = os.getenv('APP_NAME', 'DefaultApp')
        self.log_level = os.getenv('LOG_LEVEL', 'INFO')
        self.db_password = os.getenv('DB_PASSWORD')
        self.api_key = os.getenv('API_KEY')
        
        # YAML 파일 로드
        config_file = '/etc/app/config.yaml'
        if os.path.exists(config_file):
            with open(config_file, 'r') as f:
                file_config = yaml.safe_load(f)
                self.server = file_config['server']
                self.database = file_config['database']
                self.redis = file_config['redis']
                self.features = file_config['features']
        
        # Secret 파일 로드
        secret_file = '/etc/secrets/JWT_SECRET'
        if os.path.exists(secret_file):
            with open(secret_file, 'r') as f:
                self.jwt_secret = f.read().strip()
    
    def __repr__(self):
        return f"<Config app={self.app_name} log={self.log_level}>"

config = Config()

# 사용
from flask import Flask
app = Flask(__name__)

@app.route('/config')
def show_config():
    return {
        'app_name': config.app_name,
        'log_level': config.log_level,
        'database_host': config.database['host'],
        'feature_flags': config.features['feature_flags']
    }
```

---

## 🔧 실습 2: 동적 설정 Hot Reload

### Step 1: 설정 감시 및 리로드

```python
# app/hot_reload.py
import os
import time
import yaml
import hashlib
from threading import Thread

class ConfigWatcher:
    """설정 파일 감시 및 Hot Reload"""
    
    def __init__(self, config_file, reload_callback):
        self.config_file = config_file
        self.reload_callback = reload_callback
        self.last_hash = None
        self.running = True
    
    def get_file_hash(self):
        """파일 해시 계산"""
        if not os.path.exists(self.config_file):
            return None
        
        with open(self.config_file, 'rb') as f:
            return hashlib.md5(f.read()).hexdigest()
    
    def watch(self):
        """파일 변경 감시"""
        while self.running:
            current_hash = self.get_file_hash()
            
            if current_hash and current_hash != self.last_hash:
                print(f"Config file changed, reloading...")
                self.reload_callback()
                self.last_hash = current_hash
            
            time.sleep(5)  # 5초마다 체크
    
    def start(self):
        """백그라운드에서 감시 시작"""
        thread = Thread(target=self.watch, daemon=True)
        thread.start()
    
    def stop(self):
        """감시 중지"""
        self.running = False

# 사용 예
class Application:
    def __init__(self):
        self.config = self.load_config()
        
        # 설정 감시 시작
        watcher = ConfigWatcher(
            '/etc/app/config.yaml',
            self.reload_config
        )
        watcher.start()
    
    def load_config(self):
        """설정 로드"""
        with open('/etc/app/config.yaml', 'r') as f:
            config = yaml.safe_load(f)
        print(f"Loaded config: {config}")
        return config
    
    def reload_config(self):
        """설정 리로드"""
        try:
            new_config = self.load_config()
            self.config = new_config
            print("✅ Config reloaded successfully")
        except Exception as e:
            print(f"❌ Failed to reload config: {e}")

app = Application()
```

### Step 2: ConfigMap 업데이트

```bash
# 1. ConfigMap 확인
kubectl get configmap app-config -o yaml

# 2. ConfigMap 수정
kubectl edit configmap app-config

# 또는 파일에서 업데이트
kubectl create configmap app-config \
  --from-file=config.yaml=./new-config.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

# 3. Pod에 반영 (자동, 최대 60초)
# Kubernetes가 마운트된 ConfigMap 자동 업데이트

# 4. 로그 확인
kubectl logs -f <pod-name>
# Config file changed, reloading...
# ✅ Config reloaded successfully
```

---

## 🔧 실습 3: External Config Server (Spring Cloud Config)

### Step 1: Config Server

```yaml
# config-server/application.yml
server:
  port: 8888

spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          default-label: main
          search-paths: '{application}'
```

```yaml
# Git Repository 구조
config-repo/
├── myapp/
│   ├── myapp.yml              # 기본 설정
│   ├── myapp-dev.yml          # Development
│   ├── myapp-staging.yml      # Staging
│   └── myapp-prod.yml         # Production
```

### Step 2: Client 구현

```python
# app/config_client.py
import requests
import os

class ConfigClient:
    """Config Server 클라이언트"""
    
    def __init__(self, config_server_url, app_name, profile='default'):
        self.config_server_url = config_server_url
        self.app_name = app_name
        self.profile = profile
        self.config = {}
    
    def fetch_config(self):
        """Config Server에서 설정 가져오기"""
        url = f"{self.config_server_url}/{self.app_name}/{self.profile}"
        
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            
            data = response.json()
            
            # propertySources에서 설정 추출
            for source in data.get('propertySources', []):
                self.config.update(source.get('source', {}))
            
            print(f"✅ Fetched config from server: {len(self.config)} properties")
            return self.config
        
        except Exception as e:
            print(f"❌ Failed to fetch config: {e}")
            return {}
    
    def get(self, key, default=None):
        """설정 값 가져오기"""
        return self.config.get(key, default)

# 사용
CONFIG_SERVER = os.getenv('CONFIG_SERVER_URL', 'http://config-server:8888')
APP_NAME = os.getenv('APP_NAME', 'myapp')
PROFILE = os.getenv('PROFILE', 'dev')

config_client = ConfigClient(CONFIG_SERVER, APP_NAME, PROFILE)
config_client.fetch_config()

# 값 사용
db_host = config_client.get('database.host')
log_level = config_client.get('logging.level', 'INFO')
```

---

## 🔧 실습 4: HashiCorp Vault Integration

### Step 1: Vault 설정

```bash
# Vault 서버 시작 (Dev 모드)
vault server -dev

# Environment 설정
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='root'

# Secret 저장
vault kv put secret/myapp \
  db_password="MySecurePassword123!" \
  api_key="sk_live_abc123xyz789" \
  jwt_secret="super-secret-jwt-key"

# 읽기
vault kv get secret/myapp
```

### Step 2: Vault Agent Sidecar

```yaml
# deployment-with-vault.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-database: "secret/data/myapp"
        vault.hashicorp.com/agent-inject-template-database: |
          {{- with secret "secret/data/myapp" -}}
          export DB_PASSWORD="{{ .Data.data.db_password }}"
          export API_KEY="{{ .Data.data.api_key }}"
          {{- end }}
    spec:
      containers:
      - name: app
        image: myapp:latest
        command:
        - sh
        - -c
        - |
          source /vault/secrets/database
          python app.py
```

### Step 3: Python Vault Client

```python
# app/vault_client.py
import hvac
import os

class VaultClient:
    """HashiCorp Vault 클라이언트"""
    
    def __init__(self, url, token):
        self.client = hvac.Client(url=url, token=token)
    
    def get_secret(self, path):
        """Secret 읽기"""
        try:
            response = self.client.secrets.kv.v2.read_secret_version(
                path=path
            )
            return response['data']['data']
        except Exception as e:
            print(f"Failed to read secret: {e}")
            return {}
    
    def get_database_credentials(self, role):
        """동적 데이터베이스 자격증명"""
        try:
            response = self.client.secrets.database.generate_credentials(
                name=role
            )
            return {
                'username': response['data']['username'],
                'password': response['data']['password'],
                'ttl': response['lease_duration']
            }
        except Exception as e:
            print(f"Failed to generate credentials: {e}")
            return {}

# 사용
VAULT_ADDR = os.getenv('VAULT_ADDR', 'http://vault:8200')
VAULT_TOKEN = os.getenv('VAULT_TOKEN')

vault = VaultClient(VAULT_ADDR, VAULT_TOKEN)

# Static Secret
secrets = vault.get_secret('myapp')
db_password = secrets.get('db_password')

# Dynamic Credentials
db_creds = vault.get_database_credentials('myapp-role')
db_username = db_creds['username']
db_password = db_creds['password']
```

---

## 🔧 실습 5: Environment-Specific Configuration

### Step 1: 환경별 ConfigMap

```yaml
# base/configmap.yaml (공통 설정)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: "MyApp"
  CACHE_ENABLED: "true"

---
# overlays/dev/configmap.yaml (개발 환경)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: "MyApp-Dev"
  LOG_LEVEL: "DEBUG"
  DATABASE_URL: "postgresql://localhost:5432/dev_db"

---
# overlays/prod/configmap.yaml (운영 환경)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: "MyApp-Prod"
  LOG_LEVEL: "ERROR"
  DATABASE_URL: "postgresql://prod-db:5432/app"
```

### Step 2: Kustomize로 관리

```yaml
# base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml

configMapGenerator:
- name: app-config
  literals:
  - APP_NAME=MyApp
  - CACHE_ENABLED=true

---
# overlays/dev/kustomization.yaml
bases:
- ../../base

configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - LOG_LEVEL=DEBUG
  - DATABASE_URL=postgresql://localhost:5432/dev_db

---
# overlays/prod/kustomization.yaml
bases:
- ../../base

configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - LOG_LEVEL=ERROR
  - DATABASE_URL=postgresql://prod-db:5432/app
```

```bash
# 배포
kubectl apply -k overlays/dev
kubectl apply -k overlays/prod
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 설정 타입              │ 도구                        │
├──────────────────────┼────────────────────────────┤
│ 일반 설정              │ ConfigMap                  │
├──────────────────────┼────────────────────────────┤
│ 민감 정보              │ Secret, Vault              │
├──────────────────────┼────────────────────────────┤
│ 환경별 설정             │ Kustomize, Helm            │
├──────────────────────┼────────────────────────────┤
│ 동적 설정              │ Config Server, Consul      │
├──────────────────────┼────────────────────────────┤
│ Feature Flags        │ LaunchDarkly, Unleash      │
└──────────────────────┴────────────────────────────┘
```

---

## 🎓 연습 문제

### 문제 1: ConfigMap 변경이 Pod에 즉시 반영되지 않는 이유는?

<details>
<summary>정답 보기</summary>

**캐싱 메커니즘:**
```
Kubernetes는 ConfigMap을 캐시함
- 업데이트 전파: 최대 60초
- kubelet sync period 기본값

즉시 반영하려면:
1. Pod 재시작
   kubectl rollout restart deployment/myapp

2. Annotation 변경 (자동 재시작)
   kubectl patch deployment myapp \
     -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"$(date)\"}}}}}"

3. Hot Reload 구현 (앱에서)
   - 파일 변경 감지
   - 자동 리로드
```

**Best Practice:**
```python
# ConfigMap을 환경 변수로 사용 시
# → Pod 재시작 필요

# ConfigMap을 파일로 마운트 시
# → Hot Reload 가능 (앱에서 구현)
```

</details>

### 문제 2: Secret을 안전하게 관리하는 방법은?

<details>
<summary>정답 보기</summary>

**1. Kubernetes Secret (기본):**
```yaml
# Base64 인코딩 (암호화 아님!)
data:
  password: cGFzc3dvcmQ=  # "password"

# etcd 암호화 설정
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-key>
```

**2. Sealed Secrets:**
```bash
# 암호화
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# Git에 커밋 가능 (암호화됨)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
```

**3. Vault:**
```
- 중앙화된 시크릿 관리
- 동적 시크릿 생성
- Audit Log
- TTL 자동 만료
```

**4. External Secrets Operator:**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-secret
spec:
  secretStoreRef:
    name: vault-backend
  target:
    name: app-secret
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/myapp
      property: db_password
```

</details>

### 문제 3: 설정 변경을 어떻게 추적하는가?

<details>
<summary>정답 보기</summary>

**1. Git으로 관리:**
```bash
config-repo/
├── myapp-dev.yaml
├── myapp-prod.yaml
└── .git/

# 변경 이력
git log myapp-prod.yaml

# 누가, 언제, 무엇을
commit abc123
Author: John Doe
Date: 2024-01-15
  Changed DB connection pool from 10 to 20
```

**2. ConfigMap Annotations:**
```yaml
metadata:
  annotations:
    last-updated: "2024-01-15T10:30:00Z"
    updated-by: "john@example.com"
    change-reason: "Increase DB pool size"
```

**3. Audit Log:**
```bash
# Kubernetes Audit
kubectl get events --field-selector involvedObject.name=app-config

# Vault Audit
vault audit enable file file_path=/vault/logs/audit.log
```

**4. Config Server:**
```
Spring Cloud Config:
- Git 기반 → 자동 버전 관리
- /actuator/env → 현재 설정 조회
- Refresh Endpoint → 설정 리로드
```

</details>

---

## 📌 핵심 요약

```
Configuration Management 핵심:
1. 코드와 설정 분리
2. 환경별 다른 설정
3. 시크릿 암호화
4. 중앙 관리
5. 변경 추적

Best Practices:
✅ 12-Factor App 원칙
✅ ConfigMap (일반) / Secret (민감)
✅ 환경 변수로 주입
✅ Hot Reload 구현
✅ Vault로 시크릿 관리
```

---

## 📚 참고 자료

- [12-Factor App: Config](https://12factor.net/config)
- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [HashiCorp Vault](https://www.vaultproject.io/)

---

## 🤔 생각해볼 문제

1. 모든 설정을 환경 변수로 관리하는 것이 최선인가?
2. ConfigMap vs Secret - 어디까지 Secret으로 관리해야 하는가?
3. 동적 설정 변경이 항상 좋은가?

> 💡 **답변**:
> 
> **1) 환경 변수 vs 파일:**
> 
> **환경 변수 장점:**
> - 간단함
> - 12-Factor App 원칙
> - 플랫폼 독립적
> 
> **환경 변수 단점:**
> - 복잡한 설정 어려움 (JSON, YAML)
> - 개수 제한
> - 로깅 시 노출 위험
> 
> **파일이 나은 경우:**
> - 복잡한 구조 (YAML, JSON)
> - 대량의 설정
> - 인증서, 키 파일
> 
> **2) Secret 범위:**
> ```
> Secret으로:
> - 비밀번호
> - API 키
> - 인증서
> - OAuth 토큰
> 
> ConfigMap으로:
> - 호스트명
> - 포트 번호
> - 로그 레벨
> - Feature Flags
> 
> 기준: "Git에 커밋해도 되는가?"
> ```
> 
> **3) 동적 설정 주의사항:**
> ```
> 장점:
> - 재배포 불필요
> - 빠른 변경
> 
> 단점:
> - 버그 추적 어려움
> - 일관성 보장 어려움
> - 캐시 불일치
> 
> 권장:
> - Feature Flags: 동적
> - 인프라 설정: 정적 (재배포)
> ```

---

<div align="center">

**[⬅️ 이전: Graceful Shutdown](./07-Graceful-Shutdown.md)** | **[🏠 Patterns 섹션 완료!](./README.md)**

</div>
