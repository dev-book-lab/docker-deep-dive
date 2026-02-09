# 04. Secrets Management - 시크릿 관리

## 🎯 이 챕터에서 배울 것

- **Docker Secrets** - Swarm 내장 시크릿 관리
- **환경 변수 vs Secrets** - 보안 비교
- **HashiCorp Vault** - 엔터프라이즈 시크릿 관리
- **Secrets 로테이션** - 주기적 갱신 전략
- **실무 패턴** - 안전한 시크릿 배포

## 📌 왜 중요한가?

**"시크릿 관리는 애플리케이션 보안의 가장 취약한 지점을 보호합니다."**

```
잘못된 시크릿 관리 vs 올바른 시크릿 관리:

잘못된 방법 (평문 노출):
┌─────────────────────────────────────┐
│ Dockerfile                          │
│ ENV DB_PASSWORD=mysecretpass123     │
└────────────┬────────────────────────┘
             │ docker build
             ↓
┌─────────────────────────────────────┐
│ Image Layer                         │
│ → 이미지에 영구 저장                    │
│ → Registry에 푸시됨                   │
│ → 누구나 접근 가능                      │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ GitHub Repository                   │
│ docker-compose.yml:                 │
│   DB_PASSWORD=mysecretpass123       │
│ → 소스 코드 히스토리에 영구 보관           │
│ → Git clone으로 유출 가능              │
└─────────────────────────────────────┘
❌ 이미지 레이어에 저장
❌ Git 히스토리에 노출
❌ 로그에 평문 기록
❌ 로테이션 불가능

올바른 방법 (Docker Secrets):
┌─────────────────────────────────────┐
│ Secrets Store (Encrypted)           │
│ ┌─────────────────────────────────┐ │
│ │ db_password: ●●●●●●●●●●●        │ │
│ │ api_key: ●●●●●●●●●●●●●●         │ │
│ │ tls_cert: ●●●●●●●●●●●●●●        │ │
│ └─────────────────────────────────┘ │
└────────────┬────────────────────────┘
             │ TLS encrypted
             ↓
┌─────────────────────────────────────┐
│ Swarm Manager (Raft Store)          │
│ - 암호화된 상태로 저장                   │
│ - 분산 합의 알고리즘                    │
│ - 자동 복제                           │
└────────────┬────────────────────────┘
             │ TLS + mTLS
             ↓
┌─────────────────────────────────────┐
│ Container (In-Memory tmpfs)         │
│ /run/secrets/db_password            │
│ - RAM에만 존재                        │
│ - 디스크에 미기록                       │
│ - 컨테이너 종료 시 자동 삭제              │
└─────────────────────────────────────┘
✅ 전송 중 암호화 (TLS)
✅ 저장 시 암호화 (AES-256)
✅ 메모리만 존재
✅ 감사 로그
✅ 접근 제어
✅ 로테이션 가능

시크릿 관리의 핵심 가치:

1. 암호화 (Encryption):
   At Rest (저장 시):
   ┌──────────────────┐
   │ Swarm Raft Store │
   │ AES-256-GCM      │ ← 암호화된 상태로 저장
   └──────────────────┘
   
   In Transit (전송 중):
   Manager ←──TLS──→ Worker
   (mTLS 인증서 자동 관리)
   
   In Use (사용 중):
   ┌──────────────────┐
   │ tmpfs (RAM)      │ ← 평문이지만 격리됨
   │ /run/secrets/    │
   └──────────────────┘

2. 최소 권한 접근:
   Without Secrets:
   ┌──────────────────┐
   │ All Containers   │
   │ ENV DB_PASSWORD  │ ← 모든 컨테이너 접근
   └──────────────────┘
   
   With Secrets:
   ┌──────────────────┐
   │ DB Container     │ ✅ db_password
   ├──────────────────┤
   │ Web Container    │ ✅ api_key
   ├──────────────────┤
   │ Cache Container  │ ❌ no secrets
   └──────────────────┘

3. 감사 및 로테이션:
   Traditional:
   Password 변경 → 모든 곳 수동 업데이트
   
   Secrets:
   docker secret update → 자동 배포
   
   Vault:
   자동 로테이션 → 동적 시크릿

4. 취약점 제거:
   Common Leaks:
   ❌ Git 커밋
   ❌ 이미지 레이어
   ❌ 환경 변수 (docker inspect)
   ❌ 로그 파일
   ❌ 코어 덤프
   
   Secrets 사용:
   ✅ 소스 코드와 분리
   ✅ 이미지와 분리
   ✅ inspect에 미노출
   ✅ 로그 필터링
   ✅ tmpfs (메모리)

실무 시나리오:

문제 상황 - 평문 노출:
┌─────────────────────────────────────┐
│ 1. Developer 실수                    │
│    git add docker-compose.yml       │
│    git commit -m "Add DB config"    │
│    git push                         │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. GitHub Public Repository         │
│    DB_PASSWORD=prod_password_2024   │
│    → 검색 엔진 인덱싱                   │
│    → 봇 크롤링                        │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 공격자 발견 (30분 내)                │
│    GitHub Search:                   │
│    "DB_PASSWORD" filename:compose   │
│    → 수천 개 결과                      │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 4. 데이터 유출                         │
│    Production DB 접근                │
│    → 고객 정보 탈취                    │
│    → 랜섬웨어 감염                     │
└─────────────────────────────────────┘

올바른 접근 - Secrets 관리:
┌─────────────────────────────────────┐
│ 1. Developer                        │
│    echo "prod_password" | \         │
│    docker secret create db_password │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. Swarm Encrypted Storage          │
│    Secret: db_password              │
│    Value: [AES-256 encrypted]       │
│    ACL: db-service only             │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. Service Deployment               │
│    docker service create \          │
│      --secret db_password \         │
│      db-service                     │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 4. Container Runtime                │
│    /run/secrets/db_password (tmpfs) │
│    - 해당 컨테이너만 접근                │
│    - 메모리에만 존재                    │
│    - 종료 시 자동 삭제                  │
└─────────────────────────────────────┘

실제 사고 사례:
┌─────────────────────────────────────┐
│ 2019 - Capital One 데이터 유출         │
│ - 1억 명 고객 정보 유출                 │
│ - AWS 자격 증명 평문 저장               │
│ - 손실: $300M                        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ 2020 - SolarWinds 공급망 공격          │
│ - 빌드 서버 자격 증명 유출                │
│ - 18,000개 기업 감염                   │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ 2021 - Codecov 공격                  │
│ - CI/CD 시크릿 탈취                    │
│ - 수백 개 기업 영향                     │
└─────────────────────────────────────┘

방어 전략:
┌─────────────────────────────────────┐
│ Shift Left Security                 │
├─────────────────────────────────────┤
│ 1. Pre-commit Hooks                 │
│    → git-secrets                    │
│    → detect-secrets                 │
│                                     │
│ 2. CI/CD Scanning                   │
│    → GitGuardian                    │
│    → TruffleHog                     │
│                                     │
│ 3. Runtime Protection               │
│    → Docker Secrets                 │
│    → Vault                          │
│                                     │
│ 4. Monitoring                       │
│    → Audit logs                     │
│    → Access tracking                │
└─────────────────────────────────────┘
```

**실무 영향:**
- 데이터 유출 방지 → 평균 $4.24M 손실 회피
- 규정 준수 → GDPR, PCI-DSS, SOC 2 요구사항
- 자동화 → 수동 관리 오류 90% 감소
- 감사 추적 → 사고 조사 시간 단축

---

## 🔧 실습 1: Docker Secrets 기본

### Step 1: Swarm 초기화

```bash
# Swarm 모드 활성화
docker swarm init

# Swarm 상태 확인
docker info | grep Swarm
# Swarm: active

# 노드 확인
docker node ls
```

### Step 2: Secret 생성

```bash
# 방법 1: stdin으로 생성
echo "my-database-password" | docker secret create db_password -

# 방법 2: 파일에서 생성
echo "my-api-key-12345" > /tmp/api_key.txt
docker secret create api_key /tmp/api_key.txt
rm /tmp/api_key.txt  # 즉시 삭제

# 방법 3: 여러 줄 시크릿
cat > /tmp/tls_cert.pem <<EOF
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAKL0UG+mRmPhMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV
...
-----END CERTIFICATE-----
EOF
docker secret create tls_cert /tmp/tls_cert.pem
rm /tmp/tls_cert.pem

# Secret 목록 확인
docker secret ls

# 출력:
# ID              NAME          DRIVER   CREATED          UPDATED
# abc123...       db_password            30 seconds ago   30 seconds ago
# def456...       api_key                20 seconds ago   20 seconds ago
# ghi789...       tls_cert               10 seconds ago   10 seconds ago
```

### Step 3: Secret 사용

```bash
# Secret을 사용하는 서비스 생성
docker service create \
  --name postgres \
  --secret db_password \
  --env POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  postgres:alpine

# Secret이 여러 개일 경우
docker service create \
  --name webapp \
  --secret db_password \
  --secret api_key \
  --secret source=tls_cert,target=/etc/ssl/cert.pem \
  myapp:latest

# 서비스 확인
docker service ls
docker service ps webapp

# 컨테이너 내부에서 Secret 확인
# 실행 중인 컨테이너 찾기
CONTAINER_ID=$(docker ps --filter name=webapp -q | head -1)

# Secret 파일 확인
docker exec $CONTAINER_ID ls -la /run/secrets/
# total 8
# -r--r--r-- 1 root root 21 Feb 10 10:00 db_password
# -r--r--r-- 1 root root 17 Feb 10 10:00 api_key
# -r--r--r-- 1 root root 1234 Feb 10 10:00 cert.pem

# Secret 내용 읽기
docker exec $CONTAINER_ID cat /run/secrets/db_password
# my-database-password

# tmpfs 마운트 확인 (메모리에만 존재)
docker exec $CONTAINER_ID mount | grep secrets
# tmpfs on /run/secrets type tmpfs (ro,relatime)
```

### Step 4: Secret 업데이트 (로테이션)

```bash
# Secret은 불변(immutable)이므로 새로 생성 후 교체

# 1. 새 Secret 생성
echo "new-password-v2" | docker secret create db_password_v2 -

# 2. 서비스 업데이트 (롤링 업데이트)
docker service update \
  --secret-rm db_password \
  --secret-add source=db_password_v2,target=db_password \
  postgres

# 3. 이전 Secret 삭제
docker secret rm db_password

# 4. 검증
CONTAINER_ID=$(docker ps --filter name=postgres -q | head -1)
docker exec $CONTAINER_ID cat /run/secrets/db_password
# new-password-v2
```

### Step 5: Secret 삭제 및 정리

```bash
# 사용 중인 Secret은 삭제 불가
docker secret rm db_password_v2
# Error response from daemon: rpc error: code = InvalidArgument
# desc = secret 'db_password_v2' is in use by the following service: postgres

# 서비스 먼저 삭제
docker service rm postgres

# 이제 Secret 삭제 가능
docker secret rm db_password_v2

# 모든 Secret 삭제
docker secret ls -q | xargs docker secret rm
```

---

## 🔧 실습 2: 환경 변수 vs Secrets 비교

### Step 1: 환경 변수의 문제점

```bash
# 환경 변수로 패스워드 전달 (나쁜 예)
docker service create \
  --name insecure-db \
  --env POSTGRES_PASSWORD=mysecretpassword \
  postgres:alpine

# 문제 1: docker inspect로 평문 노출
docker service inspect insecure-db | grep POSTGRES_PASSWORD
# "POSTGRES_PASSWORD=mysecretpassword"

# 문제 2: 컨테이너 환경 변수 노출
CONTAINER_ID=$(docker ps --filter name=insecure-db -q | head -1)
docker exec $CONTAINER_ID env | grep PASSWORD
# POSTGRES_PASSWORD=mysecretpassword

# 문제 3: 프로세스 목록에 노출
docker exec $CONTAINER_ID ps aux | grep postgres

# 문제 4: 로그에 기록될 수 있음
docker logs $CONTAINER_ID 2>&1 | grep -i password

# 정리
docker service rm insecure-db
```

### Step 2: Secrets의 장점

```bash
# Secrets 사용 (올바른 예)
echo "mysecretpassword" | docker secret create secure_db_password -

docker service create \
  --name secure-db \
  --secret secure_db_password \
  --env POSTGRES_PASSWORD_FILE=/run/secrets/secure_db_password \
  postgres:alpine

# 장점 1: docker inspect에 미노출
docker service inspect secure-db | grep -i password
# "POSTGRES_PASSWORD_FILE=/run/secrets/secure_db_password"
# (파일 경로만 보임, 값은 없음)

# 장점 2: 환경 변수에 평문 없음
CONTAINER_ID=$(docker ps --filter name=secure-db -q | head -1)
docker exec $CONTAINER_ID env | grep PASSWORD
# POSTGRES_PASSWORD_FILE=/run/secrets/secure_db_password

# 장점 3: tmpfs (메모리)에만 존재
docker exec $CONTAINER_ID mount | grep secrets
# tmpfs on /run/secrets type tmpfs (ro,relatime)

# 장점 4: 읽기 전용
docker exec $CONTAINER_ID ls -l /run/secrets/secure_db_password
# -r--r--r-- 1 root root 16 Feb 10 10:00 secure_db_password

# 정리
docker service rm secure-db
docker secret rm secure_db_password
```

### Step 3: 비교 테이블

```bash
# 비교 스크립트 작성
cat > compare_secrets.sh <<'EOF'
#!/bin/bash

echo "=== 환경 변수 방식 ==="
docker service create --name env-test --env SECRET=my-secret-123 alpine sleep 3600 > /dev/null 2>&1
sleep 3
docker service inspect env-test --format '{{.Spec.TaskTemplate.ContainerSpec.Env}}' | grep SECRET
docker service rm env-test > /dev/null 2>&1

echo ""
echo "=== Docker Secrets 방식 ==="
echo "my-secret-123" | docker secret create test-secret - > /dev/null 2>&1
docker service create --name secret-test --secret test-secret alpine sleep 3600 > /dev/null 2>&1
sleep 3
docker service inspect secret-test --format '{{.Spec.TaskTemplate.ContainerSpec.Secrets}}'
docker service rm secret-test > /dev/null 2>&1
docker secret rm test-secret > /dev/null 2>&1
EOF

chmod +x compare_secrets.sh
./compare_secrets.sh
```

**비교 결과:**

| 항목 | 환경 변수 | Docker Secrets |
|-----|----------|----------------|
| **저장 위치** | 컨테이너 환경 | tmpfs (메모리) |
| **inspect 노출** | ✅ 평문 노출 | ❌ 미노출 |
| **프로세스 목록** | ✅ 노출 가능 | ❌ 미노출 |
| **로그 유출** | ✅ 위험 높음 | ❌ 위험 낮음 |
| **암호화 전송** | ❌ 없음 | ✅ TLS |
| **암호화 저장** | ❌ 없음 | ✅ AES-256 |
| **접근 제어** | ❌ 모든 컨테이너 | ✅ 지정된 서비스만 |
| **로테이션** | ❌ 어려움 | ✅ 쉬움 |
| **감사 로그** | ❌ 없음 | ✅ 있음 |

---

## 🔧 실습 3: HashiCorp Vault 통합

### Step 1: Vault 설치 및 초기화

```bash
# Vault 서버 실행 (Dev 모드)
docker run -d \
  --name vault \
  --cap-add=IPC_LOCK \
  -p 8200:8200 \
  -e 'VAULT_DEV_ROOT_TOKEN_ID=myroot' \
  -e 'VAULT_DEV_LISTEN_ADDRESS=0.0.0.0:8200' \
  vault:latest

# Vault CLI 환경 변수
export VAULT_ADDR='http://localhost:8200'
export VAULT_TOKEN='myroot'

# Vault 상태 확인
docker exec vault vault status

# 출력:
# Key             Value
# ---             -----
# Seal Type       shamir
# Initialized     true
# Sealed          false
# ...
```

### Step 2: KV Secrets Engine 활성화

```bash
# KV v2 엔진 활성화
docker exec vault vault secrets enable -path=secret kv-v2

# 시크릿 저장
docker exec vault vault kv put secret/database \
  username=dbuser \
  password=super-secret-password

docker exec vault vault kv put secret/api \
  key=api-key-12345 \
  token=bearer-token-67890

# 시크릿 조회
docker exec vault vault kv get secret/database

# 출력:
# ====== Data ======
# Key         Value
# ---         -----
# password    super-secret-password
# username    dbuser

# JSON 형식으로 조회
docker exec vault vault kv get -format=json secret/database | jq -r '.data.data.password'
# super-secret-password
```

### Step 3: 정책 생성

```bash
# 읽기 전용 정책
docker exec vault sh -c 'cat > /tmp/db-read-policy.hcl <<EOF
path "secret/data/database" {
  capabilities = ["read"]
}
EOF'

docker exec vault vault policy write db-read /tmp/db-read-policy.hcl

# 정책 확인
docker exec vault vault policy list
docker exec vault vault policy read db-read
```

### Step 4: AppRole 인증

```bash
# AppRole 인증 활성화
docker exec vault vault auth enable approle

# AppRole 생성
docker exec vault vault write auth/approle/role/my-app \
  token_policies="db-read" \
  token_ttl=1h \
  token_max_ttl=4h

# Role ID 획득
ROLE_ID=$(docker exec vault vault read -field=role_id auth/approle/role/my-app/role-id)
echo "Role ID: $ROLE_ID"

# Secret ID 생성
SECRET_ID=$(docker exec vault vault write -field=secret_id -f auth/approle/role/my-app/secret-id)
echo "Secret ID: $SECRET_ID"

# 로그인 (토큰 획득)
CLIENT_TOKEN=$(docker exec vault vault write -field=token auth/approle/login \
  role_id=$ROLE_ID \
  secret_id=$SECRET_ID)
echo "Client Token: $CLIENT_TOKEN"

# 토큰으로 시크릿 읽기
docker exec -e VAULT_TOKEN=$CLIENT_TOKEN vault \
  vault kv get secret/database
```

### Step 5: 컨테이너에서 Vault 사용

```bash
# Vault Agent를 사이드카로 사용하는 예시
cat > app-with-vault.yml <<'EOF'
version: '3.8'

services:
  vault-agent:
    image: vault:latest
    cap_add:
      - IPC_LOCK
    volumes:
      - vault-agent-config:/vault/config
      - shared-data:/vault/secrets
    environment:
      - VAULT_ADDR=http://vault:8200
    command: agent -config=/vault/config/agent.hcl
    networks:
      - app-network

  app:
    image: alpine:latest
    volumes:
      - shared-data:/secrets:ro
    command: sh -c "while true; do cat /secrets/database; sleep 30; done"
    depends_on:
      - vault-agent
    networks:
      - app-network

volumes:
  vault-agent-config:
  shared-data:

networks:
  app-network:
EOF

# Vault Agent 설정
cat > agent.hcl <<'EOF'
pid_file = "/tmp/pidfile"

vault {
  address = "http://vault:8200"
}

auto_auth {
  method "approle" {
    config = {
      role_id_file_path = "/vault/config/role-id"
      secret_id_file_path = "/vault/config/secret-id"
      remove_secret_id_file_after_reading = false
    }
  }

  sink "file" {
    config = {
      path = "/vault/secrets/.vault-token"
    }
  }
}

template {
  source      = "/vault/config/database.tpl"
  destination = "/vault/secrets/database"
}
EOF

# 템플릿 파일
cat > database.tpl <<'EOF'
{{ with secret "secret/database" }}
export DB_USER="{{ .Data.data.username }}"
export DB_PASS="{{ .Data.data.password }}"
{{ end }}
EOF
```

---

## 🔧 실습 4: Secrets 로테이션 전략

### Step 1: 수동 로테이션

```bash
# 초기 Secret 생성
echo "password-v1" | docker secret create db_pass_v1 -

# 서비스 생성
docker service create \
  --name db \
  --secret source=db_pass_v1,target=db_password \
  --env POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  postgres:alpine

# 로테이션 계획
cat > rotate_secret.sh <<'EOF'
#!/bin/bash

SECRET_NAME="db_pass"
CURRENT_VERSION=$(docker secret ls --filter name=$SECRET_NAME | tail -1 | awk '{print $2}' | grep -oP 'v\K[0-9]+')
NEW_VERSION=$((CURRENT_VERSION + 1))

echo "Current version: v$CURRENT_VERSION"
echo "New version: v$NEW_VERSION"

# 1. 새 Secret 생성
echo "Creating new secret..."
read -sp "Enter new password: " NEW_PASSWORD
echo
echo "$NEW_PASSWORD" | docker secret create ${SECRET_NAME}_v${NEW_VERSION} -

# 2. 서비스 업데이트
echo "Updating service..."
docker service update \
  --secret-rm ${SECRET_NAME}_v${CURRENT_VERSION} \
  --secret-add source=${SECRET_NAME}_v${NEW_VERSION},target=db_password \
  db

# 3. 검증
echo "Waiting for service to update..."
sleep 10

# 4. 이전 Secret 삭제
echo "Removing old secret..."
docker secret rm ${SECRET_NAME}_v${CURRENT_VERSION}

echo "Rotation complete!"
EOF

chmod +x rotate_secret.sh
```

### Step 2: 자동 로테이션 (Cron)

```bash
# 자동 로테이션 스크립트
cat > auto_rotate.sh <<'EOF'
#!/bin/bash

LOG_FILE="/var/log/secret_rotation.log"
SECRET_NAME="db_pass"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

rotate_secret() {
    CURRENT_VERSION=$(docker secret ls --filter name=$SECRET_NAME | tail -1 | awk '{print $2}' | grep -oP 'v\K[0-9]+')
    NEW_VERSION=$((CURRENT_VERSION + 1))
    
    # 새 패스워드 생성 (32자 랜덤)
    NEW_PASSWORD=$(openssl rand -base64 32)
    
    log "Starting rotation from v$CURRENT_VERSION to v$NEW_VERSION"
    
    # 새 Secret 생성
    echo "$NEW_PASSWORD" | docker secret create ${SECRET_NAME}_v${NEW_VERSION} - || {
        log "ERROR: Failed to create new secret"
        return 1
    }
    
    # 서비스 업데이트
    docker service update \
      --secret-rm ${SECRET_NAME}_v${CURRENT_VERSION} \
      --secret-add source=${SECRET_NAME}_v${NEW_VERSION},target=db_password \
      db || {
        log "ERROR: Failed to update service"
        docker secret rm ${SECRET_NAME}_v${NEW_VERSION}
        return 1
    }
    
    # 대기
    sleep 30
    
    # 이전 Secret 삭제
    docker secret rm ${SECRET_NAME}_v${CURRENT_VERSION} || {
        log "WARNING: Failed to remove old secret"
    }
    
    log "Rotation completed successfully"
}

rotate_secret
EOF

chmod +x auto_rotate.sh

# Cron 설정 (매달 1일 새벽 2시)
(crontab -l 2>/dev/null; echo "0 2 1 * * /path/to/auto_rotate.sh") | crontab -
```

### Step 3: Vault 동적 시크릿

```bash
# PostgreSQL Secrets Engine 활성화
docker exec vault vault secrets enable database

# PostgreSQL 연결 설정
docker exec vault vault write database/config/postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="readonly" \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mydb?sslmode=disable" \
  username="vaultadmin" \
  password="vaultpass"

# Role 생성 (동적 자격 증명)
docker exec vault vault write database/roles/readonly \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# 동적 자격 증명 생성
docker exec vault vault read database/creds/readonly

# 출력:
# Key                Value
# ---                -----
# lease_id          database/creds/readonly/abc123
# lease_duration    1h
# lease_renewable   true
# password          A1a-aBcDeFgHiJk
# username          v-root-readonly-xyz789

# 1시간 후 자동 만료됨!
```

---

## 🔧 실습 5: 실무 패턴

### Step 1: 다중 환경 Secrets 관리

```bash
# 환경별 Secret 네이밍
# dev: {name}_dev
# staging: {name}_staging
# prod: {name}_prod

# Development
echo "dev-password-123" | docker secret create db_password_dev -
echo "dev-api-key" | docker secret create api_key_dev -

# Staging
echo "staging-password-456" | docker secret create db_password_staging -
echo "staging-api-key" | docker secret create api_key_staging -

# Production
echo "prod-password-789" | docker secret create db_password_prod -
echo "prod-api-key" | docker secret create api_key_prod -

# Compose 파일에서 환경별 사용
cat > docker-compose.yml <<'EOF'
version: '3.8'

services:
  app:
    image: myapp:latest
    secrets:
      - source: db_password
        target: /run/secrets/db_password
      - source: api_key
        target: /run/secrets/api_key
    deploy:
      replicas: 3

secrets:
  db_password:
    external: true
    name: db_password_${ENV:-dev}
  api_key:
    external: true
    name: api_key_${ENV:-dev}
EOF

# 배포
ENV=dev docker stack deploy -c docker-compose.yml myapp-dev
ENV=staging docker stack deploy -c docker-compose.yml myapp-staging
ENV=prod docker stack deploy -c docker-compose.yml myapp-prod
```

### Step 2: Secret 변경 감지

```bash
# Secret 버전 추적
cat > track_secrets.sh <<'EOF'
#!/bin/bash

SECRETS_FILE="/var/log/secrets_versions.txt"

# 현재 Secret 목록과 생성 시간 기록
docker secret ls --format "{{.ID}}\t{{.Name}}\t{{.CreatedAt}}" > ${SECRETS_FILE}.new

# 이전 기록과 비교
if [ -f "${SECRETS_FILE}" ]; then
    diff ${SECRETS_FILE} ${SECRETS_FILE}.new > /dev/null
    if [ $? -ne 0 ]; then
        echo "[$(date)] Secrets changed!" | tee -a /var/log/secrets_audit.log
        diff ${SECRETS_FILE} ${SECRETS_FILE}.new | tee -a /var/log/secrets_audit.log
        
        # Slack 알림
        curl -X POST https://hooks.slack.com/services/YOUR/WEBHOOK/URL \
          -H 'Content-Type: application/json' \
          -d "{\"text\": \"Docker Secrets changed! Check /var/log/secrets_audit.log\"}"
    fi
fi

mv ${SECRETS_FILE}.new ${SECRETS_FILE}
EOF

chmod +x track_secrets.sh

# Cron (매시간)
(crontab -l; echo "0 * * * * /path/to/track_secrets.sh") | crontab -
```

### Step 3: Git Secrets Prevention

```bash
# git-secrets 설치
git clone https://github.com/awslabs/git-secrets
cd git-secrets
sudo make install

# 프로젝트에 git-secrets 설정
cd /path/to/your/project
git secrets --install

# 패턴 추가
git secrets --add 'password\s*=\s*["\047][^\s]+["\047]'
git secrets --add 'api[_-]?key\s*=\s*["\047][^\s]+["\047]'
git secrets --add '[A-Za-z0-9+/]{40,}'  # Base64
git secrets --add 'AKIA[0-9A-Z]{16}'     # AWS Access Key
git secrets --add '[0-9a-zA-Z/+]{32,}==' # Secret pattern

# AWS 패턴 추가
git secrets --add-provider -- git secrets --aws-provider

# 전체 저장소 스캔
git secrets --scan

# Pre-commit hook으로 자동 검사
git secrets --install -f

# 테스트
echo "password='my-secret-123'" > test.txt
git add test.txt
git commit -m "Test"
# [ERROR] Matched one or more prohibited patterns
```

### Step 4: Docker Image에서 Secrets 제거

```bash
# 나쁜 예 - 빌드 시 Secret 포함
cat > Dockerfile.bad <<'EOF'
FROM alpine
RUN echo "password=secret123" > /etc/config
COPY secret.key /app/
EOF

# 좋은 예 - Multi-stage build
cat > Dockerfile.good <<'EOF'
# Build stage
FROM alpine AS builder
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > ~/.npmrc
RUN npm install
RUN rm ~/.npmrc

# Runtime stage
FROM alpine
COPY --from=builder /app /app
# Secret은 runtime 단계로 전달되지 않음
EOF

# BuildKit secrets (권장)
cat > Dockerfile.buildkit <<'EOF'
# syntax=docker/dockerfile:1

FROM alpine
RUN --mount=type=secret,id=npmtoken \
  echo "//registry.npmjs.org/:_authToken=$(cat /run/secrets/npmtoken)" > ~/.npmrc && \
  npm install && \
  rm ~/.npmrc
EOF

# 빌드
DOCKER_BUILDKIT=1 docker build \
  --secret id=npmtoken,src=$HOME/.npmrc \
  -t myapp:latest \
  -f Dockerfile.buildkit .

# Secret은 최종 이미지에 포함되지 않음!
docker history myapp:latest
# /run/secrets에 대한 참조 없음
```

### Step 5: Kubernetes Secrets 마이그레이션

```bash
# Docker Secret을 Kubernetes Secret으로 변환
cat > migrate_to_k8s.sh <<'EOF'
#!/bin/bash

NAMESPACE="default"

# Docker Secrets 목록
for secret in $(docker secret ls --format "{{.Name}}"); do
    echo "Processing $secret..."
    
    # Secret 값 추출 (주의: 실제로는 추출 불가, 예시용)
    # 실제로는 원본 값을 별도로 보관해야 함
    
    # Kubernetes Secret 생성
    kubectl create secret generic $secret \
      --from-literal=value="$(cat /path/to/$secret)" \
      --namespace=$NAMESPACE
done
EOF

# Sealed Secrets 사용 (GitOps 친화적)
# 1. Sealed Secrets Controller 설치
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# 2. kubeseal CLI 설치
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/kubeseal-linux-amd64 -O kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# 3. Secret 봉인
echo -n "my-secret-password" | kubectl create secret generic db-password \
  --dry-run=client \
  --from-file=password=/dev/stdin \
  -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# 4. Git에 커밋 가능 (암호화됨)
cat sealed-secret.yaml

# 5. 배포
kubectl apply -f sealed-secret.yaml
# Controller가 자동으로 복호화하여 Secret 생성
```

---

## 💡 주요 명령어 정리

### Docker Secrets

```bash
# Secret 생성
echo "value" | docker secret create NAME -
docker secret create NAME /path/to/file

# Secret 목록
docker secret ls

# Secret 상세 정보
docker secret inspect NAME

# Secret 삭제
docker secret rm NAME

# 서비스에 Secret 추가
docker service create --secret NAME IMAGE
docker service create --secret source=NAME,target=/path IMAGE

# 서비스 업데이트 (로테이션)
docker service update --secret-rm OLD --secret-add NEW SERVICE

# Compose에서 사용
# docker-compose.yml:
# services:
#   app:
#     secrets:
#       - db_password
# secrets:
#   db_password:
#     external: true
```

### Vault

```bash
# Vault 시작
docker run -d --name vault -p 8200:8200 vault

# 환경 변수
export VAULT_ADDR='http://localhost:8200'
export VAULT_TOKEN='root-token'

# KV 엔진
vault kv put secret/path key=value
vault kv get secret/path
vault kv delete secret/path

# 정책
vault policy write NAME policy.hcl
vault policy list
vault policy read NAME

# 토큰
vault token create -policy=NAME
vault token lookup TOKEN

# AppRole
vault auth enable approle
vault write auth/approle/role/NAME token_policies=POLICY
vault read auth/approle/role/NAME/role-id
vault write -f auth/approle/role/NAME/secret-id
```

### Git Secrets

```bash
# 설치
git secrets --install

# 패턴 추가
git secrets --add PATTERN

# 스캔
git secrets --scan
git secrets --scan-history

# AWS 패턴
git secrets --register-aws
```

---

## 🎓 연습 문제

### 문제 1: 3-Tier 애플리케이션 Secrets

다음 구성요소를 가진 애플리케이션의 Secrets를 설정하세요:

- Frontend: API endpoint URL
- Backend: Database password, JWT secret
- Database: Root password, Replication password

<details>
<summary>정답 보기</summary>

```bash
# Secrets 생성
echo "https://api.example.com" | docker secret create api_endpoint -
echo "db-pass-$(openssl rand -hex 16)" | docker secret create db_password -
echo "jwt-$(openssl rand -hex 32)" | docker secret create jwt_secret -
echo "root-$(openssl rand -hex 16)" | docker secret create db_root_password -
echo "repl-$(openssl rand -hex 16)" | docker secret create db_repl_password -

# Compose 파일
cat > docker-compose.yml <<'EOF'
version: '3.8'

services:
  frontend:
    image: frontend:latest
    secrets:
      - api_endpoint
    deploy:
      replicas: 3

  backend:
    image: backend:latest
    secrets:
      - db_password
      - jwt_secret
    deploy:
      replicas: 2

  database:
    image: postgres:alpine
    secrets:
      - db_root_password
      - db_repl_password
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_root_password
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

secrets:
  api_endpoint:
    external: true
  db_password:
    external: true
  jwt_secret:
    external: true
  db_root_password:
    external: true
  db_repl_password:
    external: true
EOF

# 배포
docker stack deploy -c docker-compose.yml myapp
```

</details>

### 문제 2: Secret 로테이션 자동화

30일마다 자동으로 데이터베이스 패스워드를 로테이션하는 스크립트를 작성하세요.

<details>
<summary>정답 보기</summary>

```bash
cat > monthly_rotation.sh <<'EOF'
#!/bin/bash

SECRET_PREFIX="db_password"
LOG_FILE="/var/log/secret_rotation.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

# 현재 버전 확인
CURRENT=$(docker secret ls --filter name=$SECRET_PREFIX | tail -1 | awk '{print $2}')
VERSION=$(echo $CURRENT | grep -oP '\d{8}$' || echo "0")
NEW_VERSION=$(date +%Y%m%d)

if [ "$VERSION" == "$NEW_VERSION" ]; then
    log "Already rotated today, skipping"
    exit 0
fi

NEW_SECRET="${SECRET_PREFIX}_${NEW_VERSION}"
NEW_PASSWORD=$(openssl rand -base64 32)

log "Starting rotation: $CURRENT -> $NEW_SECRET"

# 1. 새 Secret 생성
echo "$NEW_PASSWORD" | docker secret create $NEW_SECRET - || {
    log "ERROR: Failed to create $NEW_SECRET"
    exit 1
}

# 2. 모든 서비스 업데이트
for service in $(docker service ls --format "{{.Name}}"); do
    # 해당 서비스가 Secret을 사용하는지 확인
    if docker service inspect $service --format '{{range .Spec.TaskTemplate.ContainerSpec.Secrets}}{{.SecretName}}{{end}}' | grep -q $CURRENT; then
        log "Updating service: $service"
        docker service update \
          --secret-rm $CURRENT \
          --secret-add source=$NEW_SECRET,target=db_password \
          $service || {
            log "ERROR: Failed to update $service"
        }
    fi
done

# 3. 대기
log "Waiting for services to stabilize..."
sleep 60

# 4. 이전 Secret 삭제
docker secret rm $CURRENT && log "Removed old secret: $CURRENT"

# 5. 데이터베이스 패스워드도 업데이트
CONTAINER=$(docker ps --filter name=database -q | head -1)
docker exec $CONTAINER psql -U postgres -c "ALTER USER postgres PASSWORD '$NEW_PASSWORD';"

log "Rotation completed successfully"
EOF

chmod +x monthly_rotation.sh

# Cron 설정 (매달 1일 새벽 3시)
(crontab -l; echo "0 3 1 * * /path/to/monthly_rotation.sh") | crontab -
```

</details>

### 문제 3: Vault 동적 DB 자격 증명

Vault를 사용하여 애플리케이션이 임시 데이터베이스 자격 증명을 얻도록 설정하세요.

<details>
<summary>정답 보기</summary>

```bash
# 1. PostgreSQL Secrets Engine 설정
docker exec vault vault secrets enable database

docker exec vault vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  allowed_roles="app-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/appdb?sslmode=disable" \
  username="vaultadmin" \
  password="vaultpass"

# 2. Role 생성
docker exec vault vault write database/roles/app-role \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# 3. 정책 생성
docker exec vault sh -c 'cat > /tmp/app-policy.hcl <<EOF
path "database/creds/app-role" {
  capabilities = ["read"]
}
EOF'

docker exec vault vault policy write app-policy /tmp/app-policy.hcl

# 4. AppRole 설정
docker exec vault vault auth enable approle

docker exec vault vault write auth/approle/role/my-app \
  token_policies="app-policy" \
  token_ttl=1h \
  token_max_ttl=4h

# 5. 애플리케이션 코드 (Python 예시)
cat > app.py <<'EOF'
import hvac
import os
import time

# Vault 클라이언트
client = hvac.Client(url='http://vault:8200')

# AppRole 로그인
role_id = os.environ['ROLE_ID']
secret_id = os.environ['SECRET_ID']

response = client.auth.approle.login(
    role_id=role_id,
    secret_id=secret_id
)

client.token = response['auth']['client_token']

# 동적 자격 증명 획득
db_creds = client.read('database/creds/app-role')
username = db_creds['data']['username']
password = db_creds['data']['password']

print(f"Username: {username}")
print(f"Password: {password}")

# 데이터베이스 연결
import psycopg2
conn = psycopg2.connect(
    host="postgres",
    database="appdb",
    user=username,
    password=password
)

# 애플리케이션 로직...

# 1시간 후 자격 증명 자동 만료!
EOF
```

</details>

---

## 📌 핵심 요약

### Secrets 관리 계층

```
┌──────────────────────────────────────┐
│ Level 3: Enterprise                  │
│ - HashiCorp Vault                    │
│ - AWS Secrets Manager                │
│ - Azure Key Vault                    │
│ - 동적 자격 증명                        │
│ - 자동 로테이션                         │
│ - 감사 로그                            │
├──────────────────────────────────────┤
│ Level 2: Orchestrator                │
│ - Docker Secrets (Swarm)             │
│ - Kubernetes Secrets                 │
│ - 암호화 전송/저장                       │
│ - 접근 제어                            │
│ - tmpfs 마운트                         │
├──────────────────────────────────────┤
│ Level 1: Basic                       │
│ - 환경 변수 (파일에서 로드)               │
│ - .env 파일 (.gitignore)              │
│ - 최소한의 보안                         │
└──────────────────────────────────────┘
```

### 보안 비교

| 방법 | 보안성 | 복잡도 | 비용 | 추천 |
|-----|--------|--------|------|------|
| **평문 ENV** | 🔴 매우 낮음 | 🟢 낮음 | 무료 | ❌ 절대 금지 |
| **.env 파일** | 🟠 낮음 | 🟢 낮음 | 무료 | 개발만 |
| **Docker Secrets** | 🟡 중간 | 🟡 중간 | 무료 | Swarm 환경 |
| **K8s Secrets** | 🟡 중간 | 🟡 중간 | 무료 | K8s 환경 |
| **Vault** | 🟢 높음 | 🔴 높음 | 유료/무료 | 엔터프라이즈 |
| **Cloud KMS** | 🟢 높음 | 🟡 중간 | 유료 | 클라우드 |

### 실무 Best Practices

**1. 절대 하지 말 것:**
```bash
# ❌ Dockerfile에 하드코딩
ENV DB_PASSWORD=secret123

# ❌ Git에 커밋
git add .env
git commit -m "Add secrets"

# ❌ 로그에 출력
echo "Password: $DB_PASSWORD"

# ❌ 이미지 레이어에 저장
RUN echo "password=secret" > /app/config
```

**2. 반드시 할 것:**
```bash
# ✅ Docker Secrets 사용
docker secret create db_password /path/to/secret

# ✅ .gitignore에 추가
echo ".env" >> .gitignore
echo "*.key" >> .gitignore

# ✅ 환경 변수는 파일 경로만
ENV DB_PASSWORD_FILE=/run/secrets/db_password

# ✅ 로테이션 주기 설정
# 30-90일마다 자동 로테이션
```

**3. 로테이션 전략:**
```
Critical Secrets (DB root, API keys):
- 로테이션 주기: 30일
- 방법: 자동화
- 알림: 완료 시 Slack

Normal Secrets (App tokens):
- 로테이션 주기: 90일
- 방법: 수동/자동
- 알림: 7일 전 리마인더

Low Priority (Dev credentials):
- 로테이션 주기: 180일
- 방법: 수동
- 알림: 없음
```

**4. 감사 체크리스트:**
- [ ] 모든 Secrets가 암호화됨
- [ ] Git 히스토리에 평문 없음
- [ ] 이미지 레이어에 Secrets 없음
- [ ] 로그에 Secrets 노출 없음
- [ ] 접근 제어 설정됨
- [ ] 로테이션 계획 수립됨
- [ ] 백업 암호화됨
- [ ] 감사 로그 활성화됨

### 마이그레이션 경로

```
현재 상태:
평문 ENV → .env 파일 → Git 저장소
           ↓
1단계 (즉시):
.env 파일 → .gitignore 추가
           ↓
2단계 (1주일):
.env 파일 → Docker Secrets/K8s Secrets
           ↓
3단계 (1개월):
Docker Secrets → Vault 통합
           ↓
4단계 (3개월):
Vault → 동적 자격 증명 + 자동 로테이션
```

---

<div align="center">

**[⬅️ 이전: Runtime Security](./03-Runtime-Security.md)** | **[다음: AppArmor & SELinux ➡️](./05-AppArmor-SELinux.md)**

</div>
