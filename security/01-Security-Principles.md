# 01. Security Best Practices - 보안 기본 원칙

## 🎯 이 챕터에서 배울 것

- **Docker 보안의 핵심 원칙** (Least Privilege, Defense in Depth)
- **공격 표면 분석**과 위협 모델
- **Host 보안 강화** (커널, AppArmor/SELinux)
- **Docker Daemon 보안** (TLS, socket 권한)
- **컨테이너 보안** (Capabilities, Seccomp, User Namespace)

## 📌 왜 중요한가?

**"Docker 보안은 여러 계층의 방어 메커니즘을 통해 컨테이너와 호스트를 보호합니다."**

```
보안이 없는 환경 vs 보안 강화 환경:

보안이 없는 환경:
┌─────────────────────────────────────┐
│ Host (Root)                         │
│ ┌─────────────────────────────────┐ │
│ │ Container (Root)                │ │
│ │ - 모든 Capabilities              │ │
│ │ - Privileged 모드                │ │
│ │ - Host 파일시스템 접근              │ │
│ │ - 제한 없는 시스템 콜               │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
❌ 컨테이너 탈출 가능
❌ Host 권한 획득 위험
❌ 악성 이미지 실행 가능
❌ 리소스 고갈 공격

보안 강화 환경:
┌─────────────────────────────────────┐
│ Host (Hardened)                     │
│ - AppArmor/SELinux                  │
│ - User Namespace                    │
│ - Kernel 파라미터 강화                 │
│ ┌─────────────────────────────────┐ │
│ │ Container (Unprivileged)        │ │
│ │ - UID 1000 (mapped)             │ │
│ │ - Capabilities 최소화             │ │
│ │ - Seccomp 필터                   │ │
│ │ - Read-only 파일시스템             │ │
│ │ - No new privileges             │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
✅ 격리 강화
✅ 권한 최소화
✅ 다층 방어
✅ 감사 추적

Docker 보안의 핵심 가치:

1. Least Privilege (최소 권한):
   - 비특권 사용자 실행
   - Capabilities 최소화
   - Read-only 파일시스템
   - 불필요한 바인드 마운트 제거

2. Defense in Depth (다층 방어):
   ┌──────────────────────────────┐
   │ Application Security         │
   ├──────────────────────────────┤
   │ Container Runtime (Seccomp)  │
   ├──────────────────────────────┤
   │ Image Security (Scan)        │
   ├──────────────────────────────┤
   │ Network Isolation            │
   ├──────────────────────────────┤
   │ Docker Daemon (TLS)          │
   ├──────────────────────────────┤
   │ Host OS (AppArmor)           │
   └──────────────────────────────┘
   → 한 계층 돌파해도 다른 계층이 방어

3. Immutability (불변성):
   ❌ 실행 중 수정:
   docker exec -it app bash
   apt-get install vim
   
   ✅ 이미지 재빌드:
   Dockerfile 수정 → Build → Deploy
   
4. Minimal Attack Surface:
   크기    패키지   쉘   취약점
   ubuntu:  70MB   100+  ✅   높음
   alpine:   7MB    14   ✅   중간
   distroless: 20MB   0   ❌   낮음
   scratch:  <1MB    0   ❌   최소

실무 시나리오:

공격 시나리오 1 - Container Breakout:
┌─────────────────────────────────────┐
│ Attacker in Container               │
│ (Privileged Mode)                   │
└────────────┬────────────────────────┘
             │ nsenter, chroot
┌────────────▼────────────────────────┐
│ Host Root Access                    │
│ - 모든 컨테이너 제어                    │
│ - Host 파일 접근                      │
│ - 다른 서비스 공격                      │
└─────────────────────────────────────┘

방어 메커니즘:
┌─────────────────────────────────────┐
│ Container (Unprivileged)            │
│ - User Namespace (UID 100000)       │
│ - Capabilities 제거                  │
│ - Seccomp 필터                       │
└────────────┬────────────────────────┘
             │ 탈출 시도
┌────────────▼────────────────────────┐
│ ❌ Permission Denied                │
│ ❌ Syscall Blocked                  │
│ ❌ AppArmor Violation               │
└─────────────────────────────────────┘

공격 시나리오 2 - Malicious Image:
┌─────────────────────────────────────┐
│ docker pull evil/image              │
│ docker run evil/image               │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Image 내 악성 코드                     │
│ - Cryptocurrency Miner              │
│ - Backdoor                          │
│ - Data Exfiltration                 │
└─────────────────────────────────────┘

방어 메커니즘:
┌─────────────────────────────────────┐
│ Image Security Pipeline             │
│ ┌─────────────────────────────────┐ │
│ │ 1. Vulnerability Scan           │ │
│ │    (Trivy, Clair)               │ │
│ ├─────────────────────────────────┤ │
│ │ 2. Content Trust                │ │
│ │    (Image Signature)            │ │
│ ├─────────────────────────────────┤ │
│ │ 3. Policy Enforcement           │ │
│ │    (OPA, Admission Control)     │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

---

## 🔧 실습 1: Host 보안 강화

### Step 1: 커널 보안 파라미터 확인

```bash
# 현재 보안 설정 확인
sysctl kernel.dmesg_restrict
sysctl kernel.kptr_restrict
sysctl kernel.yama.ptrace_scope
sysctl kernel.unprivileged_userns_clone

# 권장 값 확인
# kernel.dmesg_restrict = 1         (일반 사용자 커널 로그 차단)
# kernel.kptr_restrict = 2          (커널 포인터 주소 숨김)
# kernel.yama.ptrace_scope = 1      (프로세스 디버깅 제한)
# kernel.unprivileged_userns_clone = 0  (비특권 user namespace 생성 제한)
```

**출력 예시:**
```
kernel.dmesg_restrict = 0          ← 취약!
kernel.kptr_restrict = 1           ← 보통
kernel.yama.ptrace_scope = 1       ← 좋음
```

### Step 2: 보안 설정 적용

```bash
# /etc/sysctl.d/99-docker-security.conf 생성
sudo tee /etc/sysctl.d/99-docker-security.conf > /dev/null <<EOF
# Docker 보안 강화
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 1
kernel.unprivileged_userns_clone = 0

# 네트워크 보안
net.ipv4.conf.all.forwarding = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
EOF

# 적용
sudo sysctl -p /etc/sysctl.d/99-docker-security.conf

# 확인
sudo sysctl -a | grep -E 'dmesg_restrict|kptr_restrict|ptrace_scope'
```

### Step 3: AppArmor 상태 확인 (Ubuntu/Debian)

```bash
# AppArmor 설치 확인
sudo aa-status

# Docker 프로필 확인
sudo aa-status | grep docker
```

**출력 예시:**
```
apparmor module is loaded.
34 profiles are loaded.
34 profiles are in enforce mode.
   docker-default
   /usr/bin/docker
   ...
```

### Step 4: SELinux 상태 확인 (RHEL/CentOS)

```bash
# SELinux 모드 확인
getenforce

# SELinux 상태 확인
sestatus

# Docker SELinux 컨텍스트 확인
ps auxZ | grep dockerd
```

**출력 예시:**
```
Enforcing                    ← 활성화
SELinux status:  enabled
Current mode:    enforcing
Mode from config file: enforcing
```

### Step 5: User Namespace 설정

```bash
# User Namespace 지원 확인
docker info | grep "userns"

# User Namespace 활성화
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
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
  }
}
EOF

# Daemon 재시작
sudo systemctl restart docker

# 확인
docker info | grep "userns"
# Security Options:
#  userns
```

**User Namespace 효과:**
```
Without User Namespace:
┌─────────────────────────────────┐
│ Host                            │
│ UID 0 (root) ────────────────┐  │
│                              │  │
│ ┌─────────────────────┐      │  │
│ │ Container           │      │  │
│ │ UID 0 (root) ◄──────┘      │  │
│ └─────────────────────┘         │
└─────────────────────────────────┘
문제: Container root = Host root

With User Namespace:
┌─────────────────────────────────┐
│ Host                            │
│ UID 0 (root)                    │
│ UID 100000 (dockremap) ──────┐  │
│                              │  │
│ ┌─────────────────────┐      │  │
│ │ Container           │      │  │
│ │ UID 0 (root) ◄──────┘      │  │
│ │ (mapped to 100000)         │  │
│ └─────────────────────┘         │
└─────────────────────────────────┘
해결: Container root → Host 일반 사용자
```

---

## 🔧 실습 2: Docker Daemon 보안

### Step 1: Docker Socket 권한 확인

```bash
# Socket 권한 확인
ls -l /var/run/docker.sock

# 출력:
# srw-rw---- 1 root docker 0 Feb 10 10:00 /var/run/docker.sock
#
# s: socket
# rw-rw----: 660 (owner/group만 읽기/쓰기)
# root: 소유자
# docker: 그룹
```

**보안 원칙:**
```bash
# ❌ 절대 금지: 모두에게 권한 부여
sudo chmod 666 /var/run/docker.sock  # 위험!

# ❌ 절대 금지: 컨테이너에 socket 마운트
docker run -v /var/run/docker.sock:/var/run/docker.sock ...  # 위험!
```

**이유:**
```
Docker Socket = Root 권한

┌─────────────────────────────────┐
│ docker.sock 접근                 │
└────────────┬────────────────────┘
             │
             ↓
docker run --privileged -v /:/host ...
             ↓
┌─────────────────────────────────┐
│ Host 전체 파일시스템 접근            │
│ = Root 권한 획득                  │
└─────────────────────────────────┘
```

### Step 2: TLS 인증서 생성

```bash
# 작업 디렉토리 생성
mkdir -p ~/docker-certs && cd ~/docker-certs

# 1. CA 개인키 생성
openssl genrsa -aes256 -out ca-key.pem 4096

# 암호 입력 프롬프트 (예: secure-password)

# 2. CA 인증서 생성 (자체 서명)
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem

# 입력 예시:
# Country Name: KR
# State: Seoul
# Locality: Seoul
# Organization Name: MyCompany
# Organizational Unit: IT
# Common Name: Docker CA
# Email: admin@mycompany.com

# 3. 서버 개인키 생성
openssl genrsa -out server-key.pem 4096

# 4. 서버 CSR(Certificate Signing Request) 생성
openssl req -subj "/CN=docker-host" -sha256 -new -key server-key.pem -out server.csr

# 5. 서버 인증서 확장 속성 파일 생성
echo "subjectAltName = DNS:docker-host,IP:10.0.0.1,IP:127.0.0.1" > extfile.cnf
echo "extendedKeyUsage = serverAuth" >> extfile.cnf

# ⚠️ IP 주소를 실제 서버 IP로 변경

# 6. 서버 인증서 서명
openssl x509 -req -days 365 -sha256 \
  -in server.csr \
  -CA ca.pem \
  -CAkey ca-key.pem \
  -CAcreateserial \
  -out server-cert.pem \
  -extfile extfile.cnf

# 7. 클라이언트 개인키 생성
openssl genrsa -out key.pem 4096

# 8. 클라이언트 CSR 생성
openssl req -subj '/CN=client' -new -key key.pem -out client.csr

# 9. 클라이언트 확장 속성
echo "extendedKeyUsage = clientAuth" > extfile-client.cnf

# 10. 클라이언트 인증서 서명
openssl x509 -req -days 365 -sha256 \
  -in client.csr \
  -CA ca.pem \
  -CAkey ca-key.pem \
  -CAcreateserial \
  -out cert.pem \
  -extfile extfile-client.cnf

# 11. 임시 파일 제거
rm -v client.csr server.csr extfile.cnf extfile-client.cnf ca.srl

# 12. 권한 설정 (보안 강화)
chmod 0400 ca-key.pem key.pem server-key.pem
chmod 0444 ca.pem server-cert.pem cert.pem

# 13. 파일 확인
ls -l
```

**생성된 파일:**
```
ca-key.pem        (CA 개인키 - 절대 공유 금지)
ca.pem            (CA 인증서 - 클라이언트에 배포)
server-key.pem    (서버 개인키)
server-cert.pem   (서버 인증서)
key.pem           (클라이언트 개인키)
cert.pem          (클라이언트 인증서)
```

### Step 3: Docker Daemon TLS 활성화

```bash
# 인증서를 시스템 디렉토리로 복사
sudo mkdir -p /etc/docker/certs
sudo cp ca.pem server-cert.pem server-key.pem /etc/docker/certs/
sudo chmod 0400 /etc/docker/certs/server-key.pem

# daemon.json 수정
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "tls": true,
  "tlscacert": "/etc/docker/certs/ca.pem",
  "tlscert": "/etc/docker/certs/server-cert.pem",
  "tlskey": "/etc/docker/certs/server-key.pem",
  "tlsverify": true,
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"]
}
EOF

# systemd override 설정 (hosts 옵션 충돌 방지)
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/override.conf > /dev/null <<EOF
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
EOF

# Daemon 재시작
sudo systemctl daemon-reload
sudo systemctl restart docker

# 상태 확인
sudo systemctl status docker
sudo netstat -tlnp | grep 2376
```

**TLS 통신 다이어그램:**
```
Without TLS:
Client ─────→ Docker Daemon
       평문 통신 (도청 가능)

With TLS:
Client ────────────────────→ Docker Daemon
    ↓                              ↑
1. Client Cert                1. Verify Client
2. CA 서명 확인                2. Server Cert
    ↓                              ↑
    ←──────────────────────────────
         암호화 통신 (안전)
```

### Step 4: 클라이언트 TLS 설정

```bash
# 클라이언트 인증서 복사
mkdir -p ~/.docker
cp ca.pem cert.pem key.pem ~/.docker/
chmod 0400 ~/.docker/key.pem

# 환경 변수 설정
export DOCKER_HOST=tcp://docker-host:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=~/.docker

# ~/.bashrc에 추가
cat >> ~/.bashrc <<EOF

# Docker TLS
export DOCKER_HOST=tcp://docker-host:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=~/.docker
EOF

source ~/.bashrc

# 테스트
docker info

# 또는 명령어마다 옵션 지정
docker --tlsverify \
  --tlscacert=~/.docker/ca.pem \
  --tlscert=~/.docker/cert.pem \
  --tlskey=~/.docker/key.pem \
  -H=tcp://docker-host:2376 \
  ps
```

### Step 5: 방화벽 설정

```bash
# UFW (Ubuntu)
sudo ufw allow 2376/tcp comment 'Docker TLS'
sudo ufw enable

# firewalld (RHEL/CentOS)
sudo firewall-cmd --permanent --add-port=2376/tcp
sudo firewall-cmd --reload

# iptables (직접)
sudo iptables -A INPUT -p tcp --dport 2376 -j ACCEPT
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

---

## 🔧 실습 3: 최소 권한 컨테이너 실행

### Step 1: Capabilities 이해

```bash
# 기본 컨테이너의 capabilities 확인
docker run --rm alpine sh -c 'apk add -U libcap && capsh --print'

# 출력 (주요 capabilities):
# Current: = cap_chown,cap_dac_override,cap_fowner,cap_fsetid,
#            cap_kill,cap_setgid,cap_setuid,cap_setpcap,
#            cap_net_bind_service,cap_net_raw,cap_sys_chroot,
#            cap_mknod,cap_audit_write,cap_setfcap+eip
```

**Capabilities 설명:**
| Capability | 설명 | 위험도 | 제거 권장 |
|-----------|------|--------|----------|
| CAP_SYS_ADMIN | 시스템 관리 (mount, namespace 등) | 🔴 매우 높음 | ✅ |
| CAP_NET_ADMIN | 네트워크 설정 변경 | 🟠 높음 | ✅ |
| CAP_SYS_MODULE | 커널 모듈 로드 | 🔴 매우 높음 | ✅ |
| CAP_DAC_OVERRIDE | 파일 권한 무시 | 🟠 높음 | ✅ |
| CAP_NET_RAW | Raw socket 생성 (패킷 스니핑) | 🟠 높음 | ✅ |
| CAP_NET_BIND_SERVICE | 1024 미만 포트 바인딩 | 🟢 낮음 | ❌ |
| CAP_CHOWN | 파일 소유권 변경 | 🟢 낮음 | 상황에 따라 |
| CAP_SETUID | UID 변경 | 🟠 중간 | 상황에 따라 |

### Step 2: Capabilities 최소화

```bash
# 모든 capabilities 제거
docker run --rm --cap-drop=ALL alpine sh -c 'apk add -U libcap && capsh --print'

# 출력:
# Current: =
# (capabilities 없음)

# 필요한 capabilities만 추가
docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=CHOWN \
  -p 80:80 \
  nginx:alpine

# 테스트: 80 포트 바인딩 성공
curl http://localhost

# 테스트: 불필요한 작업 차단
docker run --rm --cap-drop=ALL alpine mount
# mount: permission denied (could not mount)
```

**실무 예시:**
```bash
# 웹 서버 (80 포트 필요)
docker run -d \
  --name web \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  -p 80:80 \
  nginx:alpine

# 데이터베이스 (특권 불필요)
docker run -d \
  --name db \
  --cap-drop=ALL \
  -e POSTGRES_PASSWORD=secret \
  postgres:alpine

# 로그 수집기 (파일 소유권 변경 필요)
docker run -d \
  --name logger \
  --cap-drop=ALL \
  --cap-add=CHOWN \
  --cap-add=DAC_OVERRIDE \
  -v /var/log:/logs:ro \
  fluentd:alpine
```

### Step 3: Seccomp 프로필 적용

```bash
# 기본 seccomp 프로필 확인
docker run --rm alpine grep Seccomp /proc/1/status

# 출력:
# Seccomp: 2
# (2 = 필터링 활성화)

# ❌ Seccomp 비활성화 (위험!)
docker run --rm --security-opt seccomp=unconfined alpine grep Seccomp /proc/1/status
# Seccomp: 0

# ✅ 기본 프로필 사용 (권장)
docker run --rm alpine grep Seccomp /proc/1/status
```

**커스텀 Seccomp 프로필:**
```bash
# 최소 시스템 콜만 허용하는 프로필 생성
cat > minimal-seccomp.json <<EOF
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "accept",
        "accept4",
        "access",
        "arch_prctl",
        "bind",
        "brk",
        "chmod",
        "chown",
        "clone",
        "close",
        "connect",
        "dup",
        "dup2",
        "epoll_create",
        "epoll_ctl",
        "epoll_wait",
        "execve",
        "exit",
        "exit_group",
        "fcntl",
        "fstat",
        "futex",
        "getcwd",
        "getdents",
        "getpid",
        "getsockname",
        "getsockopt",
        "ioctl",
        "listen",
        "lseek",
        "mmap",
        "mprotect",
        "munmap",
        "open",
        "openat",
        "pipe",
        "poll",
        "read",
        "readv",
        "recvfrom",
        "recvmsg",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "select",
        "sendmsg",
        "sendto",
        "set_robust_list",
        "setsockopt",
        "socket",
        "stat",
        "wait4",
        "write",
        "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
EOF

# 프로필 적용
docker run --rm \
  --security-opt seccomp=minimal-seccomp.json \
  alpine sh -c 'echo "Hello from secured container"'

# 차단된 시스템 콜 테스트
docker run --rm \
  --security-opt seccomp=minimal-seccomp.json \
  alpine mount
# Operation not permitted (mount syscall 차단됨)
```

### Step 4: AppArmor 커스텀 프로필

```bash
# 커스텀 AppArmor 프로필 생성
sudo tee /etc/apparmor.d/docker-restricted <<EOF
#include <tunables/global>

profile docker-restricted flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # 네트워크
  network inet stream,
  network inet dgram,
  
  # 파일 접근
  / r,
  /app/** rw,
  /tmp/** rw,
  /var/tmp/** rw,
  
  # 제한사항
  deny /proc/sys/** w,        # 커널 파라미터 쓰기 금지
  deny /sys/** w,              # sysfs 쓰기 금지
  deny @{PROC}/kcore r,        # 커널 메모리 읽기 금지
  deny @{PROC}/sys/kernel/** w,  # 커널 설정 쓰기 금지
  deny mount,                  # 마운트 금지
  deny umount,                 # 언마운트 금지
  deny ptrace,                 # 프로세스 추적 금지
  
  # Capabilities
  capability net_bind_service,
  capability setuid,
  capability setgid,
  capability chown,
  capability dac_override,
}
EOF

# 프로필 로드
sudo apparmor_parser -r /etc/apparmor.d/docker-restricted

# 프로필 확인
sudo aa-status | grep docker-restricted

# 프로필 적용
docker run --rm \
  --security-opt apparmor=docker-restricted \
  alpine sh -c 'echo "Restricted container"'

# 차단 테스트
docker run --rm \
  --security-opt apparmor=docker-restricted \
  alpine sh -c 'echo 1 > /proc/sys/kernel/randomize_va_space'
# sh: can't create /proc/sys/kernel/randomize_va_space: Permission denied
```

### Step 5: Read-Only 파일시스템

```bash
# Read-only 루트 파일시스템
docker run -d \
  --name readonly-nginx \
  --read-only \
  --tmpfs /var/run:rw,noexec,nosuid,size=64m \
  --tmpfs /var/cache/nginx:rw,noexec,nosuid,size=64m \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  -p 8080:80 \
  nginx:alpine

# 검증: 루트 파일시스템 쓰기 시도
docker exec readonly-nginx touch /test.txt
# touch: /test.txt: Read-only file system

# 검증: tmpfs는 쓰기 가능
docker exec readonly-nginx touch /tmp/test.txt
docker exec readonly-nginx ls -l /tmp/test.txt
# -rw-r--r--    1 root     root             0 Feb 10 10:00 /tmp/test.txt

# 마운트 정보 확인
docker exec readonly-nginx mount | grep -E '/$ |tmpfs'
```

**tmpfs 옵션 설명:**
| 옵션 | 설명 |
|-----|------|
| `rw` | 읽기/쓰기 가능 |
| `noexec` | 실행 파일 실행 금지 |
| `nosuid` | setuid/setgid 비트 무시 |
| `size=64m` | 최대 크기 제한 |

### Step 6: 비특권 사용자 실행

```dockerfile
# Dockerfile
FROM nginx:alpine

# 비특권 사용자 생성
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

# nginx를 8080 포트에서 실행하도록 설정
RUN sed -i 's/listen\s*80;/listen 8080;/' /etc/nginx/conf.d/default.conf && \
    sed -i 's/listen\s*\[::\]:80;/listen [::]:8080;/' /etc/nginx/conf.d/default.conf

# 필요한 디렉토리 권한 변경
RUN chown -R appuser:appuser \
    /var/cache/nginx \
    /var/run \
    /var/log/nginx \
    /etc/nginx/conf.d

# 비특권 사용자로 전환
USER appuser

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
```

```bash
# 빌드
docker build -t secure-nginx:unprivileged .

# 실행
docker run -d \
  --name nginx-secure \
  --user 1000:1000 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --read-only \
  --tmpfs /var/run:rw,noexec,nosuid,size=64m \
  --tmpfs /var/cache/nginx:rw,noexec,nosuid,size=64m \
  --security-opt=no-new-privileges:true \
  --security-opt apparmor=docker-default \
  -p 8080:8080 \
  secure-nginx:unprivileged

# 프로세스 확인
docker exec nginx-secure ps aux

# 출력:
# PID   USER     COMMAND
#   1   appuser  nginx: master process nginx -g daemon off
#   7   appuser  nginx: worker process
#   8   appuser  nginx: worker process

# 테스트
curl http://localhost:8080
```

### Step 7: 모든 보안 옵션 적용

```bash
# 최대 보안 컨테이너 실행
docker run -d \
  --name ultra-secure-app \
  --user 1000:1000 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --security-opt=no-new-privileges:true \
  --security-opt apparmor=docker-restricted \
  --security-opt seccomp=minimal-seccomp.json \
  --memory=256m \
  --memory-swap=256m \
  --cpus=0.5 \
  --pids-limit=100 \
  --health-cmd='wget -q --spider http://localhost:8080 || exit 1' \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  -p 8080:8080 \
  secure-nginx:unprivileged

# 보안 설정 검증
docker inspect ultra-secure-app --format='{{json .HostConfig.SecurityOpt}}' | jq .
docker inspect ultra-secure-app --format='{{json .HostConfig.CapDrop}}' | jq .
docker inspect ultra-secure-app --format='{{json .HostConfig.CapAdd}}' | jq .
docker inspect ultra-secure-app --format='{{.Config.User}}'
docker inspect ultra-secure-app --format='{{.HostConfig.ReadonlyRootfs}}'
```

**보안 계층 시각화:**
```
Ultra-Secure Container:

┌─────────────────────────────────────┐
│ Layer 7: Resource Limits            │
│ - Memory: 256MB                     │
│ - CPU: 0.5 cores                    │
│ - PIDs: 100                         │
├─────────────────────────────────────┤
│ Layer 6: Runtime Security           │
│ - User: 1000 (unprivileged)         │
│ - No new privileges                 │
├─────────────────────────────────────┤
│ Layer 5: Filesystem                 │
│ - Read-only root                    │
│ - tmpfs: noexec, nosuid             │
├─────────────────────────────────────┤
│ Layer 4: Capabilities               │
│ - Drop: ALL                         │
│ - Add: NET_BIND_SERVICE only        │
├─────────────────────────────────────┤
│ Layer 3: Seccomp                    │
│ - Minimal syscalls                  │
│ - mount, ptrace 등 차단               │
├─────────────────────────────────────┤
│ Layer 2: AppArmor                   │
│ - Restricted profile                │
│ - /proc, /sys 쓰기 차단               │
├─────────────────────────────────────┤
│ Layer 1: User Namespace             │
│ - UID remapping                     │
│ - Host isolation                    │
└─────────────────────────────────────┘
```

---

## 💡 주요 명령어 정리

### Host 보안

```bash
# 커널 파라미터
sysctl -a | grep kernel
sysctl -w kernel.dmesg_restrict=1
sysctl -p /etc/sysctl.d/99-docker-security.conf

# AppArmor
sudo aa-status
sudo aa-enforce /etc/apparmor.d/docker-default
sudo apparmor_parser -r /etc/apparmor.d/docker-custom

# SELinux
getenforce
sestatus
setenforce Enforcing

# User Namespace
docker info | grep userns
```

### Daemon 보안

```bash
# TLS 인증서 생성
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem

# Daemon 설정
sudo vim /etc/docker/daemon.json
sudo systemctl restart docker

# Socket 권한
ls -l /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock
```

### Container 보안

```bash
# Capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
docker run --rm alpine sh -c 'apk add -U libcap && capsh --print'

# Seccomp
docker run --security-opt seccomp=profile.json alpine
docker run --rm alpine grep Seccomp /proc/1/status

# AppArmor
docker run --security-opt apparmor=docker-restricted alpine

# Read-only
docker run --read-only --tmpfs /tmp nginx

# User
docker run --user 1000:1000 nginx

# No new privileges
docker run --security-opt=no-new-privileges:true nginx

# 복합 적용
docker run -d \
  --user 1000:1000 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  --security-opt=no-new-privileges:true \
  --security-opt apparmor=docker-default \
  --security-opt seccomp=custom.json \
  nginx
```

---

## 🎓 연습 문제

### 문제 1: 보안 강화 웹 서버

다음 요구사항을 만족하는 nginx 컨테이너를 실행하세요:

1. 비특권 사용자 (UID 1000) 실행
2. Read-only 파일시스템
3. 모든 Capabilities 제거 후 필요한 것만 추가
4. tmpfs는 noexec, nosuid 옵션 적용
5. AppArmor 프로필 적용
6. No new privileges

<details>
<summary>힌트 보기</summary>

- Dockerfile에서 사용자 생성 및 포트 변경
- --cap-drop=ALL, --cap-add=NET_BIND_SERVICE
- --read-only, --tmpfs
- --security-opt 활용

</details>

<details>
<summary>정답 보기</summary>

```dockerfile
# Dockerfile
FROM nginx:alpine

RUN addgroup -g 1000 webuser && \
    adduser -D -u 1000 -G webuser webuser && \
    sed -i 's/listen       80;/listen       8080;/' /etc/nginx/conf.d/default.conf && \
    chown -R webuser:webuser /var/cache/nginx /var/run /var/log/nginx

USER webuser
EXPOSE 8080
```

```bash
# 빌드
docker build -t secure-nginx .

# 실행
docker run -d \
  --name secure-web \
  --user 1000:1000 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --read-only \
  --tmpfs /var/run:rw,noexec,nosuid,size=64m \
  --tmpfs /var/cache/nginx:rw,noexec,nosuid,size=64m \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --security-opt=no-new-privileges:true \
  --security-opt apparmor=docker-default \
  -p 8080:8080 \
  secure-nginx

# 검증
docker exec secure-web ps aux
docker exec secure-web touch /test.txt  # 실패해야 함
curl http://localhost:8080  # 성공해야 함
```

</details>

### 문제 2: Seccomp 프로필 작성

다음 시스템 콜만 허용하는 seccomp 프로필을 작성하고 테스트하세요:

- 파일 I/O: read, write, open, close, stat
- 메모리: mmap, munmap, brk
- 프로세스: exit, exit_group, clone, execve
- 네트워크: socket, bind, listen, accept, connect

<details>
<summary>정답 보기</summary>

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "read",
        "write",
        "open",
        "openat",
        "close",
        "stat",
        "fstat",
        "lstat",
        "mmap",
        "munmap",
        "brk",
        "exit",
        "exit_group",
        "clone",
        "execve",
        "socket",
        "bind",
        "listen",
        "accept",
        "accept4",
        "connect",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

```bash
# 파일로 저장
cat > basic-seccomp.json <<'EOF'
...
EOF

# 테스트
docker run --rm \
  --security-opt seccomp=basic-seccomp.json \
  alpine sh -c 'echo "Hello"'

# mount 차단 확인
docker run --rm \
  --security-opt seccomp=basic-seccomp.json \
  alpine mount
# Operation not permitted
```

</details>

### 문제 3: AppArmor 프로필 생성

다음 제약사항을 가진 AppArmor 프로필을 작성하세요:

1. /app 디렉토리만 쓰기 가능
2. /proc/sys, /sys 쓰기 금지
3. mount, umount 금지
4. ptrace 금지
5. 네트워크 허용 (inet stream, dgram)

<details>
<summary>정답 보기</summary>

```bash
# /etc/apparmor.d/docker-app-restricted
sudo tee /etc/apparmor.d/docker-app-restricted <<EOF
#include <tunables/global>

profile docker-app-restricted flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # 네트워크 허용
  network inet stream,
  network inet dgram,
  network inet6 stream,
  network inet6 dgram,
  
  # 파일 접근
  / r,
  /app/** rw,
  /tmp/** rw,
  /var/tmp/** rw,
  /dev/null rw,
  /dev/zero rw,
  /dev/urandom r,
  
  # 제한사항
  deny /proc/sys/** w,
  deny /sys/** w,
  deny @{PROC}/kcore r,
  deny mount,
  deny umount,
  deny ptrace,
  
  # Capabilities
  capability net_bind_service,
  capability setuid,
  capability setgid,
}
EOF

# 로드
sudo apparmor_parser -r /etc/apparmor.d/docker-app-restricted

# 테스트
docker run --rm \
  --security-opt apparmor=docker-app-restricted \
  -v /tmp/app:/app \
  alpine sh -c 'echo "test" > /app/test.txt && cat /app/test.txt'

# /proc/sys 쓰기 차단 확인
docker run --rm \
  --security-opt apparmor=docker-app-restricted \
  alpine sh -c 'echo 1 > /proc/sys/kernel/randomize_va_space'
# Permission denied
```

</details>

### 문제 4: Docker Daemon TLS 설정

Docker Daemon에 TLS 인증을 설정하고 원격에서 안전하게 접속하세요.

<details>
<summary>힌트 보기</summary>

1. CA 인증서 생성
2. 서버 인증서 생성 (Subject Alternative Name 포함)
3. 클라이언트 인증서 생성
4. daemon.json 설정
5. systemd override
6. 클라이언트 환경 변수 설정

</details>

<details>
<summary>정답 보기</summary>

```bash
# 1. 인증서 생성 (위의 Step 2 참고)
mkdir ~/docker-tls && cd ~/docker-tls

# CA 키 및 인증서
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem

# 서버 인증서
openssl genrsa -out server-key.pem 4096
openssl req -subj "/CN=docker-host" -sha256 -new -key server-key.pem -out server.csr
echo "subjectAltName = DNS:docker-host,IP:10.0.0.1,IP:127.0.0.1" > extfile.cnf
echo "extendedKeyUsage = serverAuth" >> extfile.cnf
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -extfile extfile.cnf

# 클라이언트 인증서
openssl genrsa -out key.pem 4096
openssl req -subj '/CN=client' -new -key key.pem -out client.csr
echo "extendedKeyUsage = clientAuth" > extfile-client.cnf
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out cert.pem -extfile extfile-client.cnf

# 권한 설정
chmod 0400 ca-key.pem key.pem server-key.pem
chmod 0444 ca.pem server-cert.pem cert.pem

# 2. Daemon 설정
sudo mkdir -p /etc/docker/certs
sudo cp ca.pem server-cert.pem server-key.pem /etc/docker/certs/

sudo tee /etc/docker/daemon.json <<EOF
{
  "tls": true,
  "tlscacert": "/etc/docker/certs/ca.pem",
  "tlscert": "/etc/docker/certs/server-cert.pem",
  "tlskey": "/etc/docker/certs/server-key.pem",
  "tlsverify": true,
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"]
}
EOF

# 3. systemd override
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/override.conf <<EOF
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
EOF

# 4. 재시작
sudo systemctl daemon-reload
sudo systemctl restart docker

# 5. 클라이언트 설정
mkdir -p ~/.docker
cp ca.pem cert.pem key.pem ~/.docker/

export DOCKER_HOST=tcp://localhost:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=~/.docker

# 6. 테스트
docker info
```

</details>

---

## 📌 핵심 요약

### 보안 4대 원칙

1. **Least Privilege (최소 권한)**
   ```
   - 비특권 사용자 실행 (--user 1000:1000)
   - Capabilities 최소화 (--cap-drop=ALL)
   - Read-only 파일시스템 (--read-only)
   - No new privileges (--security-opt=no-new-privileges:true)
   ```

2. **Defense in Depth (다층 방어)**
   ```
   Application → Container Runtime → Image → Network → Daemon → Host
   각 계층에서 독립적인 보안 메커니즘 적용
   ```

3. **Immutability (불변성)**
   ```
   실행 중 수정 금지 → 이미지 재빌드 → 재배포
   ```

4. **Minimal Attack Surface (최소 공격 표면)**
   ```
   distroless/scratch 이미지 사용
   불필요한 패키지/도구 제거
   쉘 제거
   ```

### 보안 도구 매트릭스

| 계층 | 도구 | 목적 | 명령어 |
|-----|------|------|--------|
| Host | AppArmor/SELinux | MAC | `--security-opt apparmor=profile` |
| Host | User Namespace | UID 매핑 | `"userns-remap": "default"` |
| Runtime | Seccomp | Syscall 필터 | `--security-opt seccomp=profile.json` |
| Runtime | Capabilities | 권한 세분화 | `--cap-drop=ALL --cap-add=...` |
| Daemon | TLS | 통신 암호화 | `"tlsverify": true` |
| Network | Firewalls | 접근 제어 | `ufw`, `iptables` |

### 실무 체크리스트

**Host:**
- [ ] AppArmor/SELinux 활성화
- [ ] User namespace 사용
- [ ] 커널 파라미터 강화 (dmesg_restrict, kptr_restrict)
- [ ] docker 그룹 멤버 최소화

**Daemon:**
- [ ] TLS 인증 설정 (2376 포트)
- [ ] Socket 권한 관리 (660)
- [ ] 불필요한 API 비활성화
- [ ] 로그 중앙 집중화

**Container:**
- [ ] 비특권 사용자 실행 (--user)
- [ ] Capabilities 최소화 (--cap-drop=ALL)
- [ ] Seccomp 프로필 (기본 또는 커스텀)
- [ ] Read-only 파일시스템 (--read-only)
- [ ] No new privileges
- [ ] 리소스 제한 (--memory, --cpus)

**Image:**
- [ ] 최소 베이스 이미지 (alpine, distroless)
- [ ] 취약점 스캔 (Trivy, Clair)
- [ ] Multi-stage build
- [ ] .dockerignore 사용
- [ ] 레이어 최소화

**Network:**
- [ ] 네트워크 분리 (custom networks)
- [ ] 불필요한 포트 노출 금지
- [ ] TLS/mTLS 사용
- [ ] Ingress 네트워크 보안

### 보안 vs 편의성 트레이드오프

| 설정 | 보안 | 편의성 | 권장 |
|-----|------|--------|------|
| Root 사용자 | ❌ 낮음 | ✅ 높음 | 개발 환경만 |
| Privileged | ❌ 매우 낮음 | ✅ 매우 높음 | 특수 케이스만 |
| 기본 Capabilities | 🟠 중간 | ✅ 높음 | 일반적 |
| Capabilities 최소화 | ✅ 높음 | ❌ 낮음 | 프로덕션 |
| Read-only FS | ✅ 높음 | 🟠 중간 | 프로덕션 |
| Seccomp 기본 | ✅ 높음 | ✅ 높음 | 항상 |
| Seccomp 커스텀 | ✅ 매우 높음 | ❌ 낮음 | 고보안 환경 |

---

<div align="center">

**[다음: Image Security ➡️](./02-Image-Security.md)**

</div>
