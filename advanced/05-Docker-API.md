# 05. Docker API - REST API로 Docker 제어

## 🎯 이 챕터에서 배울 것

- **Docker API 구조**: Engine API의 전체 엔드포인트
- **Unix Socket 통신**: curl을 통한 API 호출
- **Container CRUD**: 생성, 시작, 정지, 삭제, 로그
- **Image 관리**: Pull, push, build, inspect
- **Network/Volume API**: 리소스 생성 및 관리
- **실시간 이벤트**: Event stream, stats monitoring
- **인증 및 보안**: TLS, API 버전 협상

## 📌 왜 중요한가?

**"Docker API를 이해하면 Docker CLI 없이도 컨테이너를 자동화하고 통합할 수 있습니다."**

```
Docker CLI는 내부적으로 API를 호출:

사용자                Docker CLI              Docker API
  │                     │                       │
  │  docker run nginx   │                       │
  │────────────────────→│                       │
  │                     │POST /containers/create│
  │                     │──────────────────────→│
  │                     │ {"Image": "nginx"}    │
  │                     │                       │
  │                     │← ContainerID          │
  │                     │                       │
  │                     │ POST /containers/{id}/start
  │                     │──────────────────────→│
  │                     │                       │
  │                     │← 204 No Content       │
  │                     │                       │
  │← nginx container    │                       │

모든 Docker 기능은 API로 제공됨:
┌────────────────────────────────────────────────────┐
│ Docker CLI (docker)                                │
│ - 사용자 친화적 인터페이스                               │
│ - 내부적으로 API 호출                                  │
└────────────────┬───────────────────────────────────┘
                 │ HTTP/Unix Socket
┌────────────────▼───────────────────────────────────┐
│ Docker Engine API                                  │
│                                                    │
│  ┌─────────────────────────────────────────────┐   │
│  │ REST API Endpoints                          │   │
│  │                                             │   │
│  │ - /containers (생성, 시작, 정지, 삭제)           │  │
│  │ - /images (pull, push, build, inspect)       │  │
│  │ - /networks (생성, 연결, 삭제)                  │  │
│  │ - /volumes (생성, 조회, 삭제)                   │  │
│  │ - /exec (컨테이너 내 명령 실행)                   │  │
│  │ - /events (실시간 이벤트 스트림)                  │  │
│  │ - /system (시스템 정보, 버전)                    │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
└────────────────┬───────────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────────┐
│ dockerd (Docker Daemon)                            │
│ - API 요청 처리                                      │
│ - containerd로 컨테이너 관리                           │
└────────────────────────────────────────────────────┘

API를 직접 사용하는 이유:
✅ Docker CLI 없이 자동화
✅ 커스텀 도구 개발
✅ CI/CD 파이프라인 통합
✅ 모니터링 시스템 구축
✅ 다른 언어로 Docker 제어

통신 방법:
┌──────────────────┬──────────────────────────────┐
│ 방법              │ 용도                          │
├──────────────────┼──────────────────────────────┤
│ Unix Socket      │ 로컬 (기본)                    │
│ /var/run/docker. │ - 빠른 통신                    │
│ sock             │ - 권한 제어 용이                │
├──────────────────┼──────────────────────────────┤
│ TCP              │ 원격                          │
│ tcp://host:2375  │ - 네트워크 접근                 │
│ (비암호화)         │ - 보안 위험 (프로덕션 X)          │
├──────────────────┼──────────────────────────────┤
│ TLS/TCP          │ 원격 (보안)                    │
│ tcp://host:2376  │ - 인증서 기반 인증               │
│                  │ - 프로덕션 환경                 │
└──────────────────┴──────────────────────────────┘
```

**실무 영향:**
- **자동화**: Bash/Python 스크립트로 컨테이너 관리
- **CI/CD**: Jenkins, GitLab CI에서 Docker 제어
- **모니터링**: Prometheus, Grafana 통합
- **커스텀 도구**: 자체 컨테이너 관리 UI 개발

---

## 🔬 Deep Dive

### 1. Docker API 구조

#### API 버전 및 엔드포인트

```
Docker Engine API:

API 버전:
- Docker 1.12+: API v1.24
- Docker 17.06+: API v1.30
- Docker 18.03+: API v1.37
- Docker 20.10+: API v1.41
- Docker 23.0+: API v1.43 (현재)

버전 협상:
클라이언트가 낮은 버전 요청 시 호환 모드로 동작
Header: "API-Version: 1.41"

주요 엔드포인트 구조:
┌─────────────────────────────────────────────────┐
│ /v1.43 (API Version)                            │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ Containers                                  │ │
│ │ POST   /containers/create                   │ │
│ │ GET    /containers/{id}/json                │ │
│ │ POST   /containers/{id}/start               │ │
│ │ POST   /containers/{id}/stop                │ │
│ │ POST   /containers/{id}/restart             │ │
│ │ POST   /containers/{id}/kill                │ │
│ │ DELETE /containers/{id}                     │ │
│ │ GET    /containers/{id}/logs                │ │
│ │ GET    /containers/{id}/stats               │ │
│ │ POST   /containers/{id}/exec                │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ Images                                      │ │
│ │ POST   /images/create (pull)                │ │
│ │ POST   /build (build from Dockerfile)       │ │
│ │ GET    /images/{name}/json                  │ │
│ │ GET    /images/json (list)                  │ │
│ │ DELETE /images/{name}                       │ │
│ │ POST   /images/{name}/push                  │ │
│ │ POST   /images/{name}/tag                   │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ Networks                                    │ │
│ │ POST   /networks/create                     │ │
│ │ GET    /networks/{id}                       │ │
│ │ DELETE /networks/{id}                       │ │
│ │ POST   /networks/{id}/connect               │ │
│ │ POST   /networks/{id}/disconnect            │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ Volumes                                     │ │
│ │ POST   /volumes/create                      │ │
│ │ GET    /volumes                             │ │
│ │ GET    /volumes/{name}                      │ │
│ │ DELETE /volumes/{name}                      │ │
│ └─────────────────────────────────────────────┘ │
│                                                 │
│ ┌─────────────────────────────────────────────┐ │
│ │ System                                      │ │
│ │ GET    /info                                │ │
│ │ GET    /version                             │ │
│ │ GET    /events                              │ │
│ │ GET    /_ping                               │ │
│ │ POST   /auth                                │ │
│ └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘

HTTP Methods:
GET    - 조회 (리소스 읽기)
POST   - 생성/실행 (상태 변경)
DELETE - 삭제
PUT    - 업데이트 (드물게 사용)
```

#### Unix Socket vs TCP

```
통신 방법 비교:

1. Unix Socket (기본):
   경로: /var/run/docker.sock
   
   ┌────────────────────────────────────────┐
   │ Client (curl, docker CLI)              │
   └────────────┬───────────────────────────┘
                │ Unix Domain Socket
                │ (파일 기반 통신)
   ┌────────────▼───────────────────────────┐
   │ dockerd                                │
   │ - 로컬 전용                              │
   │ - 파일 권한으로 접근 제어                   │
   │ - docker 그룹 사용자만 접근                │
   └────────────────────────────────────────┘
   
   장점:
   ✅ 빠른 통신 (네트워크 오버헤드 없음)
   ✅ 파일 권한으로 접근 제어
   ✅ 간단한 설정
   
   단점:
   ❌ 로컬 전용 (원격 불가)

2. TCP (원격):
   주소: tcp://0.0.0.0:2375 (비암호화)
         tcp://0.0.0.0:2376 (TLS)
   
   ┌────────────────────────────────────────┐
   │ Remote Client                          │
   └────────────┬───────────────────────────┘
                │ TCP/IP
                │ (네트워크 통신)
   ┌────────────▼───────────────────────────┐
   │ dockerd -H tcp://0.0.0.0:2375          │
   │ - 원격 접근 가능                          │
   │ - 네트워크를 통한 제어                      │
   └────────────────────────────────────────┘
   
   장점:
   ✅ 원격 접근 가능
   ✅ 여러 클라이언트 지원
   
   단점:
   ❌ 네트워크 보안 필요
   ❌ 2375 (비암호화)는 매우 위험!

3. TLS/TCP (프로덕션):
   주소: tcp://0.0.0.0:2376
   
   ┌────────────────────────────────────────┐
   │ Client (with TLS certificates)         │
   │ - ca.pem (CA 인증서)                     │
   │ - cert.pem (클라이언트 인증서)             │
   │ - key.pem (클라이언트 키)                 │
   └────────────┬───────────────────────────┘
                │ TLS (암호화 + 인증)
   ┌────────────▼───────────────────────────┐
   │ dockerd --tlsverify                    │
   │ - 인증서 검증                             │
   │ - 암호화 통신                             │
   └────────────────────────────────────────┘
   
   장점:
   ✅ 보안 (암호화 + 인증)
   ✅ 원격 접근
   
   단점:
   ❌ 인증서 관리 필요

설정 예:
# /etc/docker/daemon.json
{
  "hosts": [
    "unix:///var/run/docker.sock",
    "tcp://0.0.0.0:2376"
  ],
  "tls": true,
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem",
  "tlsverify": true,
  "tlscacert": "/etc/docker/ca.pem"
}
```

---

### 2. Container API 상세

#### Container Lifecycle API

```
Container 생명주기 API:

1. 컨테이너 생성 (Created 상태):
POST /containers/create
{
  "Image": "nginx:latest",
  "Cmd": ["nginx", "-g", "daemon off;"],
  "ExposedPorts": {
    "80/tcp": {}
  },
  "HostConfig": {
    "PortBindings": {
      "80/tcp": [{"HostPort": "8080"}]
    },
    "Memory": 536870912,
    "CpuShares": 1024
  }
}

응답:
{
  "Id": "abc123...",
  "Warnings": []
}

2. 컨테이너 시작 (Running):
POST /containers/{id}/start

응답: 204 No Content

3. 컨테이너 정지 (Stopped):
POST /containers/{id}/stop
?t=10  ← Timeout (SIGTERM 후 SIGKILL까지 대기)

응답: 204 No Content

4. 컨테이너 재시작:
POST /containers/{id}/restart
?t=10

5. 컨테이너 강제 종료:
POST /containers/{id}/kill
?signal=SIGKILL

6. 컨테이너 일시 정지:
POST /containers/{id}/pause
POST /containers/{id}/unpause

7. 컨테이너 삭제:
DELETE /containers/{id}
?v=true     ← 볼륨도 삭제
?force=true ← 실행 중이어도 삭제

응답: 204 No Content

전체 흐름:
┌─────────────────────────────────────────────┐
│                                             │
│  POST /containers/create                    │
│      │                                      │
│      ▼                                      │
│  [ Created ]                                │
│      │                                      │
│  POST /containers/{id}/start                │
│      │                                      │
│      ▼                                      │
│  [ Running ] ◄─── POST /restart             │
│      │                                      │
│      │ POST /stop or /kill                  │
│      ▼                                      │
│  [ Stopped ]                                │
│      │                                      │
│  DELETE /containers/{id}                    │
│      │                                      │
│      ▼                                      │
│  [ Removed ]                                │
│                                             │
└─────────────────────────────────────────────┘
```

#### Container 조회 및 모니터링

```
컨테이너 정보 조회:

1. 컨테이너 목록:
GET /containers/json
?all=true        ← 정지된 컨테이너 포함
?limit=10        ← 개수 제한
?filters={"status":["running"]}

응답:
[
  {
    "Id": "abc123...",
    "Names": ["/my-nginx"],
    "Image": "nginx:latest",
    "State": "running",
    "Status": "Up 2 hours",
    "Ports": [
      {
        "PrivatePort": 80,
        "PublicPort": 8080,
        "Type": "tcp"
      }
    ]
  }
]

2. 컨테이너 상세 정보:
GET /containers/{id}/json

응답:
{
  "Id": "abc123...",
  "Created": "2024-01-15T10:00:00Z",
  "Path": "nginx",
  "Args": ["-g", "daemon off;"],
  "State": {
    "Status": "running",
    "Running": true,
    "Paused": false,
    "Restarting": false,
    "OOMKilled": false,
    "Dead": false,
    "Pid": 12345,
    "ExitCode": 0,
    "StartedAt": "2024-01-15T10:00:01Z"
  },
  "Image": "sha256:def456...",
  "HostConfig": {
    "Memory": 536870912,
    "CpuShares": 1024,
    "PortBindings": {...}
  },
  "NetworkSettings": {
    "IPAddress": "172.17.0.2",
    "Ports": {...}
  }
}

3. 컨테이너 로그:
GET /containers/{id}/logs
?stdout=true
?stderr=true
?timestamps=true
?tail=100       ← 마지막 100줄
?follow=true    ← 실시간 스트리밍

응답: 로그 스트림

4. 컨테이너 리소스 사용량 (실시간):
GET /containers/{id}/stats
?stream=true    ← 지속적 스트림
?stream=false   ← 현재 값 1회

응답 (JSON 스트림):
{
  "read": "2024-01-15T10:00:00Z",
  "cpu_stats": {
    "cpu_usage": {
      "total_usage": 1234567890,
      "usage_in_usermode": 123456789,
      "usage_in_kernelmode": 12345678
    },
    "system_cpu_usage": 12345678900
  },
  "memory_stats": {
    "usage": 104857600,  ← 100MB
    "limit": 536870912   ← 512MB
  },
  "networks": {
    "eth0": {
      "rx_bytes": 1024,
      "tx_bytes": 2048
    }
  }
}

5. 컨테이너 프로세스 목록:
GET /containers/{id}/top
?ps_args=aux

응답:
{
  "Titles": ["UID", "PID", "CMD"],
  "Processes": [
    ["root", "12345", "nginx: master"],
    ["nginx", "12346", "nginx: worker"]
  ]
}

6. 컨테이너 파일시스템 변경:
GET /containers/{id}/changes

응답:
[
  {
    "Path": "/tmp/cache.txt",
    "Kind": 1  // 0: Modified, 1: Added, 2: Deleted
  }
]
```

---

### 3. Image API

#### Image 관리

```
이미지 작업:

1. 이미지 Pull:
POST /images/create
?fromImage=nginx
&tag=latest

응답 (JSON Lines 스트림):
{"status":"Pulling from library/nginx","id":"latest"}
{"status":"Pulling fs layer","progressDetail":{},"id":"a3ed95caeb02"}
{"status":"Downloading","progressDetail":{"current":1024,"total":3145728},"progress":"[=>     ]"}
{"status":"Pull complete","id":"a3ed95caeb02"}
{"status":"Digest: sha256:abc123..."}
{"status":"Status: Downloaded newer image for nginx:latest"}

2. 이미지 목록:
GET /images/json
?all=true              ← 중간 레이어 포함
?filters={"dangling":["true"]}  ← dangling 이미지만

응답:
[
  {
    "Id": "sha256:def456...",
    "RepoTags": ["nginx:latest"],
    "RepoDigests": ["nginx@sha256:abc123..."],
    "Created": 1705320000,
    "Size": 187654321,
    "VirtualSize": 187654321
  }
]

3. 이미지 상세 정보:
GET /images/{name}/json

응답:
{
  "Id": "sha256:def456...",
  "RepoTags": ["nginx:latest"],
  "Config": {
    "Image": "sha256:...",
    "Env": ["PATH=/usr/local/sbin:..."],
    "Cmd": ["nginx", "-g", "daemon off;"],
    "ExposedPorts": {"80/tcp": {}}
  },
  "Architecture": "amd64",
  "Os": "linux",
  "Size": 187654321,
  "RootFS": {
    "Type": "layers",
    "Layers": [
      "sha256:a3ed95...",
      "sha256:9f13e0...",
      "sha256:1b2c3d..."
    ]
  }
}

4. 이미지 히스토리:
GET /images/{name}/history

응답:
[
  {
    "Id": "sha256:def456...",
    "Created": 1705320000,
    "CreatedBy": "/bin/sh -c #(nop) CMD [\"nginx\"]",
    "Size": 0
  },
  {
    "Id": "sha256:abc123...",
    "Created": 1705319000,
    "CreatedBy": "/bin/sh -c apt-get update && apt-get install nginx",
    "Size": 157654321
  }
]

5. 이미지 태그:
POST /images/{name}/tag
?repo=myrepo/nginx
&tag=v1.0

6. 이미지 삭제:
DELETE /images/{name}
?force=true
?noprune=false  ← 부모 이미지 유지

응답:
[
  {"Untagged": "nginx:latest"},
  {"Deleted": "sha256:def456..."}
]

7. 이미지 Push:
POST /images/{name}/push
Header: X-Registry-Auth: <base64-encoded-auth-config>

응답 (JSON Lines 스트림):
{"status":"The push refers to repository [docker.io/myrepo/nginx]"}
{"status":"Preparing","progressDetail":{},"id":"a3ed95caeb02"}
{"status":"Pushing","progressDetail":{"current":1024,"total":3145728},"progress":"[=>     ]","id":"a3ed95caeb02"}
{"status":"Pushed","progressDetail":{},"id":"a3ed95caeb02"}
{"status":"v1.0: digest: sha256:abc123... size: 1234"}
```

#### Image Build

```
Dockerfile 기반 빌드:

POST /build
Content-Type: application/x-tar

Body: tar 아카이브 (Dockerfile + 컨텍스트)

Query Parameters:
?dockerfile=Dockerfile       ← Dockerfile 이름
?t=myimage:latest            ← 태그
?nocache=true                ← 캐시 사용 안 함
&buildargs={"ARG1":"value"}  ← 빌드 인자
&labels={"version":"1.0"}    ← 레이블

응답 (JSON Lines 스트림):
{"stream":"Step 1/5 : FROM alpine:3.18"}
{"stream":" ---\u003e abc123...\n"}
{"stream":"Step 2/5 : RUN apk add nginx"}
{"stream":" ---\u003e Running in def456..."}
{"stream":"Successfully built ghi789..."}
{"stream":"Successfully tagged myimage:latest"}

cURL 예시:
tar -czf context.tar.gz -C /path/to/context .
curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/x-tar" \
  --data-binary @context.tar.gz \
  "http://localhost/build?t=myimage:latest"
```

---

### 4. 실시간 이벤트 및 모니터링

#### Events API

```
Docker Events 스트리밍:

GET /events
?since=1705320000     ← Unix timestamp 이후
?until=1705406400     ← Unix timestamp 이전
?filters={"type":["container"],"event":["start","stop"]}

응답 (Server-Sent Events):
{"status":"start","id":"abc123...","from":"nginx:latest","Type":"container","Action":"start","Actor":{"ID":"abc123...","Attributes":{"image":"nginx:latest","name":"my-nginx"}},"time":1705320001,"timeNano":1705320001000000000}
{"status":"stop","id":"abc123...","from":"nginx:latest","Type":"container","Action":"stop","Actor":{"ID":"abc123..."},"time":1705320100}

이벤트 타입:
┌─────────────┬──────────────────────────────────┐
│ Type        │ 이벤트                             │
├─────────────┼──────────────────────────────────┤
│ container   │ create, start, stop, kill, die,  │
│             │ destroy, pause, unpause, exec    │
├─────────────┼──────────────────────────────────┤
│ image       │ pull, push, tag, untag, delete   │
├─────────────┼──────────────────────────────────┤
│ network     │ create, connect, disconnect,     │
│             │ destroy                          │
├─────────────┼──────────────────────────────────┤
│ volume      │ create, mount, unmount, destroy  │
├─────────────┼──────────────────────────────────┤
│ daemon      │ reload                           │
└─────────────┴──────────────────────────────────┘

실시간 모니터링 예:
curl --unix-socket /var/run/docker.sock \
  -X GET \
  "http://localhost/events?filters=%7B%22type%22%3A%5B%22container%22%5D%7D"

# 실행 후 다른 터미널에서:
# docker run --rm alpine echo hi
# → 이벤트 즉시 출력
```

---

### 5. Exec API

#### 컨테이너 내 명령 실행

```
Exec API (docker exec 구현):

1. Exec 인스턴스 생성:
POST /containers/{id}/exec
{
  "AttachStdin": false,
  "AttachStdout": true,
  "AttachStderr": true,
  "Tty": false,
  "Cmd": ["ls", "-la", "/tmp"]
}

응답:
{
  "Id": "exec-abc123..."
}

2. Exec 시작:
POST /exec/{exec-id}/start
{
  "Detach": false,
  "Tty": false
}

응답: 명령 출력 (스트림)

3. Exec 상태 확인:
GET /exec/{exec-id}/json

응답:
{
  "ID": "exec-abc123...",
  "Running": false,
  "ExitCode": 0,
  "ProcessConfig": {
    "entrypoint": "ls",
    "arguments": ["-la", "/tmp"]
  },
  "ContainerID": "abc123..."
}

전체 흐름:
┌──────────────────────────────────────────┐
│ 1. POST /containers/{id}/exec            │
│    → Exec ID 생성                         │
│                                          │
│ 2. POST /exec/{exec-id}/start            │
│    → 명령 실행 + 출력 스트리밍                │
│                                          │
│ 3. GET /exec/{exec-id}/json              │
│    → 종료 코드 확인                         │
└──────────────────────────────────────────┘

Interactive TTY 예:
POST /containers/{id}/exec
{
  "AttachStdin": true,
  "AttachStdout": true,
  "AttachStderr": true,
  "Tty": true,
  "Cmd": ["/bin/sh"]
}

→ WebSocket 또는 HTTP Upgrade로 interactive 세션
```

---

## 🔧 실습 1: curl로 Container 생성 및 실행

### Step 1: API 버전 확인

```bash
# Docker 버전 확인
curl --unix-socket /var/run/docker.sock http://localhost/version | jq

# 출력:
# {
#   "Version": "24.0.7",
#   "ApiVersion": "1.43",
#   "Platform": {"Name": "Docker Engine - Community"},
#   ...
# }

# System 정보
curl --unix-socket /var/run/docker.sock http://localhost/info | jq '.ServerVersion, .Containers, .Images'

# Ping (헬스체크)
curl --unix-socket /var/run/docker.sock http://localhost/_ping
# 출력: OK
```

### Step 2: 이미지 Pull

```bash
# nginx:alpine 이미지 Pull
curl --unix-socket /var/run/docker.sock \
  -X POST \
  "http://localhost/images/create?fromImage=nginx&tag=alpine" \
  | jq -c

# 출력 (JSON Lines):
# {"status":"Pulling from library/nginx","id":"alpine"}
# {"status":"Pulling fs layer","progressDetail":{},"id":"a3ed95caeb02"}
# ...
# {"status":"Status: Downloaded newer image for nginx:alpine"}

# Pull 완료 확인
curl --unix-socket /var/run/docker.sock \
  "http://localhost/images/json?filters=%7B%22reference%22%3A%5B%22nginx%3Aalpine%22%5D%7D" \
  | jq '.[0] | {RepoTags, Size}'

# 출력:
# {
#   "RepoTags": ["nginx:alpine"],
#   "Size": 41254832
# }
```

### Step 3: Container 생성

```bash
# Container 생성 (POST /containers/create)
RESPONSE=$(curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "Image": "nginx:alpine",
    "Cmd": ["nginx", "-g", "daemon off;"],
    "ExposedPorts": {
      "80/tcp": {}
    },
    "HostConfig": {
      "PortBindings": {
        "80/tcp": [{"HostPort": "8080"}]
      },
      "Memory": 268435456,
      "RestartPolicy": {
        "Name": "unless-stopped"
      }
    }
  }' \
  "http://localhost/containers/create?name=api-nginx")

# Container ID 추출
CONTAINER_ID=$(echo $RESPONSE | jq -r '.Id')
echo "Created Container: $CONTAINER_ID"

# 생성된 컨테이너 확인
curl --unix-socket /var/run/docker.sock \
  "http://localhost/containers/$CONTAINER_ID/json" \
  | jq '{Id, Created, State, Image}'
```

### Step 4: Container 시작 및 관리

```bash
# Container 시작
curl --unix-socket /var/run/docker.sock \
  -X POST \
  "http://localhost/containers/$CONTAINER_ID/start"

# HTTP 204 No Content = 성공

# 상태 확인
curl --unix-socket /var/run/docker.sock \
  "http://localhost/containers/$CONTAINER_ID/json" \
  | jq '.State | {Status, Running, Pid, StartedAt}'

# 출력:
# {
#   "Status": "running",
#   "Running": true,
#   "Pid": 12345,
#   "StartedAt": "2024-01-15T10:00:01Z"
# }

# 웹서버 테스트
curl http://localhost:8080
# 출력: Nginx 기본 페이지 HTML

# 로그 확인 (마지막 10줄)
curl --unix-socket /var/run/docker.sock \
  "http://localhost/containers/$CONTAINER_ID/logs?stdout=true&tail=10"

# Container 정지
curl --unix-socket /var/run/docker.sock \
  -X POST \
  "http://localhost/containers/$CONTAINER_ID/stop?t=5"

# Container 삭제
curl --unix-socket /var/run/docker.sock \
  -X DELETE \
  "http://localhost/containers/$CONTAINER_ID?force=true"

echo "Cleanup completed"
```

---

## 🔧 실습 2: 실시간 Stats 모니터링

### Step 1: Container 실행

```bash
# 부하를 생성할 컨테이너 실행
RESPONSE=$(curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "Image": "alpine:latest",
    "Cmd": ["sh", "-c", "while true; do echo test; sleep 1; done"],
    "HostConfig": {
      "Memory": 104857600,
      "CpuShares": 512
    }
  }' \
  "http://localhost/containers/create?name=stats-test")

CONTAINER_ID=$(echo $RESPONSE | jq -r '.Id')

curl --unix-socket /var/run/docker.sock \
  -X POST \
  "http://localhost/containers/$CONTAINER_ID/start"

echo "Container started: $CONTAINER_ID"
```

### Step 2: Stats 스트리밍 (실시간)

```bash
# 실시간 stats 모니터링
curl --unix-socket /var/run/docker.sock \
  "http://localhost/containers/$CONTAINER_ID/stats?stream=true" \
  | jq -c '{
    read,
    cpu_percent: ((.cpu_stats.cpu_usage.total_usage - .precpu_stats.cpu_usage.total_usage) / (.cpu_stats.system_cpu_usage - .precpu_stats.system_cpu_usage) * 100 | floor),
    memory_usage: (.memory_stats.usage / 1048576 | floor),
    memory_limit: (.memory_stats.limit / 1048576 | floor),
    network_rx: (.networks.eth0.rx_bytes // 0),
    network_tx: (.networks.eth0.tx_bytes // 0)
  }'

# 출력 (매초 업데이트):
# {"read":"2024-01-15T10:00:00Z","cpu_percent":5,"memory_usage":2,"memory_limit":100,"network_rx":1024,"network_tx":512}
# {"read":"2024-01-15T10:00:01Z","cpu_percent":3,"memory_usage":2,"memory_limit":100,"network_rx":1536,"network_tx":768}
# ...

# Ctrl+C로 중단
```

### Step 3: Stats 단일 스냅샷

```bash
# 현재 리소스 사용량 1회만
curl --unix-socket /var/run/docker.sock \
  "http://localhost/containers/$CONTAINER_ID/stats?stream=false" \
  | jq '{
    memory_usage_mb: (.memory_stats.usage / 1048576 | floor),
    memory_limit_mb: (.memory_stats.limit / 1048576 | floor),
    memory_percent: ((.memory_stats.usage / .memory_stats.limit) * 100 | floor),
    pids: .pids_stats.current
  }'

# 출력:
# {
#   "memory_usage_mb": 2,
#   "memory_limit_mb": 100,
#   "memory_percent": 2,
#   "pids": 2
# }

# 정리
curl --unix-socket /var/run/docker.sock \
  -X DELETE \
  "http://localhost/containers/$CONTAINER_ID?force=true"
```

---

## 🔧 실습 3: Events 실시간 모니터링

### Step 1: Events 스트림 시작

```bash
# 백그라운드로 events 모니터링 시작
curl --unix-socket /var/run/docker.sock \
  "http://localhost/events?filters=%7B%22type%22%3A%5B%22container%22%5D%7D" \
  | jq -c '{time: (.time | strftime("%Y-%m-%d %H:%M:%S")), action: .Action, id: .Actor.ID[:12], name: .Actor.Attributes.name}' &

EVENTS_PID=$!
echo "Events monitoring started (PID: $EVENTS_PID)"
```

### Step 2: Container 작업 수행

```bash
# Container 생성 → start 이벤트 발생
docker run -d --name event-test alpine sleep 30

# 출력 (events 스트림):
# {"time":"2024-01-15 10:00:00","action":"create","id":"abc123456789","name":"event-test"}
# {"time":"2024-01-15 10:00:00","action":"start","id":"abc123456789","name":"event-test"}

sleep 2

# Container 정지 → stop 이벤트 발생
docker stop event-test

# 출력:
# {"time":"2024-01-15 10:00:02","action":"kill","id":"abc123456789","name":"event-test"}
# {"time":"2024-01-15 10:00:02","action":"die","id":"abc123456789","name":"event-test"}
# {"time":"2024-01-15 10:00:02","action":"stop","id":"abc123456789","name":"event-test"}

sleep 2

# Container 삭제 → destroy 이벤트 발생
docker rm event-test

# 출력:
# {"time":"2024-01-15 10:00:04","action":"destroy","id":"abc123456789","name":"event-test"}
```

### Step 3: Events 모니터링 종료

```bash
# Events 프로세스 종료
kill $EVENTS_PID 2>/dev/null || true

echo "Events monitoring stopped"
```

---

## 🔧 실습 4: Exec API로 명령 실행

### Step 1: Container 준비

```bash
# 백그라운드 컨테이너 실행
docker run -d --name exec-test alpine sleep 300
CONTAINER_ID=$(docker ps -qf name=exec-test)
echo "Container ID: $CONTAINER_ID"
```

### Step 2: Exec 인스턴스 생성 및 실행

```bash
# 1. Exec 인스턴스 생성 (명령 등록)
EXEC_RESPONSE=$(curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "AttachStdin": false,
    "AttachStdout": true,
    "AttachStderr": true,
    "Tty": false,
    "Cmd": ["ls", "-la", "/"]
  }' \
  "http://localhost/containers/$CONTAINER_ID/exec")

EXEC_ID=$(echo $EXEC_RESPONSE | jq -r '.Id')
echo "Exec ID: $EXEC_ID"

# 2. Exec 실행 (명령 시작)
curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "Detach": false,
    "Tty": false
  }' \
  "http://localhost/exec/$EXEC_ID/start"

# 출력: ls -la / 결과
# total 64
# drwxr-xr-x   1 root root 4096 Jan 15 10:00 .
# drwxr-xr-x   1 root root 4096 Jan 15 10:00 ..
# drwxr-xr-x   2 root root 4096 Dec 28 12:00 bin
# ...

# 3. Exec 상태 확인 (종료 코드)
curl --unix-socket /var/run/docker.sock \
  "http://localhost/exec/$EXEC_ID/json" \
  | jq '{ID, Running, ExitCode}'

# 출력:
# {
#   "ID": "exec-abc123...",
#   "Running": false,
#   "ExitCode": 0
# }
```

### Step 3: 여러 명령 실행

```bash
# hostname 확인
EXEC_ID=$(curl -s --unix-socket /var/run/docker.sock \
  -X POST -H "Content-Type: application/json" \
  -d '{"Cmd": ["hostname"]}' \
  "http://localhost/containers/$CONTAINER_ID/exec" | jq -r '.Id')

curl -s --unix-socket /var/run/docker.sock \
  -X POST -H "Content-Type: application/json" \
  -d '{"Detach": false}' \
  "http://localhost/exec/$EXEC_ID/start"

# 출력: <container-hostname>

# 프로세스 목록
EXEC_ID=$(curl -s --unix-socket /var/run/docker.sock \
  -X POST -H "Content-Type: application/json" \
  -d '{"Cmd": ["ps", "aux"]}' \
  "http://localhost/containers/$CONTAINER_ID/exec" | jq -r '.Id')

curl -s --unix-socket /var/run/docker.sock \
  -X POST -H "Content-Type: application/json" \
  -d '{"Detach": false}' \
  "http://localhost/exec/$EXEC_ID/start"

# 출력:
# PID   USER     COMMAND
# 1     root     sleep 300
# ...

# 정리
docker rm -f exec-test
```

---

## 🔧 실습 5: Network 및 Volume API

### Step 1: Network 생성 및 관리

```bash
# 1. Network 생성
NETWORK_RESPONSE=$(curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "api-network",
    "Driver": "bridge",
    "IPAM": {
      "Config": [
        {
          "Subnet": "172.20.0.0/16",
          "Gateway": "172.20.0.1"
        }
      ]
    },
    "Labels": {
      "com.example.project": "api-demo"
    }
  }' \
  "http://localhost/networks/create")

NETWORK_ID=$(echo $NETWORK_RESPONSE | jq -r '.Id')
echo "Network ID: $NETWORK_ID"

# 2. Network 목록 확인
curl --unix-socket /var/run/docker.sock \
  "http://localhost/networks?filters=%7B%22name%22%3A%5B%22api-network%22%5D%7D" \
  | jq '.[0] | {Name, Driver, Scope, IPAM}'

# 3. Container 생성 및 Network 연결
CONTAINER_ID=$(curl -s --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"Image": "alpine:latest", "Cmd": ["sleep", "60"]}' \
  "http://localhost/containers/create?name=net-test" | jq -r '.Id')

curl --unix-socket /var/run/docker.sock \
  -X POST \
  "http://localhost/containers/$CONTAINER_ID/start"

# 4. Container를 Network에 연결
curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d "{
    \"Container\": \"$CONTAINER_ID\",
    \"EndpointConfig\": {
      \"IPAMConfig\": {
        \"IPv4Address\": \"172.20.0.10\"
      }
    }
  }" \
  "http://localhost/networks/$NETWORK_ID/connect"

# 5. Network 상세 확인 (연결된 컨테이너)
curl --unix-socket /var/run/docker.sock \
  "http://localhost/networks/$NETWORK_ID" \
  | jq '.Containers'

# 6. Container에서 Network 확인
docker exec net-test ip addr show eth0

# 7. 정리
curl --unix-socket /var/run/docker.sock \
  -X DELETE \
  "http://localhost/containers/$CONTAINER_ID?force=true"

curl --unix-socket /var/run/docker.sock \
  -X DELETE \
  "http://localhost/networks/$NETWORK_ID"
```

### Step 2: Volume 생성 및 사용

```bash
# 1. Volume 생성
VOLUME_RESPONSE=$(curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "api-volume",
    "Driver": "local",
    "Labels": {
      "com.example.type": "data"
    }
  }' \
  "http://localhost/volumes/create")

VOLUME_NAME=$(echo $VOLUME_RESPONSE | jq -r '.Name')
echo "Volume Name: $VOLUME_NAME"

# 2. Volume 목록
curl --unix-socket /var/run/docker.sock \
  "http://localhost/volumes" \
  | jq '.Volumes[] | select(.Name == "api-volume") | {Name, Driver, Mountpoint}'

# 3. Volume 상세 정보
curl --unix-socket /var/run/docker.sock \
  "http://localhost/volumes/$VOLUME_NAME" \
  | jq

# 4. Volume을 사용하는 Container 생성
CONTAINER_ID=$(curl -s --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d "{
    \"Image\": \"alpine:latest\",
    \"Cmd\": [\"sh\", \"-c\", \"echo data > /data/test.txt; cat /data/test.txt; sleep 30\"],
    \"HostConfig\": {
      \"Binds\": [\"$VOLUME_NAME:/data\"]
    }
  }" \
  "http://localhost/containers/create?name=vol-test" | jq -r '.Id')

curl --unix-socket /var/run/docker.sock \
  -X POST \
  "http://localhost/containers/$CONTAINER_ID/start"

# 로그 확인 (data 출력됨)
sleep 2
curl --unix-socket /var/run/docker.sock \
  "http://localhost/containers/$CONTAINER_ID/logs?stdout=true"

# 5. 정리
docker rm -f vol-test

curl --unix-socket /var/run/docker.sock \
  -X DELETE \
  "http://localhost/volumes/$VOLUME_NAME"
```

---

## 💡 주요 API 엔드포인트 정리

```bash
# ========== System ==========
GET    /_ping                          # 헬스체크
GET    /version                        # Docker 버전
GET    /info                           # 시스템 정보
GET    /events                         # 이벤트 스트림

# ========== Containers ==========
POST   /containers/create              # 컨테이너 생성
GET    /containers/json                # 컨테이너 목록
GET    /containers/{id}/json           # 컨테이너 상세
POST   /containers/{id}/start          # 시작
POST   /containers/{id}/stop           # 정지
POST   /containers/{id}/restart        # 재시작
POST   /containers/{id}/kill           # 강제 종료
POST   /containers/{id}/pause          # 일시 정지
POST   /containers/{id}/unpause        # 재개
DELETE /containers/{id}                # 삭제
GET    /containers/{id}/logs           # 로그
GET    /containers/{id}/stats          # 리소스 사용량
GET    /containers/{id}/top            # 프로세스 목록
POST   /containers/{id}/exec           # Exec 생성
POST   /exec/{id}/start                # Exec 시작
GET    /exec/{id}/json                 # Exec 정보

# ========== Images ==========
POST   /images/create                  # Pull
GET    /images/json                    # 이미지 목록
GET    /images/{name}/json             # 이미지 상세
GET    /images/{name}/history          # 히스토리
POST   /images/{name}/tag              # 태그
POST   /images/{name}/push             # Push
DELETE /images/{name}                  # 삭제
POST   /build                          # Dockerfile 빌드

# ========== Networks ==========
POST   /networks/create                # 네트워크 생성
GET    /networks                       # 네트워크 목록
GET    /networks/{id}                  # 네트워크 상세
DELETE /networks/{id}                  # 삭제
POST   /networks/{id}/connect          # 컨테이너 연결
POST   /networks/{id}/disconnect       # 컨테이너 분리

# ========== Volumes ==========
POST   /volumes/create                 # 볼륨 생성
GET    /volumes                        # 볼륨 목록
GET    /volumes/{name}                 # 볼륨 상세
DELETE /volumes/{name}                 # 삭제

# ========== curl 사용 패턴 ==========
# Unix Socket 사용 (로컬)
curl --unix-socket /var/run/docker.sock \
  -X GET \
  "http://localhost/containers/json"

# JSON Body 전송
curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"Image": "nginx"}' \
  "http://localhost/containers/create"

# Query Parameters (URL 인코딩 필요)
curl --unix-socket /var/run/docker.sock \
  "http://localhost/containers/json?all=true&filters=%7B%22status%22%3A%5B%22exited%22%5D%7D"

# 스트림 응답 (jq로 파싱)
curl --unix-socket /var/run/docker.sock \
  "http://localhost/events" | jq -c
```

---

## 🎓 연습 문제

### 문제 1: Docker CLI 명령어를 API 호출로 변환하면?

다음 Docker 명령어를 API 호출 순서로 나열하시오:
```bash
docker run -d -p 8080:80 --name web nginx
```

<details>
<summary>정답 보기</summary>

```bash
# 1. 이미지가 없으면 Pull
POST /images/create?fromImage=nginx&tag=latest

# 2. Container 생성
POST /containers/create?name=web
{
  "Image": "nginx",
  "ExposedPorts": {
    "80/tcp": {}
  },
  "HostConfig": {
    "PortBindings": {
      "80/tcp": [{"HostPort": "8080"}]
    }
  }
}

# 응답: {"Id": "abc123..."}

# 3. Container 시작
POST /containers/abc123.../start

# docker run = 위 3단계를 한 번에 수행하는 wrapper
```

**단계별 설명:**
1. **이미지 확인/Pull**: 로컬에 nginx 이미지가 없으면 레지스트리에서 다운로드
2. **Container 생성**: 
   - `Image`: 사용할 이미지
   - `ExposedPorts`: 컨테이너 내부 포트 선언 (메타데이터)
   - `HostConfig.PortBindings`: 실제 포트 매핑 (8080 → 80)
3. **Container 시작**: Created 상태에서 Running으로 전환

**-d 옵션은?**
CLI 레벨에서 처리됨. API는 항상 비동기 (detached).

**추가 옵션 매핑:**
```bash
# 메모리 제한
docker run --memory=512m nginx
→ "HostConfig": {"Memory": 536870912}

# CPU 제한
docker run --cpus=2 nginx
→ "HostConfig": {"NanoCpus": 2000000000}

# 환경 변수
docker run -e FOO=bar nginx
→ "Env": ["FOO=bar"]

# 볼륨 마운트
docker run -v /host:/container nginx
→ "HostConfig": {"Binds": ["/host:/container"]}
```

</details>

### 문제 2: API를 사용해 모든 정지된 컨테이너를 삭제하는 스크립트를 작성하시오.

<details>
<summary>정답 보기</summary>

```bash
#!/bin/bash

# 1. 정지된 컨테이너 목록 조회 (filters: status=exited)
STOPPED_CONTAINERS=$(curl -s --unix-socket /var/run/docker.sock \
  "http://localhost/containers/json?all=true&filters=%7B%22status%22%3A%5B%22exited%22%5D%7D" \
  | jq -r '.[].Id')

# 2. 각 컨테이너 삭제
for CONTAINER_ID in $STOPPED_CONTAINERS; do
  echo "Deleting container: $CONTAINER_ID"
  
  curl -s --unix-socket /var/run/docker.sock \
    -X DELETE \
    "http://localhost/containers/$CONTAINER_ID" \
    > /dev/null
  
  if [ $? -eq 0 ]; then
    echo "  ✅ Deleted"
  else
    echo "  ❌ Failed"
  fi
done

echo "Cleanup completed"
```

**Query Parameter 인코딩:**
```
원본: {"status":["exited"]}
인코딩: %7B%22status%22%3A%5B%22exited%22%5D%7D

또는 jq로 동적 생성:
FILTER=$(echo '{"status":["exited"]}' | jq -sRr @uri)
curl "...?filters=$FILTER"
```

**개선 버전 (prune API 사용):**
```bash
# Docker의 prune API 사용 (더 효율적)
curl -s --unix-socket /var/run/docker.sock \
  -X POST \
  "http://localhost/containers/prune" \
  | jq

# 출력:
# {
#   "ContainersDeleted": ["abc123...", "def456..."],
#   "SpaceReclaimed": 12345678
# }
```

**docker container prune = POST /containers/prune**

</details>

### 문제 3: Events API로 컨테이너가 OOMKilled 되는 순간을 감지하는 스크립트를 작성하시오.

<details>
<summary>정답 보기</summary>

```bash
#!/bin/bash

echo "Monitoring for OOMKilled containers..."
echo "Press Ctrl+C to stop"
echo ""

# Events API 스트림 모니터링
curl -N --unix-socket /var/run/docker.sock \
  "http://localhost/events?filters=%7B%22type%22%3A%5B%22container%22%5D%2C%22event%22%3A%5B%22die%22%5D%7D" \
  | while IFS= read -r line; do
    # JSON 파싱
    CONTAINER_ID=$(echo "$line" | jq -r '.Actor.ID[:12]')
    CONTAINER_NAME=$(echo "$line" | jq -r '.Actor.Attributes.name')
    EXIT_CODE=$(echo "$line" | jq -r '.Actor.Attributes.exitCode')
    
    # OOMKilled 확인 (exitCode 137 = SIGKILL by OOM)
    if [ "$EXIT_CODE" = "137" ]; then
      # Container 정보 조회
      INSPECT=$(curl -s --unix-socket /var/run/docker.sock \
        "http://localhost/containers/$CONTAINER_ID/json")
      
      OOM_KILLED=$(echo "$INSPECT" | jq -r '.State.OOMKilled')
      
      if [ "$OOM_KILLED" = "true" ]; then
        MEMORY_LIMIT=$(echo "$INSPECT" | jq -r '.HostConfig.Memory')
        MEMORY_LIMIT_MB=$((MEMORY_LIMIT / 1048576))
        
        echo "🔴 OOMKilled Detected!"
        echo "  Container: $CONTAINER_NAME ($CONTAINER_ID)"
        echo "  Memory Limit: ${MEMORY_LIMIT_MB}MB"
        echo "  Time: $(date)"
        echo ""
        
        # 선택: Slack 알림, 로그 전송 등
        # curl -X POST https://hooks.slack.com/...
      fi
    fi
  done
```

**테스트 방법:**
```bash
# 스크립트 실행
./oom-monitor.sh &

# OOM 유발 컨테이너 실행 (메모리 제한 10MB)
docker run --rm --memory=10m alpine sh -c \
  'dd if=/dev/zero of=/tmp/fill bs=1M count=100'

# 출력:
# 🔴 OOMKilled Detected!
#   Container: eloquent_turing (abc123456789)
#   Memory Limit: 10MB
#   Time: Mon Jan 15 10:00:00 UTC 2024
```

**Filter 설명:**
```json
{
  "type": ["container"],   // container 이벤트만
  "event": ["die"]         // die 이벤트만 (종료 시)
}
```

**OOMKilled 판별:**
1. `die` 이벤트의 `exitCode`가 137 (SIGKILL)
2. `/containers/{id}/json`의 `State.OOMKilled`가 true

**프로덕션 개선:**
- 이벤트를 파일/DB에 로깅
- 메모리 부족 패턴 분석
- 자동 메모리 증가 (Kubernetes HPA 연동)

</details>

---

## 📌 핵심 요약

```
┌──────────────────┬────────────────────────────────────┐
│ 개념              │ 설명                                │
├──────────────────┼────────────────────────────────────┤
│ Docker API       │ Docker Engine의 REST API           │
│                  │ 모든 Docker 기능을 HTTP로 제공         │
├──────────────────┼────────────────────────────────────┤
│ Unix Socket      │ /var/run/docker.sock               │
│                  │ 로컬 통신, 파일 권한 기반 접근 제어        │
├──────────────────┼────────────────────────────────────┤
│ API Version      │ v1.43 (Docker 23.0+)               │
│                  │ 하위 호환성 지원                       │
├──────────────────┼────────────────────────────────────┤
│ Container API    │ create, start, stop, kill, delete  │
│                  │ logs, stats, exec                  │
├──────────────────┼────────────────────────────────────┤
│ Image API        │ create (pull), build, push, tag    │
│                  │ inspect, history, delete           │
├──────────────────┼────────────────────────────────────┤
│ Events API       │ 실시간 이벤트 스트림                    │
│                  │ container, image, network, volume  │
├──────────────────┼────────────────────────────────────┤
│ Stats API        │ 실시간 리소스 모니터링                   │
│                  │ CPU, Memory, Network, I/O          │
├──────────────────┼────────────────────────────────────┤
│ Exec API         │ 컨테이너 내 명령 실행                   │
│                  │ create exec → start exec           │
├──────────────────┼────────────────────────────────────┤
│ Network API      │ 네트워크 생성, 연결, 분리               │
├──────────────────┼────────────────────────────────────┤
│ Volume API       │ 볼륨 생성, 조회, 삭제                  │
└──────────────────┴────────────────────────────────────┘

Docker CLI → API 매핑:
docker run          → POST /containers/create + /start
docker ps           → GET /containers/json
docker logs         → GET /containers/{id}/logs
docker exec         → POST /containers/{id}/exec + /start
docker pull         → POST /images/create
docker build        → POST /build
docker stop         → POST /containers/{id}/stop
docker rm           → DELETE /containers/{id}
docker events       → GET /events
docker stats        → GET /containers/{id}/stats
```

---

## 📚 참고 자료

- [Docker Engine API Reference](https://docs.docker.com/engine/api/latest/)
- [Docker SDK for Python](https://docker-py.readthedocs.io/)
- [Docker SDK for Go](https://pkg.go.dev/github.com/docker/docker/client)
- [API Version History](https://docs.docker.com/engine/api/version-history/)
- [Remote API Tutorial](https://docs.docker.com/engine/api/getting-started/)

---

## 🤔 생각해볼 문제

1. Docker CLI를 사용하지 않고 API만으로 프로덕션 환경을 운영할 수 있을까? 무엇이 필요할까?
2. Unix Socket 대신 TCP로 Docker API를 노출할 때 주의해야 할 보안 사항은?
3. Events API를 활용한 자동화 시스템을 어떻게 설계할 수 있을까?

> 💡 **답변**:
> 
> **1) API만으로 프로덕션 운영:**
> 
> **충분히 가능하지만 추가 구현 필요:**
> ```
> Docker CLI가 제공하는 편의 기능:
> ❌ 컬러 출력, 포맷팅
> ❌ Context 관리 (여러 Docker 호스트)
> ❌ Credential helper (레지스트리 인증)
> ❌ Compose 파일 파싱 (docker-compose)
> ❌ Buildx (멀티 플랫폼 빌드)
> 
> API로 구현 가능:
> ✅ Container CRUD (완전 지원)
> ✅ Image 관리 (pull, push, build)
> ✅ Network/Volume 관리
> ✅ 실시간 모니터링 (stats, events)
> ✅ 자동화 스크립트
> ```
> 
> **필요한 추가 구현:**
> - **인증 관리**: 레지스트리 로그인 토큰 관리
> - **오류 처리**: API 응답 코드 파싱 및 재시도 로직
> - **병렬 처리**: 여러 컨테이너 동시 관리
> - **상태 추적**: 비동기 작업 완료 대기 (pull, build)
> - **로깅**: API 호출 이력 및 감사 로그
> 
> **실무 사례:**
> - Kubernetes: CRI를 통해 containerd API 직접 사용 (Docker CLI 불필요)
> - CI/CD: Jenkins Docker Plugin은 API를 사용
> - 모니터링: Prometheus, Datadog 등은 API로 메트릭 수집
> 
> **2) TCP 노출 시 보안:**
> 
> ```
> 위험한 설정 (절대 금지):
> dockerd -H tcp://0.0.0.0:2375
> 
> 문제점:
> ❌ 암호화 없음 (평문 통신)
> ❌ 인증 없음 (누구나 접근)
> ❌ Root 권한 획득 가능
> ❌ 호스트 탈출 가능
> 
> 실제 공격 사례:
> 1. 포트 스캔으로 2375 발견
> 2. API로 privileged 컨테이너 생성
> 3. 호스트 파일시스템 마운트
> 4. Root 권한 획득
> 5. 크립토마이너 설치
> ```
> 
> **안전한 설정 (TLS):**
> ```bash
> # 1. 인증서 생성 (CA, Server, Client)
> openssl genrsa -out ca-key.pem 4096
> openssl req -new -x509 -key ca-key.pem -out ca.pem
> 
> openssl genrsa -out server-key.pem 4096
> openssl req -new -key server-key.pem -out server.csr
> openssl x509 -req -in server.csr -CA ca.pem -CAkey ca-key.pem \
>   -out server-cert.pem
> 
> # 2. dockerd 설정
> dockerd \
>   --tlsverify \
>   --tlscacert=ca.pem \
>   --tlscert=server-cert.pem \
>   --tlskey=server-key.pem \
>   -H tcp://0.0.0.0:2376
> 
> # 3. 클라이언트 접근
> curl --cacert ca.pem \
>      --cert client-cert.pem \
>      --key client-key.pem \
>      https://host:2376/containers/json
> ```
> 
> **추가 보안 조치:**
> - **방화벽**: 특정 IP만 2376 접근 허용
> - **VPN**: Private network로 제한
> - **Authorization Plugin**: API 호출 권한 제어
> - **Audit Log**: 모든 API 호출 기록
> - **TLS 버전**: TLS 1.2+ 강제
> - **정기 인증서 갱신**: Let's Encrypt, cert-manager
> 
> **3) Events API 자동화 시스템:**
> 
> ```
> 자동화 시나리오:
> 
> 1. Auto Scaling:
>    Events: container/die, container/oom
>    Action: 
>    - OOM 감지 시 메모리 증가
>    - 연속 재시작 감지 시 알림
>    - 리소스 사용률 기반 스케일링
> 
> 2. 헬스 체크:
>    Events: container/health_status
>    Action:
>    - unhealthy 감지 시 재시작
>    - 연속 실패 시 알림
>    - 자동 롤백
> 
> 3. 로깅/모니터링:
>    Events: container/start, container/stop
>    Action:
>    - 컨테이너 생명주기 로깅
>    - Slack/PagerDuty 알림
>    - Prometheus metric 생성
> 
> 4. 보안 감사:
>    Events: image/pull, container/create
>    Action:
>    - 미승인 이미지 감지
>    - Privileged 컨테이너 경고
>    - 호스트 마운트 추적
> 
> 5. 리소스 정리:
>    Events: container/die
>    Action:
>    - 오래된 exited 컨테이너 삭제
>    - Dangling 이미지 정리
>    - 미사용 볼륨 제거
> ```
> 
> **시스템 아키텍처:**
> ```
> ┌──────────────────────────────────────┐
> │ Docker Engine                        │
> │ GET /events (SSE Stream)             │
> └──────────────┬───────────────────────┘
>                │ Event Stream
> ┌──────────────▼───────────────────────┐
> │ Event Processor                      │
> │ - Event 수신 및 파싱                    │
> │ - Filter 적용                         │
> │ - Handler 라우팅                       │
> └──────────────┬───────────────────────┘
>                │
>      ┌─────────┴─────────┬────────────┐
>      │                   │            │
> ┌────▼────┐         ┌────▼────┐   ┌───▼────┐
> │ Handler │         │ Handler │   │Handler │
> │ 1       │         │ 2       │   │ 3      │
> │ (Scale) │         │ (Alert) │   │ (Log)  │
> └─────────┘         └─────────┘   └────────┘
> ```
> 
> **구현 예 (Go):**
> ```go
> client, _ := docker.NewClientWithOpts()
> events, _ := client.Events(context.Background(), types.EventsOptions{
>     Filters: filters.NewArgs(
>         filters.Arg("type", "container"),
>         filters.Arg("event", "die"),
>     ),
> })
> 
> for event := range events {
>     if event.Actor.Attributes["exitCode"] == "137" {
>         containerID := event.Actor.ID
>         inspect, _ := client.ContainerInspect(ctx, containerID)
>         
>         if inspect.State.OOMKilled {
>             // Handle OOMKilled
>             log.Printf("OOMKilled: %s", containerID)
>             sendAlert(containerID)
>         }
>     }
> }
> ```

---

<div align="center">

**[⬅️ 이전: runc](./04-runc.md)** | **[다음: Docker SDK ➡️](./06-Docker-SDK.md)**

</div>
