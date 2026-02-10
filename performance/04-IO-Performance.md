# 04. I/O Performance - I/O 성능 최적화

## 🎯 이 챕터에서 배울 것

- **Block I/O 제어** - blkio weight, throttling
- **Storage Driver 최적화** - overlay2, devicemapper 비교
- **Volume 성능** - bind mount vs named volume vs tmpfs
- **I/O 벤치마킹** - fio, dd, sysbench
- **파일시스템 튜닝** - ext4, xfs 최적화

## 📌 왜 중요한가?

**"I/O 성능은 데이터베이스와 로그 처리에서 병목의 주요 원인입니다."**

```
I/O 최적화 전 vs 후:

I/O 최적화 전:
┌─────────────────────────────────────┐
│ Host Disk (500 IOPS)                │
│                                     │
│ Container A (DB):                   │
│ - Random Write: 500 IOPS            │
│ - Latency: 100ms                    │
│ - Throughput: 50MB/s                │
│                                     │
│ Container B (Log):                  │
│ - Sequential Write: 0 IOPS (대기)    │
│ - Latency: 1000ms (10배)            │
│ - Throughput: 5MB/s                 │
└─────────────────────────────────────┘

문제:
❌ I/O 경쟁으로 로그 유실
❌ DB 성능 저하
❌ 예측 불가능한 레이턴시
❌ Disk 병목

I/O 최적화 후:
┌─────────────────────────────────────┐
│ Host Disk (2000 IOPS - SSD)         │
│                                     │
│ Container A (DB):                   │
│ - blkio-weight: 1000 (높음)          │
│ - Read: 100MB/s limit               │
│ - Write: 50MB/s limit               │
│ - Latency: 5ms                      │
│                                     │
│ Container B (Log):                  │
│ - blkio-weight: 100 (낮음)           │
│ - tmpfs 사용 (메모리)                  │
│ - Latency: 0.1ms                    │
└─────────────────────────────────────┘

결과:
✅ 격리된 I/O (QoS)
✅ DB 성능 20배 향상
✅ 로그 속도 1000배 향상
✅ 예측 가능한 성능

I/O 성능 핵심 개념:

1. I/O 유형:
   
   Sequential (순차):
   ┌──────────────────────────────┐
   │ Block: 0 → 1 → 2 → 3 → 4     │
   │ 빠름: 500MB/s (SSD)           │
   │ 예: 로그, 백업, 스트리밍          │
   └──────────────────────────────┘
   
   Random (무작위):
   ┌──────────────────────────────┐
   │ Block: 5 → 2 → 9 → 1 → 7     │
   │ 느림: 50MB/s (SSD)            │
   │ 예: DB, 인덱스, 메타데이터        │
   └──────────────────────────────┘

2. Storage Driver 비교:
   
   overlay2 (권장):
   ┌──────────────────────────────┐
   │ Layer 3: Container Layer     │
   │ Layer 2: App Layer           │
   │ Layer 1: Base Layer          │
   │ Layer 0: Kernel              │
   │                              │
   │ CoW (Copy-on-Write):         │
   │ - 읽기: 빠름 (직접)             │
   │ - 쓰기: 첫 번째만 느림           │
   └──────────────────────────────┘
   
   devicemapper:
   ┌──────────────────────────────┐
   │ Thin Pool                    │
   │ - 블록 레벨 CoW                │
   │ - 느림 (추가 레이어)             │
   │ - 복잡함                       │
   └──────────────────────────────┘

3. Volume 성능:
   
   tmpfs (메모리):
   Speed: 10GB/s
   Latency: 0.1ms
   Persistence: ❌
   Use: 임시 파일, 캐시
   
   Named Volume (SSD):
   Speed: 500MB/s
   Latency: 5ms
   Persistence: ✅
   Use: 데이터베이스
   
   Bind Mount (HDD):
   Speed: 100MB/s
   Latency: 10ms
   Persistence: ✅
   Use: 백업, 아카이브

4. I/O Scheduler:
   
   noop (SSD):
   ┌──────────────────────────────┐
   │ Request → Direct → Device    │
   │ (스케줄링 없음)                 │
   └──────────────────────────────┘
   
   deadline (균형):
   ┌──────────────────────────────┐
   │ Request → Sort → Deadline    │
   │ → Device                     │
   │ (응답시간 보장)                 │
   └──────────────────────────────┘
   
   cfq (HDD):
   ┌──────────────────────────────┐
   │ Request → Queue → Fairness   │
   │ → Device                     │
   │ (공정한 분배)                   │
   └──────────────────────────────┘

실무 시나리오:

시나리오 1 - DB 성능 저하:
┌─────────────────────────────────────┐
│ 09:00 - 정상                         │
│ DB Latency: 5ms                     │
│ Throughput: 10,000 TPS              │
└────────────┬────────────────────────┘
             │ (로그 컨테이너 시작)
┌────────────▼────────────────────────┐
│ 10:00 - 성능 저하                     │
│ DB Latency: 50ms (10배)             │
│ Throughput: 1,000 TPS (1/10)        │
│                                     │
│ 원인: I/O 경쟁                        │
│ - DB: Random Write 1000 IOPS        │
│ - Log: Sequential Write 500 IOPS    │
│ - Disk: 1500 IOPS 한계               │
└─────────────────────────────────────┘

해결:
- DB: blkio-weight 1000
- Log: tmpfs (메모리), 비동기 flush
- 결과: DB 정상화

시나리오 2 - 로그 유실:
┌─────────────────────────────────────┐
│ 고속 로깅 (10,000 logs/s)             │
│ Disk: 100MB/s limit                 │
│                                     │
│ Without tmpfs:                      │
│ → Disk Full → 로그 유실               │
│ → 병목                               │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ With tmpfs:                         │
│ → 메모리 버퍼 (10GB)                   │
│ → 비동기 Disk flush                   │
│ → 유실 없음                           │
└─────────────────────────────────────┘

시나리오 3 - Storage Driver 선택:
┌─────────────────────────────────────┐
│ overlay2:                           │
│ - Read: 500MB/s                     │
│ - Write: 400MB/s                    │
│ - Build: 2분                        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ devicemapper (loop-lvm):            │
│ - Read: 200MB/s (2.5배 느림)          │
│ - Write: 100MB/s (4배 느림)           │
│ - Build: 10분 (5배 느림)              │
└─────────────────────────────────────┘

권장: overlay2 (Linux 4.0+)
```

---

## 🔧 실습 1: Block I/O 제어

### Step 1: blkio-weight (상대적 우선순위)

```bash
# 디스크 확인
lsblk
# NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
# sda      8:0    0  100G  0 disk
# ├─sda1   8:1    0   99G  0 part /
# └─sda2   8:2    0    1G  0 part [SWAP]

# 높은 우선순위 (DB)
docker run -d --name db-high-io \
  --blkio-weight=1000 \
  postgres:15-alpine

# 낮은 우선순위 (로그)
docker run -d --name log-low-io \
  --blkio-weight=100 \
  alpine sh -c 'while true; do dd if=/dev/zero of=/tmp/test bs=1M count=100; done'

# I/O 경쟁 시 비율:
# db-high-io: 90.9% (1000/1100)
# log-low-io: 9.1% (100/1100)

# 확인 (iotop)
sudo iotop -o

# 출력:
# TID  DISK READ  DISK WRITE  COMMAND
# 123  0.00 B/s   90.00 M/s   postgres (db-high-io)
# 456  0.00 B/s   10.00 M/s   dd (log-low-io)

docker rm -f db-high-io log-low-io
```

### Step 2: I/O Throttling (절대적 제한)

```bash
# Device 확인
df / | tail -1 | awk '{print $1}'
# /dev/sda1

# Read/Write 제한
docker run -d --name throttled \
  --device-read-bps /dev/sda:10mb \
  --device-write-bps /dev/sda:5mb \
  alpine sh -c 'while true; do \
    dd if=/dev/zero of=/tmp/write bs=1M count=100; \
    dd if=/tmp/write of=/dev/null bs=1M; \
  done'

# 테스트
docker exec throttled sh -c 'dd if=/dev/zero of=/tmp/test bs=1M count=100 oflag=direct'

# 출력:
# 100+0 records in
# 100+0 records out
# 104857600 bytes (105 MB) copied, 20 s, 5.2 MB/s
#                                            ↑ 5MB/s 제한 적용

# IOPS 제한
docker run -d --name iops-limited \
  --device-read-iops /dev/sda:100 \
  --device-write-iops /dev/sda:50 \
  postgres:15-alpine

docker rm -f throttled iops-limited
```

### Step 3: I/O 모니터링

```bash
# iostat으로 I/O 확인
iostat -x 1

# 출력:
# Device  r/s   w/s   rMB/s  wMB/s  %util
# sda     120   80    12.0   8.0    45.2

# 컨테이너별 I/O 통계
docker stats --format "table {{.Name}}\t{{.BlockIO}}"

# cgroup 직접 확인
CID=$(docker inspect CONTAINER --format '{{.Id}}')
cat /sys/fs/cgroup/blkio/docker/$CID/blkio.throttle.io_service_bytes

# 출력:
# 8:0 Read 104857600
# 8:0 Write 52428800
#     ↑ Device  ↑ Operation ↑ Bytes
```

---

## 🔧 실습 2: Storage Driver 최적화

### Step 1: Storage Driver 확인

```bash
# 현재 driver 확인
docker info | grep "Storage Driver"
# Storage Driver: overlay2

# 상세 정보
docker info | grep -A 10 "Storage Driver"

# 출력:
# Storage Driver: overlay2
#  Backing Filesystem: extfs
#  Supports d_type: true
#  Native Overlay Diff: true
#  userxattr: false

# Driver별 성능 비교
cat /var/lib/docker/
# overlay2/  (권장)
# devicemapper/  (레거시)
# aufs/  (구형)
```

### Step 2: overlay2 최적화

```bash
# daemon.json 설정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

sudo systemctl restart docker

# 파일시스템 확인
df -T /var/lib/docker
# Filesystem     Type  Size  Used Avail Use% Mounted on
# /dev/sda1      ext4  100G   20G   75G  21% /

# xfs 권장 (더 빠름)
# mkfs.xfs /dev/sdb
# mount /dev/sdb /var/lib/docker
```

### Step 3: Image Layer 최적화

```bash
# 많은 레이어 (느림)
cat > Dockerfile.bad <<'EOF'
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get install -y git
RUN apt-get clean
# 6 layers
EOF

# 최적화 (빠름)
cat > Dockerfile.good <<'EOF'
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y \
      curl \
      vim \
      git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
# 2 layers
EOF

# 빌드 시간 비교
time docker build -f Dockerfile.bad -t test-bad .
# real: 2m30s

time docker build -f Dockerfile.good -t test-good .
# real: 1m10s (2배 빠름)

# I/O 비교
# bad: 많은 레이어 → 많은 I/O
# good: 적은 레이어 → 적은 I/O
```

---

## 🔧 실습 3: Volume 성능 비교

### Step 1: tmpfs (메모리)

```bash
# tmpfs 마운트
docker run -d --name tmpfs-test \
  --tmpfs /tmp:rw,size=1g,mode=1777 \
  alpine sh -c 'while true; do \
    dd if=/dev/zero of=/tmp/test bs=1M count=1000; \
  done'

# 성능 측정
docker exec tmpfs-test sh -c 'dd if=/dev/zero of=/tmp/bench bs=1M count=1000 oflag=direct'

# 출력:
# 1000+0 records in
# 1000+0 records out
# 1048576000 bytes (1.0 GB) copied, 0.1 s, 10.5 GB/s
#                                               ↑ 매우 빠름 (메모리)

docker rm -f tmpfs-test
```

### Step 2: Named Volume (SSD)

```bash
# Named volume 생성
docker volume create --driver local \
  --opt type=none \
  --opt device=/mnt/ssd \
  --opt o=bind \
  ssd-volume

# 사용
docker run -d --name volume-test \
  -v ssd-volume:/data \
  alpine sh -c 'while true; do \
    dd if=/dev/zero of=/data/test bs=1M count=1000; \
  done'

# 성능 측정
docker exec volume-test sh -c 'dd if=/dev/zero of=/data/bench bs=1M count=1000 oflag=direct'

# 출력:
# 1048576000 bytes (1.0 GB) copied, 2 s, 524 MB/s
#                                           ↑ SSD 속도

docker rm -f volume-test
```

### Step 3: Bind Mount (HDD)

```bash
# Bind mount
mkdir -p /mnt/hdd/data

docker run -d --name bind-test \
  -v /mnt/hdd/data:/data \
  alpine sh -c 'while true; do \
    dd if=/dev/zero of=/data/test bs=1M count=1000; \
  done'

# 성능 측정
docker exec bind-test sh -c 'dd if=/dev/zero of=/data/bench bs=1M count=1000 oflag=direct'

# 출력:
# 1048576000 bytes (1.0 GB) copied, 10 s, 105 MB/s
#                                            ↑ HDD 속도

docker rm -f bind-test
```

### Step 4: 성능 비교 요약

```bash
# 벤치마크 스크립트
cat > bench-volumes.sh <<'EOF'
#!/bin/bash

echo "Volume Type,Write Speed,Read Speed,Latency"

# tmpfs
docker run --rm --tmpfs /tmp:rw,size=1g \
  alpine sh -c 'dd if=/dev/zero of=/tmp/bench bs=1M count=1000 oflag=direct 2>&1' | \
  grep -oP '\d+ MB/s' | \
  awk '{print "tmpfs," $0}'

# SSD (named volume)
docker volume create ssd-bench
docker run --rm -v ssd-bench:/data \
  alpine sh -c 'dd if=/dev/zero of=/data/bench bs=1M count=1000 oflag=direct 2>&1' | \
  grep -oP '\d+ MB/s' | \
  awk '{print "SSD," $0}'
docker volume rm ssd-bench

# HDD (bind mount)
mkdir -p /tmp/hdd-bench
docker run --rm -v /tmp/hdd-bench:/data \
  alpine sh -c 'dd if=/dev/zero of=/data/bench bs=1M count=1000 oflag=direct 2>&1' | \
  grep -oP '\d+ MB/s' | \
  awk '{print "HDD," $0}'
rm -rf /tmp/hdd-bench
EOF

chmod +x bench-volumes.sh
./bench-volumes.sh

# 결과:
# tmpfs: 10,000 MB/s
# SSD:   500 MB/s
# HDD:   100 MB/s
```

---

## 🔧 실습 4: I/O 벤치마킹

### Step 1: fio (Flexible I/O Tester)

```bash
# fio 설치
docker run --rm -v $(pwd):/workdir \
  ljishen/fio \
  fio --version

# Random Read 벤치마크
docker run --rm -v /var/lib/docker:/data \
  ljishen/fio \
  fio --name=random-read \
      --directory=/data \
      --rw=randread \
      --bs=4k \
      --size=1G \
      --numjobs=4 \
      --runtime=60 \
      --time_based \
      --ioengine=libaio \
      --direct=1

# 출력:
# random-read: (g=0): rw=randread, bs=4K-4K/4K-4K/4K-4K, ioengine=libaio, iodepth=1
# ...
# read: IOPS=12.5k, BW=48.8MiB/s (51.2MB/s)(2932MiB/60001msec)
#       ↑ IOPS      ↑ Bandwidth

# Random Write 벤치마크
docker run --rm -v /var/lib/docker:/data \
  ljishen/fio \
  fio --name=random-write \
      --directory=/data \
      --rw=randwrite \
      --bs=4k \
      --size=1G \
      --numjobs=4 \
      --runtime=60 \
      --time_based \
      --ioengine=libaio \
      --direct=1

# Sequential Write
docker run --rm -v /var/lib/docker:/data \
  ljishen/fio \
  fio --name=seq-write \
      --directory=/data \
      --rw=write \
      --bs=1m \
      --size=2G \
      --numjobs=1 \
      --ioengine=libaio \
      --direct=1
```

### Step 2: sysbench

```bash
# sysbench 파일 I/O 테스트
docker run --rm -v /var/lib/docker:/data \
  severalnines/sysbench \
  sysbench fileio --file-total-size=2G prepare

docker run --rm -v /var/lib/docker:/data \
  severalnines/sysbench \
  sysbench fileio \
    --file-total-size=2G \
    --file-test-mode=rndrw \
    --time=60 \
    --max-requests=0 \
    run

# 출력:
# File operations:
#     reads/s:      1234.56
#     writes/s:     823.45
#     fsyncs/s:     2639.87
# Throughput:
#     read, MiB/s:  19.29
#     written, MiB/s: 12.86

# 정리
docker run --rm -v /var/lib/docker:/data \
  severalnines/sysbench \
  sysbench fileio --file-total-size=2G cleanup
```

### Step 3: dd (간단한 테스트)

```bash
# Sequential Write
docker run --rm -v /var/lib/docker:/data \
  alpine dd if=/dev/zero of=/data/test bs=1M count=1000 oflag=direct

# 출력:
# 1048576000 bytes (1.0 GB) copied, 2.5 s, 419 MB/s

# Sequential Read
docker run --rm -v /var/lib/docker:/data \
  alpine dd if=/data/test of=/dev/null bs=1M iflag=direct

# Random I/O (fio 사용 권장)
```

---

## 🔧 실습 5: 파일시스템 튜닝

### Step 1: ext4 최적화

```bash
# 현재 마운트 옵션 확인
mount | grep " / "

# 출력:
# /dev/sda1 on / type ext4 (rw,relatime,errors=remount-ro)

# 최적화 옵션
# - noatime: Access time 업데이트 안 함 (빠름)
# - nodiratime: Directory access time 업데이트 안 함
# - commit=60: 60초마다 fsync (기본 5초)

# /etc/fstab 수정
sudo vim /etc/fstab

# 수정:
# /dev/sda1  /  ext4  defaults,noatime,nodiratime,commit=60  0  1

# 재마운트
sudo mount -o remount /

# 확인
mount | grep " / "
# /dev/sda1 on / type ext4 (rw,noatime,nodiratime,commit=60)
```

### Step 2: I/O Scheduler 변경

```bash
# 현재 scheduler 확인
cat /sys/block/sda/queue/scheduler
# noop deadline [cfq]
#                ↑ 현재

# SSD: noop 또는 deadline 권장
echo noop | sudo tee /sys/block/sda/queue/scheduler

# HDD: deadline 또는 cfq 권장
echo deadline | sudo tee /sys/block/sda/queue/scheduler

# 영구 설정 (GRUB)
sudo vim /etc/default/grub

# 추가:
# GRUB_CMDLINE_LINUX="elevator=noop"

sudo update-grub
sudo reboot
```

### Step 3: Read-ahead 조정

```bash
# 현재 read-ahead 확인
sudo blockdev --getra /dev/sda
# 256 (기본값: 128KB)

# Sequential I/O가 많은 경우: 증가
sudo blockdev --setra 2048 /dev/sda  # 1MB

# Random I/O가 많은 경우: 감소
sudo blockdev --setra 128 /dev/sda   # 64KB

# 영구 설정
echo 'ACTION=="add|change", KERNEL=="sda", ATTR{bdi/read_ahead_kb}="1024"' | \
  sudo tee /etc/udev/rules.d/60-readahead.rules
```

---

## 💡 주요 명령어 정리

### Block I/O 제한

```bash
# Weight (상대적)
docker run --blkio-weight=500 IMAGE
docker run --blkio-weight-device=/dev/sda:500 IMAGE

# Throttling (절대적)
docker run --device-read-bps /dev/sda:10mb IMAGE
docker run --device-write-bps /dev/sda:5mb IMAGE
docker run --device-read-iops /dev/sda:100 IMAGE
docker run --device-write-iops /dev/sda:50 IMAGE
```

### Volume

```bash
# tmpfs (메모리)
docker run --tmpfs /tmp:rw,size=1g IMAGE

# Named volume
docker volume create VOLUME
docker run -v VOLUME:/data IMAGE

# Bind mount
docker run -v /host/path:/container/path IMAGE
```

### 벤치마킹

```bash
# fio
fio --name=test --rw=randread --bs=4k --size=1G

# sysbench
sysbench fileio --file-test-mode=rndrw run

# dd
dd if=/dev/zero of=test bs=1M count=1000 oflag=direct
```

---

## 🎓 연습 문제

### 문제 1: I/O 성능 진단

다음 상황에서 병목을 찾으세요:

```bash
iostat -x 1
# Device  r/s  w/s  rMB/s  wMB/s  %util
# sda     10   500  1.0    50.0   95.2
```

<details>
<summary>정답 보기</summary>

**병목: Disk Write**

분석:
- Write: 500 IOPS, 50MB/s
- Read: 10 IOPS, 1MB/s
- %util: 95.2% (거의 포화)

해결:
```bash
# 1. Write 많은 컨테이너 확인
docker stats --format "table {{.Name}}\t{{.BlockIO}}"

# 2. 로그 컨테이너라면 tmpfs 사용
docker run --tmpfs /var/log:rw,size=2g IMAGE

# 3. DB라면 SSD 사용 + Write cache
# 4. blkio-weight로 우선순위 조정
```

</details>

### 문제 2: Storage Driver 선택

새 서버에 Docker 설치 시 Storage Driver를 선택하세요.

<details>
<summary>정답 보기</summary>

**권장: overlay2**

이유:
```bash
# 1. 성능
# overlay2: Read 500MB/s, Write 400MB/s
# devicemapper: Read 200MB/s, Write 100MB/s

# 2. 안정성
# - Kernel 4.0+ 공식 지원
# - 성숙한 구현

# 3. 설정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "storage-driver": "overlay2"
}
EOF

# 4. 파일시스템
# ext4 또는 xfs (xfs 권장)
```

</details>

### 문제 3: 최적 Volume 선택

다음 각 경우에 적합한 Volume 타입을 선택하세요:

- 임시 빌드 디렉토리
- PostgreSQL 데이터
- 로그 파일

<details>
<summary>정답 보기</summary>

```bash
# 1. 임시 빌드 디렉토리: tmpfs
docker run --tmpfs /tmp/build:rw,size=4g IMAGE
# - 빠름 (메모리)
# - 재사용 불필요

# 2. PostgreSQL 데이터: Named Volume (SSD)
docker volume create --driver local \
  --opt type=none \
  --opt device=/mnt/ssd \
  --opt o=bind \
  pg-data

docker run -v pg-data:/var/lib/postgresql/data postgres
# - 영구 저장
# - 고성능 필요

# 3. 로그 파일: tmpfs + 비동기 flush
docker run --tmpfs /var/log:rw,size=2g IMAGE
# - 고속 쓰기
# - 비동기로 disk 저장
```

</details>

---

## 📌 핵심 요약

### I/O 성능 비교

| 타입 | Sequential | Random | Latency | 용도 |
|-----|------------|--------|---------|------|
| **tmpfs** | 10GB/s | 10GB/s | 0.1ms | 임시 파일 |
| **SSD** | 500MB/s | 100MB/s | 5ms | 데이터베이스 |
| **HDD** | 100MB/s | 1MB/s | 10ms | 아카이브 |

### Storage Driver 비교

```
overlay2 (권장):
✅ 빠름 (Native)
✅ 안정적
✅ 간단한 설정
❌ ext4/xfs 필요

devicemapper:
❌ 느림 (Block-level CoW)
❌ 복잡한 설정
❌ 유지보수 어려움
✅ 모든 FS 지원
```

### Volume 선택 가이드

```yaml
임시 데이터:
  type: tmpfs
  size: 적당히
  performance: 최고

데이터베이스:
  type: named_volume
  device: SSD
  performance: 높음

로그:
  type: tmpfs
  async_flush: true
  performance: 높음

백업:
  type: bind_mount
  device: HDD
  performance: 낮음 (OK)
```

### I/O 최적화 체크리스트

- [ ] Storage Driver: overlay2 사용
- [ ] 파일시스템: xfs (또는 ext4)
- [ ] 마운트 옵션: noatime, nodiratime
- [ ] I/O Scheduler: noop (SSD) 또는 deadline
- [ ] Read-ahead: 워크로드에 맞게 조정
- [ ] tmpfs: 로그, 임시 파일에 사용
- [ ] blkio: 우선순위 조정

---

## 📚 참고 자료

- [Docker Storage Drivers](https://docs.docker.com/storage/storagedriver/)
- [Linux I/O Schedulers](https://www.kernel.org/doc/Documentation/block/switching-sched.txt)
- [fio Documentation](https://fio.readthedocs.io/)
- [ext4 vs XFS Performance](https://www.phoronix.com/scan.php?page=article&item=linux-58-filesystems)

---

## 🤔 생각해볼 문제

1. Sequential I/O와 Random I/O의 성능 차이가 나는 이유는?
2. tmpfs를 사용하면 왜 빠를까?
3. Storage Driver가 성능에 영향을 미치는 이유는?

> 💡 **답변**: 1) Disk의 물리적 특성 때문, Sequential: 디스크 헤드가 연속된 위치 읽기 → Seek time 최소, HDD: 7200 RPM → 120번 seek/s 가능, Sequential은 1번 seek로 많은 데이터, Random: 매번 Seek 필요 → Seek time 누적, 예: 1000번 Random Read = 1000번 Seek = 8.3ms × 1000 = 8.3초, SSD는 Seek 없지만 Random도 느림 (내부 매핑, Wear leveling), 2) tmpfs는 RAM 사용, 메모리 속도: 10-20GB/s, Disk 속도: 100-500MB/s, 100-200배 빠름, Latency: RAM 100ns vs SSD 100,000ns (1000배), CPU가 직접 접근 → 디스크 I/O 대기 없음, 단점: 휘발성 (재부팅 시 소실), 메모리 소비, 3) Storage Driver는 파일시스템 위에 레이어 구현, overlay2: Union mount (빠름), 파일 레벨 CoW, 상위 레이어만 쓰기, devicemapper: Block 레벨 CoW (느림), Thin provisioning overhead, 복잡한 메타데이터, 예: 파일 수정 시, overlay2: 파일 복사 후 수정, devicemapper: 블록 복사 → 추가 I/O, overlay2가 2-4배 빠름

---

<div align="center">

**[⬅️ 이전: Memory Management](./03-Memory-Management.md)** | **[다음: Monitoring ➡️](./05-Monitoring.md)**

</div>
