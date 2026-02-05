# 04. Union Filesystem - 유니온 파일시스템의 비밀

## 🎯 이 챕터에서 배울 것

- **Union Filesystem**의 개념과 동작 원리
- **OverlayFS**의 내부 구조 완전 분석
- 다양한 스토리지 드라이버 (AUFS, Btrfs, ZFS) 비교
- 성능 특성과 실무 선택 기준

## 📌 왜 중요한가?

**"레이어를 어떻게 합치는가?"**

```
지금까지 배운 것:
- 이미지 = 여러 레이어의 스택 ✅
- 각 레이어는 독립적 ✅

궁금한 점:
- 여러 레이어를 어떻게 하나의 파일시스템으로 보이게 할까? 🤔
- /etc/hosts가 여러 레이어에 있으면 어느 것이 보일까? 🤔
- 파일 삭제는 어떻게 표현할까? 🤔

답: Union Filesystem!
```

**실무 영향:**
- 스토리지 드라이버 선택 → 성능 차이 최대 3배
- 파일시스템 이해 → 디버깅 능력 향상
- I/O 패턴 최적화 → 애플리케이션 성능 개선

---

## 🔬 Deep Dive

### 1. Union Filesystem이란?

#### 기본 개념

**Union Mount: 여러 디렉토리를 하나로**

```
일반 파일시스템:
/mnt/usb  →  USB 드라이브 내용만 보임

Union Filesystem:
/merged  →  dirA + dirB + dirC 합쳐서 보임
             (같은 경로는 우선순위에 따라)

┌─────────────────────────────────────┐
│          Merged View (/merged)      │  ← 사용자가 보는 통합 뷰
│  /etc/hosts (from Layer 3)          │
│  /app/config.json (from Layer 2)    │
│  /usr/bin/curl (from Layer 1)       │
└─────────────────────────────────────┘
         ↑ Union Mount
┌─────────────────────────────────────┐
│  Layer 3 (R/W) - Container Layer    │
│  ├─ /etc/hosts (modified)           │
│  └─ /app/data.db (new)              │
├─────────────────────────────────────┤
│  Layer 2 (R/O) - Application        │
│  ├─ /app/config.json                │
│  └─ /app/app.js                     │
├─────────────────────────────────────┤
│  Layer 1 (R/O) - Base Image         │
│  ├─ /usr/bin/curl                   │
│  ├─ /etc/hosts (original)           │
│  └─ /lib/* (libraries)              │
└─────────────────────────────────────┘
```

#### 핵심 기능

1. **레이어 합성 (Composition)**
   - 여러 레이어를 하나의 파일시스템으로 통합
   - 위 레이어가 아래 레이어를 덮어씀 (overlay)

2. **Copy-on-Write (CoW)**
   - 읽기: 합성된 뷰에서 읽기
   - 쓰기: 최상위 레이어로 복사 후 수정

3. **Whiteout 파일**
   - 하위 레이어의 파일 삭제 표현
   - 실제로는 삭제하지 않고 "보이지 않게" 처리

---

### 2. OverlayFS - 현대 Docker의 기본

#### 구조

**OverlayFS 디렉토리 구조**

```
/var/lib/docker/overlay2/
├─ l/                           # 심볼릭 링크 짧은 이름
│  ├─ ABC123 -> ../abc123.../diff
│  ├─ DEF456 -> ../def456.../diff
│  └─ ...
├─ abc123.../                   # 레이어 1
│  ├─ diff/                     # 실제 파일 내용
│  │  ├─ etc/
│  │  ├─ usr/
│  │  └─ ...
│  ├─ link                      # 짧은 이름 (ABC123)
│  ├─ lower                     # 아래 레이어 참조
│  └─ work/                     # 작업 디렉토리
├─ def456.../                   # 레이어 2
│  ├─ diff/
│  ├─ link
│  ├─ lower
│  └─ work/
└─ container-id/                # 컨테이너 레이어
   ├─ diff/                     # 컨테이너 변경사항
   ├─ merged/                   # 마운트 포인트 (통합 뷰)
   ├─ lower                     # 이미지 레이어들
   └─ work/                     # CoW 작업 디렉토리
```

#### OverlayFS 마운트 동작

```bash
# OverlayFS 마운트 명령어 구조
mount -t overlay overlay \
  -o lowerdir=/lower1:/lower2:/lower3,\
     upperdir=/upper,\
     workdir=/work \
  /merged

# lowerdir: 읽기 전용 레이어들 (아래→위 순서)
# upperdir: 읽기/쓰기 레이어 (최상위)
# workdir:  내부 작업용 (임시)
# merged:   통합 뷰 (실제 사용)
```

#### 파일 연산

**1. 파일 읽기**
```
시나리오: /etc/hosts 읽기

┌─────────────────────────────────┐
│  Upper Layer (R/W)              │
│  (파일 없음)                      │
├─────────────────────────────────┤
│  Lower Layer 2 (R/O)            │
│  /etc/hosts (modified)          │  ← 여기서 발견! 읽기
├─────────────────────────────────┤
│  Lower Layer 1 (R/O)            │
│  /etc/hosts (original)          │  (무시됨)
└─────────────────────────────────┘

검색 순서: Upper → Lower2 → Lower1
결과: Lower2의 /etc/hosts 반환
```

**2. 파일 쓰기 (존재하지 않는 파일)**
```
시나리오: echo "test" > /newfile.txt

1. Upper Layer에 파일 생성
┌─────────────────────────────────┐
│  Upper Layer (R/W)              │
│  /newfile.txt (new)             │  ← 여기에 생성
├─────────────────────────────────┤
│  Lower Layers (R/O)             │
│  (변경 없음)                      │
└─────────────────────────────────┘

빠름! (단순 생성만)
```

**3. 파일 수정 (존재하는 파일)**
```
시나리오: echo "new" >> /etc/hosts

1. Lower에서 파일 찾기
┌─────────────────────────────────┐
│  Lower Layer 2                  │
│  /etc/hosts (original)          │  ← 발견
└─────────────────────────────────┘

2. Upper로 복사 (Copy-up)
┌─────────────────────────────────┐
│  Upper Layer (R/W)              │
│  /etc/hosts (copied + modified) │  ← 복사 후 수정
├─────────────────────────────────┤
│  Lower Layer 2                  │
│  /etc/hosts (original)          │  (가려짐)
└─────────────────────────────────┘

느림! (전체 파일 복사 필요)
```

**4. 파일 삭제**
```
시나리오: rm /etc/hosts

1. Whiteout 파일 생성
┌─────────────────────────────────┐
│  Upper Layer (R/W)              │
│  .wh.hosts                      │  ← whiteout 마커
├─────────────────────────────────┤
│  Lower Layer                    │
│  /etc/hosts                     │  (여전히 존재하지만 보이지 않음)
└─────────────────────────────────┘

실제로는 Lower에서 삭제 안 됨 (불변)
Upper에 "삭제됨" 표시만
```

**5. 디렉토리 삭제**
```
시나리오: rm -rf /app

1. Opaque 디렉토리 생성
┌─────────────────────────────────┐
│  Upper Layer (R/W)              │
│  /app/ (opaque)                 │  ← 불투명 디렉토리
│  .wh..wh..opq                   │  ← opaque 마커
├─────────────────────────────────┤
│  Lower Layer                    │
│  /app/file1.txt                 │  (가려짐)
│  /app/file2.txt                 │  (가려짐)
└─────────────────────────────────┘

하위 레이어의 전체 디렉토리 내용 숨김
```

---

### 3. 다른 스토리지 드라이버들

#### AUFS (Another Union File System)

**특징:**
```
장점:
├─ 성숙한 기술 (오래됨)
├─ 안정적
└─ 레이어 수 제한 없음

단점:
├─ Mainline 커널에 포함 안 됨
├─ 성능이 OverlayFS보다 느림
└─ 더 이상 권장되지 않음

사용:
Docker 초기 버전의 기본값
현재는 레거시
```

**디렉토리 구조:**
```
/var/lib/docker/aufs/
├─ diff/           # 각 레이어의 변경사항
│  ├─ layer1/
│  ├─ layer2/
│  └─ ...
├─ layers/         # 레이어 의존성 정보
└─ mnt/            # 마운트 포인트
```

#### Btrfs (B-tree File System)

**특징:**
```
장점:
├─ Copy-on-Write 네이티브 지원
├─ 스냅샷 기능 강력
├─ 블록 레벨 CoW (빠름)
└─ 공간 효율적

단점:
├─ 복잡한 설정
├─ 성능 예측 어려움
└─ 특별한 파일시스템 필요

사용:
고급 스토리지 기능 필요 시
SUSE Linux 기본값
```

**동작 방식:**
```
Btrfs Subvolume 활용:

/var/lib/docker/btrfs/subvolumes/
├─ base-image         # Subvolume 1
├─ layer-2            # Subvolume 2 (base 스냅샷)
└─ container-abc      # Subvolume 3 (layer-2 스냅샷)

스냅샷 = CoW 복사
변경 사항만 새로 쓰기
```

#### ZFS (Zettabyte File System)

**특징:**
```
장점:
├─ 강력한 데이터 무결성
├─ 압축/중복 제거 지원
├─ 스냅샷/클론 고속
└─ 엔터프라이즈급 안정성

단점:
├─ 메모리 사용량 높음
├─ Linux 메인라인 아님 (라이선스 문제)
└─ 설정 복잡

사용:
데이터 무결성이 중요한 환경
프로덕션 서버
```

#### Device Mapper

**특징:**
```
장점:
├─ 블록 레벨 CoW
└─ 안정적

단점:
├─ 설정 복잡
├─ 성능 오버헤드
└─ 디스크 공간 관리 어려움

사용:
RHEL/CentOS 구버전 기본값
현재는 권장 안 됨
```

---

## 💻 실습

### 실습 1: OverlayFS 직접 사용해보기

```bash
# 디렉토리 준비
mkdir -p /tmp/overlay-test/{lower1,lower2,upper,work,merged}

# Lower Layer 1: 기본 파일들
echo "Lower1: file1" > /tmp/overlay-test/lower1/file1.txt
echo "Lower1: file2" > /tmp/overlay-test/lower1/file2.txt
echo "Lower1: common" > /tmp/overlay-test/lower1/common.txt

# Lower Layer 2: 일부 파일 추가/덮어쓰기
echo "Lower2: file3" > /tmp/overlay-test/lower2/file3.txt
echo "Lower2: common" > /tmp/overlay-test/lower2/common.txt  # 덮어쓰기

# OverlayFS 마운트
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/overlay-test/lower2:/tmp/overlay-test/lower1,\
     upperdir=/tmp/overlay-test/upper,\
     workdir=/tmp/overlay-test/work \
  /tmp/overlay-test/merged

# 통합 뷰 확인
ls /tmp/overlay-test/merged/
# file1.txt  file2.txt  file3.txt  common.txt

cat /tmp/overlay-test/merged/common.txt
# Lower2: common  ← lower2가 lower1을 덮어씀

# 파일 수정 (Copy-on-Write)
echo "Modified!" >> /tmp/overlay-test/merged/file1.txt

# Upper에 복사됨 확인
ls /tmp/overlay-test/upper/
# file1.txt  ← 수정된 파일만 Upper에 존재

cat /tmp/overlay-test/upper/file1.txt
# Lower1: file1
# Modified!

# Lower는 변경 없음
cat /tmp/overlay-test/lower1/file1.txt
# Lower1: file1  ← 원본 그대로

# 파일 삭제
rm /tmp/overlay-test/merged/file2.txt

# Whiteout 파일 생성 확인
ls -la /tmp/overlay-test/upper/
# c---------. 1 root root 0, 0 ... .wh.file2.txt  ← whiteout

# 정리
sudo umount /tmp/overlay-test/merged
```

### 실습 2: Docker에서 OverlayFS 구조 탐색

```bash
# 컨테이너 실행
docker run -d --name overlay-explore nginx

# 컨테이너 ID 확인
CONTAINER_ID=$(docker inspect -f '{{.Id}}' overlay-explore)

# OverlayFS 디렉토리 찾기
OVERLAY_DIR=$(docker inspect -f '{{.GraphDriver.Data.MergedDir}}' overlay-explore)
echo $OVERLAY_DIR
# /var/lib/docker/overlay2/abc123.../merged

# 상위 디렉토리로 이동
cd $(dirname $OVERLAY_DIR)
pwd
# /var/lib/docker/overlay2/abc123...

# 구조 확인
tree -L 2
# .
# ├── diff/       ← 컨테이너 레이어 변경사항
# ├── link        ← 짧은 이름 (심볼릭 링크용)
# ├── lower       ← 하위 레이어 목록
# ├── merged/     ← 통합 뷰 (실제 컨테이너 파일시스템)
# └── work/       ← 내부 작업 디렉토리

# Lower 레이어 확인
cat lower
# l/ABC123:l/DEF456:l/GHI789
# → 3개의 하위 레이어

# 컨테이너에서 파일 수정
docker exec overlay-explore sh -c 'echo "test" > /test.txt'

# diff에 추가됨 확인
ls diff/
# test.txt  ← 새로 생성된 파일

# merged에도 보임
ls merged/
# test.txt  (+ nginx 파일들)

# 정리
docker rm -f overlay-explore
```

### 실습 3: 스토리지 드라이버 비교 벤치마크

```bash
# 현재 스토리지 드라이버 확인
docker info | grep "Storage Driver"
# Storage Driver: overlay2

# 파일 생성 성능 테스트
docker run --rm alpine sh -c '
  time for i in $(seq 1 1000); do
    echo "test" > /file$i.txt
  done
'
# real: 0m0.8s (overlay2)

# 파일 수정 성능 테스트 (Copy-on-Write)
docker run --rm ubuntu sh -c '
  # 1MB 파일 1000개 생성 (이미지 레이어)
  for i in $(seq 1 1000); do
    dd if=/dev/zero of=/file$i bs=1M count=1 2>/dev/null
  done
  
  # 각 파일의 첫 바이트만 수정 (CoW 발생)
  time for i in $(seq 1 1000); do
    echo "x" > /file$i
  done
'
# real: 0m5.2s (1000MB 복사 오버헤드)

# 작은 파일 vs 큰 파일 CoW 비교
docker run --rm alpine sh -c '
  # 작은 파일 (1KB)
  echo "small" > /small.txt
  time echo "x" >> /small.txt
  # real: 0m0.001s
  
  # 큰 파일 (100MB)
  dd if=/dev/zero of=/large bs=1M count=100 2>/dev/null
  time echo "x" >> /large
  # real: 0m0.5s (100MB 복사!)
'
```

### 실습 4: Whiteout 파일 직접 확인

```bash
# 컨테이너 실행
docker run -d --name whiteout-test alpine sleep 3600

# 파일 삭제
docker exec whiteout-test rm /etc/os-release

# 컨테이너 레이어 찾기
CONTAINER_LAYER=$(docker inspect -f '{{.GraphDriver.Data.UpperDir}}' whiteout-test)

# Whiteout 파일 확인
sudo ls -la $CONTAINER_LAYER/etc/
# c---------. 1 root root 0, 0 ... .wh.os-release

# 파일 타입 확인 (character device 0,0)
sudo stat $CONTAINER_LAYER/etc/.wh.os-release
# File: .wh.os-release
# Size: 0               Blocks: 0          IO Block: 4096   character special file
# Device: 0,0           Inode: ...

# 디렉토리 삭제 (opaque)
docker exec whiteout-test rm -rf /usr/share/apk

# Opaque 마커 확인
sudo ls -la $CONTAINER_LAYER/usr/share/
# c---------. 1 root root 0, 0 ... apk/
# c---------. 1 root root 0, 0 ... .wh..wh..opq

# 정리
docker rm -f whiteout-test
```

---

## 🔥 실전 적용

### 시나리오 1: 스토리지 드라이버 선택

**요구사항별 선택 가이드:**

```bash
# 1. 일반적인 경우 → OverlayFS
cat > /etc/docker/daemon.json <<EOF
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

# 2. SLES/OpenSUSE → Btrfs
{
  "storage-driver": "btrfs"
}

# 3. ZFS Pool 있는 경우 → ZFS
{
  "storage-driver": "zfs"
}

# 적용
sudo systemctl restart docker
docker info | grep "Storage Driver"
```

### 시나리오 2: 성능 문제 디버깅

**문제: 컨테이너에서 파일 쓰기가 느림**

```bash
# 1. 스토리지 드라이버 확인
docker info | grep "Storage Driver"

# 2. 레이어 수 확인 (많으면 느림)
docker history myimage --no-trunc
# → 100개 레이어?? 너무 많음!

# 3. 해결: 멀티 스테이지 빌드로 레이어 최소화
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM node:18-alpine
COPY --from=builder /app/dist ./dist
# → 최종 이미지는 레이어 2-3개만
```

**문제: 큰 파일 수정 시 컨테이너 시작 느림**

```bash
# 원인: CoW로 인한 복사 오버헤드

# 해결 1: Volume 사용 (CoW 우회)
docker run -v /host/data:/data myimage

# 해결 2: 큰 파일은 이미지에 포함 안 함
# 빌드 시:
RUN wget large-file.zip && \
    unzip large-file.zip && \
    rm large-file.zip  # 같은 레이어에서 삭제!
```

### 시나리오 3: 디스크 공간 절약

**레이어 공유 활용:**

```bash
# Before: 각 앱마다 별도 베이스 이미지
FROM ubuntu:22.04  # App1
FROM ubuntu:20.04  # App2  ← 다른 버전
FROM ubuntu:22.04  # App3

# 디스크: 3 × 80MB = 240MB

# After: 동일한 베이스 이미지 사용
FROM ubuntu:22.04  # App1
FROM ubuntu:22.04  # App2  ← 레이어 공유!
FROM ubuntu:22.04  # App3  ← 레이어 공유!

# 디스크: 80MB + 앱 레이어들만
# → 160MB 절약!
```

### 시나리오 4: 개발 환경 최적화

**Bind Mount vs Volume vs Copy**

```bash
# 개발 중: Bind Mount (호스트 파일 직접 사용)
docker run -v $(pwd):/app myimage
# 장점: 코드 수정 즉시 반영
# 단점: 느린 I/O (특히 Mac/Windows)

# 빌드 시: Copy (이미지에 포함)
COPY . /app
# 장점: 빠른 I/O
# 단점: 코드 수정마다 재빌드

# 최적: 하이브리드
# Dockerfile:
COPY package*.json ./
RUN npm install
# 소스는 bind mount:
docker run -v $(pwd)/src:/app/src myimage
```

---

## ⚡ 고급 팁

### 1. inode 사용량 모니터링

```bash
# Overlay2는 많은 inode 사용
df -i /var/lib/docker
# Inodes: 10000000 used / 50000000 total

# inode 부족 시 컨테이너 생성 실패
# 해결: XFS 파일시스템 사용 (더 많은 inode)
mkfs.xfs -i size=512 /dev/sdb1
```

### 2. Overlay 레이어 깊이 제한

```bash
# OverlayFS는 레이어 128개 제한
# 확인:
docker history myimage | wc -l
# 150  ← 문제!

# 해결: 이미지 flatten (squash)
docker build --squash -t myimage:squashed .
# → 모든 레이어를 하나로 압축
```

### 3. 커널 파라미터 최적화

```bash
# /etc/sysctl.conf
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=512

# 적용
sudo sysctl -p
```

---

## 🚫 안티패턴

### 1. Lower 디렉토리 직접 수정

```bash
# ❌ 절대 하지 말 것
sudo rm -rf /var/lib/docker/overlay2/abc123/diff/some-file
# → 이미지 손상!

# ✅ 올바른 방법
docker exec container rm /some-file
# 또는 새 이미지 빌드
```

### 2. 너무 많은 레이어

```dockerfile
# ❌ 나쁜 예: 1000개 레이어
FROM alpine
RUN apk add package1
RUN apk add package2
RUN apk add package3
... (997번 반복)

# 문제:
# - 느린 이미지 pull
# - 느린 컨테이너 시작
# - OverlayFS 한계 도달 가능

# ✅ 좋은 예: 레이어 최소화
FROM alpine
RUN apk add --no-cache \
    package1 \
    package2 \
    ... \
    package1000
```

### 3. 스토리지 드라이버 무시

```bash
# ❌ 기본값 그대로 사용
# (일부 환경에서는 비효율적)

# ✅ 환경에 맞게 선택
# Ubuntu: overlay2
# SLES: btrfs
# ZFS pool: zfs
```

---

## 📊 성능 비교 요약

| 드라이버 | 읽기 성능 | 쓰기 성능 | 메모리 | 안정성 | 추천도 |
|---------|----------|----------|--------|--------|--------|
| **overlay2** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ 1순위 |
| **aufs** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⚠️ 레거시 |
| **btrfs** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | 🔧 특수 목적 |
| **zfs** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | 🔧 엔터프라이즈 |
| **devicemapper** | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ❌ 비권장 |

---

## 🎓 핵심 정리

### Union Filesystem 원리

```
1. 레이어 합성
   ├─ 여러 레이어를 하나로 통합
   ├─ 위 레이어가 아래 레이어 덮어씀
   └─ 읽기: 위→아래 순서로 검색

2. Copy-on-Write
   ├─ 읽기: 빠름 (직접 읽기)
   ├─ 쓰기: 느림 (복사 필요)
   └─ 큰 파일 수정 주의

3. Whiteout 파일
   ├─ 삭제 표현 방법
   ├─ .wh.<filename>
   └─ 하위 레이어 파일 숨김
```

### 실무 가이드

```
1. 스토리지 드라이버 선택
   ├─ 일반: overlay2 (최우선)
   ├─ SUSE: btrfs
   └─ 특수: zfs (데이터 무결성)

2. 성능 최적화
   ├─ 레이어 수 최소화
   ├─ 큰 파일은 Volume 사용
   └─ 자주 쓰는 데이터는 tmpfs

3. 디버깅
   ├─ docker info
   ├─ /var/lib/docker/overlay2 탐색
   └─ mount | grep overlay
```

---

## 📚 참고 자료

- [OverlayFS Documentation](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html)
- [Docker Storage Drivers](https://docs.docker.com/storage/storagedriver/)
- [AUFS vs OverlayFS Performance](https://github.com/docker/docker/issues/21304)
- [Btrfs for Docker](https://docs.docker.com/storage/storagedriver/btrfs-driver/)

---

**🤔 생각해볼 문제**

1. OverlayFS에서 `/etc/hosts`를 3번 수정하면 몇 개의 복사본이 생길까?
2. 왜 Docker는 AUFS에서 OverlayFS로 전환했을까?
3. Kubernetes에서 컨테이너 간 파일 공유가 안 되는 이유는?

> 💡 **답변**: 1) 1개 (Upper Layer에만), 2) 커널 메인라인 포함 + 성능, 3) 각 컨테이너는 독립 Union Mount

---

<div align="center">

**[⬅️ 이전: Image Layers](./03-Image-Layers.md)** | **[다음: Namespaces ➡️](./05-Namespaces.md)**

</div>
