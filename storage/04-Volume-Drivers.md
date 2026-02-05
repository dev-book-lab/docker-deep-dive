# 04. Volume Drivers - 볼륨 드라이버

## 🎯 이 챕터에서 배울 것

- Docker **Volume Driver** 시스템
- **NFS**, **GlusterFS**, **Ceph** 등 원격 스토리지 통합
- 클라우드 스토리지 연동 (**AWS EBS**, **Azure Disk**)
- 커스텀 드라이버 **개발** 및 실전 활용

## 📌 왜 중요한가?

**"로컬 스토리지만으로는 엔터프라이즈 요구사항을 충족할 수 없습니다."**

```
로컬 드라이버 (local):
┌─────────────────────────────────┐
│ Container                       │
│ /app/data                       │
└────────┬────────────────────────┘
         │
┌────────▼────────────────────────┐
│ Local Disk (Single Host)        │
│ /var/lib/docker/volumes/        │
│ ✅ 간단                          │
│ ❌ 단일 호스트 제한                 │
│ ❌ 고가용성 없음                   │
│ ❌ 확장성 없음                     │
└─────────────────────────────────┘

볼륨 드라이버 (NFS/Ceph/etc):
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Container 1 │  │ Container 2 │  │ Container 3 │
│ /app/data   │  │ /app/data   │  │ /app/data   │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
           ┌────────────▼────────────┐
           │ Remote Storage          │
           │ - NFS / GlusterFS       │
           │ - Ceph / AWS EBS        │
           │ ✅ 멀티 호스트 공유         │
           │ ✅ 고가용성               │
           │ ✅ 확장 가능              │
           └─────────────────────────┘

엔터프라이즈 요구사항:
- 컨테이너 마이그레이션 (호스트 간)
- 데이터 공유 (여러 컨테이너)
- 고가용성 (HA)
- 백업/복구
- 성능 최적화
```

**실무 영향:**
- 확장성: 여러 호스트에서 동일 볼륨 접근
- 가용성: 스토리지 장애 시 자동 복구
- 성능: 워크로드에 최적화된 스토리지
- 운영: 중앙 집중식 관리

---

## 🔬 Deep Dive

### 1. Volume Driver 시스템

#### 기본 구조

```
Docker Volume Plugin System:
┌──────────────────────────────────┐
│ Docker Engine                    │
│ ┌──────────────────────────────┐ │
│ │ Volume Management API        │ │
│ └────────────┬─────────────────┘ │
└──────────────┼───────────────────┘
               │
    ┌──────────▼──────────┐
    │ Plugin Framework    │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────────────────────────┐
    │ Volume Drivers                          │
    ├─────────────────────────────────────────┤
    │ - local (기본)                           │
    │ - nfs                                   │
    │ - ceph (rexray/ceph)                    │
    │ - glusterfs                             │
    │ - aws-ebs (rexray/ebs)                  │
    │ - azure-disk                            │
    │ - vsphere                               │
    │ - custom (사용자 정의)                     │
    └──────────┬──────────────────────────────┘
               │
    ┌──────────▼──────────┐
    │ Storage Backend     │
    │ - NFS Server        │
    │ - Ceph Cluster      │
    │ - GlusterFS         │
    │ - Cloud Storage     │
    └─────────────────────┘
```

#### 드라이버 확인

```bash
# 사용 가능한 드라이버
docker plugin ls

# 볼륨 드라이버 확인
docker volume ls --format "{{.Driver}}" | sort | uniq
# local

# 특정 드라이버의 볼륨
docker volume ls --filter driver=local
```

---

### 2. NFS (Network File System)

#### NFS 서버 설정

```bash
# Ubuntu/Debian에서 NFS 서버 설치
sudo apt-get update
sudo apt-get install -y nfs-kernel-server

# NFS 공유 디렉토리 생성
sudo mkdir -p /mnt/nfs_share
sudo chown nobody:nogroup /mnt/nfs_share
sudo chmod 777 /mnt/nfs_share

# NFS exports 설정
sudo tee /etc/exports << EOF
/mnt/nfs_share *(rw,sync,no_subtree_check,no_root_squash)
EOF

# NFS 서버 재시작
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

# 방화벽 (필요시)
sudo ufw allow from any to any port nfs
```

#### NFS 볼륨 사용

```bash
# NFS 볼륨 생성
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/mnt/nfs_share \
  nfs-vol

# 볼륨 확인
docker volume inspect nfs-vol
# {
#     "Driver": "local",
#     "Labels": {},
#     "Mountpoint": "/var/lib/docker/volumes/nfs-vol/_data",
#     "Name": "nfs-vol",
#     "Options": {
#         "device": ":/mnt/nfs_share",
#         "o": "addr=192.168.1.100,rw",
#         "type": "nfs"
#     },
#     "Scope": "local"
# }

# 컨테이너에서 사용
docker run -d --name web1 \
  -v nfs-vol:/data \
  nginx:alpine

docker run -d --name web2 \
  -v nfs-vol:/data \
  nginx:alpine

# 데이터 공유 테스트
docker exec web1 sh -c 'echo "Shared data" > /data/test.txt'
docker exec web2 cat /data/test.txt
# Shared data
# ✅ 공유됨!

# 정리
docker rm -f web1 web2
docker volume rm nfs-vol
```

#### Docker Compose with NFS

```yaml
version: '3.8'

services:
  web1:
    image: nginx:alpine
    volumes:
      - nfs-data:/usr/share/nginx/html

  web2:
    image: nginx:alpine
    volumes:
      - nfs-data:/usr/share/nginx/html

volumes:
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/mnt/nfs_share"
```

---

### 3. REX-Ray (다중 스토리지 플랫폼)

#### REX-Ray 설치

```bash
# REX-Ray 설치
curl -sSL https://rexray.io/install | sh

# 설정 파일
sudo tee /etc/rexray/config.yml << EOF
libstorage:
  service: ebs
ebs:
  accessKey: YOUR_ACCESS_KEY
  secretKey: YOUR_SECRET_KEY
  region: us-east-1
EOF

# REX-Ray 서비스 시작
sudo systemctl start rexray
sudo systemctl enable rexray

# 확인
sudo rexray volume ls
```

#### AWS EBS 볼륨

```bash
# EBS 볼륨 생성
docker volume create \
  --driver rexray/ebs \
  --opt size=10 \
  ebs-vol

# 볼륨 확인
docker volume inspect ebs-vol

# 사용
docker run -d --name db \
  -v ebs-vol:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8

# EBS 볼륨은 컨테이너와 함께 이동 가능
# 다른 호스트에서도 동일 볼륨 마운트 가능
```

---

### 4. GlusterFS

#### GlusterFS 클러스터 설정

**Node 1, 2, 3에서:**

```bash
# GlusterFS 설치
sudo apt-get update
sudo apt-get install -y glusterfs-server

# 서비스 시작
sudo systemctl start glusterd
sudo systemctl enable glusterd

# 방화벽
sudo ufw allow 24007/tcp
sudo ufw allow 24008/tcp
sudo ufw allow 49152:49251/tcp
```

**Node 1에서 (마스터):**

```bash
# 피어 추가
sudo gluster peer probe node2
sudo gluster peer probe node3

# 피어 상태 확인
sudo gluster peer status

# 볼륨 생성 (replica 3 - 복제)
sudo gluster volume create gv0 \
  replica 3 \
  node1:/data/glusterfs/gv0/brick1 \
  node2:/data/glusterfs/gv0/brick1 \
  node3:/data/glusterfs/gv0/brick1

# 볼륨 시작
sudo gluster volume start gv0

# 볼륨 정보
sudo gluster volume info gv0
```

#### Docker with GlusterFS

```bash
# GlusterFS 플러그인 설치
docker plugin install \
  --alias glusterfs \
  trajano/glusterfs-volume-plugin \
  --grant-all-permissions

# GlusterFS 볼륨 생성
docker volume create \
  --driver glusterfs \
  --opt glusterserver=node1 \
  --opt glustervolume=gv0 \
  gluster-vol

# 사용
docker run -d --name app1 \
  -v gluster-vol:/data \
  nginx:alpine

docker run -d --name app2 \
  -v gluster-vol:/data \
  nginx:alpine

# 데이터 공유 및 복제
docker exec app1 sh -c 'echo "Replicated data" > /data/test.txt'
docker exec app2 cat /data/test.txt
# Replicated data

# Node 1 장애 발생해도 Node 2, 3에서 데이터 접근 가능!
```

---

### 5. Ceph (분산 스토리지)

#### Ceph 클러스터 (간단 설정)

```bash
# Ceph 설치 (Ubuntu)
sudo apt-get update
sudo apt-get install -y ceph ceph-common

# Ceph 설정 (간단한 단일 노드 예시)
sudo ceph-deploy new node1
sudo ceph-deploy install node1
sudo ceph-deploy mon create-initial
sudo ceph-deploy admin node1
sudo ceph-deploy mgr create node1
sudo ceph-deploy osd create --data /dev/sdb node1

# 상태 확인
sudo ceph -s
```

#### Docker with Ceph RBD

```bash
# Ceph RBD 플러그인
docker plugin install \
  --alias rbd \
  wetopi/rbd \
  --grant-all-permissions

# Ceph pool 생성
sudo ceph osd pool create docker 128

# RBD 볼륨 생성
docker volume create \
  --driver rbd \
  --opt pool=docker \
  --opt size=10240 \
  ceph-vol

# 사용
docker run -d --name db \
  -v ceph-vol:/var/lib/mysql \
  mysql:8

# 특징:
# - 블록 스토리지 (고성능)
# - 스냅샷 지원
# - 복제 (replica)
# - 분산 저장
```

---

## 💻 실습

### 실습 1: NFS 공유 스토리지

#### 환경 구성

```bash
# 1. NFS 서버 설정 (Docker로 간단히)
docker run -d --name nfs-server \
  --privileged \
  -p 2049:2049 \
  -e SHARED_DIRECTORY=/data \
  -v $(pwd)/nfs-data:/data \
  itsthenetwork/nfs-server-alpine:latest

# 2. NFS 서버 IP 확인
NFS_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nfs-server)
echo "NFS Server: $NFS_IP"

# 3. NFS 볼륨 생성 (클라이언트)
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=$NFS_IP,rw,nfsvers=4 \
  --opt device=:/data \
  shared-nfs

# 4. 여러 컨테이너에서 사용
docker run -d --name writer \
  -v shared-nfs:/data \
  alpine sh -c 'while true; do echo "$(date): Writer" >> /data/log.txt; sleep 5; done'

docker run -d --name reader1 \
  -v shared-nfs:/data \
  alpine sh -c 'while true; do tail -n 5 /data/log.txt; sleep 5; done'

docker run -d --name reader2 \
  -v shared-nfs:/data \
  alpine sh -c 'while true; do tail -n 5 /data/log.txt; sleep 5; done'

# 5. 로그 확인
docker logs -f reader1
# writer가 쓴 내용을 reader1, reader2 모두 읽음
# ✅ 실시간 공유!

# 6. NFS 서버에서 직접 확인
docker exec nfs-server cat /data/log.txt

# 정리
docker rm -f writer reader1 reader2 nfs-server
docker volume rm shared-nfs
```

---

### 실습 2: 로컬 드라이버 옵션

#### 다양한 옵션 테스트

```bash
# 1. 기본 로컬 볼륨
docker volume create basic-vol

# 2. tmpfs (메모리)
docker volume create \
  --driver local \
  --opt type=tmpfs \
  --opt device=tmpfs \
  --opt o=size=100m,uid=1000 \
  tmpfs-vol

# 3. Bind (호스트 디렉토리)
mkdir /tmp/bind-data

docker volume create \
  --driver local \
  --opt type=none \
  --opt o=bind \
  --opt device=/tmp/bind-data \
  bind-vol

# 4. NFS (원격)
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/mnt/nfs \
  nfs-vol

# 5. CIFS/SMB (Windows 공유)
docker volume create \
  --driver local \
  --opt type=cifs \
  --opt o=username=user,password=pass,addr=192.168.1.100 \
  --opt device=//192.168.1.100/share \
  smb-vol

# 비교
docker volume ls --format "table {{.Name}}\t{{.Driver}}\t{{.Scope}}"

# 정리
docker volume rm basic-vol tmpfs-vol bind-vol
rm -rf /tmp/bind-data
```

---

### 실습 3: Docker Compose 멀티 드라이버

#### 복합 스토리지 전략

```yaml
# docker-compose.yml
version: '3.8'

services:
  # 데이터베이스 (로컬 SSD)
  db:
    image: postgres:14-alpine
    volumes:
      # 고성능 로컬 스토리지
      - db-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret

  # 애플리케이션 (NFS 공유)
  app:
    image: nginx:alpine
    volumes:
      # 정적 파일 (NFS 공유)
      - app-static:/usr/share/nginx/html
      
      # 로그 (로컬)
      - app-logs:/var/log/nginx
    ports:
      - "80:80"

  # 캐시 (Tmpfs)
  redis:
    image: redis:alpine
    volumes:
      # 메모리 기반 (tmpfs)
      - type: tmpfs
        target: /data
        tmpfs:
          size: 256m

  # 백업 (NFS)
  backup:
    image: alpine
    volumes:
      # 백업 저장소 (NFS)
      - backup-data:/backups
    command: >
      sh -c "
        while true; do
          tar czf /backups/backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /source .
          sleep 3600
        done
      "

volumes:
  # 로컬 (기본)
  db-data:
    driver: local
    
  app-logs:
    driver: local

  # NFS (공유)
  app-static:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/mnt/nfs/static"
  
  backup-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/mnt/nfs/backups"
```

```bash
# 실행
docker-compose up -d

# 볼륨 확인
docker volume ls

# 드라이버별 분류
docker volume ls --format "{{.Name}}: {{.Driver}}"

# 정리
docker-compose down -v
```

---

### 실습 4: 플러그인 관리

#### 플러그인 설치 및 사용

```bash
# 1. 사용 가능한 플러그인 검색
# Docker Hub 또는 Docker Store에서

# 2. 플러그인 설치 예시 (vieux/sshfs)
docker plugin install \
  --grant-all-permissions \
  vieux/sshfs

# 3. 플러그인 확인
docker plugin ls
# ID         NAME             ENABLED
# abc123...  vieux/sshfs      true

# 4. SSHFS 볼륨 생성
docker volume create \
  --driver vieux/sshfs \
  -o sshcmd=user@host:/path \
  -o password=secret \
  ssh-vol

# 5. 사용
docker run -d --name app \
  -v ssh-vol:/data \
  nginx:alpine

# 6. 플러그인 비활성화
docker plugin disable vieux/sshfs

# 7. 플러그인 제거
docker plugin rm vieux/sshfs

# 8. 플러그인 업그레이드
docker plugin upgrade vieux/sshfs
```

---

## 🔥 실전 적용

### 시나리오 1: 하이브리드 스토리지 전략

**요구사항:**
- 데이터베이스: 고성능 로컬 SSD
- 정적 파일: NFS 공유 (CDN 연동)
- 로그: 중앙 집중식 (NFS)
- 캐시: 메모리 (Tmpfs)
- 백업: 클라우드 (S3)

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  # PostgreSQL (로컬 NVMe SSD)
  postgres:
    image: postgres:14-alpine
    volumes:
      - type: volume
        source: db-data
        target: /var/lib/postgresql/data
        volume:
          nocopy: true
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    deploy:
      resources:
        limits:
          memory: 4G
      placement:
        constraints:
          - node.labels.storage == nvme

  # Web Server (NFS 정적 파일)
  web:
    image: nginx:alpine
    volumes:
      # 정적 파일 (NFS)
      - type: volume
        source: static-files
        target: /usr/share/nginx/html
        read_only: true
      
      # 업로드 (NFS)
      - type: volume
        source: uploads
        target: /var/www/uploads
      
      # 로그 (NFS - 중앙 집중)
      - type: volume
        source: web-logs
        target: /var/log/nginx
    ports:
      - "80:80"

  # Redis (Tmpfs)
  redis:
    image: redis:alpine
    command: redis-server --save ""
    volumes:
      - type: tmpfs
        target: /data
        tmpfs:
          size: 2G
    deploy:
      resources:
        limits:
          memory: 2G

  # Elasticsearch (로컬 SSD)
  elasticsearch:
    image: elasticsearch:8.5.0
    volumes:
      - es-data:/usr/share/elasticsearch/data
    environment:
      - discovery.type=single-node
    deploy:
      placement:
        constraints:
          - node.labels.storage == ssd

  # 백업 서비스 (S3)
  backup:
    image: amazon/aws-cli
    volumes:
      - db-data:/source/db:ro
      - uploads:/source/uploads:ro
    entrypoint: >
      sh -c "
        while true; do
          aws s3 sync /source/db s3://my-bucket/backups/db/
          aws s3 sync /source/uploads s3://my-bucket/backups/uploads/
          sleep 86400
        done
      "
    environment:
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_KEY}

volumes:
  # 로컬 NVMe (데이터베이스)
  db-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/nvme/postgres

  # 로컬 SSD (Elasticsearch)
  es-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/ssd/elasticsearch

  # NFS (정적 파일 - CDN origin)
  static-files:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs.example.com,rw,nfsvers=4
      device: ":/exports/static"

  # NFS (업로드)
  uploads:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs.example.com,rw,nfsvers=4
      device: ":/exports/uploads"

  # NFS (로그 - 중앙 집중)
  web-logs:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs.example.com,rw,nfsvers=4
      device: ":/exports/logs/web"
```

**성능 특성:**

```
┌──────────────┬────────────┬─────────┬────────┐
│ 서비스         │ 스토리지     │ IOPS    │ 용도    │
├──────────────┼────────────┼─────────┼────────┤
│ PostgreSQL   │ NVMe       │ 100,000 │ 고성능   │
├──────────────┼────────────┼─────────┼────────┤
│ Elasticsearch│ SSD        │ 20,000  │ 검색    │
├──────────────┼────────────┼─────────┼────────┤
│ Redis        │ Tmpfs      │ 500,000+│ 캐시    │
├──────────────┼────────────┼─────────┼────────┤
│ Static Files │ NFS        │ 1,000   │ 공유    │
├──────────────┼────────────┼─────────┼────────┤
│ Logs         │ NFS        │ 500     │ 중앙화   │
└──────────────┴────────────┴─────────┴────────┘
```

---

### 시나리오 2: 멀티 클라우드 스토리지

**요구사항:**
- 데이터 지역성 (GDPR)
- 클라우드 벤더 종속 회피
- 재해 복구 (DR)

**구성:**

```yaml
version: '3.8'

services:
  # EU 리전 (AWS EBS)
  app-eu:
    image: myapp:latest
    volumes:
      - type: volume
        source: eu-data
        target: /app/data
    deploy:
      placement:
        constraints:
          - node.labels.region == eu-west-1

  # US 리전 (Azure Disk)
  app-us:
    image: myapp:latest
    volumes:
      - type: volume
        source: us-data
        target: /app/data
    deploy:
      placement:
        constraints:
          - node.labels.region == us-east-1

  # APAC 리전 (GCP Persistent Disk)
  app-apac:
    image: myapp:latest
    volumes:
      - type: volume
        source: apac-data
        target: /app/data
    deploy:
      placement:
        constraints:
          - node.labels.region == asia-northeast1

volumes:
  # AWS EBS (EU)
  eu-data:
    driver: rexray/ebs
    driver_opts:
      size: 100
      volumeType: gp3
      iops: 3000
      encrypted: "true"

  # Azure Disk (US)
  us-data:
    driver: rexray/azured
    driver_opts:
      size: 100
      storageAccountType: Premium_LRS

  # GCP Persistent Disk (APAC)
  apac-data:
    driver: rexray/gcepd
    driver_opts:
      size: 100
      type: pd-ssd
```

---

### 시나리오 3: 고가용성 스토리지

**요구사항:**
- 데이터 복제 (3개 복사본)
- 자동 장애 복구
- 성능 최적화

**GlusterFS 설정:**

```yaml
version: '3.8'

services:
  # 애플리케이션 (여러 노드)
  app:
    image: myapp:latest
    volumes:
      - type: volume
        source: replicated-data
        target: /app/data
    deploy:
      replicas: 10
      update_config:
        parallelism: 2
        delay: 10s
      placement:
        max_replicas_per_node: 2

  # 데이터베이스 (레플리카)
  db-primary:
    image: postgres:14-alpine
    volumes:
      - db-primary-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret
    deploy:
      placement:
        constraints:
          - node.labels.db-role == primary

  db-replica:
    image: postgres:14-alpine
    volumes:
      - db-replica-data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.labels.db-role == replica

volumes:
  # GlusterFS (replica 3)
  replicated-data:
    driver: glusterfs
    driver_opts:
      glusterserver: node1,node2,node3
      glustervolume: gv0

  # Ceph RBD (복제)
  db-primary-data:
    driver: rbd
    driver_opts:
      pool: postgres
      size: 100
      fstype: xfs

  db-replica-data:
    driver: rbd
    driver_opts:
      pool: postgres
      size: 100
      fstype: xfs
```

---

## ⚡ 볼륨 드라이버 선택 가이드

### 드라이버 비교

```
┌───────────┬─────────┬─────────┬─────────┬──────────┬───────────┐
│ 기능       │Local    │ NFS     │Ceph     │Gluster   │Cloud      │
├───────────┼─────────┼─────────┼─────────┼──────────┼───────────┤
│ 공유       │ ❌      │ ✅      │ ✅      │ ✅       │ ✅       │
├───────────┼─────────┼─────────┼─────────┼──────────┼───────────┤
│ 복제       │ ❌      │ ❌      │ ✅      │ ✅       │ ✅       │
├───────────┼─────────┼─────────┼─────────┼──────────┼───────────┤
│ 성능       │ ⭐⭐⭐⭐│ ⭐⭐   │ ⭐⭐⭐ │ ⭐⭐     │ ⭐⭐⭐    │
├───────────┼─────────┼─────────┼─────────┼──────────┼───────────┤
│ 복잡도      │ ⭐     │ ⭐⭐    │ ⭐⭐⭐⭐│ ⭐⭐⭐  │ ⭐⭐      │
├───────────┼─────────┼─────────┼─────────┼──────────┼───────────┤
│ 비용       │ 낮음     │ 낮음     │ 중간     │ 중간       │ 높음       │
└───────────┴─────────┴─────────┴─────────┴──────────┴───────────┘
```

### 선택 기준

```
Local:
✅ 단일 호스트
✅ 최고 성능
✅ 간단한 설정
❌ 공유 불가

NFS:
✅ 간단한 공유
✅ 범용적
✅ 낮은 비용
❌ 단일 실패점
❌ 성능 제한

Ceph:
✅ 고가용성
✅ 복제
✅ 확장성
⚠️ 복잡한 설정
⚠️ 리소스 많이 필요

GlusterFS:
✅ 간단한 복제
✅ 확장 가능
✅ 오픈소스
⚠️ 성능 (네트워크)

Cloud (EBS/Disk):
✅ 관리 불필요
✅ 백업/스냅샷
✅ 고가용성
❌ 비용
❌ 벤더 종속
```

---

## 🚫 안티패턴

### 1. 모든 것을 NFS로

```yaml
# ❌ 모든 볼륨을 NFS로
volumes:
  db-data:  # 데이터베이스도 NFS?
    driver: local
    driver_opts:
      type: nfs
      ...
# 성능 저하!

# ✅ 용도별 드라이버
volumes:
  db-data:  # 고성능 로컬
    driver: local
  
  static-files:  # 공유 NFS
    driver: local
    driver_opts:
      type: nfs
      ...
```

### 2. 드라이버 옵션 누락

```bash
# ❌ 옵션 없음
docker volume create --driver local nfs-vol
# 실제로 NFS가 아님!

# ✅ 명시적 옵션
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=...,rw \
  --opt device=:... \
  nfs-vol
```

### 3. 백업 전략 없음

```yaml
# ❌ 원격 스토리지라고 백업 안 함
volumes:
  data:
    driver: nfs
# NFS 서버 장애 시?

# ✅ 정기 백업
services:
  backup:
    image: backup-tool
    volumes:
      - data:/source:ro
      - backup-storage:/backups
    # 정기 백업 스크립트
```

### 4. 성능 테스트 생략

```bash
# ❌ 테스트 없이 프로덕션 적용
# 성능 문제 발견!

# ✅ 벤치마크
docker run --rm \
  -v test-vol:/data \
  ljishen/fio \
  fio --name=test ...
```

---

## 🎓 핵심 정리

### 1. Volume Driver 개념

```
역할:
- Docker Volume API 구현
- 다양한 스토리지 백엔드 연결
- 추상화 레이어 제공

종류:
- local (기본)
- nfs, cifs
- ceph, glusterfs
- cloud (ebs, azured, gcepd)
- custom
```

### 2. 주요 드라이버

```
NFS:
- 네트워크 파일 시스템
- 간단한 공유
- Linux/Unix 표준

Ceph:
- 분산 스토리지
- 블록/오브젝트/파일
- 고가용성

GlusterFS:
- 분산 파일시스템
- 복제
- 확장 가능

Cloud:
- AWS EBS
- Azure Disk
- GCP PD
- 관리형
```

### 3. 선택 기준

```
성능 우선:
→ Local (NVMe/SSD)

공유 필요:
→ NFS (간단)
→ GlusterFS (복제)

고가용성:
→ Ceph
→ Cloud

비용 중요:
→ Local
→ NFS
```

### 4. 핵심 명령어

```bash
# 볼륨 생성 (드라이버 지정)
docker volume create --driver <d> --opt ... <n>

# 플러그인 관리
docker plugin install
docker plugin ls
docker plugin enable/disable

# 볼륨 확인
docker volume inspect <n>
docker volume ls --filter driver=<d>
```

---

## 📚 참고 자료

- [Docker Volume Plugins](https://docs.docker.com/engine/extend/plugins_volume/)
- [REX-Ray](https://rexray.io/)
- [NFS](https://linux.die.net/man/5/nfs)
- [Ceph](https://docs.ceph.com/)
- [GlusterFS](https://www.gluster.org/)

---

## 🤔 생각해볼 문제

1. NFS 볼륨을 여러 컨테이너가 동시에 쓰면 안전할까?
2. Ceph와 GlusterFS의 근본적인 차이는?
3. 클라우드 스토리지 드라이버의 성능 오버헤드는?

> 💡 **답변**: 1) 애플리케이션에 따라 다름 - 읽기만: 안전, 동시 쓰기: 파일 잠금 메커니즘 필요, NFS는 기본적으로 파일 잠금 지원하지만 완벽하지 않음, 데이터베이스 같은 경우 절대 여러 인스턴스가 동일 NFS 볼륨 사용하면 안 됨 (데이터 손상), 해결: 애플리케이션 레벨 동기화, 또는 블록 스토리지 사용 (Ceph RBD), 2) Ceph는 object storage 기반으로 블록/파일/오브젝트 모두 지원, CRUSH 알고리즘으로 데이터 분산, GlusterFS는 파일시스템 기반, 더 간단하지만 Ceph보다 기능 적음, Ceph가 더 복잡하지만 확장성과 성능 우수, 3) 네트워크 레이턴시 + API 호출 오버헤드 - 로컬 대비 10-50% 느림, 특히 작은 파일 많을 때 더 큼, 하지만 대용량 순차 I/O는 비슷, 장점은 관리 편의성과 고가용성

---

<div align="center">

**[⬅️ 이전: Tmpfs Mounts](./03-Tmpfs-Mounts.md)** | **[다음: Storage Drivers ➡️](./05-Storage-Drivers.md)**

</div>
