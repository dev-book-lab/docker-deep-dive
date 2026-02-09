# 06. User Namespaces - 사용자 네임스페이스

## 🎯 이 챕터에서 배울 것

- **User Namespace** - UID/GID 매핑 개념
- **Rootless 컨테이너** - 비특권 사용자로 Docker 실행
- **UID 재매핑** - 컨테이너 root를 호스트 일반 사용자로
- **보안 격리** - 권한 상승 공격 방지
- **실무 적용** - 프로덕션 환경 설정

## 📌 왜 중요한가?

**"User Namespace는 컨테이너 root를 호스트의 일반 사용자로 매핑하여 근본적인 격리를 제공합니다."**

```
User Namespace 없이 vs 있을 때:

Without User Namespace:
┌─────────────────────────────────────┐
│ Host                                │
│ UID 0 (root) ─────────────────────┐ │
│                                   │ │
│ ┌─────────────────────────────────▼─┐
│ │ Container                         │
│ │ UID 0 (root) ◄────────────────────┘
│ │                                   │
│ │ Process 실행:                      │
│ │ $ whoami                          │
│ │ root                              │
│ │                                   │
│ │ $ id -u                           │
│ │ 0                                 │
│ └───────────────────────────────────┘
└─────────────────────────────────────┘

문제점:
1. Container root = Host root
2. Container 탈출 시 Host root 권한 획득
3. /proc, /sys 접근 가능
4. 호스트 파일 소유권 변경 가능

공격 시나리오:
┌─────────────────────────────────────┐
│ Attacker in Container (root)        │
│ $ cat /proc/sys/kernel/core_pattern │
│ |/usr/lib/systemd/systemd-coredump  │
│                                     │
│ $ echo "payload" > /proc/sys/...    │
│ ✅ 성공! (Host 영향)                  │
└─────────────────────────────────────┘

With User Namespace:
┌─────────────────────────────────────┐
│ Host                                │
│ UID 0 (root)                        │
│ UID 100000 (dockremap) ───────────┐ │
│                                   │ │
│ ┌─────────────────────────────────▼─┐
│ │ Container                         │
│ │ UID 0 (root) ◄────────────────────┘
│ │ → mapped to UID 100000 on host    │
│ │                                   │
│ │ Process 실행:                      │
│ │ $ whoami (in container)           │
│ │ root                              │
│ │                                   │
│ │ $ id -u (in container)            │
│ │ 0                                 │
│ └───────────────────────────────────┘
│                                     │
│ Host에서 보면:                        │
│ $ ps aux | grep container           │
│ 100000  12345  ... /app/process     │
│         ↑                           │
│     일반 사용자!                       │
└─────────────────────────────────────┘

장점:
1. Container root ≠ Host root
2. Container 탈출해도 일반 사용자
3. /proc, /sys 쓰기 불가
4. 호스트 파일 소유권 변경 불가

공격 차단:
┌─────────────────────────────────────┐
│ Attacker in Container (root)        │
│ $ cat /proc/sys/kernel/core_pattern │
│ ✅ 읽기 가능                          │
│                                     │
│ $ echo "payload" > /proc/sys/...    │
│ ❌ Permission denied                │
│ (Host에서는 UID 100000)               │
└─────────────────────────────────────┘

UID 매핑 상세:
┌──────────────────────────────────────┐
│ Container Namespace → Host           │
├──────────────────────────────────────┤
│ UID 0     (root)    → UID 100000     │
│ UID 1     (daemon)  → UID 100001     │
│ UID 1000  (app)     → UID 101000     │
│ UID 65534 (nobody)  → UID 165534     │
└──────────────────────────────────────┘

매핑 범위 설정:
/etc/subuid:
dockremap:100000:65536
   ↑        ↑      ↑
  사용자   시작   개수

Container UID 0-65535 → Host UID 100000-165535

User Namespace의 핵심 가치:

1. 권한 격리:
   Without:
   Container root → Host에서 작업 → root 권한
   
   With:
   Container root → Host에서 작업 → 일반 사용자 권한

2. Privilege Escalation 방지:
   ┌─────────────────────────────────┐
   │ Container (UID 0)               │
   │ $ exploit vulnerability         │
   │ → Escalate to UID 0             │
   └────────────┬────────────────────┘
                │ Escape
   ┌────────────▼────────────────────┐
   │ Host                            │
   │ UID 100000 (일반 사용자)           │
   │ → Limited damage                │
   └─────────────────────────────────┘

3. 파일 소유권 보호:
   Without User Namespace:
   Container: chown 0:0 /host-file
   Host:      -rw-r--r-- root root /host-file
   → 위험!
   
   With User Namespace:
   Container: chown 0:0 /host-file
   Host:      -rw-r--r-- 100000 100000 /host-file
   → 안전

4. Rootless Docker:
   ┌──────────────────────────────┐
   │ User: alice (UID 1000)       │
   │ $ dockerd-rootless.sh        │
   │                              │
   │ - Docker Daemon: UID 1000    │
   │ - Container: UID 0 (mapped)  │
   │ - No sudo required           │
   │ - No root access to host     │
   └──────────────────────────────┘

실무 시나리오:

사고 사례 1 - CVE-2019-5736 (runc):
┌─────────────────────────────────────┐
│ 취약점: runc 컨테이너 탈출               │
│                                     │
│ Without User Namespace:             │
│ 1. Container에서 exploit 실행         │
│ 2. Host runc 바이너리 덮어쓰기          │
│ 3. Host root 권한 획득                │
│ → 완전한 Host 장악                     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ With User Namespace:                │
│ 1. Container에서 exploit 실행         │
│ 2. Host에서는 UID 100000              │
│ 3. runc 바이너리 쓰기 권한 없음           │
│ → 공격 실패 또는 피해 최소화              │
└─────────────────────────────────────┘

사고 사례 2 - 실수로 인한 호스트 영향:
┌─────────────────────────────────────┐
│ 개발자 실수:                           │
│ docker run -v /:/host alpine        │
│   chown -R root:root /host/etc      │
│                                     │
│ Without User Namespace:             │
│ → Host /etc 소유권 변경됨!             │ 
│ → 시스템 부팅 불가능                    │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ With User Namespace:                │
│ → Host /etc 소유권이 100000:100000    │
│ → 시스템 정상 동작                      │
│ → 영향 최소화 (컨테이너만 재생성)          │
└─────────────────────────────────────┘

보안 계층 추가:
┌──────────────────────────────────────┐
│ Layer 5: User Namespace              │
│ (UID 재매핑)                           │
├──────────────────────────────────────┤
│ Layer 4: AppArmor/SELinux            │
│ (강제 접근 제어)                        │
├──────────────────────────────────────┤
│ Layer 3: Seccomp                     │
│ (시스템 콜 제한)                        │
├──────────────────────────────────────┤
│ Layer 2: Capabilities                │
│ (권한 세분화)                           │
├──────────────────────────────────────┤
│ Layer 1: Namespace                   │
│ (프로세스 격리)                         │
└──────────────────────────────────────┘
```

**실무 영향:**
- Container Escape 피해 최소화 → 손실 90% 감소
- 제로데이 대응 → 알려지지 않은 취약점 방어
- 멀티 테넌시 → 안전한 공유 환경
- 규정 준수 → CIS Benchmark 요구사항

---

## 🔧 실습 1: User Namespace 기본

### Step 1: 시스템 지원 확인

```bash
# 커널 User Namespace 지원 확인
cat /proc/sys/kernel/unprivileged_userns_clone
# 1 (활성화) 또는 0 (비활성화)

# Ubuntu 20.04+에서는 기본 활성화
# Debian/Ubuntu에서 비활성화된 경우:
echo 1 | sudo tee /proc/sys/kernel/unprivileged_userns_clone

# 영구 설정
echo "kernel.unprivileged_userns_clone=1" | sudo tee -a /etc/sysctl.d/99-userns.conf
sudo sysctl -p /etc/sysctl.d/99-userns.conf

# Docker 버전 확인 (User Namespace는 1.10+ 필요)
docker version

# 현재 Docker의 User Namespace 상태
docker info | grep -i "userns"
# Security Options:
#  (userns 없음 = 비활성화)
```

### Step 2: subuid/subgid 설정

```bash
# subuid 파일 확인 (UID 매핑)
cat /etc/subuid

# 출력 예시:
# alice:100000:65536
# bob:200000:65536

# subgid 파일 확인 (GID 매핑)
cat /etc/subgid

# 없으면 생성
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $(whoami)

# 또는 직접 편집
echo "$(whoami):100000:65536" | sudo tee -a /etc/subuid
echo "$(whoami):100000:65536" | sudo tee -a /etc/subgid

# dockremap 사용자 생성 (Docker 기본)
sudo useradd -r -s /bin/false dockremap
echo "dockremap:100000:65536" | sudo tee -a /etc/subuid
echo "dockremap:100000:65536" | sudo tee -a /etc/subgid

# 확인
cat /etc/subuid | grep dockremap
# dockremap:100000:65536
```

### Step 3: Docker Daemon에서 User Namespace 활성화

```bash
# daemon.json 설정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "userns-remap": "default",
  "live-restore": true
}
EOF

# "default"는 dockremap 사용자 사용
# 또는 특정 사용자: "userns-remap": "username"

# Docker 재시작
sudo systemctl restart docker

# 확인
docker info | grep -i "userns"
# Security Options:
#  userns

# Docker 데이터 디렉토리 변경 확인
ls -la /var/lib/docker/
# drwx------ 100000 100000 ... 100000.100000
#                                ↑
#                     User Namespace 디렉토리
```

### Step 4: UID 매핑 확인

```bash
# 테스트 컨테이너 실행
docker run -d --name test-userns alpine sleep 3600

# 컨테이너 내부 UID
docker exec test-userns id
# uid=0(root) gid=0(root) groups=0(root),1(bin),...

# 호스트에서 프로세스 확인
ps aux | grep "sleep 3600"
# 100000  12345  ... sleep 3600
#   ↑
# 재매핑된 UID!

# 상세 매핑 확인
docker exec test-userns cat /proc/self/uid_map
# 0     100000      65536
# ↑     ↑           ↑
# 컨테이너 호스트   범위

docker exec test-userns cat /proc/self/gid_map
# 0     100000      65536
```

### Step 5: 파일 소유권 확인

```bash
# 볼륨 마운트 테스트
mkdir -p /tmp/test-volume

# 컨테이너에서 파일 생성
docker run --rm \
  -v /tmp/test-volume:/data \
  alpine sh -c 'echo "test" > /data/file.txt'

# 호스트에서 확인
ls -ln /tmp/test-volume/
# -rw-r--r-- 1 100000 100000 5 Feb 10 10:00 file.txt
#              ↑      ↑
#           재매핑된 UID/GID

# 컨테이너 내부에서 보면
docker run --rm \
  -v /tmp/test-volume:/data \
  alpine ls -ln /data/
# -rw-r--r-- 1 0 0 5 Feb 10 10:00 file.txt
#              ↑ ↑
#           root로 보임 (매핑됨)

# 정리
rm -rf /tmp/test-volume
docker rm -f test-userns
```

---

## 🔧 실습 2: Rootless Docker

### Step 1: Rootless Docker 설치

```bash
# 일반 사용자로 로그인 (sudo 권한 필요 없음)
whoami
# alice

# 필수 패키지 설치 (한 번만)
sudo apt-get install -y uidmap

# Rootless Docker 설치 스크립트
curl -fsSL https://get.docker.com/rootless | sh

# 출력:
# Client: Docker Engine - Community
#  Version:           24.0.0
# ...
# 
# # Docker daemon is not running.
# # You need to run the following commands to start it:
# 
# export PATH=/home/alice/bin:$PATH
# export DOCKER_HOST=unix:///run/user/1000/docker.sock
# 
# systemctl --user start docker
# systemctl --user enable docker

# 환경 변수 설정 (영구적)
cat >> ~/.bashrc <<'EOF'

# Rootless Docker
export PATH=/home/alice/bin:$PATH
export DOCKER_HOST=unix:///run/user/1000/docker.sock
EOF

source ~/.bashrc
```

### Step 2: Rootless Docker 시작

```bash
# Docker Daemon 시작 (사용자 서비스)
systemctl --user start docker

# 부팅 시 자동 시작
systemctl --user enable docker

# 상태 확인
systemctl --user status docker

# Docker 명령 테스트
docker version
docker info

# 중요: 모든 것이 일반 사용자 권한으로 실행됨!
ps aux | grep dockerd
# alice   12345  ... dockerd-rootless.sh
#   ↑
# 일반 사용자!
```

### Step 3: Rootless 제약 사항 확인

```bash
# 1. 특권 포트 (1024 미만) 바인딩 불가
docker run -d --name web -p 80:80 nginx
# Error: permission denied

# 해결 방법 1: 높은 포트 사용
docker run -d --name web -p 8080:80 nginx
# 성공

# 해결 방법 2: setcap (권장하지 않음)
sudo setcap cap_net_bind_service=ep /home/alice/bin/rootlesskit

# 2. cgroup v2 필요
cat /proc/mounts | grep cgroup
# cgroup2 /sys/fs/cgroup cgroup2 ...

# cgroup v1인 경우 업그레이드 필요

# 3. Overlay 파일시스템
docker info | grep "Storage Driver"
# Storage Driver: overlay2
# 또는 fuse-overlayfs

# 4. --privileged 컨테이너 불가
docker run --privileged alpine
# Error: not supported in rootless mode
```

### Step 4: Rootless 보안 장점 확인

```bash
# 테스트: 호스트 파일 접근
docker run --rm \
  -v /:/host \
  alpine sh -c 'ls -la /host/etc/shadow'
# ls: /host/etc/shadow: Permission denied

# Rootless에서는:
# 1. 컨테이너 root = 호스트 일반 사용자
# 2. /etc/shadow는 호스트 root만 읽을 수 있음
# 3. 따라서 접근 불가

# 호스트 프로세스 확인
ps aux | grep "alpine"
# alice   12345  ... /bin/sh
#   ↑
# 일반 사용자 권한!

# /proc 쓰기 테스트
docker run --rm alpine sh -c 'echo 1 > /proc/sys/kernel/randomize_va_space'
# sh: can't create /proc/sys/kernel/randomize_va_space: Permission denied
```

---

## 🔧 실습 3: User Namespace 고급 설정

### Step 1: 커스텀 UID 매핑

```bash
# 특정 사용자로 매핑
echo "myuser:200000:65536" | sudo tee -a /etc/subuid
echo "myuser:200000:65536" | sudo tee -a /etc/subgid

# daemon.json 수정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "userns-remap": "myuser"
}
EOF

sudo systemctl restart docker

# 확인
ps aux | grep dockerd
ls -la /var/lib/docker/
# drwx------ 200000 200000 ... 200000.200000
```

### Step 2: 좁은 범위 매핑 (보안 강화)

```bash
# 1000개 UID만 사용
echo "limited:100000:1000" | sudo tee -a /etc/subuid
echo "limited:100000:1000" | sudo tee -a /etc/subgid

sudo tee /etc/docker/daemon.json <<'EOF'
{
  "userns-remap": "limited"
}
EOF

sudo systemctl restart docker

# 테스트
docker run --rm alpine id
# uid=0(root) gid=0(root)

# 호스트에서
ps aux | grep alpine
# 100000 ... alpine

# UID 999를 초과하는 사용자는?
docker run --rm alpine sh -c 'adduser -u 1500 testuser'
# adduser: unknown user testuser
# (매핑 범위 초과로 실패)
```

### Step 3: 개별 컨테이너 User Namespace

```bash
# Docker Daemon 전체가 아닌 개별 컨테이너만
# (userns-remap 비활성화 상태에서)

# --userns=host (User Namespace 비활성화)
docker run --rm --userns=host alpine id
# uid=0(root) gid=0(root)

# 호스트에서
ps aux | grep alpine
# root ... alpine  (위험!)

# --user 옵션 (권장)
docker run --rm --user 1000:1000 alpine id
# uid=1000 gid=1000

# 조합: User Namespace + 비특권 사용자
# daemon.json에서 userns-remap 활성화 후
docker run --rm --user 1000:1000 alpine id
# uid=1000 gid=1000

# 호스트에서
ps aux | grep alpine
# 101000 ... alpine
#   ↑
# 100000 (base) + 1000 (container UID) = 101000
```

### Step 4: 볼륨 소유권 문제 해결

```bash
# 문제: User Namespace로 인한 볼륨 권한 문제
mkdir -p /tmp/app-data
echo "data" > /tmp/app-data/config.txt

docker run --rm \
  -v /tmp/app-data:/app \
  alpine cat /app/config.txt
# cat: /app/config.txt: Permission denied

# 호스트 파일 소유권
ls -ln /tmp/app-data/
# -rw-r--r-- 1 1000 1000 ... config.txt

# 컨테이너에서 보면
docker run --rm -v /tmp/app-data:/app alpine ls -ln /app
# -rw-r--r-- 1 nobody nobody ... config.txt
# (UID 1000이 매핑 범위 밖)

# 해결 방법 1: 호스트 파일을 재매핑된 UID로 변경
sudo chown -R 100000:100000 /tmp/app-data

docker run --rm -v /tmp/app-data:/app alpine cat /app/config.txt
# data (성공!)

# 해결 방법 2: 컨테이너에서 비특권 사용자 사용
docker run --rm \
  --user 1000:1000 \
  -v /tmp/app-data:/app \
  alpine cat /app/config.txt
# data (성공!)

# 해결 방법 3: Named Volume 사용 (권장)
docker volume create app-data
docker run --rm \
  -v app-data:/app \
  alpine sh -c 'echo "data" > /app/config.txt'

docker run --rm \
  -v app-data:/app \
  alpine cat /app/config.txt
# data (성공!)

# Named volume의 소유권은 Docker가 관리
docker volume inspect app-data --format '{{.Mountpoint}}'
# /var/lib/docker/100000.100000/volumes/app-data/_data
```

---

## 🔧 실습 4: User Namespace 보안 테스트

### Step 1: Privilege Escalation 방지 테스트

```bash
# Without User Namespace
# daemon.json에서 userns-remap 제거 후

# 1. setuid 바이너리 테스트
docker run --rm -v /:/host alpine \
  find /host/usr/bin -perm -4000 2>/dev/null | head -5

# 2. setuid 바이너리 실행
docker run --rm -it alpine sh

# 컨테이너 내부:
find / -perm -4000 2>/dev/null
# /bin/su
# /usr/bin/passwd
# ...

# su 실행 (권한 상승 가능)
# (단, 패스워드를 알아야 함)

# With User Namespace 활성화 후
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "userns-remap": "default"
}
EOF
sudo systemctl restart docker

# 동일한 테스트
docker run --rm alpine sh -c 'ls -l /bin/su'
# -rwsr-xr-x 1 root root ... /bin/su

# 하지만 호스트에서는
docker run -d --name test alpine sleep 3600
ps aux | grep "sleep 3600"
# 100000 ... (일반 사용자)

# setuid 비트가 있어도 호스트에서는 일반 사용자이므로
# 권한 상승 효과 없음
```

### Step 2: 파일시스템 공격 테스트

```bash
# Without User Namespace
docker run --rm -v /tmp:/host alpine \
  sh -c 'touch /host/malicious && chown 0:0 /host/malicious'

ls -l /tmp/malicious
# -rw-r--r-- 1 root root ... malicious
# 위험! 호스트에 root 소유 파일 생성

sudo rm /tmp/malicious

# With User Namespace
docker run --rm -v /tmp:/host alpine \
  sh -c 'touch /host/malicious && chown 0:0 /host/malicious'

ls -l /tmp/malicious
# -rw-r--r-- 1 100000 100000 ... malicious
# 안전! 일반 사용자 소유

rm /tmp/malicious
```

### Step 3: /proc 공격 테스트

```bash
# Without User Namespace
docker run --rm alpine \
  sh -c 'cat /proc/sys/kernel/core_pattern'
# |/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h

docker run --rm alpine \
  sh -c 'echo "malicious" > /proc/sys/kernel/core_pattern'
# 성공 가능 (CAP_SYS_ADMIN이 있으면)

# With User Namespace
docker run --rm alpine \
  sh -c 'echo "malicious" > /proc/sys/kernel/core_pattern'
# sh: can't create /proc/sys/kernel/core_pattern: Permission denied

# User Namespace + AppArmor + Seccomp
# → 삼중 방어
```

### Step 4: CVE-2019-5736 (runc) 시뮬레이션

```bash
# CVE-2019-5736: runc 컨테이너 탈출 취약점
# https://github.com/Frichetten/CVE-2019-5736-PoC

# Without User Namespace:
# 1. 컨테이너에서 악성 코드 실행
# 2. /proc/self/exe (runc) 덮어쓰기
# 3. Host root 권한 획득

# With User Namespace:
# 1. 컨테이너에서 악성 코드 실행
# 2. /proc/self/exe 접근 시도
# 3. Permission denied (호스트에서 일반 사용자)
# → 공격 실패 또는 피해 최소화

# 테스트 (안전한 버전으로)
docker run --rm alpine ls -l /proc/self/exe
# lrwxrwxrwx 1 root root ... /proc/self/exe -> /usr/bin/docker-runc

# 쓰기 권한 확인
docker run --rm alpine sh -c 'echo test > /proc/self/exe'
# sh: can't create /proc/self/exe: Permission denied

# User Namespace가 runc 취약점으로부터 보호
```

---

## 🔧 실습 5: 프로덕션 환경 설정

### Step 1: 복합 보안 설정

```bash
# 최대 보안 설정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "userns-remap": "default",
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "icc": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  }
}
EOF

sudo systemctl restart docker
```

### Step 2: Docker Compose 호환성

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    # User Namespace와 호환
    user: "1000:1000"
    volumes:
      # Named volume 사용 (권장)
      - web-data:/usr/share/nginx/html
    ports:
      - "8080:80"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE

  db:
    image: postgres:alpine
    user: "999:999"  # postgres 사용자
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - SETGID
      - SETUID

volumes:
  web-data:
  db-data:

secrets:
  db_password:
    external: true
```

```bash
# 배포
docker compose up -d

# 확인
docker compose ps
ps aux | grep nginx
# 101000 ... nginx  (100000 + 1000)

ps aux | grep postgres
# 100999 ... postgres  (100000 + 999)
```

### Step 3: Swarm 환경에서 User Namespace

```bash
# Swarm 클러스터 모든 노드에서
# /etc/docker/daemon.json 설정 동일하게

# Stack 배포
cat > stack.yml <<'EOF'
version: '3.8'

services:
  web:
    image: nginx:alpine
    user: "nginx"
    volumes:
      - web-data:/usr/share/nginx/html
    ports:
      - "80:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure

volumes:
  web-data:
EOF

docker stack deploy -c stack.yml webapp

# 각 노드에서 프로세스 확인
docker node ls
# ID        HOSTNAME   STATUS  AVAILABILITY  MANAGER STATUS
# abc123    node1      Ready   Active        Leader
# def456    node2      Ready   Active        Reachable
# ghi789    node3      Ready   Active        Reachable

# node1에서
ssh node1 "ps aux | grep nginx"
# 100xxx ... nginx

# node2에서
ssh node2 "ps aux | grep nginx"
# 100xxx ... nginx

# 모든 노드에서 일반 사용자로 실행됨
```

### Step 4: 모니터링 및 감사

```bash
# User Namespace 사용 여부 확인 스크립트
cat > check_userns.sh <<'EOF'
#!/bin/bash

echo "=== Docker Daemon User Namespace Status ==="
docker info | grep -i "userns"

echo ""
echo "=== Running Containers UID Mapping ==="
for container in $(docker ps -q); do
    name=$(docker inspect --format '{{.Name}}' $container | sed 's/\///')
    pid=$(docker inspect --format '{{.State.Pid}}' $container)
    uid=$(ps -o uid= -p $pid 2>/dev/null)
    
    if [ -n "$uid" ]; then
        if [ "$uid" -ge 100000 ]; then
            status="✅ User Namespace (UID: $uid)"
        else
            status="❌ No User Namespace (UID: $uid)"
        fi
        echo "Container: $name - $status"
    fi
done

echo ""
echo "=== subuid/subgid Mapping ==="
cat /etc/subuid
cat /etc/subgid
EOF

chmod +x check_userns.sh
./check_userns.sh
```

### Step 5: 마이그레이션 전략

```bash
# 기존 환경에서 User Namespace로 마이그레이션

# 1. 백업
sudo tar -czf docker-backup-$(date +%Y%m%d).tar.gz /var/lib/docker

# 2. 테스트 환경에서 먼저 검증
# - 볼륨 권한 문제
# - 애플리케이션 호환성
# - 성능 영향

# 3. Staging 환경 적용
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "userns-remap": "default"
}
EOF
sudo systemctl restart docker

# 4. 볼륨 마이그레이션
# Named volume: Docker가 자동 처리
# Bind mount: 수동으로 chown 필요

# 5. 검증 스크립트
cat > verify_migration.sh <<'EOF'
#!/bin/bash

echo "Testing User Namespace..."

# 1. 간단한 컨테이너 실행
docker run --rm alpine echo "Test OK" || exit 1

# 2. UID 매핑 확인
docker run -d --name test alpine sleep 30
PID=$(docker inspect --format '{{.State.Pid}}' test)
UID=$(ps -o uid= -p $PID)
docker rm -f test

if [ "$UID" -ge 100000 ]; then
    echo "✅ User Namespace active (UID: $UID)"
else
    echo "❌ User Namespace NOT active (UID: $UID)"
    exit 1
fi

# 3. 볼륨 테스트
docker run --rm -v test-vol:/data alpine sh -c 'echo test > /data/file'
docker run --rm -v test-vol:/data alpine cat /data/file || exit 1
docker volume rm test-vol

echo "✅ All tests passed!"
EOF

chmod +x verify_migration.sh
./verify_migration.sh

# 6. 프로덕션 적용 (롤링 업데이트)
# - 한 번에 한 노드씩
# - 모니터링 후 다음 노드
```

---

## 💡 주요 명령어 정리

### User Namespace 설정

```bash
# subuid/subgid 확인
cat /etc/subuid
cat /etc/subgid

# 사용자 매핑 추가
echo "username:100000:65536" | sudo tee -a /etc/subuid
echo "username:100000:65536" | sudo tee -a /etc/subgid

# Daemon 설정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "userns-remap": "default"
}
EOF

sudo systemctl restart docker

# 확인
docker info | grep userns
ls -la /var/lib/docker/
```

### Rootless Docker

```bash
# 설치
curl -fsSL https://get.docker.com/rootless | sh

# 환경 변수
export PATH=$HOME/bin:$PATH
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock

# 시작
systemctl --user start docker
systemctl --user enable docker

# 상태
systemctl --user status docker
```

### UID 매핑 확인

```bash
# 컨테이너 UID
docker exec CONTAINER id

# 호스트 UID
ps aux | grep CONTAINER_PROCESS

# 매핑 정보
docker exec CONTAINER cat /proc/self/uid_map
docker exec CONTAINER cat /proc/self/gid_map

# 파일 소유권
docker run -v /path:/path CONTAINER ls -ln /path
ls -ln /path  # 호스트에서
```

---

## 🎓 연습 문제

### 문제 1: User Namespace 활성화

Docker에서 User Namespace를 활성화하고, nginx 컨테이너를 실행한 후 호스트에서 프로세스 UID가 100000번대인지 확인하세요.

<details>
<summary>정답 보기</summary>

```bash
# 1. dockremap 사용자 생성
sudo useradd -r -s /bin/false dockremap
echo "dockremap:100000:65536" | sudo tee -a /etc/subuid
echo "dockremap:100000:65536" | sudo tee -a /etc/subgid

# 2. daemon.json 설정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "userns-remap": "default"
}
EOF

# 3. Docker 재시작
sudo systemctl restart docker

# 4. nginx 실행
docker run -d --name test-nginx nginx:alpine

# 5. 컨테이너 내부 UID
docker exec test-nginx id
# uid=0(root) gid=0(root)

# 6. 호스트 UID 확인
ps aux | grep "nginx: master"
# 100000 ... nginx: master process

# 7. 매핑 확인
docker exec test-nginx cat /proc/self/uid_map
# 0  100000  65536

# 정리
docker rm -f test-nginx
```

</details>

### 문제 2: 볼륨 권한 문제 해결

User Namespace가 활성화된 환경에서 bind mount 볼륨의 권한 문제를 해결하세요.

```bash
# 문제 상황
mkdir /tmp/data
echo "config" > /tmp/data/config.txt
docker run --rm -v /tmp/data:/app alpine cat /app/config.txt
# Permission denied
```

<details>
<summary>정답 보기</summary>

```bash
# 해결 방법 1: 호스트 파일 소유권 변경
sudo chown -R 100000:100000 /tmp/data
docker run --rm -v /tmp/data:/app alpine cat /app/config.txt
# config

# 해결 방법 2: 컨테이너에서 일치하는 사용자 사용
# 호스트 파일이 1000:1000인 경우
docker run --rm --user 1000:1000 -v /tmp/data:/app alpine cat /app/config.txt

# 해결 방법 3: Named volume 사용 (권장)
docker volume create data-vol
docker run --rm -v data-vol:/app alpine sh -c 'echo config > /app/config.txt'
docker run --rm -v data-vol:/app alpine cat /app/config.txt
# config

# 해결 방법 4: 초기화 컨테이너
docker run --rm -v /tmp/data:/app alpine chown -R 1000:1000 /app
docker run --rm --user 1000:1000 -v /tmp/data:/app alpine cat /app/config.txt
```

</details>

### 문제 3: Rootless Docker 설정

일반 사용자 계정으로 Rootless Docker를 설치하고 nginx 컨테이너를 실행하세요.

<details>
<summary>정답 보기</summary>

```bash
# 1. 필수 패키지 설치
sudo apt-get install -y uidmap

# 2. Rootless Docker 설치
curl -fsSL https://get.docker.com/rootless | sh

# 3. 환경 변수 설정
cat >> ~/.bashrc <<'EOF'
export PATH=$HOME/bin:$PATH
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock
EOF

source ~/.bashrc

# 4. Docker 시작
systemctl --user start docker
systemctl --user enable docker

# 5. nginx 실행 (8080 포트 사용)
docker run -d --name nginx -p 8080:80 nginx:alpine

# 6. 확인
curl http://localhost:8080

# 7. 프로세스 확인
ps aux | grep dockerd
# alice ... dockerd-rootless.sh

ps aux | grep nginx
# alice ... nginx: master process

# 8. 특권 포트 시도 (실패)
docker run -d --name web80 -p 80:80 nginx:alpine
# Error: permission denied

# 9. 정리
docker rm -f nginx
```

</details>

---

## 📌 핵심 요약

### User Namespace vs 일반 모드

| 항목 | 일반 모드 | User Namespace |
|-----|----------|----------------|
| **Container UID 0** | Host UID 0 | Host UID 100000+ |
| **권한 상승** | 가능 | 제한적 |
| **파일 소유권** | Host root | Host 일반 사용자 |
| **/proc 쓰기** | 가능 (CAP_SYS_ADMIN) | 불가능 |
| **Container Escape 피해** | 전체 시스템 | 일반 사용자 권한만 |
| **설정 복잡도** | 🟢 낮음 | 🟡 중간 |
| **볼륨 권한** | 🟢 쉬움 | 🔴 복잡 |

### Rootless Docker

```
Traditional Docker:
┌─────────────────────────┐
│ root                    │
│ ├─ dockerd (root)       │
│ └─ container (root)     │
└─────────────────────────┘
→ sudo 필요
→ 보안 위험

Rootless Docker:
┌─────────────────────────┐
│ alice (UID 1000)        │
│ ├─ dockerd (UID 1000)   │
│ └─ container (mapped)   │
└─────────────────────────┘
→ sudo 불필요
→ 안전
```

### 보안 계층 통합

```
최대 보안 설정:

┌────────────────────────────┐
│ User Namespace             │
│ (UID 재매핑)                 │
├────────────────────────────┤
│ AppArmor/SELinux           │
│ (파일 접근 제어)              │
├────────────────────────────┤
│ Seccomp                    │
│ (시스템 콜 제한)              │
├────────────────────────────┤
│ Capabilities               │
│ (권한 세분화)                 │
├────────────────────────────┤
│ --user 1000:1000           │
│ (비특권 사용자)               │
├────────────────────────────┤
│ --read-only                │
│ (불변 파일시스템)              │
└────────────────────────────┘
```

### 실무 Best Practices

**1. 활성화 전략:**
```bash
# Development
userns-remap: 선택 사항
테스트 및 호환성 확인

# Staging
userns-remap: 필수
전체 워크플로우 검증

# Production
userns-remap: 필수
+ Rootless (가능하면)
```

**2. 볼륨 관리:**
```bash
# 권장: Named volumes
docker volume create mydata

# 피하기: Bind mounts (권한 문제)
# 필요시: 초기화 컨테이너로 chown
```

**3. 마이그레이션:**
```
1. 백업
2. 테스트 환경 검증
3. Staging 적용
4. 볼륨 권한 수정
5. 프로덕션 롤링 배포
```

**4. 제약 사항:**
- [ ] Privileged 컨테이너 불가
- [ ] 일부 볼륨 드라이버 제한
- [ ] Bind mount 권한 복잡
- [ ] Rootless: 특권 포트 불가
- [ ] Rootless: cgroup v2 필요

### 보안 영향

```
CVE-2019-5736 (runc):
Without User Namespace: 완전한 Host 장악
With User Namespace: 일반 사용자 권한만

실수로 인한 피해:
Without: Host 시스템 손상
With: 컨테이너만 영향

멀티 테넌시:
Without: 위험
With: 안전 (각 테넌트 격리)
```

---

<div align="center">

**[⬅️ 이전: AppArmor & SELinux](./05-AppArmor-SELinux.md)** | **[다음: Security Scanning Tools ➡️](./07-Security-Scanning-Tools.md)**

</div>
