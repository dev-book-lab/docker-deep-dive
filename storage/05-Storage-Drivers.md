# 05. Storage Drivers - 스토리지 드라이버

## 🎯 이 챕터에서 배울 것

- Docker **Storage Driver** 시스템
- **overlay2**, **btrfs**, **zfs** 등 드라이버 비교
- 이미지 레이어와 **Copy-on-Write** 메커니즘
- 성능 최적화와 **드라이버 선택** 가이드

## 📌 왜 중요한가?

**"Storage Driver는 Docker 이미지와 컨테이너의 핵심 작동 방식을 결정합니다."**

```
Storage Driver (스토리지 드라이버):
- 이미지 레이어 관리
- 컨테이너 레이어 관리
- Copy-on-Write (CoW) 구현
- 파일시스템 추상화

Volume Driver vs Storage Driver:
┌─────────────────────────────────────┐
│ Volume Driver                       │
│ - 데이터 영속성                        │
│ - /var/lib/docker/volumes/          │
│ - 사용자 데이터                        │
│ - 선택: local, nfs, ceph, etc        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Storage Driver                      │
│ - 이미지/컨테이너 레이어                 │
│ - /var/lib/docker/<driver>/         │
│ - Docker 내부 작동                    │
│ - 선택: overlay2, btrfs, zfs, etc    │
└─────────────────────────────────────┘

이미지 레이어 구조:
┌─────────────────────────────────────┐
│ Container Layer (R/W)               │ ← Storage Driver
├─────────────────────────────────────┤
│ Image Layer 3 (R/O)                 │ ↓ 관리
├─────────────────────────────────────┤
│ Image Layer 2 (R/O)                 │ ↓
├─────────────────────────────────────┤
│ Image Layer 1 (R/O) [Base]          │ ↓
└─────────────────────────────────────┘

성능 비교 (참고):
┌──────────────┬──────────┬──────────┬──────────┐
│ 드라이버       │ 읽기      │ 쓰기      │ 안정성     │
├──────────────┼──────────┼──────────┼──────────┤
│ overlay2     │ ⭐⭐⭐⭐│ ⭐⭐⭐   │ ⭐⭐⭐⭐│
├──────────────┼──────────┼──────────┼──────────┤
│ btrfs        │ ⭐⭐⭐   │ ⭐⭐⭐⭐│ ⭐⭐⭐  │
├──────────────┼──────────┼──────────┼──────────┤
│ zfs          │ ⭐⭐⭐   │ ⭐⭐⭐⭐│ ⭐⭐⭐⭐│
├──────────────┼──────────┼──────────┼──────────┤
│ devicemapper │ ⭐⭐     │ ⭐⭐    │ ⭐⭐⭐   │
└──────────────┴──────────┴──────────┴──────────┘
```

**실무 영향:**
- 성능: 읽기/쓰기 속도, IOPS
- 안정성: 데이터 무결성, 장애 복구
- 호환성: 커널 버전, 파일시스템
- 운영: 디스크 사용량, 관리 복잡도

---

## 🔬 Deep Dive

### 1. Storage Driver 개념

#### 기본 구조

```
Storage Driver 역할:
┌──────────────────────────────────────┐
│ Docker Engine                        │
│ ┌──────────────────────────────────┐ │
│ │ Storage Driver Interface         │ │
│ └────────────┬─────────────────────┘ │
└──────────────┼───────────────────────┘
               │
    ┌──────────▼──────────┐
    │ Storage Driver      │
    │ (overlay2 등)       │
    └──────────┬──────────┘
               │
    ┌──────────▼──────────┐
    │ Host Filesystem     │
    │ (ext4, xfs, etc)    │
    └─────────────────────┘

주요 기능:
1. 이미지 레이어 저장
2. 컨테이너 레이어 생성
3. Copy-on-Write 구현
4. 레이어 공유
5. 디스크 공간 관리
```

#### 현재 드라이버 확인

```bash
# 현재 사용 중인 스토리지 드라이버
docker info | grep "Storage Driver"
# Storage Driver: overlay2

# 상세 정보
docker info --format '{{.Driver}}'
# overlay2

# 드라이버별 정보
docker info --format '{{json .DriverStatus}}' | jq
```

---

### 2. overlay2 (권장)

#### 개념

```
overlay2:
- OverlayFS 기반
- 리눅스 커널 3.18+ 지원
- Docker 기본 드라이버 (18.06+)
- 가장 빠르고 안정적

구조:
┌──────────────────────────────────────┐
│ Merged View (Container)              │
│ /var/lib/docker/overlay2/merged/     │
└────────────┬─────────────────────────┘
             │ Union Mount
    ┌────────▼─────────┬────────────────┐
    │                  │                │
┌───▼─────────┐ ┌──────▼──────┐ ┌──────▼──────┐
│ Upper Layer │ │ Lower Layer │ │ Lower Layer │
│ (R/W)       │ │ (R/O)       │ │ (R/O)       │
│ Container   │ │ Image L3    │ │ Image L2    │
└─────────────┘ └─────────────┘ └─────────────┘

특징:
- 2개 레이어만 (upper + lower)
- inode 효율적
- page cache 공유
- 높은 성능
```

#### overlay2 상세 정보

```bash
# overlay2 디렉토리 구조
ls -la /var/lib/docker/overlay2/
# drwx------  3 root root 4096 Jan 15 10:00 abc123.../
# drwx------  3 root root 4096 Jan 15 10:01 def456.../
# drwx------  2 root root 4096 Jan 15 10:02 l/

# 각 레이어 구조
ls -la /var/lib/docker/overlay2/abc123.../
# diff/     # 실제 파일 (변경사항)
# link      # 심볼릭 링크 이름
# lower     # 하위 레이어 참조
# merged/   # 마운트 포인트
# work/     # OverlayFS 작업 디렉토리

# 컨테이너 실행
docker run -d --name test alpine sleep 3600

# 컨테이너 레이어 찾기
CONTAINER_ID=$(docker inspect test --format '{{.Id}}')
find /var/lib/docker/overlay2 -name $CONTAINER_ID

# 레이어 정보
docker inspect test --format '{{json .GraphDriver}}' | jq
# {
#   "Data": {
#     "LowerDir": "/var/lib/docker/overlay2/.../diff",
#     "MergedDir": "/var/lib/docker/overlay2/.../merged",
#     "UpperDir": "/var/lib/docker/overlay2/.../diff",
#     "WorkDir": "/var/lib/docker/overlay2/.../work"
#   },
#   "Name": "overlay2"
# }

docker rm -f test
```

#### Copy-on-Write 동작

```bash
# 이미지 pull
docker pull nginx:alpine

# 컨테이너 시작
docker run -d --name web nginx:alpine

# 컨테이너에서 파일 수정
docker exec web sh -c 'echo "Modified" > /usr/share/nginx/html/index.html'

# 레이어 확인
LAYER=$(docker inspect web --format '{{.GraphDriver.Data.UpperDir}}')
sudo ls -lh $LAYER/usr/share/nginx/html/
# -rw-r--r-- 1 root root 9 Jan 15 10:00 index.html

# 원본 이미지는 변경되지 않음
docker run --rm nginx:alpine cat /usr/share/nginx/html/index.html
# <!DOCTYPE html>...
# (원본 그대로)

docker rm -f web
```

---

### 3. btrfs

#### 개념

```
btrfs:
- B-tree File System
- Copy-on-Write 파일시스템
- 스냅샷, 압축 지원
- 서브볼륨 기반

구조:
┌──────────────────────────────────────┐
│ Btrfs Filesystem                     │
│ ┌──────────────────────────────────┐ │
│ │ Subvolume 1 (Image Layer 1)      │ │
│ └──────────────────────────────────┘ │
│ ┌──────────────────────────────────┐ │
│ │ Subvolume 2 (Image Layer 2)      │ │
│ │ (snapshot of Subvolume 1)        │ │
│ └──────────────────────────────────┘ │
│ ┌──────────────────────────────────┐ │
│ │ Subvolume 3 (Container Layer)    │ │
│ │ (snapshot of Image Layers)       │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘

특징:
- 네이티브 스냅샷 (빠름)
- 압축 (space-saving)
- 체크섬 (무결성)
- 서브볼륨 (격리)
```

#### btrfs 설정

```bash
# btrfs 파일시스템 생성 (빈 디스크 필요)
sudo mkfs.btrfs /dev/sdb

# 마운트
sudo mkdir /mnt/btrfs
sudo mount /dev/sdb /mnt/btrfs

# Docker 데이터 디렉토리 변경
sudo systemctl stop docker

# 기존 데이터 이동
sudo mv /var/lib/docker /var/lib/docker.bak

# btrfs에 Docker 디렉토리 생성
sudo mkdir /mnt/btrfs/docker
sudo ln -s /mnt/btrfs/docker /var/lib/docker

# Docker daemon 설정
sudo tee /etc/docker/daemon.json << EOF
{
  "storage-driver": "btrfs"
}
EOF

# Docker 시작
sudo systemctl start docker

# 확인
docker info | grep "Storage Driver"
# Storage Driver: btrfs
```

#### btrfs 장점

```bash
# 스냅샷 빠름
docker pull nginx:alpine

# 시간 측정
time docker run --rm nginx:alpine echo "test"
# 매우 빠름 (스냅샷 기반)

# 서브볼륨 확인
sudo btrfs subvolume list /mnt/btrfs
# ID 256 gen 123 top level 5 path docker/btrfs/subvolumes/...

# 디스크 사용량 (압축 시)
sudo btrfs filesystem df /mnt/btrfs
# Data, single: total=1.00GiB, used=256.00MiB
```

---

### 4. zfs

#### 개념

```
ZFS:
- Zettabyte File System
- Copy-on-Write
- 스냅샷, 클론
- 압축, 중복 제거
- 데이터 무결성

구조:
┌──────────────────────────────────────┐
│ ZFS Pool                             │
│ ┌──────────────────────────────────┐ │
│ │ Dataset 1 (Image Layer 1)        │ │
│ └──────────────────────────────────┘ │
│ ┌──────────────────────────────────┐ │
│ │ Clone 1 (Image Layer 2)          │ │
│ │ (snapshot of Dataset 1)          │ │
│ └──────────────────────────────────┘ │
│ ┌──────────────────────────────────┐ │
│ │ Clone 2 (Container Layer)        │ │
│ │ (snapshot of Image Layers)       │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘

특징:
- 엔터프라이즈급 안정성
- 압축/중복제거
- 스냅샷/클론 (빠름)
- 체크섬 (비트 부패 방지)
- ARC 캐시 (성능)
```

#### ZFS 설정

```bash
# ZFS 설치 (Ubuntu)
sudo apt-get update
sudo apt-get install -y zfsutils-linux

# ZFS pool 생성
sudo zpool create -f zpool-docker /dev/sdb

# Dataset 생성
sudo zfs create -o mountpoint=/var/lib/docker zpool-docker/docker

# Docker daemon 설정
sudo systemctl stop docker

sudo tee /etc/docker/daemon.json << EOF
{
  "storage-driver": "zfs"
}
EOF

sudo systemctl start docker

# 확인
docker info | grep "Storage Driver"
# Storage Driver: zfs

# ZFS 상태
sudo zpool status
sudo zfs list
```

#### ZFS 고급 기능

```bash
# 압축 활성화
sudo zfs set compression=lz4 zpool-docker/docker

# 중복 제거 (메모리 많이 필요)
sudo zfs set dedup=on zpool-docker/docker

# 스냅샷
sudo zfs snapshot zpool-docker/docker@backup-$(date +%Y%m%d)

# 스냅샷 목록
sudo zfs list -t snapshot

# 클론 (컨테이너 빠른 생성)
sudo zfs clone zpool-docker/docker@backup-20240115 \
  zpool-docker/docker-clone
```

---

### 5. devicemapper (레거시)

#### 개념

```
devicemapper:
- LVM 기반
- Docker 초기 드라이버
- 더 이상 권장되지 않음
- overlay2로 대체됨

모드:
1. loop-lvm (기본, 느림)
   - 파일 기반
   - 테스트용만

2. direct-lvm (프로덕션)
   - 블록 디바이스 직접 사용
   - 더 나은 성능

주의:
⚠️ Docker 18.09+ 부터 deprecated
⚠️ overlay2 사용 권장
⚠️ 복잡한 설정
⚠️ 성능 문제
```

---

## 💻 실습

### 실습 1: Storage Driver 변경

#### 현재 드라이버 확인

```bash
# 현재 드라이버
docker info --format '{{.Driver}}'
# overlay2

# 이미지/컨테이너 목록
docker images
docker ps -a

# 현재 데이터 디렉토리
ls -lh /var/lib/docker/overlay2/ | head -10
```

#### overlay2 → btrfs 변경 (실습용)

```bash
# 주의: 기존 이미지/컨테이너 모두 삭제됨!

# 1. Docker 중지
sudo systemctl stop docker

# 2. 기존 데이터 백업
sudo mv /var/lib/docker /var/lib/docker.bak

# 3. btrfs 파일시스템 준비 (가상 디스크)
dd if=/dev/zero of=/tmp/btrfs.img bs=1G count=10
sudo losetup /dev/loop0 /tmp/btrfs.img
sudo mkfs.btrfs /dev/loop0
sudo mkdir /mnt/docker-btrfs
sudo mount /dev/loop0 /mnt/docker-btrfs

# 4. Docker 디렉토리 생성
sudo mkdir /mnt/docker-btrfs/docker
sudo ln -s /mnt/docker-btrfs/docker /var/lib/docker

# 5. daemon.json 설정
sudo tee /etc/docker/daemon.json << EOF
{
  "storage-driver": "btrfs"
}
EOF

# 6. Docker 시작
sudo systemctl start docker

# 7. 확인
docker info | grep "Storage Driver"
# Storage Driver: btrfs

# 8. 테스트
docker run hello-world

# 9. 서브볼륨 확인
sudo btrfs subvolume list /mnt/docker-btrfs/

# 복원 (원래대로)
sudo systemctl stop docker
sudo umount /mnt/docker-btrfs
sudo losetup -d /dev/loop0
rm /tmp/btrfs.img
sudo rm -rf /var/lib/docker
sudo mv /var/lib/docker.bak /var/lib/docker
sudo rm /etc/docker/daemon.json
sudo systemctl start docker
```

---

### 실습 2: Copy-on-Write 성능 테스트

#### 테스트 스크립트

```bash
# 테스트 이미지 빌드
cat > Dockerfile << 'EOF'
FROM alpine
RUN dd if=/dev/zero of=/bigfile bs=1M count=100
EOF

docker build -t test-cow .

# 벤치마크 함수
benchmark_cow() {
  local driver=$1
  echo "=== Testing $driver ==="
  
  # 컨테이너 시작 시간
  echo "Container startup:"
  time for i in {1..10}; do
    docker run --rm test-cow echo "test" > /dev/null
  done
  
  # 쓰기 성능
  echo "Write performance:"
  time docker run --rm test-cow \
    dd if=/dev/zero of=/test bs=1M count=100 2>&1 | grep copied
}

# overlay2 테스트
benchmark_cow "overlay2"

# 결과:
# Container startup: real 0m5.234s
# 104857600 bytes (105 MB, 100 MiB) copied, 0.123 s, 851 MB/s

# 정리
docker rmi test-cow
```

---

### 실습 3: 레이어 공유 확인

#### 이미지 레이어 분석

```bash
# 1. 베이스 이미지
docker pull alpine:latest

# 2. 커스텀 이미지들
cat > Dockerfile.app1 << 'EOF'
FROM alpine:latest
RUN apk add --no-cache nginx
CMD ["nginx"]
EOF

cat > Dockerfile.app2 << 'EOF'
FROM alpine:latest
RUN apk add --no-cache python3
CMD ["python3"]
EOF

docker build -t app1 -f Dockerfile.app1 .
docker build -t app2 -f Dockerfile.app2 .

# 3. 레이어 확인
docker history app1 --no-trunc
docker history app2 --no-trunc

# 4. 디스크 사용량
docker system df -v

# 출력:
# Images space usage:
# REPOSITORY   TAG       SIZE      SHARED SIZE
# app1         latest    10MB      7MB
# app2         latest    12MB      7MB
# alpine       latest    7MB       7MB

# Shared Size 7MB = alpine 베이스 레이어 공유!

# 5. 레이어 디렉토리 확인
docker inspect app1 --format '{{json .GraphDriver.Data.LowerDir}}' | jq
docker inspect app2 --format '{{json .GraphDriver.Data.LowerDir}}' | jq
# 동일한 alpine 레이어 참조

# 정리
docker rmi app1 app2 alpine
```

---

### 실습 4: Storage Driver 벤치마크

#### 종합 벤치마크

```bash
# 벤치마크 스크립트
cat > storage-bench.sh << 'EOF'
#!/bin/bash

DRIVER=$1
ITERATIONS=100

echo "=== Storage Driver: $DRIVER ==="

# 1. 이미지 Pull (캐시 제거 후)
echo "Image pull performance:"
docker rmi -f nginx:alpine 2>/dev/null
time docker pull nginx:alpine

# 2. 컨테이너 시작 (cold start)
echo "Container cold start:"
time docker run --rm nginx:alpine echo "test"

# 3. 컨테이너 시작 (warm start)
echo "Container warm start:"
time for i in $(seq 1 10); do
  docker run --rm nginx:alpine echo "test" > /dev/null
done

# 4. 쓰기 성능
echo "Write performance:"
time docker run --rm nginx:alpine \
  sh -c 'for i in $(seq 1 1000); do echo test > /tmp/file$i; done'

# 5. 읽기 성능
echo "Read performance:"
time docker run --rm nginx:alpine \
  sh -c 'for i in $(seq 1 1000); do cat /etc/passwd > /dev/null; done'

# 6. 디스크 사용량
echo "Disk usage:"
docker system df

echo ""
EOF

chmod +x storage-bench.sh

# overlay2 벤치마크
./storage-bench.sh overlay2

# 결과 예시:
# Image pull: real 0m12.345s
# Cold start: real 0m0.456s
# Warm start: real 0m2.345s (10회)
# Write: real 0m0.678s
# Read: real 0m0.234s

rm storage-bench.sh
```

---

## 🔥 실전 적용

### 시나리오 1: 고성능 CI/CD

**요구사항:**
- 빈번한 이미지 빌드
- 빠른 컨테이너 시작
- 많은 레이어 공유

**최적 구성: overlay2**

```json
# /etc/docker/daemon.json
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

**이유:**
```
- 최고 성능 (읽기/쓰기)
- 빠른 컨테이너 시작
- 레이어 공유 효율적
- 안정적
- 커널 지원 보편적
```

---

### 시나리오 2: 엔터프라이즈 프로덕션

**요구사항:**
- 데이터 무결성 최우선
- 스냅샷 백업
- 압축/중복제거
- 장기 운영

**최적 구성: ZFS**

```json
# /etc/docker/daemon.json
{
  "storage-driver": "zfs",
  "storage-opts": [
    "zfs.fsname=zpool-docker/docker"
  ]
}
```

```bash
# ZFS 최적화
sudo zfs set compression=lz4 zpool-docker/docker
sudo zfs set atime=off zpool-docker/docker
sudo zfs set checksum=sha256 zpool-docker/docker

# 자동 스냅샷 (cron)
0 2 * * * /usr/sbin/zfs snapshot zpool-docker/docker@daily-$(date +\%Y\%m\%d)
```

**이유:**
```
- 비트 부패 방지 (체크섬)
- 스냅샷 (빠르고 공간 효율적)
- 압축 (디스크 절약)
- ARC 캐시 (성능)
- 엔터프라이즈급 안정성
```

---

### 시나리오 3: 대용량 이미지 환경

**요구사항:**
- ML/AI 이미지 (10GB+)
- 많은 변형 이미지
- 디스크 공간 절약

**최적 구성: btrfs (압축)**

```json
# /etc/docker/daemon.json
{
  "storage-driver": "btrfs"
}
```

```bash
# btrfs 압축 활성화
sudo btrfs property set /var/lib/docker compression zstd

# 압축 레벨 조정
sudo btrfs property set /var/lib/docker compression zstd:3
```

**효과:**
```
Before (overlay2):
- 100 images × 10GB = 1TB
- 공유 레이어: 800GB
- 실제 사용: 200GB

After (btrfs + compression):
- 압축률: 50%
- 실제 사용: 100GB
- 절약: 50%
```

---

### 시나리오 4: 개발 환경 (Docker Desktop)

**플랫폼별 권장:**

```
Linux:
✅ overlay2 (기본)
- 최고 성능
- 완벽 지원

Mac (Docker Desktop):
✅ overlay2 (VM 내부)
- virtiofs 공유
- 성능 제한 (VM)

Windows (Docker Desktop):
✅ overlay2 (WSL2)
- ext4 on VHDX
- 양호한 성능

Windows (Hyper-V):
⚠️ overlay2 (VM)
- 느림
- WSL2 권장
```

---

## ⚡ Storage Driver 선택 가이드

### 드라이버 상세 비교

```
┌────────────┬──────────┬─────────┬─────────┬─────────┬──────────┐
│ 특징        │overlay2  │btrfs    │ zfs     │devicemap│ 권장      │
├────────────┼──────────┼─────────┼─────────┼─────────┼──────────┤
│ 성능(읽기)   │ ⭐⭐⭐⭐│ ⭐⭐⭐  │ ⭐⭐⭐ │ ⭐⭐    │ overlay2 │
├────────────┼──────────┼─────────┼─────────┼─────────┼──────────┤
│ 성능(쓰기)   │ ⭐⭐⭐  │ ⭐⭐⭐⭐│ ⭐⭐⭐⭐│ ⭐⭐   │ btrfs    │
├────────────┼─────────┼──────────┼─────────┼─────────┼──────────┤
│ 안정성       │ ⭐⭐⭐⭐│⭐⭐⭐  │⭐⭐⭐⭐ │⭐⭐⭐  │zfs       │
├────────────┼─────────┼──────────┼─────────┼─────────┼──────────┤
│ 압축        │ ❌      │ ✅       │ ✅      │ ❌     │ zfs      │
├────────────┼─────────┼──────────┼─────────┼─────────┼──────────┤
│ 중복제거     │ ❌      │ ❌       │ ✅      │ ❌     │ zfs      │
├────────────┼─────────┼──────────┼─────────┼─────────┼──────────┤
│ 스냅샷       │ ❌      │ ✅      │ ✅      │ ✅      │ zfs      │
├────────────┼─────────┼──────────┼─────────┼─────────┼──────────┤
│ 복잡도       │ ⭐     │ ⭐⭐     │ ⭐⭐⭐  │ ⭐⭐⭐⭐│ overlay2│
├────────────┼─────────┼──────────┼─────────┼─────────┼──────────┤
│ 커널 지원    │ 3.18+   │ 3.9+     │모듈 필요   │ 2.6+   │ overlay2 │
└────────────┴─────────┴──────────┴─────────┴─────────┴──────────┘
```

### 선택 기준

```
일반 용도 (권장):
→ overlay2
- 최고 성능
- 간단한 설정
- 보편적 지원

엔터프라이즈:
→ ZFS
- 데이터 무결성
- 스냅샷/백업
- 압축/중복제거

대용량 이미지:
→ btrfs (압축)
- 디스크 절약
- 빠른 스냅샷

레거시:
→ devicemapper
- 피하기 (deprecated)
```

### 파일시스템 호환성

```
overlay2:
✅ ext4, xfs (권장)
⚠️ btrfs (비권장, 버그 있음)
❌ zfs (지원 안 됨)

btrfs:
✅ btrfs 파일시스템 필요

zfs:
✅ ZFS pool 필요

devicemapper:
✅ 모든 파일시스템
⚠️ 복잡한 설정
```

---

## 🚫 안티패턴

### 1. 잘못된 파일시스템

```bash
# ❌ overlay2 on btrfs
# btrfs 위에서 overlay2 사용 (버그 있음)
sudo mkfs.btrfs /dev/sdb
sudo mount /dev/sdb /var/lib/docker
# overlay2 + btrfs = 문제 발생 가능

# ✅ 올바른 조합
# ext4/xfs + overlay2
# 또는 btrfs 드라이버 사용
```

### 2. loop-lvm 프로덕션 사용

```json
// ❌ loop-lvm (테스트용만)
{
  "storage-driver": "devicemapper",
  "storage-opts": [
    "dm.loopmetadatasize=2G",
    "dm.loopdatasize=100G"
  ]
}

// ✅ overlay2 사용
{
  "storage-driver": "overlay2"
}
```

### 3. 드라이버 무분별 변경

```bash
# ❌ 데이터 백업 없이 변경
sudo vi /etc/docker/daemon.json  # overlay2 → zfs
sudo systemctl restart docker
# 모든 이미지/컨테이너 소실!

# ✅ 올바른 절차
# 1. 데이터 백업
docker save $(docker images -q) > images.tar
# 2. 드라이버 변경
# 3. 이미지 복원
docker load < images.tar
```

### 4. 압축 과다 사용

```bash
# ❌ 높은 압축 레벨
sudo zfs set compression=gzip-9 zpool/docker
# CPU 과부하

# ✅ 적절한 압축
sudo zfs set compression=lz4 zpool/docker
# 빠르고 효율적
```

---

## 🎓 핵심 정리

### 1. Storage Driver 역할

```
기능:
- 이미지 레이어 관리
- 컨테이너 레이어 관리
- Copy-on-Write 구현
- 디스크 공간 최적화

Volume Driver와 차이:
- Storage Driver: 내부 (이미지/컨테이너)
- Volume Driver: 외부 (사용자 데이터)
```

### 2. 주요 드라이버

```
overlay2:
- 기본/권장
- 최고 성능
- 간단

btrfs:
- 스냅샷/압축
- 쓰기 빠름
- 중간 복잡도

zfs:
- 엔터프라이즈
- 데이터 무결성
- 압축/중복제거
- 복잡

devicemapper:
- Deprecated
- 사용 금지
```

### 3. Copy-on-Write

```
동작:
1. 읽기: 하위 레이어 직접
2. 쓰기: 상위 레이어로 복사 후 수정
3. 삭제: 상위 레이어에 whiteout 파일

장점:
- 레이어 공유
- 빠른 시작
- 디스크 절약

단점:
- 첫 쓰기 느림
- 공간 중복 가능
```

### 4. 핵심 명령어

```bash
# 현재 드라이버
docker info | grep "Storage Driver"

# 드라이버 변경
# /etc/docker/daemon.json
{
  "storage-driver": "<driver>"
}

# 레이어 정보
docker history <image>
docker inspect <container> --format '{{json .GraphDriver}}'

# 디스크 사용량
docker system df -v
```

---

## 📚 참고 자료

- [Docker Storage Drivers](https://docs.docker.com/storage/storagedriver/)
- [overlay2 Driver](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)
- [btrfs](https://btrfs.wiki.kernel.org/)
- [ZFS on Linux](https://zfsonlinux.org/)

---

## 🤔 생각해볼 문제

1. overlay2가 다른 드라이버보다 빠른 이유는?
2. btrfs에서 압축이 성능에 미치는 영향은?
3. Storage Driver를 변경하면 기존 이미지는?

> 💡 **답변**: 1) OverlayFS는 커널 레벨에서 네이티브 지원 - 단순한 Union Mount (2개 레이어만), inode 공유로 메모리 효율적, page cache 공유로 읽기 빠름, 복잡한 스택 없음 (devicemapper는 LVM → Device Mapper → 파일시스템), 2) LZ4 압축 (기본): 압축 오버헤드 < 디스크 I/O 감소, 오히려 빠를 수 있음 (압축된 데이터 = 적은 I/O), GZIP-9 같은 높은 레벨: CPU 과부하, 느려짐, 권장: lz4 또는 zstd:1~3, 3) 모두 소실됨! - Storage Driver 변경 = 새 스토리지 백엔드, 기존 레이어 접근 불가, 복원 방법: docker save로 미리 백업, 변경 후 docker load로 복원, 또는 registry 사용

---

<div align="center">

**[⬅️ 이전: Volume Drivers](./04-Volume-Drivers.md)** | **[다음: Data Persistence ➡️](./06-Data-Persistence.md)**

</div>
