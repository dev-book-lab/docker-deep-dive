# 03. Registry Setup - Private Registry 구축

## 🎯 이 챕터에서 배울 것

- **Registry 종류**: Docker Hub, Harbor, ECR, GCR
- **Private Registry**: 자체 레지스트리 구축
- **보안 설정**: TLS, 인증, 권한 관리
- **고가용성**: Replication, Backup
- **성능 최적화**: Cache, CDN
- **실전 구현**: 프로덕션급 Registry

## 📌 왜 중요한가?

**"Private Registry는 이미지를 안전하게 관리하고 배포 속도를 향상시킵니다."**

```
Registry의 핵심:

Public Registry (Docker Hub):
┌─────────────────────────────────────────────────┐
│ Docker Hub (Public)                             │
│                                                 │
│ 장점:                                            │
│ ✅ 무료 (Public 이미지)                           │
│ ✅ 설정 불필요                                    │
│ ✅ 글로벌 CDN                                    │
│ ✅ 자동 빌드                                      │
│                                                 │
│ 단점:                                            │
│ ❌ Rate Limit (익명: 100 pulls/6시간)             │
│ ❌ Private 이미지 제한 (무료: 1개)                  │
│ ❌ 느린 속도 (해외 서버)                            │
│ ❌ 보안 우려 (Public)                             │
│ ❌ 회사 정책 위반 가능                              │
└─────────────────────────────────────────────────┘

Private Registry:
┌─────────────────────────────────────────────────┐
│ Private Registry (Self-hosted or Managed)       │
│                                                 │
│ 장점:                                            │
│ ✅ 무제한 이미지                                   │
│ ✅ Rate Limit 없음                               │
│ ✅ 빠른 속도 (로컬/VPC)                            │
│ ✅ 완전한 제어                                    │
│ ✅ 보안 (내부망)                                  │
│ ✅ 규정 준수                                      │
│                                                 │
│ 단점:                                            │
│ ❌ 구축/관리 필요                                  │
│ ❌ 스토리지 비용                                   │
│ ❌ 고가용성 구성 필요                               │
└─────────────────────────────────────────────────┘

Registry 선택:
┌──────────────┬──────────────┬──────────────┐
│ 요구사항       │ 솔루션         │ 비용          │
├──────────────┼──────────────┼──────────────┤
│ 간단한 시작     │ Docker Hub   │ 무료          │
├──────────────┼──────────────┼──────────────┤
│ 클라우드       │ ECR, GCR, ACR│ 저장량 기준     │
├──────────────┼──────────────┼──────────────┤
│ 자체 호스팅     │ Harbor       │ 인프라 비용     │
├──────────────┼──────────────┼──────────────┤
│ 간단 Self     │ Registry     │ 인프라 비용     │
└──────────────┴──────────────┴──────────────┘

Private Registry 아키텍처:
┌─────────────────────────────────────────────────┐
│ Private Registry Architecture                   │
│                                                 │
│  ┌────────────┐    ┌────────────┐               │
│  │ Developer  │    │   CI/CD    │               │
│  │   Laptop   │    │   Server   │               │
│  └─────┬──────┘    └─────┬──────┘               │
│        │ Push/Pull       │ Push                 │
│        └────────┬────────┘                      │
│                 ▼                               │
│        ┌──────────────────┐                     │
│        │ Load Balancer    │                     │
│        │ (TLS Termination)│                     │
│        └────────┬─────────┘                     │
│                 │                               │
│        ┌────────┴─────────┐                     │
│        ▼                  ▼                     │
│  ┌───────────┐      ┌───────────┐               │
│  │ Registry  │      │ Registry  │               │
│  │   Node 1  │◄────►│   Node 2  │               │
│  └─────┬─────┘      └─────┬─────┘               │
│        │                  │                     │
│        └────────┬─────────┘                     │
│                 ▼                               │
│        ┌──────────────────┐                     │
│        │ Storage Backend  │                     │
│        │ (S3, NFS, Local) │                     │
│        └──────────────────┘                     │
│                                                 │
│  ┌────────────────────────────────────────┐     │
│  │ Optional Components:                   │     │
│  │ - Notary (Image Signing)               │     │
│  │ - Clair/Trivy (Vulnerability Scan)     │     │
│  │ - LDAP/OAuth (Authentication)          │     │
│  │ - Redis (Cache)                        │     │
│  └────────────────────────────────────────┘     │
└─────────────────────────────────────────────────┘

보안 레이어:
1. Network (VPC, Firewall)
2. TLS (HTTPS)
3. Authentication (Basic, Token, OAuth)
4. Authorization (RBAC)
5. Image Signing (Notary, Cosign)
6. Vulnerability Scanning
```

**실무 영향:**
- **속도**: 로컬 네트워크로 빠른 pull/push
- **보안**: 내부망에서만 접근 가능
- **비용**: 무제한 이미지, Rate Limit 없음
- **규정**: GDPR, HIPAA 등 준수

---

## 🔬 Deep Dive

### 1. Registry 종류

#### Docker Registry (공식)

```bash
# 가장 단순한 Registry
docker run -d -p 5000:5000 --name registry registry:2

# 장점:
✅ 간단한 설정
✅ 가벼움
✅ 표준 구현

# 단점:
❌ 웹 UI 없음
❌ 사용자 관리 없음
❌ 취약점 스캔 없음
```

#### Harbor (CNCF)

```yaml
# 엔터프라이즈급 Registry
features:
  - 웹 UI
  - RBAC
  - Vulnerability Scanning (Trivy)
  - Image Signing (Notary)
  - Replication
  - Webhook
  - LDAP/OAuth

# 프로덕션 권장!
```

#### Cloud Registry

```bash
# AWS ECR
aws ecr create-repository --repository-name myapp

# Google GCR
gcloud artifacts repositories create myapp \
  --repository-format=docker

# Azure ACR
az acr create --name myregistry --sku Basic
```

---

### 2. 보안 설정

#### TLS 인증서

```bash
# Self-signed (개발용)
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout domain.key -x509 -days 365 \
  -out domain.crt

# Let's Encrypt (프로덕션)
certbot certonly --standalone -d registry.example.com
```

#### 인증

```yaml
# Basic Auth
htpasswd -Bc htpasswd user1
htpasswd -B htpasswd user2

# Token Auth (JWT)
auth:
  token:
    realm: "Registry"
    service: "Docker registry"
    issuer: "Auth Service"
```

---

## 🔧 실습 1: Docker Registry 기본 구축

### Step 1: 간단한 Registry 시작

```bash
# 1. Registry 실행
docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  registry:2

# 2. 이미지 태그 및 푸시
docker pull alpine
docker tag alpine localhost:5000/alpine
docker push localhost:5000/alpine

# 3. 확인
docker pull localhost:5000/alpine

# 4. Catalog API
curl http://localhost:5000/v2/_catalog
# {"repositories":["alpine"]}

# 5. 태그 확인
curl http://localhost:5000/v2/alpine/tags/list
# {"name":"alpine","tags":["latest"]}
```

### Step 2: Persistent Storage

```bash
# 볼륨 마운트 (데이터 보존)
docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /opt/registry:/var/lib/registry \
  registry:2

# 또는 Docker Compose
cat > docker-compose.yml <<EOF
version: '3.8'

services:
  registry:
    image: registry:2
    ports:
      - 5000:5000
    volumes:
      - ./data:/var/lib/registry
    restart: always
EOF

docker-compose up -d
```

---

## 🔧 실습 2: TLS 및 인증 설정

### Step 1: TLS 인증서 생성

```bash
# 1. 디렉토리 생성
mkdir -p certs auth

# 2. Self-signed 인증서 생성 (개발용)
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout certs/domain.key \
  -x509 -days 365 \
  -out certs/domain.crt \
  -subj "/C=US/ST=State/L=City/O=Org/CN=registry.local"

# 3. 로컬 머신에 인증서 신뢰 (macOS)
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain certs/domain.crt

# Linux
sudo cp certs/domain.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

### Step 2: Basic Auth 설정

```bash
# 1. htpasswd 설치
sudo apt-get install apache2-utils  # Ubuntu
brew install httpd                   # macOS

# 2. 사용자 생성
htpasswd -Bc auth/htpasswd admin
# Password: admin123

htpasswd -B auth/htpasswd developer
# Password: dev123

# 3. Docker Compose 설정
cat > docker-compose.yml <<EOF
version: '3.8'

services:
  registry:
    image: registry:2
    ports:
      - 5000:5000
    environment:
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
    volumes:
      - ./data:/var/lib/registry
      - ./certs:/certs
      - ./auth:/auth
    restart: always
EOF

docker-compose up -d
```

### Step 3: 인증 테스트

```bash
# 1. 로그인
docker login registry.local:5000
# Username: admin
# Password: admin123

# 2. 이미지 푸시
docker tag alpine registry.local:5000/alpine
docker push registry.local:5000/alpine

# 3. 로그아웃 후 테스트
docker logout registry.local:5000
docker pull registry.local:5000/alpine
# Error: authentication required

# 4. 다시 로그인
docker login registry.local:5000 -u developer -p dev123
docker pull registry.local:5000/alpine
# Success!
```

---

## 🔧 실습 3: Harbor 구축 (프로덕션급)

### Step 1: Harbor 설치

```bash
# 1. Harbor 다운로드
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-offline-installer-v2.10.0.tgz

tar xvf harbor-offline-installer-v2.10.0.tgz
cd harbor

# 2. 설정 파일 복사
cp harbor.yml.tmpl harbor.yml

# 3. 설정 편집
vi harbor.yml
```

```yaml
# harbor.yml
hostname: harbor.example.com

# HTTPS 설정
https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key

# 초기 관리자 비밀번호
harbor_admin_password: Harbor12345

# 데이터베이스 비밀번호
database:
  password: root123
  max_idle_conns: 100
  max_open_conns: 900

# 스토리지
data_volume: /data

# Clair (취약점 스캔)
trivy:
  ignore_unfixed: false
  skip_update: false
  insecure: false
```

### Step 2: Harbor 실행

```bash
# 1. 설치 스크립트 실행
sudo ./install.sh --with-trivy

# 2. 상태 확인
docker-compose ps

# 3. 웹 UI 접속
# https://harbor.example.com
# Username: admin
# Password: Harbor12345
```

### Step 3: 프로젝트 및 사용자 생성

```bash
# 웹 UI에서:
# 1. Projects → New Project
#    Name: myproject
#    Access Level: Private

# 2. Administration → Users → New User
#    Username: developer
#    Email: dev@example.com
#    Password: Dev123456

# 3. myproject → Members → Add
#    developer → Developer role
```

### Step 4: Harbor 사용

```bash
# 1. 로그인
docker login harbor.example.com
# Username: developer
# Password: Dev123456

# 2. 이미지 태그
docker tag myapp:v1.0.0 harbor.example.com/myproject/myapp:v1.0.0

# 3. 푸시
docker push harbor.example.com/myproject/myapp:v1.0.0

# 4. 웹 UI에서 확인
# Projects → myproject → Repositories
# - 취약점 스캔 자동 실행
# - 이미지 서명 (선택)
# - Replication (선택)
```

---

## 🔧 실습 4: Registry with S3 Backend

### Step 1: S3 스토리지 설정

```yaml
# config.yml
version: 0.1
log:
  level: info
storage:
  s3:
    accesskey: AKIAIOSFODNN7EXAMPLE
    secretkey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
    region: us-east-1
    bucket: my-docker-registry
    encrypt: true
    secure: true
  cache:
    blobdescriptor: redis
  maintenance:
    uploadpurging:
      enabled: true
      age: 168h
      interval: 24h
      dryrun: false
  delete:
    enabled: true
http:
  addr: :5000
  secret: supersecretstring
  headers:
    X-Content-Type-Options: [nosniff]
redis:
  addr: redis:6379
  db: 0
  password: ""
```

### Step 2: Docker Compose with S3

```yaml
# docker-compose.yml
version: '3.8'

services:
  registry:
    image: registry:2
    ports:
      - 5000:5000
    environment:
      REGISTRY_STORAGE: s3
      REGISTRY_STORAGE_S3_REGION: us-east-1
      REGISTRY_STORAGE_S3_BUCKET: my-docker-registry
      REGISTRY_STORAGE_S3_ACCESSKEY: ${AWS_ACCESS_KEY_ID}
      REGISTRY_STORAGE_S3_SECRETKEY: ${AWS_SECRET_ACCESS_KEY}
      REGISTRY_STORAGE_CACHE_BLOBDESCRIPTOR: redis
      REGISTRY_REDIS_ADDR: redis:6379
    depends_on:
      - redis
    restart: always

  redis:
    image: redis:alpine
    restart: always

  # 선택: UI
  registry-ui:
    image: joxit/docker-registry-ui:latest
    ports:
      - 8080:80
    environment:
      REGISTRY_URL: http://registry:5000
      DELETE_IMAGES: true
      REGISTRY_TITLE: My Docker Registry
    depends_on:
      - registry
```

---

## 🔧 실습 5: Registry Garbage Collection

### Step 1: 이미지 삭제 활성화

```yaml
# config.yml
storage:
  delete:
    enabled: true
```

### Step 2: 이미지 삭제

```bash
# 1. 이미지 매니페스트 확인
curl -v -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://localhost:5000/v2/myapp/manifests/v1.0.0

# 2. Digest 추출
# Docker-Content-Digest: sha256:abc123...

# 3. 삭제
curl -X DELETE \
  http://localhost:5000/v2/myapp/manifests/sha256:abc123...

# 4. Garbage Collection 실행
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml

# 5. 디스크 공간 확인
docker exec registry du -sh /var/lib/registry
```

### Step 3: 자동화 (Cron)

```bash
# gc.sh
#!/bin/bash
docker exec registry bin/registry garbage-collect \
  --delete-untagged \
  /etc/docker/registry/config.yml

# Crontab
# 매일 새벽 2시 실행
0 2 * * * /opt/registry/gc.sh
```

---

## 🔧 실습 6: Registry Mirroring (Cache)

### Step 1: Pull-through Cache 설정

```yaml
# config.yml
version: 0.1
proxy:
  remoteurl: https://registry-1.docker.io
  username: dockerhubuser
  password: dockerhubpassword
storage:
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
```

### Step 2: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  registry-mirror:
    image: registry:2
    ports:
      - 5001:5000
    environment:
      REGISTRY_PROXY_REMOTEURL: https://registry-1.docker.io
      REGISTRY_PROXY_USERNAME: ${DOCKERHUB_USERNAME}
      REGISTRY_PROXY_PASSWORD: ${DOCKERHUB_PASSWORD}
    volumes:
      - ./mirror-data:/var/lib/registry
    restart: always
```

### Step 3: Docker Daemon 설정

```json
// /etc/docker/daemon.json
{
  "registry-mirrors": ["http://localhost:5001"]
}
```

```bash
# Docker 재시작
sudo systemctl restart docker

# 테스트
docker pull nginx
# Cache에서 가져옴 (첫 번째는 느림, 이후는 빠름)
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ Registry 선택         │ 용도                        │
├──────────────────────┼────────────────────────────┤
│ Docker Registry      │ 간단한 Private Registry      │
├──────────────────────┼────────────────────────────┤
│ Harbor               │ 엔터프라이즈 프로덕션            │
├──────────────────────┼────────────────────────────┤
│ AWS ECR              │ AWS 환경                    │
├──────────────────────┼────────────────────────────┤
│ Google GCR           │ GCP 환경                    │
├──────────────────────┼────────────────────────────┤
│ Azure ACR            │ Azure 환경                  │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. TLS 필수
2. 인증 활성화
3. RBAC 설정
4. 정기 Garbage Collection
5. Backup
```

---

## 🎓 연습 문제

### 문제 1: Registry가 디스크 공간을 너무 많이 사용한다면?

<details>
<summary>정답 보기</summary>

**원인:**
```bash
# 삭제된 이미지도 실제론 남아있음
# Garbage Collection 필요
```

**해결:**
```bash
# 1. 삭제 활성화
storage:
  delete:
    enabled: true

# 2. GC 실행
docker exec registry bin/registry garbage-collect \
  --delete-untagged \
  /etc/docker/registry/config.yml

# 3. 정기 실행 (Cron)
0 2 * * * /opt/registry/gc.sh
```

**추가 최적화:**
```bash
# Retention Policy
# Harbor: Project → Policy
# - 최근 N개만 유지
# - N일 이상 된 태그 삭제
```

</details>

### 문제 2: Registry 고가용성을 어떻게 구성하는가?

<details>
<summary>정답 보기</summary>

**구성:**
```
┌──────────────┐
│ Load Balancer│
└──────┬───────┘
       │
   ┌───┴───┐
   ▼       ▼
┌─────┐ ┌─────┐
│Reg 1│ │Reg 2│
└──┬──┘ └──┬──┘
   │       │
   └───┬───┘
       ▼
┌──────────┐
│ S3/NFS   │
└──────────┘
```

**구현:**
```yaml
# 1. 공유 스토리지 (S3)
storage:
  s3:
    bucket: my-registry

# 2. Redis (Cache 공유)
redis:
  addr: redis-cluster:6379

# 3. 여러 Registry 인스턴스
docker-compose scale registry=3

# 4. Load Balancer
nginx:
  upstream:
    - registry1:5000
    - registry2:5000
    - registry3:5000
```

</details>

### 문제 3: Registry 간 이미지 동기화는?

<details>
<summary>정답 보기</summary>

**Harbor Replication:**
```yaml
# Harbor Web UI
# Replications → New Replication Rule

Source:
  Registry: harbor-dc1.example.com
  Resource Filter: myproject/**

Destination:
  Registry: harbor-dc2.example.com

Trigger:
  - Manual
  - Scheduled (Cron)
  - Event Based (Push)
```

**수동 동기화:**
```bash
#!/bin/bash
# sync-registries.sh

SOURCE=harbor-dc1.example.com
DEST=harbor-dc2.example.com

# 이미지 목록
IMAGES=$(curl -u admin:password \
  https://$SOURCE/api/v2.0/projects/myproject/repositories)

for IMAGE in $IMAGES; do
  # Pull
  docker pull $SOURCE/$IMAGE
  
  # Retag
  docker tag $SOURCE/$IMAGE $DEST/$IMAGE
  
  # Push
  docker push $DEST/$IMAGE
done
```

</details>

---

## 📌 핵심 요약

```
Private Registry 핵심:
1. 보안 (TLS + Auth)
2. 고가용성 (HA)
3. 성능 (Cache, CDN)
4. 관리 (GC, Backup)
5. 모니터링

Best Practices:
✅ Harbor (프로덕션)
✅ TLS 필수
✅ S3 Backend
✅ 정기 GC
✅ Replication
```

---

## 📚 참고 자료

- [Docker Registry Documentation](https://docs.docker.com/registry/)
- [Harbor Documentation](https://goharbor.io/docs/)
- [OCI Distribution Spec](https://github.com/opencontainers/distribution-spec)

---

## 🤔 생각해볼 문제

1. Docker Hub Rate Limit을 우회하는 방법은?
2. Multi-region Registry 전략은?
3. Registry의 백업 전략은?

> 💡 **답변**:
> 
> **1) Rate Limit 우회:**
> 
> ```bash
> # 방법 1: Pull-through Cache
> registry-mirror → Docker Hub
> 
> # 방법 2: Docker Hub Pro
> - 무제한 Pull
> - $5/월
> 
> # 방법 3: Base Image 복사
> docker pull nginx
> docker tag nginx my-registry/nginx
> docker push my-registry/nginx
> ```
> 
> **2) Multi-region:**
> 
> ```
> US-East Registry ←→ EU-West Registry
>                ↕
>           Asia Registry
> 
> - Replication (Harbor)
> - DNS Routing (가까운 곳)
> - 자동 Failover
> ```
> 
> **3) Backup:**
> 
> ```bash
> # 1. 메타데이터 (Harbor)
> harbor-db → PostgreSQL Backup
> 
> # 2. 이미지 (S3)
> S3 Versioning + Lifecycle
> 
> # 3. 정기 백업
> 0 0 * * * backup-registry.sh
> ```

---

<div align="center">

**[⬅️ 이전: Image Tagging](./02-Image-Tagging.md)** | **[다음: Automated Testing ➡️](./04-Automated-Testing.md)**

</div>
