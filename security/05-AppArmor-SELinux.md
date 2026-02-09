# 05. AppArmor & SELinux - 강제 접근 제어

## 🎯 이 챕터에서 배울 것

- **MAC (Mandatory Access Control)** - 강제 접근 제어 개념
- **AppArmor** - 프로필 기반 접근 제어
- **SELinux** - 컨텍스트 기반 보안
- **프로필 작성** - 커스텀 보안 정책
- **실무 적용** - 프로덕션 환경 설정

## 📌 왜 중요한가?

**"MAC 시스템은 DAC의 한계를 극복하고 커널 레벨에서 강제적으로 보안을 적용합니다."**

```
DAC vs MAC 비교:

DAC (Discretionary Access Control):
┌─────────────────────────────────────┐
│ Traditional Linux Permissions       │
│                                     │
│ Owner: rwx (읽기/쓰기/실행)            │
│ Group: r-x                          │
│ Other: r--                          │
│                                     │
│ 문제:                                │
│ 1. 소유자가 권한 변경 가능               │
│ 2. Root는 모든 제한 우회               │
│ 3. 프로세스가 사용자 권한 상속            │
│ 4. 세밀한 제어 불가능                   │
└─────────────────────────────────────┘

예시 취약점:
┌─────────────────────────────────────┐
│ Container (root)                    │
│ chmod 777 /host-volume              │ ✅ 성공
│ cat /proc/sys/kernel/randomize_...  │ ✅ 성공
│ mount /dev/sda1 /mnt                │ ✅ 성공 (CAP_SYS_ADMIN)
└─────────────────────────────────────┘
❌ DAC만으로는 격리 불충분

MAC (Mandatory Access Control):
┌─────────────────────────────────────┐
│ AppArmor / SELinux                  │
│                                     │
│ 커널 레벨 강제 정책                     │
│ - 관리자만 정책 변경 가능                │
│ - Root도 정책 우회 불가능               │
│ - 프로세스별 세밀한 제어                 │
│ - 파일/네트워크/Capabilities 제한       │
└─────────────────────────────────────┘

AppArmor 프로필 적용:
┌─────────────────────────────────────┐
│ Container (root + AppArmor)         │
│ chmod 777 /host-volume              │ ❌ Denied
│ cat /proc/sys/kernel/randomize_...  │ ❌ Denied
│ mount /dev/sda1 /mnt                │ ❌ Denied
│                                     │
│ /app/** rw                          │ ✅ Allowed
│ /tmp/** rw                          │ ✅ Allowed
└─────────────────────────────────────┘
✅ MAC로 강제 격리

AppArmor vs SELinux:

AppArmor (Path-based):
┌─────────────────────────────────────┐
│ Profile: docker-nginx               │
│                                     │
│ /etc/nginx/** r,                    │
│ /var/log/nginx/** w,                │
│ /usr/sbin/nginx ix,                 │
│                                     │
│ deny /proc/sys/** w,                │
│ deny /sys/** w,                     │
│                                     │
│ 장점:                                │
│ - 경로 기반으로 이해 쉬움                │
│ - 프로필 작성 간단                      │
│ - Ubuntu 기본 탑재                    │
│                                     │
│ 단점:                                │
│ - 심볼릭 링크 취약                      │
│ - SELinux보다 제어 범위 좁음            │
└─────────────────────────────────────┘

SELinux (Label-based):
┌─────────────────────────────────────┐
│ Context: container_t                │
│                                     │
│ Type Enforcement:                   │
│ allow container_t container_file_t  │
│   : file { read write };            │
│                                     │
│ File Context:                       │
│ /var/lib/docker/  system_u:object_r │
│   :container_file_t:s0              │
│                                     │
│ 장점:                                │
│ - 레이블 기반으로 우회 어려움              │
│ - 매우 세밀한 제어                      │
│ - RHEL/CentOS 기본                   │
│                                     │
│ 단점:                                │
│ - 학습 곡선 높음                       │
│ - 정책 작성 복잡                       │
│ - 디버깅 어려움                        │
└─────────────────────────────────────┘

실무 시나리오:

공격 시나리오 - DAC만 적용:
┌─────────────────────────────────────┐
│ 1. 공격자가 컨테이너 진입                 │
│    docker exec -it webapp bash      │
│    (취약점 exploit 후)                │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. 권한 상승 시도                      │
│    find / -perm -4000 2>/dev/null   │
│    (setuid 바이너리 찾기)              │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 호스트 파일시스템 접근                │
│    cat /proc/sys/kernel/core_...    │
│    echo "malicious" > /etc/passwd   │
│    → 성공! (DAC 우회)                 │
└─────────────────────────────────────┘

방어 - MAC 적용:
┌─────────────────────────────────────┐
│ 1. 공격자가 컨테이너 진입                │
│    docker exec -it webapp bash      │
│    (AppArmor 프로필 적용됨)            │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. 권한 상승 시도                      │
│    find / -perm -4000 2>/dev/null   │
│    → AppArmor: Permission denied    │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 호스트 파일시스템 접근                 │
│    cat /proc/sys/kernel/core_...    │
│    → AppArmor: Permission denied    │
│    echo "test" > /etc/passwd        │
│    → AppArmor: Permission denied    │
│    ✅ 공격 차단!                      │
└─────────────────────────────────────┘

보안 계층 구조:
┌──────────────────────────────────────┐
│ Layer 4: Application Security        │
│ (입력 검증, 인증/인가)                   │
├──────────────────────────────────────┤
│ Layer 3: MAC (AppArmor/SELinux)      │
│ (강제 접근 제어)                        │
├──────────────────────────────────────┤
│ Layer 2: Seccomp + Capabilities      │
│ (시스템 콜 제한)                        │
├──────────────────────────────────────┤
│ Layer 1: Namespace + cgroup          │
│ (프로세스 격리)                         │
├──────────────────────────────────────┤
│ Layer 0: DAC (파일 권한)               │
│ (기본 접근 제어)                        │
└──────────────────────────────────────┘

Defense in Depth:
각 계층이 독립적으로 작동하여
하나가 뚫려도 다른 계층이 방어
```

**실무 영향:**
- Container Escape 방지 → 보안 사고 95% 감소
- 제로데이 대응 → 알려지지 않은 취약점도 차단
- 규정 준수 → PCI-DSS, HIPAA 요구사항
- 감사 추적 → 모든 거부된 접근 기록

---

## 🔧 실습 1: AppArmor 기본

### Step 1: AppArmor 설치 및 확인

```bash
# Ubuntu/Debian에서 AppArmor 확인
sudo systemctl status apparmor

# 출력:
# ● apparmor.service - Load AppArmor profiles
#      Loaded: loaded
#      Active: active (exited)

# AppArmor 모듈 확인
sudo aa-status

# 출력:
# apparmor module is loaded.
# 34 profiles are loaded.
# 34 profiles are in enforce mode.
#    docker-default
#    /usr/bin/docker
#    ...

# Docker 기본 프로필 확인
sudo aa-status | grep docker

# 커널 파라미터 확인
cat /sys/module/apparmor/parameters/enabled
# Y

# 컨테이너의 AppArmor 프로필 확인
docker run --rm alpine cat /proc/1/attr/current
# docker-default (enforce)
```

### Step 2: 기본 프로필 분석

```bash
# Docker 기본 프로필 위치
sudo cat /etc/apparmor.d/docker

# 또는
docker run --rm alpine sh -c 'cat /proc/1/attr/current'

# 기본 프로필의 주요 규칙:
# - /proc/sys/** w 금지
# - /sys/** w 금지
# - mount/umount 금지
# - ptrace 제한
# - signal 제한

# 프로필이 차단하는 것 테스트
docker run --rm alpine sh -c 'echo 1 > /proc/sys/kernel/randomize_va_space'
# sh: can't create /proc/sys/kernel/randomize_va_space: Read-only file system
# (실제로는 AppArmor가 차단)

# AppArmor 로그 확인
sudo dmesg | grep apparmor | tail -20
# 또는
sudo journalctl -k | grep apparmor | tail -20
```

### Step 3: 커스텀 프로필 작성

```bash
# 웹 애플리케이션용 프로필
sudo tee /etc/apparmor.d/docker-webapp <<'EOF'
#include <tunables/global>

profile docker-webapp flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # 네트워크 허용
  network inet stream,
  network inet dgram,
  network inet6 stream,
  network inet6 dgram,
  network unix stream,
  network unix dgram,

  # 파일시스템 접근
  / r,
  /app/** rw,
  /tmp/** rw,
  /var/tmp/** rw,
  /var/log/app/** w,
  /run/** rw,
  
  # 실행 파일
  /bin/** rix,
  /usr/bin/** rix,
  /lib/** rix,
  /usr/lib/** rix,
  
  # 디바이스
  /dev/null rw,
  /dev/zero rw,
  /dev/random r,
  /dev/urandom r,
  /dev/tty rw,
  /dev/pts/** rw,
  
  # /proc 읽기 허용
  /proc/ r,
  /proc/** r,
  /proc/sys/kernel/hostname r,
  /proc/sys/kernel/domainname r,
  
  # /sys 읽기 허용
  /sys/fs/cgroup/** r,
  
  # 명시적 거부
  deny /proc/sys/** w,              # 커널 파라미터 쓰기
  deny /sys/** w,                    # sysfs 쓰기
  deny @{PROC}/kcore r,              # 커널 메모리
  deny @{PROC}/kmem r,
  deny @{PROC}/mem r,
  deny @{PROC}/sys/kernel/modprobe w,
  deny mount,                        # 마운트 금지
  deny umount,                       # 언마운트 금지
  deny pivot_root,                   # pivot_root 금지
  deny ptrace (trace),               # 프로세스 추적 제한
  
  # Capabilities
  capability chown,
  capability dac_override,
  capability fowner,
  capability fsetid,
  capability kill,
  capability setgid,
  capability setuid,
  capability setpcap,
  capability net_bind_service,
  capability sys_chroot,
  capability mknod,
  capability audit_write,
  capability setfcap,
  
  # 위험한 Capabilities 거부
  deny capability sys_admin,
  deny capability sys_module,
  deny capability sys_rawio,
  deny capability sys_boot,
  deny capability sys_time,
  deny capability mac_admin,
  deny capability mac_override,
}
EOF

# 프로필 파싱 및 로드
sudo apparmor_parser -r /etc/apparmor.d/docker-webapp

# 프로필 확인
sudo aa-status | grep docker-webapp
#   docker-webapp (enforce)
```

### Step 4: 프로필 적용 및 테스트

```bash
# 프로필 적용하여 컨테이너 실행
docker run --rm \
  --security-opt apparmor=docker-webapp \
  --name test-webapp \
  alpine sh

# 컨테이너 내부에서 테스트:

# 1. 허용된 작업
echo "test" > /app/test.txt           # ✅ 성공
cat /proc/cpuinfo                     # ✅ 성공
ls /tmp                               # ✅ 성공

# 2. 거부된 작업
echo 1 > /proc/sys/kernel/randomize_va_space  # ❌ Permission denied
mount -t tmpfs tmpfs /mnt             # ❌ Permission denied
echo 1 > /sys/kernel/debug/tracing/tracing_on # ❌ Permission denied

# AppArmor 거부 로그 확인
sudo dmesg | grep -i apparmor | grep -i denied | tail -10

# 출력 예시:
# audit: type=1400 audit(...): apparmor="DENIED" operation="mount" 
#   profile="docker-webapp" name="/mnt/" pid=12345 comm="mount"
```

### Step 5: Complain 모드 (학습 모드)

```bash
# Complain 모드 프로필 (차단하지 않고 로깅만)
sudo tee /etc/apparmor.d/docker-webapp-complain <<'EOF'
#include <tunables/global>

profile docker-webapp-complain flags=(attach_disconnected,mediate_deleted,complain) {
  #include <abstractions/base>
  
  # 모든 것을 허용하되 로깅
}
EOF

# 로드
sudo apparmor_parser -r /etc/apparmor.d/docker-webapp-complain

# Complain 모드로 실행
docker run --rm \
  --security-opt apparmor=docker-webapp-complain \
  alpine sh -c '
    mount -t tmpfs tmpfs /mnt
    echo 1 > /proc/sys/kernel/randomize_va_space
    echo "All allowed in complain mode"
  '

# 로그 확인하여 필요한 권한 파악
sudo aa-logprof

# aa-logprof 사용:
# - 로그를 분석하여 필요한 권한 제안
# - 대화형으로 프로필 업데이트
# - Enforce 모드로 전환 가능
```

---

## 🔧 실습 2: SELinux 기본

### Step 1: SELinux 설치 및 확인 (RHEL/CentOS)

```bash
# SELinux 상태 확인
getenforce

# 출력:
# Enforcing    (강제 모드)
# Permissive   (허용 모드, 로깅만)
# Disabled     (비활성화)

# 상세 상태
sestatus

# 출력:
# SELinux status:                 enabled
# SELinuxfs mount:                /sys/fs/selinux
# SELinux root directory:         /etc/selinux
# Loaded policy name:             targeted
# Current mode:                   enforcing
# Mode from config file:          enforcing
# Policy MLS status:              enabled
# Policy deny_unknown status:     allowed
# Memory protection checking:     actual (secure)
# Max kernel policy version:      33

# Docker와 SELinux
docker info | grep -i selinux
# Security Options:
#  selinux

# 프로세스의 SELinux 컨텍스트
ps auxZ | grep dockerd
# system_u:system_r:container_runtime_t:s0 root ... /usr/bin/dockerd
```

### Step 2: SELinux 컨텍스트 이해

```bash
# 파일 컨텍스트 확인
ls -Z /var/lib/docker

# 출력:
# system_u:object_r:container_var_lib_t:s0 containers
# system_u:object_r:container_var_lib_t:s0 image
# system_u:object_r:container_var_lib_t:s0 volumes

# 컨테이너 프로세스 컨텍스트
docker run -d --name test-nginx nginx
ps auxZ | grep nginx

# 출력:
# system_u:system_r:container_t:s0:c123,c456 root ... nginx

# SELinux 컨텍스트 형식:
# user:role:type:level
#
# user    - system_u, unconfined_u, user_u
# role    - system_r, object_r, unconfined_r
# type    - container_t, container_file_t (주요 보안 속성)
# level   - s0 (감도 레벨), c123,c456 (카테고리)
```

### Step 3: SELinux 정책 확인

```bash
# 컨테이너 관련 타입 확인
seinfo -t | grep container

# 출력:
# container_t
# container_file_t
# container_runtime_t
# container_init_t
# container_var_lib_t
# ...

# 특정 타입의 규칙 확인
sesearch -A -s container_t -t container_file_t

# 출력:
# allow container_t container_file_t:file { read write open create ... };
# allow container_t container_file_t:dir { read write add_name ... };

# 거부 규칙 확인
sesearch -A -s container_t -t etc_t

# 일반적으로 제한됨:
# 컨테이너는 /etc 파일에 쓰기 불가
```

### Step 4: 볼륨 마운트와 SELinux

```bash
# 문제 상황: SELinux 때문에 볼륨 접근 불가
docker run --rm -v /tmp/data:/data alpine ls /data
# ls: /data: Permission denied

# 해결 방법 1: :z 플래그 (private label)
docker run --rm -v /tmp/data:/data:z alpine sh -c 'echo "test" > /data/file.txt'
# 성공

# 해결 방법 2: :Z 플래그 (shared label)
docker run --rm -v /tmp/data:/data:Z alpine ls /data

# 차이점:
# :z  - 여러 컨테이너가 공유 가능
# :Z  - 단일 컨테이너만 접근 (더 안전)

# 레이블 확인
ls -Z /tmp/data
# system_u:object_r:container_file_t:s0:c123,c456 /tmp/data

# 수동 레이블 변경
sudo chcon -Rt container_file_t /tmp/data

# 또는 semanage로 영구적 설정
sudo semanage fcontext -a -t container_file_t "/tmp/data(/.*)?"
sudo restorecon -Rv /tmp/data
```

### Step 5: SELinux 비활성화 (비권장)

```bash
# Permissive 모드로 전환 (임시)
sudo setenforce 0
getenforce
# Permissive

# 다시 Enforcing으로
sudo setenforce 1

# 영구적 비활성화 (재부팅 필요, 비권장)
sudo sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sudo reboot

# 특정 도메인만 Permissive
sudo semanage permissive -a container_t

# 확인
semodule -l | grep permissive
```

---

## 🔧 실습 3: 고급 AppArmor 프로필

### Step 1: 데이터베이스 프로필

```bash
# PostgreSQL 프로필
sudo tee /etc/apparmor.d/docker-postgres <<'EOF'
#include <tunables/global>

profile docker-postgres flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/postgres>

  # 네트워크
  network inet stream,
  network inet6 stream,
  network unix stream,

  # PostgreSQL 데이터 디렉토리
  /var/lib/postgresql/** rwk,
  /var/run/postgresql/** rwk,
  
  # 설정 파일
  /etc/postgresql/** r,
  
  # 로그
  /var/log/postgresql/** w,
  
  # 임시 파일
  /tmp/** rw,
  /var/tmp/** rw,
  
  # 실행 파일
  /usr/lib/postgresql/** rix,
  /usr/bin/postgres rix,
  /usr/bin/pg_* rix,
  
  # 디바이스
  /dev/null rw,
  /dev/zero rw,
  /dev/urandom r,
  /dev/shm/** rw,
  
  # /proc
  /proc/ r,
  /proc/** r,
  
  # Deny dangerous operations
  deny /proc/sys/** w,
  deny /sys/** w,
  deny mount,
  deny umount,
  deny ptrace,
  
  # Capabilities (PostgreSQL specific)
  capability chown,
  capability dac_override,
  capability fowner,
  capability fsetid,
  capability setgid,
  capability setuid,
  capability sys_resource,
  capability ipc_lock,
  
  # Deny dangerous capabilities
  deny capability sys_admin,
  deny capability sys_module,
  deny capability sys_rawio,
}
EOF

sudo apparmor_parser -r /etc/apparmor.d/docker-postgres

# 테스트
docker run -d \
  --name secure-postgres \
  --security-opt apparmor=docker-postgres \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:alpine

# 검증
docker logs secure-postgres
# PostgreSQL init process complete; ready for start up.

docker exec secure-postgres psql -U postgres -c "SELECT version();"
# PostgreSQL 15.x ...
```

### Step 2: 네트워크 제한 프로필

```bash
# 외부 네트워크 차단 프로필
sudo tee /etc/apparmor.d/docker-isolated <<'EOF'
#include <tunables/global>

profile docker-isolated flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # 로컬 네트워크만 허용
  network unix stream,
  network unix dgram,
  
  # 외부 네트워크 차단
  deny network inet,
  deny network inet6,
  
  # 파일시스템
  /app/** rw,
  /tmp/** rw,
  
  # /proc, /sys 쓰기 금지
  deny /proc/sys/** w,
  deny /sys/** w,
  deny mount,
  deny umount,
}
EOF

sudo apparmor_parser -r /etc/apparmor.d/docker-isolated

# 테스트
docker run --rm \
  --security-opt apparmor=docker-isolated \
  alpine ping -c 1 google.com
# ping: socket: Permission denied

docker run --rm \
  --security-opt apparmor=docker-isolated \
  alpine wget http://google.com
# wget: socket: Permission denied
```

### Step 3: 읽기 전용 프로필

```bash
# 파일시스템 읽기만 가능한 프로필
sudo tee /etc/apparmor.d/docker-readonly <<'EOF'
#include <tunables/global>

profile docker-readonly flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # 네트워크
  network inet stream,
  network inet6 stream,

  # 읽기 전용
  /** r,
  
  # 쓰기 허용 (최소한)
  /tmp/** rw,
  /dev/null w,
  /dev/zero w,
  /proc/self/fd/** w,
  
  # 나머지 모두 쓰기 금지
  deny /** w,
  
  # Capabilities
  capability setgid,
  capability setuid,
  
  # 위험한 작업 금지
  deny mount,
  deny umount,
  deny ptrace,
  deny /proc/sys/** w,
  deny /sys/** w,
}
EOF

sudo apparmor_parser -r /etc/apparmor.d/docker-readonly

# 테스트
docker run --rm \
  --security-opt apparmor=docker-readonly \
  alpine sh -c 'echo "test" > /tmp/file.txt && cat /tmp/file.txt'
# test

docker run --rm \
  --security-opt apparmor=docker-readonly \
  alpine sh -c 'echo "fail" > /etc/test.txt'
# sh: can't create /etc/test.txt: Permission denied
```

---

## 🔧 실습 4: SELinux 커스텀 정책

### Step 1: 커스텀 타입 생성

```bash
# 커스텀 컨테이너 타입 정책 파일 생성
cat > myapp_container.te <<'EOF'
policy_module(myapp_container, 1.0.0)

require {
    type container_t;
    type container_file_t;
    type http_port_t;
    class tcp_socket name_bind;
    class file { read write open };
}

# 새로운 타입 정의
type myapp_container_t;
type myapp_container_file_t;

# container_t에서 전환 허용
allow container_t myapp_container_t:process transition;

# myapp_container_t가 파일 읽기/쓰기 가능
allow myapp_container_t myapp_container_file_t:file { read write open create };

# HTTP 포트 바인딩 허용
allow myapp_container_t http_port_t:tcp_socket name_bind;

# /tmp 디렉토리 접근
allow myapp_container_t tmp_t:dir { read write add_name };
allow myapp_container_t tmp_t:file { read write create open };
EOF

# 정책 컴파일 및 로드
checkmodule -M -m -o myapp_container.mod myapp_container.te
semodule_package -o myapp_container.pp -m myapp_container.mod
sudo semodule -i myapp_container.pp

# 확인
semodule -l | grep myapp_container
# myapp_container	1.0.0
```

### Step 2: 파일 컨텍스트 관리

```bash
# 애플리케이션 디렉토리 레이블 설정
sudo mkdir -p /opt/myapp
sudo semanage fcontext -a -t myapp_container_file_t "/opt/myapp(/.*)?"
sudo restorecon -Rv /opt/myapp

# 확인
ls -Z /opt/myapp
# unconfined_u:object_r:myapp_container_file_t:s0 /opt/myapp

# 컨테이너에서 사용
docker run --rm \
  -v /opt/myapp:/app:z \
  --security-opt label=type:myapp_container_t \
  alpine ls -Z /app

# 레이블이 올바르게 적용되었는지 확인
```

### Step 3: 포트 레이블 관리

```bash
# 커스텀 포트 레이블 추가
# (예: 8080 포트를 http_port_t로 설정)
sudo semanage port -a -t http_port_t -p tcp 8080

# 확인
sudo semanage port -l | grep http_port_t
# http_port_t    tcp    80, 443, 8080, ...

# 컨테이너에서 8080 포트 바인딩 테스트
docker run -d \
  --name web8080 \
  --security-opt label=type:myapp_container_t \
  -p 8080:80 \
  nginx:alpine

# 성공적으로 바인딩됨
curl http://localhost:8080
```

### Step 4: SELinux 문제 해결

```bash
# 1. audit 로그 확인
sudo ausearch -m avc -ts recent

# 출력:
# type=AVC msg=audit(...): avc:  denied  { write } for  pid=12345 
#   comm="nginx" name="access.log" dev="sda1" ino=98765 
#   scontext=system_u:system_r:container_t:s0 
#   tcontext=system_u:object_r:var_log_t:s0 tclass=file permissive=0

# 2. audit2allow로 정책 생성
sudo ausearch -m avc -ts recent | audit2allow -M myapp_fix

# 출력:
# ******************** IMPORTANT ***********************
# To make this policy package active, execute:
# 
# semodule -i myapp_fix.pp

# 3. 생성된 정책 확인
cat myapp_fix.te

# 4. 정책 적용
sudo semodule -i myapp_fix.pp

# 5. 재테스트
```

---

## 🔧 실습 5: 프로덕션 배포

### Step 1: Docker Compose with AppArmor

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    security_opt:
      - apparmor:docker-webapp
    volumes:
      - ./html:/usr/share/nginx/html:ro
    ports:
      - "80:80"
    deploy:
      replicas: 3

  api:
    image: myapi:latest
    security_opt:
      - apparmor:docker-webapp
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    volumes:
      - api-data:/app/data
    environment:
      - DB_HOST=db
    depends_on:
      - db

  db:
    image: postgres:alpine
    security_opt:
      - apparmor:docker-postgres
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - SETGID
      - SETUID
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password

volumes:
  api-data:
  db-data:

secrets:
  db_password:
    external: true
```

### Step 2: Swarm with AppArmor

```bash
# Stack 배포 스크립트
cat > deploy.sh <<'EOF'
#!/bin/bash

set -e

echo "Deploying AppArmor profiles..."
for node in $(docker node ls -q); do
    # 각 노드에 프로필 복사
    docker node update --label-add apparmor=enabled $node
done

echo "Creating secrets..."
echo "$(openssl rand -base64 32)" | docker secret create db_password -

echo "Deploying stack..."
docker stack deploy -c docker-compose.yml myapp

echo "Verifying deployment..."
sleep 10
docker stack services myapp

echo "Checking AppArmor..."
for container in $(docker ps -q --filter "label=com.docker.stack.namespace=myapp"); do
    echo "Container: $container"
    docker exec $container cat /proc/1/attr/current
done

echo "Deployment complete!"
EOF

chmod +x deploy.sh
./deploy.sh
```

### Step 3: 모니터링 및 알림

```bash
# AppArmor 거부 모니터링 스크립트
cat > monitor_apparmor.sh <<'EOF'
#!/bin/bash

LOG_FILE="/var/log/apparmor_denials.log"
LAST_CHECK_FILE="/var/run/apparmor_last_check"

# 마지막 체크 시간
if [ -f "$LAST_CHECK_FILE" ]; then
    LAST_CHECK=$(cat "$LAST_CHECK_FILE")
else
    LAST_CHECK=$(date -d "1 hour ago" +%s)
fi

# 현재 시간
NOW=$(date +%s)

# 새로운 거부 이벤트 찾기
DENIALS=$(sudo dmesg | grep -i "apparmor.*denied" | tail -100)

if [ -n "$DENIALS" ]; then
    echo "[$(date)] New AppArmor denials detected:" >> "$LOG_FILE"
    echo "$DENIALS" >> "$LOG_FILE"
    
    # Slack 알림
    curl -X POST https://hooks.slack.com/services/YOUR/WEBHOOK/URL \
      -H 'Content-Type: application/json' \
      -d "{
        \"text\": \"⚠️ AppArmor Denials Detected\",
        \"attachments\": [{
          \"color\": \"warning\",
          \"text\": \"Check /var/log/apparmor_denials.log for details\"
        }]
      }"
fi

# 마지막 체크 시간 업데이트
echo "$NOW" > "$LAST_CHECK_FILE"
EOF

chmod +x monitor_apparmor.sh

# Cron 설정 (매 10분)
(crontab -l; echo "*/10 * * * * /path/to/monitor_apparmor.sh") | crontab -
```

### Step 4: 자동화된 프로필 업데이트

```bash
# 프로필 자동 배포 스크립트
cat > update_profiles.sh <<'EOF'
#!/bin/bash

PROFILE_DIR="/etc/apparmor.d"
PROFILES=(
    "docker-webapp"
    "docker-postgres"
    "docker-isolated"
    "docker-readonly"
)

for profile in "${PROFILES[@]}"; do
    echo "Updating profile: $profile"
    
    # 프로필 파일 존재 확인
    if [ ! -f "$PROFILE_DIR/$profile" ]; then
        echo "ERROR: Profile $profile not found"
        continue
    fi
    
    # 문법 검사
    if ! sudo apparmor_parser -Q "$PROFILE_DIR/$profile"; then
        echo "ERROR: Profile $profile has syntax errors"
        continue
    fi
    
    # 로드
    sudo apparmor_parser -r "$PROFILE_DIR/$profile"
    
    if [ $? -eq 0 ]; then
        echo "✓ Profile $profile updated successfully"
    else
        echo "✗ Failed to update profile $profile"
    fi
done

# AppArmor 상태 확인
echo "Current AppArmor status:"
sudo aa-status | grep -E "^(apparmor module|profiles)"
EOF

chmod +x update_profiles.sh
```

---

## 💡 주요 명령어 정리

### AppArmor

```bash
# 상태 확인
sudo aa-status
sudo systemctl status apparmor

# 프로필 관리
sudo apparmor_parser -r /etc/apparmor.d/profile-name  # 로드
sudo apparmor_parser -R /etc/apparmor.d/profile-name  # 제거
sudo aa-complain /etc/apparmor.d/profile-name         # Complain 모드
sudo aa-enforce /etc/apparmor.d/profile-name          # Enforce 모드

# 로그 분석
sudo aa-logprof                    # 대화형 프로필 업데이트
sudo dmesg | grep apparmor         # 커널 로그
sudo journalctl -k | grep apparmor # systemd 로그

# Docker에서 사용
docker run --security-opt apparmor=PROFILE IMAGE
docker run --security-opt apparmor=unconfined IMAGE  # 비활성화

# 컨테이너 프로필 확인
docker exec CONTAINER cat /proc/1/attr/current
```

### SELinux

```bash
# 상태 확인
getenforce
sestatus
sudo semodule -l

# 모드 변경
sudo setenforce 0  # Permissive
sudo setenforce 1  # Enforcing

# 컨텍스트 관리
ls -Z FILE
ps auxZ
chcon -t TYPE FILE
restorecon -Rv PATH

# 정책 관리
semodule -i MODULE.pp     # 설치
semodule -r MODULE        # 제거
semodule -l | grep NAME   # 확인

# 파일 컨텍스트
semanage fcontext -a -t TYPE "PATH(/.*)?"
semanage fcontext -l | grep PATH
restorecon -Rv PATH

# 포트 관리
semanage port -a -t TYPE -p PROTOCOL PORT
semanage port -l | grep PORT

# 문제 해결
ausearch -m avc -ts recent
audit2allow -a
audit2allow -M MODULE_NAME < audit.log

# Docker에서 사용
docker run -v /path:/path:z IMAGE    # Private label
docker run -v /path:/path:Z IMAGE    # Shared label
docker run --security-opt label=type:TYPE IMAGE
docker run --security-opt label=disable IMAGE  # 비활성화
```

---

## 🎓 연습 문제

### 문제 1: 웹 서버 프로필 작성

nginx 컨테이너를 위한 AppArmor 프로필을 작성하세요:

요구사항:
- /etc/nginx 읽기만 가능
- /var/log/nginx 쓰기 가능
- /usr/share/nginx/html 읽기만 가능
- /proc/sys, /sys 쓰기 금지
- 네트워크 허용 (TCP 80, 443)

<details>
<summary>정답 보기</summary>

```bash
sudo tee /etc/apparmor.d/docker-nginx <<'EOF'
#include <tunables/global>

profile docker-nginx flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # 네트워크
  network inet stream,
  network inet6 stream,

  # Nginx 설정 (읽기 전용)
  /etc/nginx/** r,
  
  # 로그 (쓰기 가능)
  /var/log/nginx/** w,
  
  # HTML 파일 (읽기 전용)
  /usr/share/nginx/html/** r,
  
  # 런타임
  /var/run/nginx.pid rw,
  /run/nginx.pid rw,
  
  # 실행 파일
  /usr/sbin/nginx rix,
  /lib/** rix,
  /usr/lib/** rix,
  
  # 임시 파일
  /var/cache/nginx/** rw,
  /tmp/** rw,
  
  # 디바이스
  /dev/null rw,
  /dev/urandom r,
  
  # /proc (읽기만)
  /proc/ r,
  /proc/** r,
  
  # 명시적 거부
  deny /proc/sys/** w,
  deny /sys/** w,
  deny mount,
  deny umount,
  deny ptrace,
  
  # Capabilities
  capability chown,
  capability setgid,
  capability setuid,
  capability net_bind_service,
  
  deny capability sys_admin,
}
EOF

# 로드
sudo apparmor_parser -r /etc/apparmor.d/docker-nginx

# 테스트
docker run -d \
  --name secure-nginx \
  --security-opt apparmor=docker-nginx \
  -p 80:80 \
  nginx:alpine

# 검증
curl http://localhost
docker exec secure-nginx sh -c 'echo test > /proc/sys/kernel/hostname'  # 실패
```

</details>

### 문제 2: SELinux 볼륨 마운트 문제 해결

SELinux가 활성화된 시스템에서 다음 에러가 발생합니다:

```bash
docker run -v /data/app:/app myapp:latest
# Permission denied when accessing /app
```

이 문제를 해결하세요.

<details>
<summary>정답 보기</summary>

```bash
# 방법 1: :z 플래그 사용 (권장)
docker run -v /data/app:/app:z myapp:latest

# 방법 2: 수동 레이블 변경
sudo chcon -Rt container_file_t /data/app
docker run -v /data/app:/app myapp:latest

# 방법 3: 영구적 컨텍스트 설정
sudo semanage fcontext -a -t container_file_t "/data/app(/.*)?"
sudo restorecon -Rv /data/app
docker run -v /data/app:/app myapp:latest

# 방법 4: SELinux 비활성화 (비권장)
docker run --security-opt label=disable -v /data/app:/app myapp:latest

# 검증
ls -Z /data/app
# system_u:object_r:container_file_t:s0 /data/app
```

</details>

### 문제 3: AppArmor Complain 모드에서 Enforce 모드로 전환

애플리케이션을 Complain 모드로 실행하여 필요한 권한을 파악한 후, Enforce 모드 프로필을 작성하세요.

<details>
<summary>정답 보기</summary>

```bash
# 1. Complain 모드 프로필
sudo tee /etc/apparmor.d/docker-myapp <<'EOF'
#include <tunables/global>

profile docker-myapp flags=(attach_disconnected,mediate_deleted,complain) {
  #include <abstractions/base>
}
EOF

sudo apparmor_parser -r /etc/apparmor.d/docker-myapp

# 2. Complain 모드로 실행
docker run -d \
  --name myapp-test \
  --security-opt apparmor=docker-myapp \
  myapp:latest

# 3. 애플리케이션 사용 (모든 기능 테스트)
# ... 실제 워크로드 실행 ...

# 4. 로그 분석
sudo aa-logprof

# aa-logprof가 대화형으로:
# - 로그에서 접근 패턴 분석
# - 필요한 권한 제안
# - 프로필 자동 업데이트

# 5. Enforce 모드로 전환
sudo aa-enforce /etc/apparmor.d/docker-myapp

# 6. 재테스트
docker restart myapp-test

# 7. 문제 없으면 프로덕션 배포
docker run -d \
  --name myapp-prod \
  --security-opt apparmor=docker-myapp \
  myapp:latest
```

</details>

---

## 📌 핵심 요약

### AppArmor vs SELinux

| 특성 | AppArmor | SELinux |
|-----|----------|---------|
| **접근 방식** | Path-based | Label-based |
| **설정 파일** | 텍스트 프로필 | 정책 모듈 |
| **복잡도** | 🟢 낮음 | 🔴 높음 |
| **보안 수준** | 🟡 중간 | 🟢 높음 |
| **학습 곡선** | 🟢 쉬움 | 🔴 어려움 |
| **기본 배포** | Ubuntu/Debian | RHEL/CentOS |
| **Docker 지원** | ✅ 우수 | ✅ 우수 |

### 프로필 작성 워크플로우

```
1. Complain 모드로 시작
   ↓
2. 애플리케이션 실행 및 테스트
   ↓
3. 로그 분석 (aa-logprof)
   ↓
4. 필요한 권한 파악
   ↓
5. Enforce 모드 프로필 작성
   ↓
6. 테스트 환경 검증
   ↓
7. 프로덕션 배포
   ↓
8. 모니터링 및 조정
```

### 보안 계층 통합

```
완벽한 보안:

┌────────────────────────────────┐
│ AppArmor/SELinux (MAC)         │
│ - 파일/네트워크 접근 제어            │
├────────────────────────────────┤
│ Seccomp                        │
│ - 시스템 콜 필터링                 │
├────────────────────────────────┤
│ Capabilities                   │
│ - 권한 세분화                     │
├────────────────────────────────┤
│ User Namespace                 │
│ - UID 매핑                      │
├────────────────────────────────┤
│ Read-only Filesystem           │
│ - 파일시스템 불변성                │
└────────────────────────────────┘
```

### 실무 체크리스트

**AppArmor:**
- [ ] 모든 프로덕션 컨테이너에 프로필 적용
- [ ] Complain 모드로 개발/테스트
- [ ] Enforce 모드로 프로덕션
- [ ] 거부 이벤트 모니터링
- [ ] 정기적 프로필 검토
- [ ] 자동화된 배포

**SELinux:**
- [ ] Enforcing 모드 유지
- [ ] 볼륨에 올바른 레이블 (:z/:Z)
- [ ] 커스텀 포트 레이블 관리
- [ ] audit 로그 모니터링
- [ ] 정책 백업
- [ ] 문서화

### 문제 해결 플로우

```
문제 발생
    ↓
로그 확인
- AppArmor: dmesg, aa-logprof
- SELinux: ausearch, audit2allow
    ↓
원인 파악
- 거부된 작업 식별
- 필요한 권한 확인
    ↓
해결
- 프로필/정책 업데이트
- 또는 컨텍스트 수정
    ↓
테스트
    ↓
배포
```

---

## 📚 참고 자료

- [AppArmor Documentation](https://gitlab.com/apparmor/apparmor/-/wikis/Documentation)
- [SELinux User's and Administrator's Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux/)
- [Docker and AppArmor](https://docs.docker.com/engine/security/apparmor/)
- [Docker and SELinux](https://docs.docker.com/engine/security/selinux/)
- [Linux Security Modules](https://www.kernel.org/doc/html/latest/admin-guide/LSM/index.html)

---

## 🤔 생각해볼 문제

1. AppArmor와 SELinux, 둘 중 어느 것이 더 안전할까?
2. MAC(Mandatory Access Control)이 DAC(Discretionary Access Control)보다 안전한 이유는?
3. 프로필/정책을 너무 엄격하게 설정하면 어떤 문제가 발생할까?

> 💡 **답변**: 1) 안전성은 비슷, 차이는 접근 방식: AppArmor - 경로 기반(path-based), 설정 간단, Ubuntu/Debian 기본, 학습 곡선 낮음, SELinux - 레이블 기반(label-based), 더 세밀한 제어, RHEL/CentOS/Fedora 기본, 복잡하지만 강력, 실제로는 "올바르게 설정된 것"이 더 안전, 조직 환경에 맞는 것 선택(익숙한 것, 기본 제공되는 것), 2) DAC는 파일 소유자가 권한 결정: Owner가 world-readable 설정 가능, 사용자 실수로 보안 구멍, 권한 상승 시 모든 파일 접근, MAC는 시스템 정책이 강제: 관리자가 정책 정의, 사용자/프로세스가 변경 불가, Root라도 정책 위반 불가, 다층 방어(LSM + DAC), 예: SELinux로 Nginx는 /var/www만 읽기 가능, Root로 실행해도 /etc/shadow 접근 불가, 3) 문제: 정상 동작 차단(False positive), 애플리케이션 업데이트 시 깨짐, 운영 부담 증가(constant troubleshooting), 결국 비활성화(보안 포기), Best Practice: Complain/Permissive 모드에서 시작, 로그 분석 후 점진적 강화, 화이트리스트 접근(필요한 것만 허용), 자동화된 프로필 생성 도구 활용

---

<div align="center">

**[⬅️ 이전: Secrets Management](./04-Secrets-Management.md)** | **[다음: User Namespaces ➡️](./06-User-Namespaces.md)**

</div>
