# 01. Container Runtime - 컨테이너 런타임 심화

## 🎯 이 챕터에서 배울 것

- **OCI Runtime Specification** 구조와 핵심 개념
- **runc, crun, kata-containers** 등 런타임 비교
- **config.json**과 **rootfs**로 컨테이너 직접 생성
- Linux **Namespace**와 **Cgroup**이 런타임에서 동작하는 방식
- Docker 명령어 없이 **Low-level 컨테이너 실행**

## 📌 왜 중요한가?

**"OCI Runtime Spec을 이해하면 모든 컨테이너 런타임의 동작을 파악할 수 있습니다."**

```
Docker CLI에서 컨테이너가 실행되기까지:

┌──────────────────────────────────────────────────────┐
│ User: docker run nginx                               │
└──────────────┬───────────────────────────────────────┘
               │
┌──────────────▼───────────────────────────────────────┐
│ Docker CLI (docker)                                  │
│ - 사용자 명령어 파싱                                     │
│ - Docker Daemon에 API 요청                            │
└──────────────┬───────────────────────────────────────┘
               │ REST API (Unix Socket)
┌──────────────▼───────────────────────────────────────┐
│ Docker Daemon (dockerd)                              │
│ - 이미지 관리, 네트워크, 볼륨                              │
│ - 컨테이너 생명주기 관리                                  │
└──────────────┬───────────────────────────────────────┘
               │ gRPC
┌──────────────▼───────────────────────────────────────┐
│ containerd                                           │
│ - 이미지 pull/push                                    │
│ - 컨테이너 실행 관리                                     │
│ - OCI 이미지 → OCI 번들 변환                             │
└──────────────┬───────────────────────────────────────┘
               │ OCI Bundle (config.json + rootfs)
┌──────────────▼───────────────────────────────────────┐
│ Container Runtime (runc)          ← 이 챕터의 핵심!     │
│ - OCI Runtime Spec 구현체                              │
│ - Namespace, Cgroup 설정                              │
│ - 실제 프로세스 생성                                     │
└──────────────┬───────────────────────────────────────┘
               │ clone() + execve()
┌──────────────▼───────────────────────────────────────┐
│ Container Process (PID 1)                            │
│ - 격리된 환경에서 실행                                    │
│ - 자체 namespace, cgroup                              │
└──────────────────────────────────────────────────────┘

핵심 포인트:
Docker = High-level Runtime (사용자 편의)
containerd = Mid-level Runtime (컨테이너 관리)
runc = Low-level Runtime (실제 컨테이너 생성)

왜 분리되어 있는가?
┌───────────────────┐     ┌───────────────────┐
│Before (Docker 1.x)│     │ After (현재)       │
│                   │     │                   │
│ dockerd           │     │ dockerd           │
│ ├── 이미지 관리      │     │ ├── 이미지/네트워크   │
│ ├── 네트워크        │     │                   │
│ ├── 볼륨           │     │ containerd        │
│ ├── 컨테이너 실행    │     │ ├── 이미지 관리      │
│ └── 모든 것        │     │ ├── 컨테이너 관리     │
│                   │     │                   │
│ ❌ 모놀리식         │     │ runc              │
│ ❌ dockerd 재시작  │     │ └── 컨테이너 생성     │
│    = 모든 컨테이너   │     │                   │
│    중단            │     │ ✅ 모듈화          │
└───────────────────┘     │ ✅ dockerd 재시작  │
                          │    = 컨테이너 유지   │
                          └───────────────────┘
```

**실무 영향:**
- 컨테이너 런타임 문제 진단 능력
- Kubernetes CRI 이해의 기초
- 보안 강화를 위한 런타임 선택
- Docker 없이 컨테이너 실행 가능

---

## 🔬 Deep Dive

### 1. OCI (Open Container Initiative) 개요

#### OCI란?

```
OCI의 탄생 배경:

2013: Docker 등장
      ↓
2014: Docker 사실상 표준
      ↓
2015: 컨테이너 표준화 필요성 대두
      ↓
┌─────────────────────────────────────────┐
│ OCI (Open Container Initiative) 설립     │
│                                         │
│ 목표: 컨테이너 포맷과 런타임의 표준화            │
│                                         │
│ 3가지 표준:                               │
│ ┌─────────────────────────────────────┐ │
│ │ 1. Runtime Spec                     │ │
│ │    - 컨테이너 실행 방법 정의              │ │
│ │    - config.json + rootfs           │ │
│ │    - 구현체: runc, crun, youki        │ │
│ └─────────────────────────────────────┘ │
│ ┌─────────────────────────────────────┐ │
│ │ 2. Image Spec                       │ │
│ │    - 컨테이너 이미지 포맷 정의            │ │
│ │    - manifest, config, layers       │ │
│ └─────────────────────────────────────┘ │
│ ┌─────────────────────────────────────┐ │
│ │ 3. Distribution Spec                │ │
│ │    - 이미지 배포 방법 정의               │ │
│ │    - Registry API                   │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘

OCI의 가치:
- 벤더 종속 방지 (Docker 독점 X)
- 런타임 교체 가능 (runc ↔ crun)
- 이미지 호환성 (Docker ↔ Podman)
- 생태계 확장 (Kubernetes CRI 연동)
```

#### 주요 런타임 비교

```
Low-level Container Runtime 비교:

┌─────────┬────────────┬──────────┬─────────────┬──────────────┐
│ 런타임    │ 언어        │ 특징      │ 보안         │ 사용 사례      │
├─────────┼────────────┼──────────┼─────────────┼──────────────┤
│ runc    │ Go         │ 표준 구현  │ 프로세스 격리   │ Docker 기본   │
├─────────┼────────────┼──────────┼─────────────┼──────────────┤
│ crun    │ C          │ 경량/빠름  │ 프로세스 격리   │ Podman 기본   │
├─────────┼────────────┼──────────┼─────────────┼──────────────┤
│ youki   │ Rust       │ 메모리 안전│ 프로세스 격리   │ 실험적        │
├─────────┼────────────┼──────────┼─────────────┼──────────────┤
│ kata    │ Go+QEMU    │ VM 격리   │ 하드웨어 격리   │ 멀티테넌트     │
├─────────┼────────────┼──────────┼─────────────┼──────────────┤
│ gVisor  │ Go         │ 커널 에뮬  │ 커널 격리      │ 클라우드       │
└─────────┴────────────┴──────────┴─────────────┴──────────────┘

격리 수준 비교:
                     보안 ↑
                          │
  kata-containers ────────┤ VM 수준 격리
                          │ (별도 커널)
                          │
  gVisor ─────────────────┤ 커널 에뮬레이션
                          │ (사용자 공간 커널)
                          │
  runc / crun ────────────┤ Namespace/Cgroup
                          │ (커널 공유)
                          │
                    성능 → │
```

---

### 2. OCI Runtime Spec 상세

#### OCI Bundle 구조

```
OCI Bundle:
컨테이너를 실행하기 위한 최소 파일 구조

my-container/                    ← OCI Bundle
├── config.json                  ← 런타임 설정 (필수)
│   ├── ociVersion               ← OCI Spec 버전
│   ├── process                  ← 실행할 프로세스
│   │   ├── terminal             ← TTY 여부
│   │   ├── user                 ← UID/GID
│   │   ├── args                 ← 실행 명령어
│   │   ├── env                  ← 환경 변수
│   │   └── cwd                  ← 작업 디렉토리
│   ├── root                     ← rootfs 경로
│   │   ├── path                 ← "rootfs"
│   │   └── readonly             ← 읽기 전용 여부
│   ├── hostname                 ← 호스트명
│   ├── mounts                   ← 마운트 포인트
│   ├── linux                    ← Linux 전용 설정
│   │   ├── namespaces           ← Namespace 목록
│   │   ├── resources            ← Cgroup 리소스
│   │   ├── seccomp              ← Seccomp 프로파일
│   │   └── maskedPaths          ← 숨길 경로
│   └── hooks                    ← 생명주기 훅
│       ├── prestart             ← 시작 전
│       ├── createRuntime        ← 런타임 생성 시
│       ├── poststart            ← 시작 후
│       └── poststop             ← 종료 후
│
└── rootfs/                      ← 루트 파일시스템 (필수)
    ├── bin/
    ├── etc/
    ├── lib/
    ├── proc/                    ← (마운트될 경로)
    ├── sys/                     ← (마운트될 경로)
    ├── tmp/
    ├── usr/
    └── var/
```

#### Container Lifecycle (컨테이너 생명주기)

```
OCI Runtime Spec이 정의하는 컨테이너 상태:

┌──────────┐  create  ┌──────────┐  start  ┌──────────┐
│ Creating │─────────→│ Created  │────────→│ Running  │
└──────────┘          └──────────┘         └─────┬────┘
                                                 │
                           kill/exit             │
                      ┌──────────┐               │
                      │ Stopped  │←──────────────┘
                      └──────────┘

각 상태 설명:
┌────────────┬────────────────────────────────────────┐
│ 상태        │ 설명                                    │
├────────────┼────────────────────────────────────────┤
│ Creating   │ 컨테이너 환경 구성 중                       │
│            │ - Namespace 생성                        │
│            │ - Cgroup 설정                           │
│            │ - rootfs 마운트                          │
├────────────┼────────────────────────────────────────┤
│ Created    │ 환경 구성 완료, 프로세스 미실행               │
│            │ - 훅(prestart) 실행 대기                  │
│            │ - 사용자 프로세스 아직 시작 안 됨             │
├────────────┼────────────────────────────────────────┤
│ Running    │ 사용자 프로세스 실행 중                     │
│            │ - PID 1이 동작 중                        │
│            │ - 훅(poststart) 실행됨                   │
├────────────┼────────────────────────────────────────┤
│ Stopped    │ 프로세스 종료                             │
│            │ - 정상 종료 또는 시그널                     │
│            │ - 훅(poststop) 실행됨                    │
│            │ - 리소스 정리 대기                         │
└────────────┴────────────────────────────────────────┘

런타임 명령어와 상태 전이:
runc create  → Creating → Created
runc start   → Created  → Running
runc kill    → Running  → Stopped
runc delete  → Stopped  → (제거)

docker run = create + start 를 한 번에 수행
```

---

### 3. config.json 핵심 필드

#### 전체 구조

```json
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": true,
    "user": {
      "uid": 0,
      "gid": 0
    },
    "args": [
      "/bin/sh"
    ],
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "TERM=xterm"
    ],
    "cwd": "/",
    "capabilities": {
      "bounding": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
      "effective": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
      "permitted": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"]
    },
    "rlimits": [
      {
        "type": "RLIMIT_NOFILE",
        "hard": 1024,
        "soft": 1024
      }
    ]
  },
  "root": {
    "path": "rootfs",
    "readonly": false
  },
  "hostname": "my-container",
  "mounts": [
    {
      "destination": "/proc",
      "type": "proc",
      "source": "proc"
    },
    {
      "destination": "/dev",
      "type": "tmpfs",
      "source": "tmpfs",
      "options": ["nosuid", "strictatime", "mode=755", "size=65536k"]
    },
    {
      "destination": "/sys",
      "type": "sysfs",
      "source": "sysfs",
      "options": ["nosuid", "noexec", "nodev", "ro"]
    }
  ],
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "network"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"},
      {"type": "cgroup"}
    ],
    "resources": {
      "memory": {
        "limit": 536870912
      },
      "cpu": {
        "shares": 1024,
        "quota": 100000,
        "period": 100000
      }
    },
    "maskedPaths": [
      "/proc/kcore",
      "/proc/keys",
      "/proc/timer_list"
    ],
    "readonlyPaths": [
      "/proc/asound",
      "/proc/bus",
      "/proc/fs",
      "/proc/sysrq-trigger"
    ]
  }
}
```

#### 각 필드 설명

```
config.json 핵심 필드 맵:

┌─ process (실행할 프로세스)
│  ├── args: ["/bin/sh"]         ← docker CMD에 해당
│  ├── env: ["PATH=..."]        ← docker ENV에 해당
│  ├── cwd: "/"                 ← docker WORKDIR에 해당
│  ├── user: {uid: 0, gid: 0}  ← docker USER에 해당
│  ├── capabilities             ← --cap-add/--cap-drop
│  └── rlimits                  ← --ulimit
│
├─ root (루트 파일시스템)
│  ├── path: "rootfs"           ← 이미지 레이어 합친 것
│  └── readonly: false          ← --read-only
│
├─ hostname                     ← --hostname
│
├─ mounts (마운트 포인트)
│  ├── /proc (proc)             ← 프로세스 정보
│  ├── /dev (tmpfs)             ← 디바이스 파일
│  ├── /sys (sysfs)             ← 시스템 정보
│  └── 사용자 정의 마운트          ← -v 옵션에 해당
│
└─ linux (Linux 전용)
   ├── namespaces               ← 격리 설정
   │   ├── pid                  ← 프로세스 격리
   │   ├── network              ← 네트워크 격리
   │   ├── ipc                  ← IPC 격리
   │   ├── uts                  ← 호스트명 격리
   │   ├── mount                ← 파일시스템 격리
   │   └── cgroup               ← Cgroup 격리
   ├── resources                ← 리소스 제한
   │   ├── memory.limit         ← --memory
   │   └── cpu.shares/quota     ← --cpus, --cpu-shares
   ├── maskedPaths              ← 숨길 /proc 경로
   └── readonlyPaths            ← 읽기 전용 /proc 경로

Docker 옵션 → config.json 매핑:
docker run --memory=512m       → linux.resources.memory.limit
docker run --cpus=2            → linux.resources.cpu.quota
docker run --hostname=myhost   → hostname
docker run --read-only         → root.readonly
docker run --cap-drop=ALL      → process.capabilities (비움)
docker run --user=1000:1000    → process.user
```

---

### 4. Namespace와 런타임

#### 6가지 Namespace

```
컨테이너 런타임이 생성하는 Namespace:

┌─────────────────────────────────────────────────────────┐
│ Host (기본 Namespace)                                    │
│                                                         │
│  ┌──── Container A ────────────────────────────────┐    │
│  │ PID NS:    PID 1 (실제: PID 3847)                │    │
│  │ NET NS:    eth0 (172.17.0.2)                    │    │
│  │ MNT NS:    / → rootfs (독립 파일시스템)             │    │
│  │ UTS NS:    hostname = "container-a"             │    │
│  │ IPC NS:    독립 공유 메모리/세마포어                  │    │
│  │ CGROUP NS: /docker/<container-id>               │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌──── Container B ────────────────────────────────┐    │
│  │ PID NS:    PID 1 (실제: PID 5123)                │    │
│  │ NET NS:    eth0 (172.17.0.3)                    │    │
│  │ MNT NS:    / → rootfs (독립 파일시스템)             │    │
│  │ UTS NS:    hostname = "container-b"             │    │
│  │ IPC NS:    독립 공유 메모리/세마포어                  │    │
│  │ CGROUP NS: /docker/<container-id>               │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│ 호스트에서 보면:                                            │
│ PID 3847 (container-a의 PID 1)                          │
│ PID 5123 (container-b의 PID 1)                          │
└─────────────────────────────────────────────────────────┘

Namespace 공유 옵션:
docker run --pid=host        → PID namespace 공유
docker run --network=host    → Network namespace 공유
docker run --ipc=host        → IPC namespace 공유
docker run --pid=container:X → 다른 컨테이너와 PID 공유
```

---

## 🔧 실습 1: runc로 컨테이너 직접 생성

### Step 1: runc 설치 및 확인

```bash
# runc 버전 확인 (Docker와 함께 설치됨)
runc --version
# runc version 1.1.x
# commit: ...
# spec: 1.0.2-dev
# go: go1.20.x

# Docker가 사용하는 runc 경로
which runc
# /usr/bin/runc (또는 /usr/sbin/runc)

# containerd가 사용하는 runc 확인
docker info | grep -i runtime
# Default Runtime: runc
# Runtimes: runc
```

### Step 2: OCI Bundle 생성

```bash
# 작업 디렉토리 생성
mkdir -p /tmp/my-container/rootfs
cd /tmp/my-container

# Docker 이미지에서 rootfs 추출
docker export $(docker create alpine:3.18) | tar -C rootfs -xf -

# rootfs 구조 확인
ls rootfs/
# bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

# OCI 기본 설정 생성
runc spec

# 생성된 파일 확인
ls
# config.json  rootfs

# config.json 내용 확인
cat config.json | python3 -m json.tool | head -30
```

### Step 3: 컨테이너 실행

```bash
# 컨테이너 생성 + 시작 (foreground)
cd /tmp/my-container
sudo runc run my-first-container

# 컨테이너 내부에서:
# / # hostname
# runc
# / # id
# uid=0(root) gid=0(root)
# / # ps aux
# PID   USER     TIME  COMMAND
#     1 root      0:00 /bin/sh
#     7 root      0:00 ps aux
# / # cat /etc/os-release
# NAME="Alpine Linux"
# / # exit

# 컨테이너가 종료됨
```

### Step 4: 생명주기 단계별 실행

```bash
cd /tmp/my-container

# 1. config.json에서 terminal을 false로 변경 (백그라운드 실행용)
cat > config.json << 'CONFIGEOF'
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": false,
    "user": {"uid": 0, "gid": 0},
    "args": ["sleep", "300"],
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "TERM=xterm"
    ],
    "cwd": "/"
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "hostname": "lifecycle-demo",
  "mounts": [
    {"destination": "/proc", "type": "proc", "source": "proc"},
    {"destination": "/dev", "type": "tmpfs", "source": "tmpfs",
     "options": ["nosuid", "strictatime", "mode=755", "size=65536k"]},
    {"destination": "/sys", "type": "sysfs", "source": "sysfs",
     "options": ["nosuid", "noexec", "nodev", "ro"]}
  ],
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"},
      {"type": "cgroup"}
    ]
  }
}
CONFIGEOF

# 2. CREATE: 컨테이너 생성 (프로세스 미실행)
sudo runc create lifecycle-demo

# 3. 상태 확인
sudo runc state lifecycle-demo
# {
#   "ociVersion": "1.0.2",
#   "id": "lifecycle-demo",
#   "pid": 12345,
#   "status": "created",    ← Created 상태
#   ...
# }

# 4. START: 프로세스 실행
sudo runc start lifecycle-demo

# 5. 상태 확인
sudo runc state lifecycle-demo
# {
#   ...
#   "status": "running",    ← Running 상태
#   ...
# }

# 6. 컨테이너 목록
sudo runc list
# ID               PID     STATUS   BUNDLE                  CREATED
# lifecycle-demo   12345   running  /tmp/my-container       2024-...

# 7. KILL: 컨테이너 종료
sudo runc kill lifecycle-demo SIGTERM

# 8. 상태 확인
sudo runc state lifecycle-demo
# {
#   ...
#   "status": "stopped",    ← Stopped 상태
#   ...
# }

# 9. DELETE: 컨테이너 삭제
sudo runc delete lifecycle-demo

# 10. 확인
sudo runc list
# (빈 목록)
```

---

## 🔧 실습 2: config.json 커스터마이징

### Step 1: 리소스 제한이 적용된 컨테이너

```bash
mkdir -p /tmp/resource-container/rootfs
cd /tmp/resource-container

# rootfs 준비
docker export $(docker create alpine:3.18) | tar -C rootfs -xf -

# 리소스 제한이 포함된 config.json
cat > config.json << 'CONFIGEOF'
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": false,
    "user": {"uid": 0, "gid": 0},
    "args": ["sh", "-c", "echo 'Memory limit test' && cat /sys/fs/cgroup/memory.max 2>/dev/null || cat /sys/fs/cgroup/memory/memory.limit_in_bytes 2>/dev/null && echo 'CPU quota:' && cat /sys/fs/cgroup/cpu.max 2>/dev/null || cat /sys/fs/cgroup/cpu/cpu.cfs_quota_us 2>/dev/null && sleep 60"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "cwd": "/",
    "capabilities": {
      "bounding": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
      "effective": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
      "permitted": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"]
    }
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "hostname": "resource-limited",
  "mounts": [
    {"destination": "/proc", "type": "proc", "source": "proc"},
    {"destination": "/dev", "type": "tmpfs", "source": "tmpfs",
     "options": ["nosuid", "strictatime", "mode=755", "size=65536k"]},
    {"destination": "/sys", "type": "sysfs", "source": "sysfs",
     "options": ["nosuid", "noexec", "nodev"]}
  ],
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"},
      {"type": "cgroup"}
    ],
    "resources": {
      "memory": {
        "limit": 134217728
      },
      "cpu": {
        "shares": 512,
        "quota": 50000,
        "period": 100000
      },
      "pids": {
        "limit": 100
      }
    }
  }
}
CONFIGEOF

# 실행
sudo runc run resource-limited
# 출력:
# Memory limit test
# 134217728          ← 128MB (134217728 bytes)
# CPU quota:
# 50000 100000       ← CPU 50% (quota/period)

# 정리
cd /tmp && sudo rm -rf /tmp/resource-container
```

### Step 2: Docker 컨테이너의 실제 config.json 확인

```bash
# Docker로 컨테이너 실행
docker run -d --name inspect-demo \
  --memory=256m \
  --cpus=1.5 \
  --hostname=demo-host \
  --read-only \
  alpine sleep 300

# Docker가 생성한 OCI Bundle 위치 찾기
CONTAINER_ID=$(docker inspect inspect-demo --format '{{.Id}}')

# containerd의 Bundle 확인 (Docker + containerd 환경)
sudo ls /run/containerd/io.containerd.runtime.v2.task/moby/${CONTAINER_ID}/
# config.json  rootfs  ...

# Docker가 생성한 config.json 확인
sudo cat /run/containerd/io.containerd.runtime.v2.task/moby/${CONTAINER_ID}/config.json \
  | python3 -m json.tool | head -50

# 주요 필드 추출
echo "=== Memory Limit ==="
sudo cat /run/containerd/io.containerd.runtime.v2.task/moby/${CONTAINER_ID}/config.json \
  | python3 -c "import sys,json; c=json.load(sys.stdin); print(c['linux']['resources']['memory']['limit'])"
# 268435456 (256MB)

echo "=== CPU Quota ==="
sudo cat /run/containerd/io.containerd.runtime.v2.task/moby/${CONTAINER_ID}/config.json \
  | python3 -c "import sys,json; c=json.load(sys.stdin); print(c['linux']['resources']['cpu']['quota'])"
# 150000 (1.5 CPU)

echo "=== Namespaces ==="
sudo cat /run/containerd/io.containerd.runtime.v2.task/moby/${CONTAINER_ID}/config.json \
  | python3 -c "import sys,json; c=json.load(sys.stdin); [print(n['type']) for n in c['linux']['namespaces']]"
# pid
# network
# ipc
# uts
# mount
# cgroup

echo "=== Hostname ==="
sudo cat /run/containerd/io.containerd.runtime.v2.task/moby/${CONTAINER_ID}/config.json \
  | python3 -c "import sys,json; c=json.load(sys.stdin); print(c.get('hostname', 'N/A'))"
# demo-host

# 정리
docker rm -f inspect-demo
```

---

## 🔧 실습 3: Namespace 직접 확인

### Step 1: 컨테이너의 Namespace 탐색

```bash
# 컨테이너 실행
docker run -d --name ns-demo alpine sleep 300

# 컨테이너 PID 확인
CONTAINER_PID=$(docker inspect ns-demo --format '{{.State.Pid}}')
echo "Container PID: $CONTAINER_PID"

# 호스트에서 컨테이너의 Namespace 확인
sudo ls -la /proc/${CONTAINER_PID}/ns/
# lrwxrwxrwx 1 root root 0 ... cgroup -> 'cgroup:[4026532xxx]'
# lrwxrwxrwx 1 root root 0 ... ipc -> 'ipc:[4026532xxx]'
# lrwxrwxrwx 1 root root 0 ... mnt -> 'mnt:[4026532xxx]'
# lrwxrwxrwx 1 root root 0 ... net -> 'net:[4026532xxx]'
# lrwxrwxrwx 1 root root 0 ... pid -> 'pid:[4026532xxx]'
# lrwxrwxrwx 1 root root 0 ... uts -> 'uts:[4026532xxx]'

# 호스트의 Namespace와 비교
sudo ls -la /proc/1/ns/
# 번호가 다름 = 격리됨

# PID Namespace 비교
echo "Host PID NS: $(readlink /proc/1/ns/pid)"
echo "Container PID NS: $(sudo readlink /proc/${CONTAINER_PID}/ns/pid)"
# 다른 inode 번호 → 격리 확인

# nsenter로 컨테이너 Namespace 진입
sudo nsenter --target ${CONTAINER_PID} --mount --uts --ipc --net --pid -- /bin/sh

# 컨테이너 내부에서:
# / # hostname
# (컨테이너 ID)
# / # ps aux
# PID   USER     TIME  COMMAND
#     1 root      0:00 sleep 300
#     8 root      0:00 /bin/sh
# / # ip addr
# (컨테이너 네트워크)
# / # exit

# 정리
docker rm -f ns-demo
```

### Step 2: Namespace 공유 실험

```bash
# 컨테이너 A 시작
docker run -d --name container-a alpine sleep 300

# 컨테이너 B: A의 PID namespace 공유
docker run -d --name container-b --pid=container:container-a alpine sleep 300

# 컨테이너 B에서 A의 프로세스가 보이는지 확인
docker exec container-b ps aux
# PID   USER     TIME  COMMAND
#     1 root      0:00 sleep 300    ← container-a의 프로세스
#     8 root      0:00 sleep 300    ← container-b의 프로세스
#    15 root      0:00 ps aux

# 컨테이너 A에서도 B의 프로세스가 보임
docker exec container-a ps aux
# PID   USER     TIME  COMMAND
#     1 root      0:00 sleep 300    ← container-a
#     8 root      0:00 sleep 300    ← container-b
#    16 root      0:00 ps aux

# Network namespace는 여전히 분리됨
docker exec container-a hostname
# (container-a ID)
docker exec container-b hostname
# (container-b ID)

# 정리
docker rm -f container-a container-b
```

---

## 🔧 실습 4: 대체 런타임 사용

### Step 1: crun 설치 및 사용

```bash
# crun 설치 (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y crun

# crun 버전 확인
crun --version
# crun version x.x.x
# spec: 1.0.0

# crun으로 컨테이너 실행
mkdir -p /tmp/crun-test/rootfs
cd /tmp/crun-test
docker export $(docker create alpine:3.18) | tar -C rootfs -xf -
runc spec

# config.json의 terminal을 false로, args를 변경
python3 -c "
import json
with open('config.json') as f: c = json.load(f)
c['process']['terminal'] = False
c['process']['args'] = ['echo', 'Hello from crun!']
with open('config.json', 'w') as f: json.dump(c, f, indent=2)
"

# crun으로 실행
sudo crun run crun-test
# Hello from crun!

# 정리
cd /tmp && sudo rm -rf /tmp/crun-test
```

### Step 2: Docker에서 대체 런타임 설정

```bash
# Docker daemon에 crun 런타임 추가
sudo cat > /etc/docker/daemon.json << 'EOF'
{
  "runtimes": {
    "crun": {
      "path": "/usr/bin/crun"
    }
  }
}
EOF

# Docker 재시작
sudo systemctl restart docker

# 사용 가능한 런타임 확인
docker info | grep -A 5 Runtimes
# Runtimes: crun runc
# Default Runtime: runc

# crun으로 컨테이너 실행
docker run --rm --runtime=crun alpine echo "Running with crun!"
# Running with crun!

# 성능 비교: runc vs crun
echo "=== runc ==="
time docker run --rm --runtime=runc alpine echo "runc"

echo "=== crun ==="
time docker run --rm --runtime=crun alpine echo "crun"
# crun이 일반적으로 더 빠름 (C 언어로 작성)

# daemon.json 원복 (선택)
sudo rm /etc/docker/daemon.json
sudo systemctl restart docker
```

---

## 🔧 실습 5: Hooks (생명주기 훅) 활용

### Step 1: 컨테이너 이벤트에 훅 연결

```bash
mkdir -p /tmp/hook-demo/rootfs
cd /tmp/hook-demo

# rootfs 준비
docker export $(docker create alpine:3.18) | tar -C rootfs -xf -

# 훅 스크립트 생성 (호스트에서 실행됨)
sudo mkdir -p /usr/local/hooks

sudo cat > /usr/local/hooks/prestart.sh << 'HOOKEOF'
#!/bin/bash
echo "[$(date)] PRESTART: Container starting (PID: $1)" >> /tmp/hook-demo/hook.log
HOOKEOF

sudo cat > /usr/local/hooks/poststart.sh << 'HOOKEOF'
#!/bin/bash
echo "[$(date)] POSTSTART: Container started" >> /tmp/hook-demo/hook.log
HOOKEOF

sudo cat > /usr/local/hooks/poststop.sh << 'HOOKEOF'
#!/bin/bash
echo "[$(date)] POSTSTOP: Container stopped" >> /tmp/hook-demo/hook.log
HOOKEOF

sudo chmod +x /usr/local/hooks/*.sh

# 훅이 포함된 config.json
cat > config.json << 'CONFIGEOF'
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": false,
    "user": {"uid": 0, "gid": 0},
    "args": ["sh", "-c", "echo 'Container running' && sleep 5 && echo 'Container done'"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "cwd": "/"
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "hostname": "hook-demo",
  "mounts": [
    {"destination": "/proc", "type": "proc", "source": "proc"},
    {"destination": "/dev", "type": "tmpfs", "source": "tmpfs",
     "options": ["nosuid", "strictatime", "mode=755", "size=65536k"]},
    {"destination": "/sys", "type": "sysfs", "source": "sysfs",
     "options": ["nosuid", "noexec", "nodev", "ro"]}
  ],
  "hooks": {
    "prestart": [{
      "path": "/usr/local/hooks/prestart.sh"
    }],
    "poststart": [{
      "path": "/usr/local/hooks/poststart.sh"
    }],
    "poststop": [{
      "path": "/usr/local/hooks/poststop.sh"
    }]
  },
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"}
    ]
  }
}
CONFIGEOF

# 컨테이너 실행
sudo runc run hook-demo

# 훅 로그 확인
cat /tmp/hook-demo/hook.log
# [Wed Jan 15 10:00:01 UTC 2024] PRESTART: Container starting (PID: ...)
# [Wed Jan 15 10:00:01 UTC 2024] POSTSTART: Container started
# [Wed Jan 15 10:00:06 UTC 2024] POSTSTOP: Container stopped
# ✅ 생명주기 훅이 순서대로 실행됨!

# 정리
cd /tmp && sudo rm -rf /tmp/hook-demo
sudo rm -rf /usr/local/hooks
```

---

## 🔥 실전 시나리오: Docker가 실행하는 실제 과정 추적

```bash
# Docker의 컨테이너 생성 과정을 추적

# 1. Docker 이벤트 모니터링 (터미널 1)
docker events --filter type=container &
EVENTS_PID=$!

# 2. strace로 runc 호출 추적 (터미널 2, 선택사항)
# sudo strace -f -e trace=clone,execve -p $(pgrep containerd) &

# 3. 컨테이너 실행
docker run --rm --name trace-demo alpine echo "Traced!"

# 이벤트 출력:
# ... container create ... (container created)
# ... container attach ... (attach to stream)
# ... container start ...  (start container)
# ... container die ...    (process exited)
# ... container destroy ... (cleanup)

# 4. containerd 로그에서 runc 호출 확인
sudo journalctl -u containerd --since "1 minute ago" --no-pager | grep -i "runc\|runtime"

# 5. 이벤트 모니터링 종료
kill $EVENTS_PID 2>/dev/null

# Docker run의 내부 과정 정리:
# docker run alpine echo "hi"
#   → dockerd: POST /containers/create
#   → dockerd: POST /containers/{id}/start
#   → containerd: 이미지 → OCI Bundle 변환
#   → containerd: runc create {id}
#   → runc: namespace/cgroup 설정, rootfs 마운트
#   → runc: runc start {id}
#   → runc: clone() → execve("echo", ["hi"])
#   → 프로세스 실행 → 종료
#   → containerd: runc delete {id}
#   → dockerd: 컨테이너 정리
```

---

## 🚫 안티패턴

### 1. 프로덕션에서 runc 직접 사용

```bash
# ❌ 프로덕션에서 runc 직접 관리
sudo runc run production-app
# - 네트워크 설정 수동
# - 로깅 없음
# - 모니터링 없음
# - 재시작 정책 없음

# ✅ Docker/containerd 사용 (관리 기능 포함)
docker run -d --restart=always \
  --log-driver=json-file \
  --name production-app \
  myapp
```

### 2. Namespace 격리 무시

```bash
# ❌ 모든 Namespace를 호스트와 공유
docker run --pid=host --network=host --ipc=host myapp
# 격리 없음 = 보안 위험!

# ✅ 필요한 것만 최소한으로
docker run --network=host myapp  # 네트워크만 공유 (성능 필요 시)
```

### 3. 런타임 버전 미확인

```bash
# ❌ runc 버전 확인 없이 운영
# 취약점 (CVE-2024-21626 등) 미패치

# ✅ 정기적 버전 확인 및 업데이트
runc --version
docker info | grep -i "runc\|runtime"
# 보안 업데이트 즉시 적용
```

---

## 🎓 연습 문제

### 문제 1: OCI Bundle의 필수 구성요소는?

<details>
<summary>정답 보기</summary>

OCI Bundle은 **2가지** 필수 구성요소로 이루어집니다:

1. **config.json**: 컨테이너 런타임 설정 파일
   - 실행할 프로세스 (args, env, cwd)
   - rootfs 경로
   - Namespace, Cgroup 설정
   - 마운트 포인트

2. **rootfs/**: 루트 파일시스템 디렉토리
   - 컨테이너의 / (루트) 디렉토리
   - 이미지 레이어를 합친 결과

```
my-bundle/
├── config.json    ← 필수
└── rootfs/        ← 필수
    ├── bin/
    ├── etc/
    └── ...
```

이 두 가지만 있으면 `runc run` 으로 컨테이너를 실행할 수 있습니다.

</details>

### 문제 2: `docker run --memory=512m --cpus=2 nginx`는 config.json에서 어떤 필드에 매핑되는가?

<details>
<summary>정답 보기</summary>

```json
{
  "linux": {
    "resources": {
      "memory": {
        "limit": 536870912
      },
      "cpu": {
        "quota": 200000,
        "period": 100000
      }
    }
  }
}
```

- `--memory=512m` → `linux.resources.memory.limit`: 536870912 (512 × 1024 × 1024 bytes)
- `--cpus=2` → `linux.resources.cpu.quota`: 200000 (2 × period)
  - period 기본값: 100000 (100ms)
  - quota = cpus × period = 2 × 100000 = 200000
  - 즉, 100ms 주기 중 200ms 사용 가능 = CPU 2코어

Docker는 사용자 친화적 옵션을 OCI Runtime Spec의 정확한 리소스 설정으로 변환합니다.

</details>

### 문제 3: runc create와 runc run의 차이점은?

<details>
<summary>정답 보기</summary>

| 명령어 | 동작 | 상태 변화 |
|-------|------|----------|
| `runc create` | 컨테이너 환경만 구성 (프로세스 미실행) | → Created |
| `runc start` | Created 상태의 컨테이너 프로세스 시작 | Created → Running |
| `runc run` | create + start를 한 번에 수행 | → Running |

**create/start 분리의 장점:**
1. **Hooks 실행 타이밍 제어**: prestart 훅이 create와 start 사이에 실행됨
2. **네트워크 설정**: containerd가 create 후 네트워크를 설정하고 start
3. **검증**: 환경 구성 후 문제가 없는지 확인 가능
4. **오케스트레이션**: Kubernetes가 Pod의 모든 컨테이너를 create 후 순서대로 start

Docker의 `docker run`은 내부적으로 create → (hook/network 설정) → start 순서로 동작합니다.

</details>

---

## 📌 핵심 요약

```
┌─────────────────┬─────────────────────────────────────┐
│ 개념             │ 설명                                 │
├─────────────────┼─────────────────────────────────────┤
│ OCI             │ 컨테이너 표준 (Runtime/Image/Dist)     │
├─────────────────┼─────────────────────────────────────┤
│ OCI Bundle      │ config.json + rootfs                │
├─────────────────┼─────────────────────────────────────┤
│ config.json     │ 프로세스, NS, Cgroup, 마운트 설정        │
├─────────────────┼─────────────────────────────────────┤
│ runc            │ OCI Runtime Spec 참조 구현체 (Go)      │
├─────────────────┼─────────────────────────────────────┤
│ crun            │ 경량 대체 런타임 (C)                    │
├─────────────────┼─────────────────────────────────────┤
│ kata-containers │ VM 기반 격리 (보안 강화)                │
├─────────────────┼─────────────────────────────────────┤
│ Lifecycle       │ Creating→Created→Running→Stopped    │
├─────────────────┼─────────────────────────────────────┤
│ Hooks           │ prestart, poststart, poststop       │
├─────────────────┼─────────────────────────────────────┤
│ Namespace       │ PID, NET, MNT, UTS, IPC, CGROUP     │
├─────────────────┼─────────────────────────────────────┤
│ Docker 내부     │ dockerd → containerd → runc          │
└─────────────────┴─────────────────────────────────────┘
```

---

## 💡 주요 명령어 정리

```bash
# runc 기본 명령어
runc spec                          # 기본 config.json 생성
runc run <name>                    # 컨테이너 실행 (create+start)
runc create <name>                 # 컨테이너 생성 (프로세스 미실행)
runc start <name>                  # Created 컨테이너 시작
runc state <name>                  # 컨테이너 상태 확인
runc list                          # 컨테이너 목록
runc kill <name> <signal>          # 시그널 전송
runc delete <name>                 # 컨테이너 삭제
runc exec <name> <cmd>             # 실행 중 컨테이너에 명령 실행

# rootfs 추출
docker export $(docker create <image>) | tar -C rootfs -xf -

# Namespace 확인
ls -la /proc/<pid>/ns/             # 프로세스의 namespace 확인
nsenter --target <pid> --all       # 다른 namespace 진입

# Docker 런타임 정보
docker info | grep Runtime         # 현재 런타임 확인
docker run --runtime=<name>        # 특정 런타임으로 실행
```

---

## 📚 참고 자료

- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [runc GitHub](https://github.com/opencontainers/runc)
- [crun GitHub](https://github.com/containers/crun)
- [kata-containers](https://katacontainers.io/)
- [gVisor](https://gvisor.dev/)
- [containerd Architecture](https://containerd.io/docs/)

---

## 🤔 생각해볼 문제

1. 왜 Docker는 모놀리식에서 dockerd/containerd/runc로 분리했을까?
2. kata-containers가 runc보다 안전한 이유는? 성능 트레이드오프는?
3. Kubernetes는 왜 Docker 대신 containerd를 직접 사용하게 되었을까?

> 💡 **답변**: 1) 초기 Docker는 모든 기능이 dockerd 하나에 있었음. dockerd를 재시작하면 모든 컨테이너가 중단되는 문제 발생. 분리 후 dockerd를 업데이트해도 containerd가 컨테이너를 유지. 또한 OCI 표준 준수로 런타임 교체 가능, 보안 표면 축소 (각 컴포넌트 최소 권한). 2) kata-containers는 각 컨테이너를 경량 VM에서 실행. 별도의 커널을 사용하므로 커널 취약점 영향 없음. 하지만 VM 부팅 오버헤드 (수백 ms vs runc의 수십 ms), 메모리 오버헤드 (VM당 수십 MB), 중첩 가상화 필요 등의 트레이드오프. 멀티테넌트 환경 (클라우드)에서 주로 사용. 3) Docker → containerd → runc 경로에서 Docker (dockerd)는 이미지 빌드, 네트워크 등 Kubernetes가 자체 관리하는 기능을 중복 제공. Kubernetes v1.24부터 dockershim 제거, CRI (Container Runtime Interface)를 통해 containerd 직접 사용. 이로써 호출 경로 단축 (kubelet → containerd → runc), 리소스 절약, 유지보수 단순화.

---

<div align="center">

**[⬅️ 이전: Performance 섹션](../performance/08-Benchmarking.md)** | **[다음: OCI Specification ➡️](./02-OCI-Specification.md)**

</div>
