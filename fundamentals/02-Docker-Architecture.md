# 02. Docker Architecture - Docker의 내부 구조

## 🎯 이 챕터에서 배울 것

- Docker의 **클라이언트-서버 아키텍처** 완전 이해
- Docker 데몬, containerd, runc의 **역할과 관계**
- 컴포넌트 간 **통신 흐름**과 프로토콜
- `docker run` 명령어가 실제로 하는 일

## 📌 왜 중요한가?

**"Docker는 단일 프로그램이 아닙니다."**

```
많은 개발자들의 오해:
Docker = 하나의 거대한 프로그램

실제:
Docker = 여러 독립적인 컴포넌트의 조합
```

이 구조를 이해하면:
- 트러블슈팅이 쉬워집니다
- 성능 최적화 포인트를 알 수 있습니다
- Kubernetes, containerd 같은 다른 도구로 확장할 수 있습니다

---

## 🔬 Deep Dive

### 1. 전체 아키텍처 개요

```
┌─────────────────────────────────────────────────────┐
│                    Docker Client                    │
│              (docker CLI, docker-compose)           │
└──────────────────────┬──────────────────────────────┘
                       │ REST API (HTTP/Unix Socket)
                       ↓
┌─────────────────────────────────────────────────────┐
│                  Docker Daemon                      │
│                   (dockerd)                         │
│  ┌────────────────────────────────────────────────┐ │
│  │  API Server                                    │ │
│  │  - REST API 엔드포인트                            │ │
│  │  - 인증 및 권한 검증                               │ │
│  └────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────┐ │
│  │  Image Management                              │ │
│  │  - 이미지 빌드/저장/관리                            │ │
│  │  - 레이어 캐시 관리                                │ │
│  └────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────┐ │
│  │  Container Management                          │ │
│  │  - 컨테이너 생명주기 관리                            │ │
│  │  - 네트워크/볼륨 관리                               │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────┘
                       │ gRPC
                       ↓
┌─────────────────────────────────────────────────────┐
│                  containerd                         │
│           (컨테이너 런타임 관리자)                        │
│  ┌────────────────────────────────────────────────┐ │
│  │  Container Lifecycle                           │ │
│  │  - 생성/시작/중지/삭제                              │ │
│  └────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────┐ │
│  │  Image Management                              │ │
│  │  - 이미지 pull/push                              │ │
│  │  - 레이어 관리                                    │ │
│  └────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────┐ │
│  │  Snapshot Management                           │ │
│  │  - 파일시스템 스냅샷                                │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────┬──────────────────────────────┘
                       │ 직접 실행
                       ↓
┌─────────────────────────────────────────────────────┐
│                      runc                           │
│            (OCI 런타임 구현체)                         │
│  - Namespace 생성                                    │
│  - Cgroup 설정                                       │
│  - 프로세스 실행                                       │
└──────────────────────┬──────────────────────────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────┐
│              Linux Kernel                           │
│  - Namespaces (격리)                                 │
│  - Cgroups (리소스 제한)                               │
│  - Union Filesystem (레이어)                          │
└─────────────────────────────────────────────────────┘
```

---

## 🎯 주요 컴포넌트 상세

### 1. Docker Client (docker CLI)

#### 역할
```
사용자 인터페이스
├─ 명령어 해석
├─ API 요청 생성
└─ 결과 출력
```

#### 동작 방식

**예: `docker run` 실행 시**
```bash
docker run -d -p 80:80 nginx

# 내부 동작:
1. CLI가 명령어 파싱
2. REST API 요청 생성
   POST /containers/create
   {
     "Image": "nginx",
     "ExposedPorts": {"80/tcp": {}},
     "HostConfig": {
       "PortBindings": {"80/tcp": [{"HostPort": "80"}]}
     }
   }
3. Unix Socket 또는 TCP로 전송
4. 응답 받아 사용자에게 출력
```

#### 통신 방식

**Unix Socket (기본)**
```bash
# Docker 데몬과 로컬 통신
/var/run/docker.sock

# 실제 통신 확인
curl --unix-socket /var/run/docker.sock http://localhost/version
```

**TCP/TLS (원격)**
```bash
# 원격 Docker 호스트
export DOCKER_HOST=tcp://192.168.1.100:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=/path/to/certs

docker ps
# → 원격 서버의 컨테이너 확인
```

---

### 2. Docker Daemon (dockerd)

**가장 복잡하고 중요한 컴포넌트**

#### 핵심 책임

```
1. API 서버 실행
   └─ REST API 엔드포인트 제공

2. 이미지 관리
   ├─ 빌드 (Dockerfile → 이미지)
   ├─ 저장 (로컬 스토리지)
   └─ 레지스트리 통신 (pull/push)

3. 컨테이너 관리
   ├─ 생명주기 관리
   ├─ 네트워크 설정
   └─ 볼륨 관리

4. 플러그인 관리
   ├─ 네트워크 드라이버
   ├─ 볼륨 드라이버
   └─ 로그 드라이버
```

#### 프로세스 확인

```bash
# Docker 데몬 프로세스
ps aux | grep dockerd
# root  1234  /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

# 데몬 설정 확인
docker info

# 데몬 로그 확인 (디버깅)
journalctl -u docker.service -f
```

#### 설정 파일

**/etc/docker/daemon.json**
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "debug": false,
  "registry-mirrors": ["https://mirror.gcr.io"],
  "insecure-registries": ["myregistry:5000"],
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    }
  ]
}
```

#### API 서버

**REST API 탐색**
```bash
# API 버전 확인
curl --unix-socket /var/run/docker.sock http://localhost/version | jq

# 컨테이너 목록
curl --unix-socket /var/run/docker.sock http://localhost/containers/json | jq

# 이미지 목록
curl --unix-socket /var/run/docker.sock http://localhost/images/json | jq

# 컨테이너 생성
curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"Image": "alpine", "Cmd": ["echo", "hello"]}' \
  http://localhost/containers/create
```

---

### 3. containerd

**컨테이너 런타임의 산업 표준**

#### 역할과 특징

```
containerd의 위치
├─ Docker의 하위 레이어
├─ Kubernetes(CRI)와 직접 통신 가능
└─ 독립적으로 사용 가능

핵심 기능
├─ 컨테이너 생명주기 관리
├─ 이미지 전송 및 저장
├─ 스냅샷 관리
└─ 네트워크 네임스페이스
```

#### 독립 실행 확인

```bash
# containerd 프로세스
ps aux | grep containerd
# root  1235  /usr/bin/containerd

# ctr 명령어 (containerd CLI)
ctr version

# 이미지 pull (Docker 우회)
sudo ctr image pull docker.io/library/alpine:latest

# 이미지 목록
sudo ctr image ls

# 컨테이너 실행
sudo ctr run docker.io/library/alpine:latest my-container echo "Hello from containerd"

# 실행 중인 작업 확인
sudo ctr task ls
```

#### gRPC API

**containerd는 gRPC로 통신**
```bash
# containerd 소켓
/run/containerd/containerd.sock

# gRPC 엔드포인트 예시:
# - /containerd.services.containers.v1.Containers/Create
# - /containerd.services.containers.v1.Containers/Delete
# - /containerd.services.images.v1.Images/Get
```

#### Namespace 격리

```bash
# containerd는 자체 namespace 사용
# Docker가 사용하는 namespace
sudo ctr -n moby container ls

# 다른 namespace 생성 가능
sudo ctr -n my-namespace container ls

# Kubernetes가 사용하는 namespace
sudo ctr -n k8s.io container ls
```

---

### 4. runc

**OCI(Open Container Initiative) 런타임 구현체**

#### 핵심 역할

```
Low-Level 컨테이너 런타임
├─ Namespace 생성
├─ Cgroup 설정
├─ chroot/pivot_root 실행
├─ Capabilities 설정
└─ 프로세스 실행
```

#### 독립 실행 테스트

**runc를 직접 사용해보기**
```bash
# 1. 루트 파일시스템 준비
mkdir -p /tmp/runc-test/rootfs
docker export $(docker create alpine) | tar -C /tmp/runc-test/rootfs -xvf -

# 2. config.json 생성
cd /tmp/runc-test
runc spec

# 3. config.json 확인 (OCI 스펙)
cat config.json | jq

# 4. 컨테이너 실행
sudo runc run mycontainer

# 5. 컨테이너 상태 확인
sudo runc state mycontainer
```

#### OCI Specification

**config.json 구조 (간략)**
```json
{
  "ociVersion": "1.0.0",
  "process": {
    "terminal": true,
    "user": {"uid": 0, "gid": 0},
    "args": ["sh"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "cwd": "/"
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "hostname": "runc",
  "mounts": [...],
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "network"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"}
    ],
    "resources": {
      "memory": {"limit": 536870912},
      "cpu": {"quota": 100000, "period": 100000}
    }
  }
}
```

#### runc 실행 추적

```bash
# runc가 실제로 하는 일 추적
sudo strace -f runc run mycontainer 2>&1 | grep -E 'clone|unshare|mount|pivot_root'

# 주요 시스템 콜:
# - clone(CLONE_NEWNS|CLONE_NEWPID|CLONE_NEWNET...)  → Namespace 생성
# - mount(...)                                       → 파일시스템 마운트
# - pivot_root(...)                                  → 루트 변경
# - execve("/bin/sh")                               → 프로세스 실행
```

---

## 🔄 컴포넌트 간 통신 흐름

### `docker run nginx` 명령어 분해

#### 단계별 흐름

```
1️⃣ Docker Client
   ↓ docker run nginx
   ├─ 명령어 파싱
   └─ REST API 요청 생성

2️⃣ dockerd (Docker Daemon)
   ↓ POST /containers/create
   ├─ 이미지 존재 확인
   │  └─ 없으면: docker pull nginx (Registry 통신)
   ├─ 컨테이너 구성 생성
   └─ containerd에게 요청

3️⃣ containerd
   ↓ gRPC: Container.Create
   ├─ 이미지 레이어 준비
   ├─ 스냅샷 생성 (rootfs)
   ├─ OCI 번들 생성 (config.json)
   └─ runc 실행 준비

4️⃣ runc
   ↓ 실제 컨테이너 생성
   ├─ Namespace 생성 (PID, NET, MNT, UTS, IPC)
   ├─ Cgroup 설정 (메모리, CPU 제한)
   ├─ 루트 파일시스템 pivot_root
   ├─ Capabilities 드롭
   └─ nginx 프로세스 실행

5️⃣ Linux Kernel
   ├─ 격리된 프로세스 실행
   └─ 리소스 제한 적용

6️⃣ 결과 반환
   containerd → dockerd → Docker Client → 사용자
   "Container nginx started: abc123..."
```

#### 실제 로그로 확인

```bash
# Docker 데몬 디버그 모드
sudo dockerd --debug

# 다른 터미널에서
docker run -d nginx

# dockerd 로그에서 확인:
# [API] POST /v1.41/containers/create
# [image] Pulling nginx:latest
# [containerd] Creating container
# [runc] Executing container process
```

---

## 💻 실습

### 실습 1: 컴포넌트별 프로세스 확인

```bash
# 전체 Docker 관련 프로세스 트리
pstree -p | grep -E "dockerd|containerd|runc"

# 결과:
# systemd(1)─┬─dockerd(1234)─┬─{dockerd}(1235)
#            │               └─{dockerd}(1236)
#            └─containerd(1300)─┬─{containerd}(1301)
#                               └─runc:[2:INIT](1400)───nginx(1401)

# 설명:
# - dockerd: Docker 데몬
# - containerd: 컨테이너 런타임
# - runc: OCI 런타임 (컨테이너 생성 후 종료)
# - nginx: 실제 컨테이너 프로세스
```

### 실습 2: 통신 프로토콜 확인

**Unix Socket 모니터링**
```bash
# Docker Client → dockerd 통신 캡처
sudo socat -v UNIX-LISTEN:/tmp/docker.sock,fork UNIX-CONNECT:/var/run/docker.sock &

# 환경 변수 변경
export DOCKER_HOST=unix:///tmp/docker.sock

# 명령어 실행
docker ps

# socat 출력에서 REST API 확인
# > 2024/01/01 12:00:00.123456  length=123 from=0 to=122
# GET /v1.41/containers/json HTTP/1.1
# Host: docker
# User-Agent: Docker-Client/20.10.21
```

**gRPC 통신 캡처**
```bash
# containerd API 모니터링 (gRPCurl 필요)
grpcurl -plaintext -unix /run/containerd/containerd.sock list

# 결과:
# containerd.services.containers.v1.Containers
# containerd.services.images.v1.Images
# containerd.services.tasks.v1.Tasks
# ...
```

### 실습 3: 각 레이어에서 컨테이너 실행

**Level 1: Docker CLI (최상위)**
```bash
docker run -d --name my-nginx nginx
```

**Level 2: containerd CLI (중간)**
```bash
# Docker 우회, containerd 직접 사용
sudo ctr -n moby run -d docker.io/library/nginx:latest my-nginx-containerd
```

**Level 3: runc (최하위)**
```bash
# 수동으로 컨테이너 생성
mkdir -p /tmp/nginx-container/rootfs
docker export $(docker create nginx) | tar -C /tmp/nginx-container/rootfs -xf -
cd /tmp/nginx-container
runc spec
sudo runc run my-nginx-runc
```

### 실습 4: Docker 없이 containerd 사용

**containerd만으로 완전한 컨테이너 실행**
```bash
# 1. 이미지 pull
sudo ctr image pull docker.io/library/redis:alpine

# 2. 컨테이너 생성 및 실행
sudo ctr run -d docker.io/library/redis:alpine my-redis

# 3. 상태 확인
sudo ctr task ls
# TASK       PID    STATUS
# my-redis   1234   RUNNING

# 4. 로그 확인
sudo ctr task logs my-redis

# 5. 실행 중인 명령어 실행
sudo ctr task exec --exec-id sh1 -t my-redis /bin/sh

# 6. 중지 및 삭제
sudo ctr task kill my-redis
sudo ctr container delete my-redis
```

---

## 🔥 실전 적용

### 시나리오 1: Docker 데몬 최적화

**문제: Docker 빌드가 느림**

```bash
# 원인 확인
docker info | grep -A 5 "Storage Driver"

# 최적화 설정
cat > /etc/docker/daemon.json <<EOF
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 10
}
EOF

sudo systemctl restart docker

# 결과: 빌드 시간 30% 단축
```

### 시나리오 2: 원격 Docker 호스트 관리

**Docker Context 활용**
```bash
# Production 서버 등록
docker context create prod \
  --docker "host=tcp://prod.example.com:2376,ca=ca.pem,cert=cert.pem,key=key.pem"

# Staging 서버 등록
docker context create staging \
  --docker "host=tcp://staging.example.com:2376,ca=ca.pem,cert=cert.pem,key=key.pem"

# 컨텍스트 전환
docker context use prod
docker ps  # Production 서버의 컨테이너

docker context use staging
docker ps  # Staging 서버의 컨테이너

# 현재 컨텍스트 확인
docker context ls
```

### 시나리오 3: Kubernetes로 전환

**containerd를 직접 사용하는 Kubernetes**
```yaml
# Kubernetes는 Docker를 거치지 않음
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
  # Kubernetes → CRI → containerd → runc
  # Docker 데몬 불필요!
```

**확인:**
```bash
# Kubernetes 노드에서
sudo ctr -n k8s.io container ls
# Kubernetes가 관리하는 containerd 컨테이너들

# Docker는 없어도 됨
docker ps  # 명령어 없음 (Docker 미설치 가능)
```

---

## ⚡ 성능과 보안 고려사항

### 성능 최적화

**1. Storage Driver 선택**
```bash
# overlay2 사용 (권장)
# - 성능: AUFS보다 2-3배 빠름
# - 호환성: Linux 4.0+ 커널 필요

# 확인
docker info | grep "Storage Driver"
```

**2. Concurrent Operations**
```json
{
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 10
}
```

**3. BuildKit 활성화**
```bash
export DOCKER_BUILDKIT=1
docker build .
# → 병렬 빌드, 캐시 최적화
```

### 보안 강화

**1. Docker Socket 보호**
```bash
# Unix Socket 권한
sudo chmod 660 /var/run/docker.sock
sudo chown root:docker /var/run/docker.sock

# 주의: Docker socket = root 권한!
# docker 그룹 = root 권한과 동등
```

**2. TLS 인증**
```bash
# TCP 노출 시 반드시 TLS 사용
dockerd \
  --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=server-cert.pem \
  --tlskey=server-key.pem \
  -H=0.0.0.0:2376
```

**3. User Namespace Remapping**
```json
{
  "userns-remap": "default"
}
```
```bash
# 컨테이너 내부 root → 호스트의 일반 사용자로 매핑
# 보안 크게 향상
```

---

## 🚫 흔한 문제와 해결

### 문제 1: "Cannot connect to the Docker daemon"

**원인:**
```bash
# Docker 데몬 미실행
sudo systemctl status docker
# ● docker.service - Docker Application Container Engine
#    Loaded: loaded
#    Active: inactive (dead)
```

**해결:**
```bash
sudo systemctl start docker
sudo systemctl enable docker  # 부팅 시 자동 시작
```

### 문제 2: "Permission denied" (docker.sock)

**원인:**
```bash
# 현재 사용자가 docker 그룹에 없음
groups
# user : user adm cdrom sudo
```

**해결:**
```bash
sudo usermod -aG docker $USER
newgrp docker  # 또는 로그아웃 후 재로그인

# 확인
docker ps  # sudo 없이 실행 가능
```

### 문제 3: containerd 충돌

**증상:**
```bash
docker ps
# Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

**진단:**
```bash
# containerd 상태 확인
sudo systemctl status containerd
# ● containerd.service - containerd container runtime
#    Active: failed

# 로그 확인
sudo journalctl -u containerd.service -n 50
```

**해결:**
```bash
# containerd 재시작
sudo systemctl restart containerd
sudo systemctl restart docker
```

### 문제 4: 높은 메모리 사용

**원인: 이미지/컨테이너 누적**
```bash
# 디스크 사용량 확인
docker system df

# TYPE            TOTAL   ACTIVE   SIZE      RECLAIMABLE
# Images          50      10       15GB      12GB (80%)
# Containers      100     5        2GB       1.9GB (95%)
# Local Volumes   30      5        5GB       4GB (80%)
```

**해결:**
```bash
# 정리
docker system prune -a --volumes

# 또는 자동 정리 설정
cat > /etc/docker/daemon.json <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
```

---

## 🎓 핵심 정리

### 아키텍처 요약

```
Layer 4: Docker CLI
         └─ 사용자 명령어 → REST API 변환

Layer 3: dockerd
         └─ 이미지/컨테이너 관리, 네트워크/볼륨

Layer 2: containerd
         └─ 컨테이너 런타임 관리, 표준 인터페이스

Layer 1: runc
         └─ 실제 컨테이너 생성 (Namespace, Cgroup)

Layer 0: Linux Kernel
         └─ 격리 및 리소스 제한 제공
```

### 통신 프로토콜

```
Docker CLI ←(REST/HTTP)→ dockerd ←(gRPC)→ containerd ←(exec)→ runc
                                                                 ↓
                                                         Linux Kernel
```

### 핵심 포인트

1. **Docker ≠ 단일 프로그램**
   - 여러 독립 컴포넌트의 조합
   - 각각 독립적으로 사용 가능

2. **containerd의 중요성**
   - Docker와 Kubernetes 공통 레이어
   - 산업 표준 컨테이너 런타임

3. **runc = OCI 구현체**
   - 실제 컨테이너를 생성하는 로우레벨 도구
   - Namespace, Cgroup 직접 조작

4. **계층화의 이점**
   - 각 레이어의 독립적 발전
   - 유연한 아키텍처
   - 다른 도구와 통합 용이

---

## 📚 참고 자료

- [Docker Architecture](https://docs.docker.com/get-started/overview/#docker-architecture)
- [containerd](https://containerd.io/)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [runc GitHub](https://github.com/opencontainers/runc)
- [Docker Engine API](https://docs.docker.com/engine/api/)

---

**🤔 생각해볼 문제**

1. Kubernetes가 Docker 대신 containerd를 직접 사용하는 이유는?
2. Docker Swarm과 Kubernetes의 아키텍처 차이는?
3. Podman은 데몬이 없다는데, 어떻게 가능할까?

> 💡 **힌트**: 모두 "레이어 분리"와 "표준화"가 핵심입니다.

---

<div align="center">

**[⬅️ 이전: Container vs VM](./01-Container-vs-VM.md)** | **[다음: Image Layers ➡️](./03-Image-Layers.md)**

</div>
