# 05. Namespaces - 컨테이너 격리의 핵심

## 🎯 이 챕터에서 배울 것

- Linux **Namespaces**의 개념과 종류
- 각 Namespace의 **격리 범위**와 동작 원리
- Namespace 실제 생성 및 **격리 확인 실습**
- Docker가 Namespace를 활용하는 방법

## 📌 왜 중요한가?

**"컨테이너의 격리는 어떻게 이루어지는가?"**

```
질문:
- 컨테이너 내부에서 ps aux 했을 때 왜 다른 프로세스가 안 보일까?
- 컨테이너마다 다른 네트워크 인터페이스를 가질 수 있는 이유는?
- 컨테이너 안에서 hostname을 바꿔도 호스트에 영향이 없는 이유는?

답: Namespaces!
```

**실무 영향:**
- 보안: 격리 수준 이해 → 안전한 컨테이너 설계
- 디버깅: 네트워크/프로세스 문제 해결
- 최적화: Namespace 공유로 성능 향상

---

## 🔬 Deep Dive

### 1. Namespace란?

#### 개념

**Linux Namespace = 격리된 시스템 리소스 뷰**

```
일반 프로세스:
모든 프로세스가 같은 시스템 뷰를 공유
├─ 같은 프로세스 목록
├─ 같은 네트워크 인터페이스
├─ 같은 파일시스템
└─ 같은 호스트명

Namespace 사용:
각 프로세스가 독립적인 시스템 뷰를 가짐
├─ 자신만의 프로세스 목록 (PID Namespace)
├─ 자신만의 네트워크 (Network Namespace)
├─ 자신만의 파일시스템 (Mount Namespace)
└─ 자신만의 호스트명 (UTS Namespace)

→ 가상화 없이 격리!
```

#### 역사

```
2002: Mount Namespace (첫 번째)
2006: UTS, IPC Namespace
2008: PID, Network, User Namespace
2013: User Namespace 완성
2016: Cgroup Namespace

Docker (2013):
→ 이미 대부분의 Namespace 사용 가능
→ 완벽한 타이밍!
```

---

### 2. Namespace 종류 (7가지)

#### 📋 전체 목록

| Namespace | 격리 대상 | Flag | 설명 |
|-----------|----------|------|------|
| **PID** | Process ID | `CLONE_NEWPID` | 프로세스 ID 격리 |
| **NET** | Network | `CLONE_NEWNET` | 네트워크 스택 격리 |
| **MNT** | Mount | `CLONE_NEWNS` | 파일시스템 마운트 격리 |
| **UTS** | Unix Timesharing | `CLONE_NEWUTS` | 호스트명/도메인명 격리 |
| **IPC** | Inter-Process Communication | `CLONE_NEWIPC` | 프로세스 간 통신 격리 |
| **USER** | User ID | `CLONE_NEWUSER` | UID/GID 격리 |
| **CGROUP** | Control Groups | `CLONE_NEWCGROUP` | Cgroup 뷰 격리 |

---

### 3. PID Namespace - 프로세스 격리

#### 동작 원리

```
호스트 시스템:
PID 1    → systemd (init)
PID 1234 → dockerd
PID 1500 → container process
PID 1501 → nginx (container)

컨테이너 내부 (PID Namespace):
PID 1    → nginx (container의 init)
PID 2    → worker process
...

→ 같은 프로세스가 다른 PID!
```

#### 특징

1. **PID 1의 특별함**
```
PID 1 = Init 프로세스
├─ 시그널 처리 특수
│  └─ SIGKILL 무시 (커널만 종료 가능)
├─ 좀비 프로세스 reaping
│  └─ 부모 없는 프로세스의 부모가 됨
└─ 컨테이너 생명주기와 직결
   └─ PID 1 종료 = 컨테이너 종료
```

2. **프로세스 가시성**
```
부모 Namespace → 자식 Namespace: 보임
자식 Namespace → 부모 Namespace: 안 보임
자식 Namespace → 형제 Namespace: 안 보임

호스트에서:
ps aux | grep nginx
→ PID 1500 nginx (전체 보임)

컨테이너에서:
ps aux
→ PID 1 nginx (자기 것만 보임)
```

#### 실습

```bash
# 1. 컨테이너 실행
docker run -d --name pid-test nginx

# 2. 호스트에서 프로세스 확인
ps aux | grep nginx
# root  12345  nginx: master process
# nginx 12346  nginx: worker process

# 3. 컨테이너 내부에서 확인
docker exec pid-test ps aux
# PID   USER     COMMAND
# 1     root     nginx: master process
# 7     nginx    nginx: worker process

# 4. 같은 프로세스, 다른 PID!
# 호스트: 12345
# 컨테이너: 1

# 5. 호스트에서 컨테이너 프로세스 종료 가능
sudo kill 12345
# → 컨테이너 종료됨 (PID 1 죽었으므로)
```

---

### 4. Network Namespace - 네트워크 격리

#### 동작 원리

```
각 Network Namespace는 독립적인 네트워크 스택:
├─ 네트워크 인터페이스 (lo, eth0, ...)
├─ 라우팅 테이블
├─ iptables 규칙
├─ 소켓 (listening port)
└─ /proc/net, /sys/class/net

┌──────────────────────────────────────┐
│  Host Network Namespace              │
│  ├─ lo (127.0.0.1)                   │
│  ├─ eth0 (192.168.1.100)             │
│  └─ docker0 (172.17.0.1) bridge      │
└──────────────────────────────────────┘
        │
        ├─ veth pair (가상 이더넷)
        │
┌──────────────────────────────────────┐
│  Container Network Namespace         │
│  ├─ lo (127.0.0.1)                   │
│  └─ eth0@if10 (172.17.0.2)           │
│     └─ veth to host                  │
└──────────────────────────────────────┘
```

#### veth pair

```
veth (Virtual Ethernet) = 가상 케이블
├─ 한쪽 끝: 호스트 (docker0 bridge)
└─ 다른 끝: 컨테이너 (eth0)

패킷 전송:
Container eth0 → veth (host) → docker0 → host eth0 → 외부
```

#### 실습

```bash
# 1. 컨테이너 실행
docker run -d --name net-test nginx

# 2. 호스트 네트워크 인터페이스
ip addr
# 1: lo
# 2: eth0: 192.168.1.100
# 3: docker0: 172.17.0.1
# 10: veth1234@if9  ← 컨테이너로 가는 veth

# 3. 컨테이너 네트워크 인터페이스
docker exec net-test ip addr
# 1: lo: 127.0.0.1
# 9: eth0@if10: 172.17.0.2  ← veth의 다른 끝

# 4. 라우팅 테이블
docker exec net-test ip route
# default via 172.17.0.1 dev eth0
# 172.17.0.0/16 dev eth0

# 5. listening 포트
docker exec net-test netstat -tlnp
# tcp  0.0.0.0:80  LISTEN  1/nginx

# 호스트에서는 안 보임 (다른 Namespace)
netstat -tlnp | grep :80
# (없음)

# 6. 네트워크 Namespace 직접 확인
sudo ls /var/run/docker/netns/
# abc123def456...

sudo ip netns list
# abc123def456 (id: 0)

# 7. Namespace에서 명령어 실행
NETNS=$(docker inspect -f '{{.NetworkSettings.SandboxKey}}' net-test)
sudo nsenter --net=$NETNS ip addr
# 컨테이너 네트워크와 동일!
```

---

### 5. Mount Namespace - 파일시스템 격리

#### 동작 원리

```
각 Mount Namespace는 독립적인 마운트 테이블
├─ 다른 Namespace의 마운트 안 보임
├─ 마운트/언마운트 독립적
└─ 루트 파일시스템 변경 가능 (chroot)

호스트:
/
├─ /home
├─ /var
└─ /usr

컨테이너:
/  (완전히 다른 루트!)
├─ /app
├─ /etc
└─ /usr (컨테이너 버전)
```

#### Propagation

```
마운트 전파 설정:
├─ private: 전파 없음 (기본값)
├─ shared: 양방향 전파
├─ slave: 부모→자식 전파만
└─ unbindable: bind mount 불가

예: Volume mount
docker run -v /host:/container:shared
→ /host 마운트가 컨테이너에도 전파
```

#### 실습

```bash
# 1. 호스트 마운트 확인
mount | grep /var/lib/docker
# overlay on /var/lib/docker/overlay2/.../merged

# 2. 컨테이너 실행
docker run -it --name mount-test ubuntu bash

# 3. 컨테이너 마운트 확인 (컨테이너 내부)
mount
# overlay on / type overlay
# proc on /proc type proc
# tmpfs on /dev type tmpfs
# ...

# 4. 호스트와 완전히 다름!
# 호스트:
df -h
# /dev/sda1  100G  50G  50G  /

# 컨테이너:
df -h
# overlay    100G  50G  50G  /  ← 같아 보이지만 다른 뷰

# 5. 컨테이너에서 마운트
mount -t tmpfs tmpfs /mnt
# → 호스트에 영향 없음 (격리됨)

# 6. 호스트에서 확인
mount | grep tmpfs | grep /mnt
# (없음)
```

---

### 6. UTS Namespace - 호스트명 격리

#### 동작 원리

```
UTS = Unix Timesharing System
격리 대상:
├─ Hostname
└─ Domain name (NIS domain)

각 컨테이너 = 독립적인 호스트명
```

#### 실습

```bash
# 1. 호스트 호스트명
hostname
# myhost

# 2. 컨테이너 실행
docker run -it --name uts-test --hostname mycontainer ubuntu bash

# 3. 컨테이너 호스트명 확인
hostname
# mycontainer  ← 다름!

# 4. 호스트명 변경
hostname changed-name

# 5. 호스트에서 확인
hostname
# myhost  ← 변경 없음 (격리됨)

# 6. /etc/hostname 파일
cat /etc/hostname
# changed-name (컨테이너 내부)

# 호스트:
cat /etc/hostname
# myhost (변경 없음)
```

---

### 7. IPC Namespace - 프로세스 간 통신 격리

#### 동작 원리

```
격리 대상:
├─ System V IPC
│  ├─ 공유 메모리 (shmem)
│  ├─ 세마포어 (semaphore)
│  └─ 메시지 큐 (message queue)
└─ POSIX 메시지 큐

같은 IPC Namespace → IPC 통신 가능
다른 IPC Namespace → IPC 통신 불가
```

#### 실습

```bash
# 1. 호스트에서 공유 메모리 생성
ipcmk -M 1024
# Shared memory id: 12345

# 확인
ipcs -m
# shmid  owner  bytes
# 12345  root   1024

# 2. 컨테이너에서 확인
docker run --rm ubuntu ipcs -m
# (없음)  ← 다른 IPC Namespace

# 3. IPC Namespace 공유
docker run --ipc=host ubuntu ipcs -m
# shmid  owner  bytes
# 12345  root   1024  ← 보임!

# 4. 컨테이너 간 IPC 공유
docker run -d --name ipc-master ubuntu sleep 3600
docker run -it --ipc=container:ipc-master ubuntu bash

# 같은 IPC Namespace 사용 → 통신 가능
```

---

### 8. User Namespace - UID/GID 격리

#### 동작 원리

```
UID/GID 매핑:
컨테이너 내부    →    호스트
UID 0 (root)    →    UID 1000 (user)
UID 1000        →    UID 2000
...

→ 컨테이너에서 root여도 호스트에서는 일반 사용자!
→ 보안 크게 향상
```

#### 특징

```
User Namespace 사용 시:
├─ 컨테이너 root ≠ 호스트 root
├─ Capabilities 제한
└─ 파일 권한 자동 변환

없이:
├─ 컨테이너 root = 호스트 root (위험!)
└─ 컨테이너 탈출 시 전체 권한 획득 가능
```

#### 실습

```bash
# 1. User Namespace 없이 (기본)
docker run --rm ubuntu id
# uid=0(root) gid=0(root)

# 호스트에서 프로세스 확인
ps aux | grep sleep
# root  12345  sleep  ← root로 실행 (위험!)

# 2. User Namespace 활성화
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}

sudo systemctl restart docker

# 3. 다시 실행
docker run -d --name user-test ubuntu sleep 3600

# 컨테이너 내부:
docker exec user-test id
# uid=0(root) gid=0(root)  ← 여전히 root

# 호스트에서:
ps aux | grep sleep
# dockrema+  12345  sleep  ← 매핑된 사용자!

# 4. 실제 UID 확인
docker exec user-test cat /proc/self/uid_map
# 0  100000  65536
# 컨테이너 UID 0 → 호스트 UID 100000

# 5. 파일 권한 확인
docker exec user-test touch /test.txt
docker exec user-test ls -l /test.txt
# -rw-r--r-- 1 root root  (컨테이너 관점)

# 호스트에서:
CONTAINER_ID=$(docker inspect -f '{{.Id}}' user-test)
sudo ls -ln /var/lib/docker/100000.100000/overlay2/$CONTAINER_ID/diff/test.txt
# -rw-r--r-- 1 100000 100000  (호스트 관점)
```

---

### 9. Cgroup Namespace - Cgroup 뷰 격리

#### 동작 원리

```
Cgroup Namespace 없이:
컨테이너에서 /proc/self/cgroup 확인
→ 호스트의 전체 cgroup 경로 보임
→ 정보 유출 가능

Cgroup Namespace 사용:
→ 자신의 cgroup만 보임
→ 상대 경로로 표시
```

#### 실습

```bash
# 1. Cgroup Namespace 없이 (Docker 기본)
docker run --rm ubuntu cat /proc/self/cgroup
# 12:memory:/docker/abc123def456...
# → 전체 경로 노출

# 2. Cgroup Namespace 사용
docker run --rm --cgroupns=private ubuntu cat /proc/self/cgroup
# 12:memory:/
# → 루트 경로로 보임 (격리됨)

# 3. 호스트와 비교
cat /proc/self/cgroup
# 12:memory:/user.slice/user-1000.slice
```

---

## 💻 고급 실습

### 실습 1: Namespace 직접 생성 (unshare)

```bash
# unshare: 새 Namespace 생성

# 1. PID Namespace 생성
sudo unshare --pid --fork --mount-proc bash

# 새 shell에서:
ps aux
# PID   USER     COMMAND
# 1     root     bash  ← PID 1!
# 2     root     ps

# 2. Network Namespace 생성
sudo unshare --net bash

# 새 shell에서:
ip addr
# 1: lo: <LOOPBACK> (DOWN)
# → 네트워크 인터페이스 없음!

# 3. 모든 Namespace 격리
sudo unshare --pid --net --mount --uts --ipc --fork bash

# 완전히 격리된 환경!
hostname isolated
ps aux  # 자기만 보임
ip addr  # 네트워크 없음
```

### 실습 2: Namespace 진입 (nsenter)

```bash
# nsenter: 기존 Namespace 진입

# 1. 컨테이너 실행
docker run -d --name ns-target nginx

# 2. 컨테이너 PID 확인
PID=$(docker inspect -f '{{.State.Pid}}' ns-target)
echo $PID
# 12345

# 3. 모든 Namespace 진입
sudo nsenter --target $PID --all

# 컨테이너와 동일한 환경!
ps aux  # 컨테이너 프로세스 보임
ip addr  # 컨테이너 네트워크

# 4. 특정 Namespace만 진입
# PID Namespace만
sudo nsenter --target $PID --pid --mount ps aux

# Network Namespace만
sudo nsenter --target $PID --net ip addr
```

### 실습 3: Docker의 Namespace 설정

```bash
# 1. Host Namespace 공유 (격리 해제)
# PID Namespace 공유
docker run --pid=host ubuntu ps aux
# → 호스트의 모든 프로세스 보임!

# Network Namespace 공유
docker run --net=host nginx
# → 호스트의 네트워크 사용 (포트 충돌 주의)

# IPC Namespace 공유
docker run --ipc=host ubuntu ipcs

# 2. 다른 컨테이너 Namespace 공유
docker run -d --name container1 nginx
docker run --pid=container:container1 --net=container:container1 ubuntu

# container1과 같은 PID, Network Namespace
# → 프로세스, 네트워크 공유

# 3. 완전 격리 (기본값)
docker run ubuntu
# 모든 Namespace 새로 생성
```

---

## 🔥 실전 적용

### 시나리오 1: 디버깅 컨테이너

**문제: 프로덕션 컨테이너에 디버깅 툴이 없음**

```bash
# 기존 컨테이너
docker run -d --name prod-app \
  --memory=512m \
  myapp:latest

# 디버깅 컨테이너 (같은 환경)
docker run -it \
  --pid=container:prod-app \
  --net=container:prod-app \
  --volumes-from prod-app \
  ubuntu bash

# 이제 디버깅 툴 사용 가능:
apt-get update && apt-get install -y strace tcpdump

# 앱 프로세스 추적
ps aux  # prod-app 프로세스 보임
strace -p <pid>  # 시스템 콜 추적
tcpdump -i eth0  # 네트워크 패킷 캡처
```

### 시나리오 2: 보안 강화 (User Namespace)

```bash
# Before: 위험한 설정
docker run --rm -v /:/host ubuntu

# 컨테이너 내부:
echo "HACKED" > /host/etc/passwd
# → 호스트 파일 수정 가능! (root이므로)

# After: User Namespace 사용
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}

docker run --rm -v /:/host ubuntu

# 컨테이너 내부:
echo "HACKED" > /host/etc/passwd
# Permission denied!
# → 호스트에서는 일반 사용자이므로
```

### 시나리오 3: 네트워크 격리

```bash
# 마이크로서비스 격리
docker network create app-net

# Frontend (외부 노출)
docker run -d --name frontend \
  --network app-net \
  -p 80:80 \
  frontend:latest

# Backend (내부만)
docker run -d --name backend \
  --network app-net \
  backend:latest

# Database (완전 격리)
docker run -d --name db \
  --network app-net \
  postgres:latest

# 네트워크 격리 확인:
# Frontend → Backend: 가능
docker exec frontend ping backend  # OK

# Frontend → 외부: 가능
docker exec frontend ping 8.8.8.8  # OK

# Backend → 외부: 차단 가능
docker run -d --name backend-secure \
  --network app-net \
  --network-alias backend \
  --cap-drop=NET_RAW \
  backend:latest

docker exec backend-secure ping 8.8.8.8  # 실패
```

---

## ⚡ 보안 고려사항

### 1. Namespace 공유의 위험

```bash
# ❌ 위험: Host PID Namespace 공유
docker run --pid=host ubuntu

# 컨테이너에서:
kill -9 1  # 호스트의 init 종료 시도 가능!

# ✅ 안전: 기본 격리
docker run ubuntu  # 각자의 PID Namespace
```

### 2. User Namespace 미사용 위험

```bash
# ❌ 기본 설정 (위험)
docker run -v /:/host ubuntu
# → 컨테이너 root = 호스트 root

# ✅ User Namespace 사용 (안전)
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}
# → 컨테이너 root ≠ 호스트 root
```

### 3. Capabilities 제한

```bash
# Namespace만으로는 부족
# Capabilities도 함께 제한

docker run --rm \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  nginx
# → 포트 바인딩만 가능, 나머지 제한
```

---

## 🚫 안티패턴

### 1. 불필요한 Namespace 공유

```bash
# ❌ 나쁜 예
docker run --pid=host --net=host --privileged myapp
# → 격리 거의 없음 (VM과 차이 없음)

# ✅ 좋은 예
docker run myapp
# → 필요한 것만 노출 (포트 등)
```

### 2. 여러 프로세스를 한 컨테이너에

```dockerfile
# ❌ 안티패턴
FROM ubuntu
RUN apt-get install -y nginx mysql supervisor
CMD ["/usr/bin/supervisord"]
# → 하나의 PID Namespace에 여러 서비스
# → 격리 이점 상실

# ✅ 올바른 패턴
# docker-compose.yml
services:
  nginx:
    image: nginx
  mysql:
    image: mysql
# → 각자의 Namespace
```

---

## 🎓 핵심 정리

### Namespace 종류

```
7가지 Namespace:
├─ PID: 프로세스 격리
├─ NET: 네트워크 격리
├─ MNT: 파일시스템 격리
├─ UTS: 호스트명 격리
├─ IPC: 프로세스 간 통신 격리
├─ USER: UID/GID 격리
└─ CGROUP: Cgroup 뷰 격리
```

### 격리 수준

```
완전 격리 (기본):
docker run myapp
→ 모든 Namespace 새로 생성

부분 공유:
docker run --pid=host myapp
→ PID Namespace만 공유

격리 없음 (위험):
docker run --pid=host --net=host --privileged myapp
→ 거의 VM 수준
```

### 보안 체크리스트

```
✅ User Namespace 사용
✅ 불필요한 Namespace 공유 금지
✅ Capabilities 최소화
✅ 네트워크 격리 (사용자 정의 네트워크)
✅ 읽기 전용 파일시스템
```

---

## 🔗 다음 단계

Namespace를 마스터했습니다! 다음 챕터:

- **[06. Cgroups](06-Cgroups.md)**: 리소스 제한 메커니즘
- **[security/03-Runtime-Security](../security/03-Runtime-Security.md)**: 런타임 보안
- **[networking/01-Network-Fundamentals](../networking/01-Network-Fundamentals.md)**: 네트워크 심화

---

## 📚 참고 자료

- [Linux Namespaces man page](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Docker Security](https://docs.docker.com/engine/security/)
- [User Namespaces](https://docs.docker.com/engine/security/userns-remap/)
- [Namespaces in Operation](https://lwn.net/Articles/531114/)

---

**🤔 생각해볼 문제**

1. `--pid=host`로 실행된 컨테이너는 정말 안전할까?
2. User Namespace를 사용하면 왜 Volume 마운트 권한 문제가 생길까?
3. Kubernetes Pod 내 컨테이너들은 어떤 Namespace를 공유할까?

> 💡 **답변**: 1) 아니오 (호스트 프로세스 종료 가능), 2) UID 매핑 때문, 3) Network, IPC, UTS (PID는 옵션)
