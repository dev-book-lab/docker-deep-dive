# 07. Custom Plugins - Docker 플러그인 개발

## 🎯 이 챕터에서 배울 것

- **Plugin 아키텍처**: Docker의 플러그인 시스템 이해
- **Volume Plugin**: 커스텀 스토리지 드라이버 개발
- **Network Plugin**: CNI 기반 네트워크 플러그인
- **Authorization Plugin**: 접근 제어 플러그인
- **Plugin API**: HTTP/JSON-RPC 인터페이스
- **배포 및 관리**: Plugin 패키징 및 설치

## 📌 왜 중요한가?

**"플러그인을 통해 Docker의 핵심 기능을 확장하고 커스터마이징할 수 있습니다."**

```
Docker의 확장 포인트:

┌─────────────────────────────────────────────────────┐
│                   Docker Engine                     │
│                                                     │
│  ┌────────────────────────────────────────────┐     │
│  │ Core Functionality                         │     │
│  │ - Container Runtime                        │     │
│  │ - Image Management                         │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ┌────────────────────────────────────────────┐     │
│  │ Built-in Drivers                           │     │
│  │ - Volume: local                            │     │
│  │ - Network: bridge, host, overlay           │     │
│  │ - Logging: json-file, syslog               │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ┌────────────────────────────────────────────┐     │
│  │ Plugin Interface ◄─── 확장 포인트!            │     │
│  │                                            │     │
│  │ - Volume Plugin API                        │     │
│  │ - Network Plugin API                       │     │
│  │ - Authorization Plugin API                 │     │
│  │ - Logging Plugin API                       │     │
│  └────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────┘
                     │
                     │ HTTP/Unix Socket
                     ▼
┌─────────────────────────────────────────────────────┐
│ Custom Plugins (외부 프로세스)                         │
│                                                     │
│ ┌──────────────┐  ┌──────────────┐  ┌───────────┐   │
│ │ Volume       │  │ Network      │  │ AuthZ     │   │
│ │ Plugin       │  │ Plugin       │  │ Plugin    │   │
│ │              │  │              │  │           │   │
│ │ - NFS        │  │ - Calico     │  │ - LDAP    │   │
│ │ - GlusterFS  │  │ - Weave      │  │ - OAuth   │   │
│ │ - S3         │  │ - Cilium     │  │ - Custom  │   │
│ └──────────────┘  └──────────────┘  └───────────┘   │
└─────────────────────────────────────────────────────┘

Plugin의 장점:
✅ Docker 재컴파일 없이 기능 확장
✅ 언어 독립적 (HTTP API)
✅ 독립 프로세스 (안정성)
✅ 생태계 활용 (공개 플러그인)

Plugin 종류:
┌──────────────────┬──────────────────────────────┐
│ 타입              │ 용도                          │
├──────────────────┼──────────────────────────────┤
│ Volume           │ 커스텀 스토리지 백엔드             │
│                  │ (NFS, S3, Ceph)              │
├──────────────────┼──────────────────────────────┤
│ Network          │ CNI 네트워크 드라이버            │
│                  │ (Calico, Weave, Cilium)      │
├──────────────────┼──────────────────────────────┤
│ Authorization    │ 접근 제어 (RBAC, LDAP)         │
├──────────────────┼──────────────────────────────┤
│ Logging          │ 로그 드라이버                   │
│                  │ (Splunk, Fluentd)            │
├──────────────────┼──────────────────────────────┤
│ Metrics          │ 메트릭 수집기                   │
└──────────────────┴──────────────────────────────┘
```

**실무 영향:**
- **엔터프라이즈 통합**: 기존 스토리지/네트워크 인프라 연동
- **보안 강화**: 커스텀 인증/인가 정책
- **멀티테넌시**: 격리된 리소스 제공
- **클라우드 통합**: AWS EBS, Azure Disk 등

---

## 🔬 Deep Dive

### 1. Plugin 아키텍처

#### Plugin 통신 방식

```
Docker ↔ Plugin 통신:

┌──────────────────────────────────────────────────┐
│ Docker Engine                                    │
│                                                  │
│  1. Plugin Discovery                             │
│     /run/docker/plugins/<name>.sock 확인          │
│     또는 /etc/docker/plugins/<name>.spec          │
│                                                  │
│  2. Plugin Activation                            │
│     POST /Plugin.Activate                        │
│     → 응답: {"Implements": ["VolumeDriver"]}      │
│                                                  │
│  3. API Calls                                    │
│     POST /VolumeDriver.Create                    │
│     POST /VolumeDriver.Mount                     │
│     POST /VolumeDriver.Unmount                   │
│     ...                                          │
└──────────────────┬───────────────────────────────┘
                   │ Unix Socket
                   │ (또는 HTTP)
┌──────────────────▼───────────────────────────────┐
│ Plugin Process                                   │
│                                                  │
│  HTTP Server                                     │
│  - Listen on Unix Socket                         │
│  - Handle JSON-RPC requests                      │
│  - Return JSON responses                         │
│                                                  │
│  Plugin Logic                                    │
│  - Volume: 스토리지 조작                            │
│  - Network: 네트워크 설정                           │
│  - AuthZ: 권한 검증                                │
└──────────────────────────────────────────────────┘

Plugin 설정 파일 예:
/etc/docker/plugins/myplugin.spec
{
  "Name": "myplugin",
  "Addr": "unix:///run/docker/plugins/myplugin.sock"
}

또는 직접 Socket 생성:
/run/docker/plugins/myplugin.sock
```

#### Plugin 생명주기

```
Plugin Lifecycle:

1. 등록 (Registration):
   - Socket 파일 생성
   - 또는 .spec 파일 배치

2. 활성화 (Activation):
   Docker → Plugin
   POST /Plugin.Activate
   
   Plugin → Docker
   {"Implements": ["VolumeDriver", "NetworkDriver"]}

3. 사용 (Usage):
   - Volume: Create, Mount, Unmount, Remove
   - Network: CreateNetwork, CreateEndpoint, Join, Leave
   - AuthZ: AuthZReq, AuthZRes

4. 비활성화 (Deactivation):
   - Docker 종료 시
   - Plugin 프로세스 종료

5. 제거 (Removal):
   - Socket 파일 삭제
   - 리소스 정리
```

---

### 2. Volume Plugin

#### Volume Plugin API

```
Volume Driver Interface:

┌──────────────────────────────────────────────────┐
│ Volume Plugin API                                │
├──────────────────────────────────────────────────┤
│ POST /Plugin.Activate                            │
│   → {"Implements": ["VolumeDriver"]}             │
├──────────────────────────────────────────────────┤
│ POST /VolumeDriver.Create                        │
│   Request: {"Name": "vol1", "Opts": {...}}       │
│   Response: {"Err": ""}                          │
├──────────────────────────────────────────────────┤
│ POST /VolumeDriver.Remove                        │
│   Request: {"Name": "vol1"}                      │
│   Response: {"Err": ""}                          │
├──────────────────────────────────────────────────┤
│ POST /VolumeDriver.Mount                         │
│   Request: {"Name": "vol1", "ID": "..."}         │
│   Response: {"Mountpoint":"/mnt/vol1", "Err": ""}│
├──────────────────────────────────────────────────┤
│ POST /VolumeDriver.Unmount                       │
│   Request: {"Name": "vol1", "ID": "..."}         │
│   Response: {"Err": ""}                          │
├──────────────────────────────────────────────────┤
│ POST /VolumeDriver.Get                           │
│   Request: {"Name": "vol1"}                      │
│   Response: {"Volume": {...}, "Err": ""}         │
├──────────────────────────────────────────────────┤
│ POST /VolumeDriver.List                          │
│   Request: {}                                    │
│   Response: {"Volumes": [...], "Err": ""}        │
├──────────────────────────────────────────────────┤
│ POST /VolumeDriver.Path                          │
│   Request: {"Name": "vol1"}                      │
│   Response: {"Mountpoint":"/mnt/vol1", "Err": ""}│
├──────────────────────────────────────────────────┤
│ POST /VolumeDriver.Capabilities                  │
│   Response: {"Capabilities": {"Scope": "local"}} │
└──────────────────────────────────────────────────┘

Volume 사용 흐름:
docker volume create --driver=myplugin vol1
  → POST /VolumeDriver.Create {"Name": "vol1"}

docker run -v vol1:/data myapp
  → POST /VolumeDriver.Mount {"Name": "vol1", "ID": "container-id"}
  → 컨테이너에 /data 마운트

docker stop ...
  → POST /VolumeDriver.Unmount {"Name": "vol1", "ID": "container-id"}

docker volume rm vol1
  → POST /VolumeDriver.Remove {"Name": "vol1"}
```

---

### 3. Network Plugin

#### Network Plugin API

```
Network Driver Interface:

┌──────────────────────────────────────────────────┐
│ Network Plugin API                               │
├──────────────────────────────────────────────────┤
│ POST /Plugin.Activate                            │
│   → {"Implements": ["NetworkDriver"]}            │
├──────────────────────────────────────────────────┤
│ POST /NetworkDriver.GetCapabilities              │
│   Response: {"Scope": "local"}                   │
│   (local: 단일 호스트, global: 멀티 호스트)            │
├──────────────────────────────────────────────────┤
│ POST /NetworkDriver.CreateNetwork                │
│   Request: {"NetworkID": "...", "Options": {...}}│
│   Response: {"Err": ""}                          │
├──────────────────────────────────────────────────┤
│ POST /NetworkDriver.DeleteNetwork                │
│   Request: {"NetworkID": "..."}                  │
│   Response: {"Err": ""}                          │
├──────────────────────────────────────────────────┤
│ POST /NetworkDriver.CreateEndpoint               │
│   Request: {"NetworkID": "...", "EndpointID": "..."}│
│   Response: {"Interface": {...}, "Err": ""}      │
├──────────────────────────────────────────────────┤
│ POST /NetworkDriver.DeleteEndpoint               │
│   Request: {"NetworkID": "...", "EndpointID": "..."}│
│   Response: {"Err": ""}                          │
├──────────────────────────────────────────────────┤
│ POST /NetworkDriver.Join                         │
│   Request: {"NetworkID": "...", "EndpointID": "...",│
│             "SandboxKey": "..."}                 │
│   Response: {"InterfaceName": {...}, "Err": ""}  │
├──────────────────────────────────────────────────┤
│ POST /NetworkDriver.Leave                        │
│   Request: {"NetworkID": "...", "EndpointID": "..."}│
│   Response: {"Err": ""}                          │
└──────────────────────────────────────────────────┘

Network 사용 흐름:
docker network create --driver=myplugin mynet
  → POST /NetworkDriver.CreateNetwork

docker run --network=mynet myapp
  → POST /NetworkDriver.CreateEndpoint
  → POST /NetworkDriver.Join (네트워크 연결)

docker stop ...
  → POST /NetworkDriver.Leave
  → POST /NetworkDriver.DeleteEndpoint

docker network rm mynet
  → POST /NetworkDriver.DeleteNetwork
```

---

### 4. Authorization Plugin

#### Authorization Plugin API

```
Authorization Plugin Interface:

┌──────────────────────────────────────────────────┐
│ Authorization Plugin API                         │
├──────────────────────────────────────────────────┤
│ POST /Plugin.Activate                            │
│   → {"Implements": ["authz"]}                    │
├──────────────────────────────────────────────────┤
│ POST /AuthZPlugin.AuthZReq                       │
│   (요청 전 검증)                                    │
│   Request: {                                     │
│     "User": "alice",                             │
│     "RequestMethod": "POST",                     │
│     "RequestURI": "/containers/create",          │
│     "RequestBody": {...},                        │
│     "RequestHeaders": {...}                      │
│   }                                              │
│   Response: {                                    │
│     "Allow": true,                               │
│     "Msg": "Allowed",                            │
│     "Err": ""                                    │
│   }                                              │
├──────────────────────────────────────────────────┤
│ POST /AuthZPlugin.AuthZRes                       │
│   (응답 후 검증)                                  │
│   Request: {                                     │
│     "User": "alice",                             │
│     "RequestMethod": "POST",                     │
│     "RequestURI": "/containers/create",          │
│     "ResponseStatusCode": 201,                   │
│     "ResponseBody": {...},                       │
│     "ResponseHeaders": {...}                     │
│   }                                              │
│   Response: {                                    │
│     "Allow": true,                               │
│     "Msg": "Allowed",                            │
│     "Err": ""                                    │
│   }                                              │
└──────────────────────────────────────────────────┘

AuthZ 흐름:
1. 사용자가 Docker 명령 실행
   docker run nginx

2. Docker → AuthZ Plugin (요청 전)
   POST /AuthZPlugin.AuthZReq
   {"User": "alice", "RequestURI": "/containers/create"}

3. Plugin이 정책 확인
   - RBAC 규칙 체크
   - LDAP 그룹 확인
   - 시간/IP 제한 확인

4. Allow/Deny 응답
   {"Allow": true/false}

5. Docker 실행 또는 거부

6. Docker → AuthZ Plugin (응답 후)
   POST /AuthZPlugin.AuthZRes
   (민감한 정보 필터링 등)
```

---

## 🔧 실습 1: 간단한 Volume Plugin 구현 (Python)

### Step 1: Volume Plugin HTTP Server

```python
# simple_volume_plugin.py
from flask import Flask, request, jsonify
import os
import json

app = Flask(__name__)

# 볼륨 저장 디렉토리
VOLUME_BASE = "/tmp/myplugin/volumes"
os.makedirs(VOLUME_BASE, exist_ok=True)

# 볼륨 메타데이터 저장
volumes = {}

@app.route('/Plugin.Activate', methods=['POST'])
def activate():
    """Plugin 활성화"""
    return jsonify({
        "Implements": ["VolumeDriver"]
    })

@app.route('/VolumeDriver.Create', methods=['POST'])
def create():
    """볼륨 생성"""
    data = request.get_json()
    name = data.get('Name')
    opts = data.get('Opts', {})
    
    print(f"Creating volume: {name}")
    
    # 볼륨 디렉토리 생성
    volume_path = os.path.join(VOLUME_BASE, name)
    os.makedirs(volume_path, exist_ok=True)
    
    # 메타데이터 저장
    volumes[name] = {
        "Name": name,
        "Mountpoint": volume_path,
        "Options": opts,
        "RefCount": 0
    }
    
    return jsonify({"Err": ""})

@app.route('/VolumeDriver.Remove', methods=['POST'])
def remove():
    """볼륨 삭제"""
    data = request.get_json()
    name = data.get('Name')
    
    print(f"Removing volume: {name}")
    
    if name in volumes:
        # RefCount 확인 (사용 중인지)
        if volumes[name]['RefCount'] > 0:
            return jsonify({"Err": "Volume is in use"})
        
        # 디렉토리 삭제
        volume_path = volumes[name]['Mountpoint']
        try:
            os.rmdir(volume_path)
        except:
            pass
        
        del volumes[name]
    
    return jsonify({"Err": ""})

@app.route('/VolumeDriver.Mount', methods=['POST'])
def mount():
    """볼륨 마운트"""
    data = request.get_json()
    name = data.get('Name')
    id = data.get('ID')
    
    print(f"Mounting volume: {name} for container {id[:12]}")
    
    if name not in volumes:
        return jsonify({"Err": "Volume not found"})
    
    # RefCount 증가
    volumes[name]['RefCount'] += 1
    
    return jsonify({
        "Mountpoint": volumes[name]['Mountpoint'],
        "Err": ""
    })

@app.route('/VolumeDriver.Unmount', methods=['POST'])
def unmount():
    """볼륨 언마운트"""
    data = request.get_json()
    name = data.get('Name')
    id = data.get('ID')
    
    print(f"Unmounting volume: {name} from container {id[:12]}")
    
    if name in volumes:
        volumes[name]['RefCount'] -= 1
    
    return jsonify({"Err": ""})

@app.route('/VolumeDriver.Get', methods=['POST'])
def get():
    """볼륨 정보 조회"""
    data = request.get_json()
    name = data.get('Name')
    
    if name not in volumes:
        return jsonify({"Err": "Volume not found"})
    
    return jsonify({
        "Volume": {
            "Name": volumes[name]['Name'],
            "Mountpoint": volumes[name]['Mountpoint'],
            "Status": {}
        },
        "Err": ""
    })

@app.route('/VolumeDriver.List', methods=['POST'])
def list_volumes():
    """볼륨 목록"""
    volume_list = [
        {
            "Name": v['Name'],
            "Mountpoint": v['Mountpoint']
        }
        for v in volumes.values()
    ]
    
    return jsonify({
        "Volumes": volume_list,
        "Err": ""
    })

@app.route('/VolumeDriver.Path', methods=['POST'])
def path():
    """볼륨 경로"""
    data = request.get_json()
    name = data.get('Name')
    
    if name not in volumes:
        return jsonify({"Err": "Volume not found"})
    
    return jsonify({
        "Mountpoint": volumes[name]['Mountpoint'],
        "Err": ""
    })

@app.route('/VolumeDriver.Capabilities', methods=['POST'])
def capabilities():
    """Plugin 기능"""
    return jsonify({
        "Capabilities": {
            "Scope": "local"
        }
    })

if __name__ == '__main__':
    # Unix Socket에서 Listen
    socket_path = "/run/docker/plugins/myplugin.sock"
    
    # 기존 소켓 제거
    if os.path.exists(socket_path):
        os.remove(socket_path)
    
    # 디렉토리 생성
    os.makedirs(os.path.dirname(socket_path), exist_ok=True)
    
    # Unix Socket으로 실행
    from werkzeug.serving import run_simple
    import socket as py_socket
    
    sock = py_socket.socket(py_socket.AF_UNIX, py_socket.SOCK_STREAM)
    sock.bind(socket_path)
    os.chmod(socket_path, 0o660)
    
    print(f"Volume plugin listening on {socket_path}")
    
    run_simple('unix://' + socket_path, 0, app, threaded=True)
```

### Step 2: Plugin 실행 및 테스트

```bash
# 1. Plugin 실행 (백그라운드)
sudo python3 simple_volume_plugin.py &

# 2. Socket 확인
ls -la /run/docker/plugins/
# srw-rw---- 1 root docker ... myplugin.sock

# 3. 볼륨 생성
docker volume create --driver=myplugin myvol

# 4. 볼륨 확인
docker volume ls | grep myvol
# myplugin  myvol

docker volume inspect myvol

# 5. 볼륨 사용
docker run -it --rm -v myvol:/data alpine sh

# 컨테이너 내부에서:
# / # echo "Hello from custom volume" > /data/test.txt
# / # cat /data/test.txt
# Hello from custom volume
# / # exit

# 6. 데이터 확인 (호스트에서)
sudo cat /tmp/myplugin/volumes/myvol/test.txt
# Hello from custom volume

# 7. 정리
docker volume rm myvol

# Plugin 종료
sudo pkill -f simple_volume_plugin
```

---

## 🔧 실습 2: Authorization Plugin (Go)

### Step 1: AuthZ Plugin 구현

```go
// authz_plugin.go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net"
    "net/http"
    "os"
    "strings"
)

type ActivateResponse struct {
    Implements []string `json:"Implements"`
}

type AuthZRequest struct {
    User              string                 `json:"User"`
    UserAuthNMethod   string                 `json:"UserAuthNMethod"`
    RequestMethod     string                 `json:"RequestMethod"`
    RequestURI        string                 `json:"RequestURI"`
    RequestBody       interface{}            `json:"RequestBody"`
    RequestHeaders    map[string]string      `json:"RequestHeaders"`
}

type AuthZResponse struct {
    Allow bool   `json:"Allow"`
    Msg   string `json:"Msg"`
    Err   string `json:"Err"`
}

// 정책: alice는 컨테이너 생성만, bob은 모든 작업 가능
func checkPolicy(user, method, uri string) (bool, string) {
    log.Printf("Checking policy: user=%s, method=%s, uri=%s", user, method, uri)
    
    // bob은 모든 작업 허용
    if user == "bob" {
        return true, "Admin user"
    }
    
    // alice는 컨테이너 조회 및 생성만
    if user == "alice" {
        if method == "GET" {
            return true, "Read access allowed"
        }
        if method == "POST" && strings.Contains(uri, "/containers/create") {
            return true, "Create container allowed"
        }
        return false, "User alice can only read and create containers"
    }
    
    // 기본: 거부
    return false, "User not authorized"
}

func activate(w http.ResponseWriter, r *http.Request) {
    resp := ActivateResponse{
        Implements: []string{"authz"},
    }
    json.NewEncoder(w).Encode(resp)
}

func authZReq(w http.ResponseWriter, r *http.Request) {
    var req AuthZRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // 정책 확인
    allow, msg := checkPolicy(req.User, req.RequestMethod, req.RequestURI)
    
    resp := AuthZResponse{
        Allow: allow,
        Msg:   msg,
        Err:   "",
    }
    
    if !allow {
        log.Printf("❌ DENIED: %s", msg)
    } else {
        log.Printf("✅ ALLOWED: %s", msg)
    }
    
    json.NewEncoder(w).Encode(resp)
}

func authZRes(w http.ResponseWriter, r *http.Request) {
    // 응답 검증 (필요 시 민감 정보 필터링)
    resp := AuthZResponse{
        Allow: true,
        Msg:   "Response allowed",
        Err:   "",
    }
    json.NewEncoder(w).Encode(resp)
}

func main() {
    socketPath := "/run/docker/plugins/authz.sock"
    
    // 기존 소켓 제거
    os.Remove(socketPath)
    
    // HTTP 핸들러
    http.HandleFunc("/Plugin.Activate", activate)
    http.HandleFunc("/AuthZPlugin.AuthZReq", authZReq)
    http.HandleFunc("/AuthZPlugin.AuthZRes", authZRes)
    
    // Unix Socket Listener
    listener, err := net.Listen("unix", socketPath)
    if err != nil {
        log.Fatal(err)
    }
    defer listener.Close()
    
    // 권한 설정
    os.Chmod(socketPath, 0660)
    
    log.Printf("AuthZ plugin listening on %s", socketPath)
    log.Fatal(http.Serve(listener, nil))
}
```

### Step 2: Plugin 빌드 및 테스트

```bash
# 1. 빌드
go build -o authz_plugin authz_plugin.go

# 2. 실행
sudo ./authz_plugin &

# 3. dockerd 재시작 (Authorization Plugin 활성화)
# /etc/docker/daemon.json 수정:
{
  "authorization-plugins": ["authz"]
}

sudo systemctl restart docker

# 4. 테스트
# alice로 실행 (제한됨)
docker --user alice ps  # ✅ 허용 (GET)
docker --user alice run nginx  # ✅ 허용 (POST /containers/create)
docker --user alice rm container  # ❌ 거부 (DELETE)

# bob으로 실행 (모든 권한)
docker --user bob rm container  # ✅ 허용

# Plugin 로그 확인
# ✅ ALLOWED: Read access allowed
# ✅ ALLOWED: Create container allowed
# ❌ DENIED: User alice can only read and create containers
```

---

## 🔧 실습 3: Plugin 패키징 및 배포

### Step 1: Plugin 메타데이터 작성

```json
// config.json
{
  "description": "Simple Volume Plugin",
  "documentation": "https://example.com/docs",
  "entrypoint": ["/usr/bin/myplugin"],
  "env": [
    {
      "name": "DEBUG",
      "settable": ["value"],
      "value": "0"
    }
  ],
  "interface": {
    "types": ["docker.volumedriver/1.0"],
    "socket": "myplugin.sock"
  },
  "linux": {
    "capabilities": ["CAP_SYS_ADMIN"],
    "devices": null
  },
  "mounts": [
    {
      "source": "/var/lib/myplugin",
      "destination": "/data",
      "type": "bind",
      "options": ["rbind"]
    }
  ],
  "network": {
    "type": "host"
  },
  "propagatedMount": "/data"
}
```

### Step 2: rootfs 준비

```bash
# 1. Plugin 디렉토리 생성
mkdir -p myplugin-rootfs/{usr/bin,var/lib/myplugin}

# 2. 실행 파일 복사
cp simple_volume_plugin.py myplugin-rootfs/usr/bin/myplugin
chmod +x myplugin-rootfs/usr/bin/myplugin

# 3. 의존성 포함 (필요 시)
# pip install -t myplugin-rootfs/usr/lib/python3/site-packages flask

# 4. config.json 복사
cp config.json myplugin-rootfs/
```

### Step 3: Plugin 생성 및 활성화

```bash
# 1. Plugin 생성
docker plugin create myplugin:v1.0 myplugin-rootfs/

# 2. Plugin 목록 확인
docker plugin ls

# 3. Plugin 활성화
docker plugin enable myplugin:v1.0

# 4. Plugin 사용
docker volume create --driver=myplugin:v1.0 testvol

# 5. Plugin 비활성화/제거
docker plugin disable myplugin:v1.0
docker plugin rm myplugin:v1.0

# 6. Plugin 업그레이드
docker plugin upgrade myplugin:v1.0 myplugin:v2.0
```

---

## 🔧 실습 4: Network Plugin (간단한 예)

### Step 1: 기본 Network Plugin

```python
# simple_network_plugin.py
from flask import Flask, request, jsonify
import os
import subprocess

app = Flask(__name__)

networks = {}
endpoints = {}

@app.route('/Plugin.Activate', methods=['POST'])
def activate():
    return jsonify({
        "Implements": ["NetworkDriver"]
    })

@app.route('/NetworkDriver.GetCapabilities', methods=['POST'])
def capabilities():
    return jsonify({
        "Scope": "local"
    })

@app.route('/NetworkDriver.CreateNetwork', methods=['POST'])
def create_network():
    data = request.get_json()
    network_id = data.get('NetworkID')
    
    print(f"Creating network: {network_id[:12]}")
    
    # Linux Bridge 생성
    bridge_name = f"br-{network_id[:12]}"
    subprocess.run(['ip', 'link', 'add', bridge_name, 'type', 'bridge'])
    subprocess.run(['ip', 'link', 'set', bridge_name, 'up'])
    
    networks[network_id] = {
        "ID": network_id,
        "Bridge": bridge_name
    }
    
    return jsonify({"Err": ""})

@app.route('/NetworkDriver.DeleteNetwork', methods=['POST'])
def delete_network():
    data = request.get_json()
    network_id = data.get('NetworkID')
    
    print(f"Deleting network: {network_id[:12]}")
    
    if network_id in networks:
        bridge_name = networks[network_id]['Bridge']
        subprocess.run(['ip', 'link', 'set', bridge_name, 'down'])
        subprocess.run(['ip', 'link', 'del', bridge_name])
        del networks[network_id]
    
    return jsonify({"Err": ""})

@app.route('/NetworkDriver.CreateEndpoint', methods=['POST'])
def create_endpoint():
    data = request.get_json()
    network_id = data.get('NetworkID')
    endpoint_id = data.get('EndpointID')
    
    print(f"Creating endpoint: {endpoint_id[:12]} on {network_id[:12]}")
    
    # veth pair 생성
    veth_host = f"veth{endpoint_id[:7]}"
    veth_container = f"veth{endpoint_id[7:14]}"
    
    subprocess.run(['ip', 'link', 'add', veth_host, 'type', 'veth', 
                    'peer', 'name', veth_container])
    
    # Host측 veth를 bridge에 연결
    bridge_name = networks[network_id]['Bridge']
    subprocess.run(['ip', 'link', 'set', veth_host, 'master', bridge_name])
    subprocess.run(['ip', 'link', 'set', veth_host, 'up'])
    
    endpoints[endpoint_id] = {
        "NetworkID": network_id,
        "VethHost": veth_host,
        "VethContainer": veth_container
    }
    
    return jsonify({
        "Interface": {
            "MacAddress": "",
            "Address": "",
            "AddressIPv6": ""
        },
        "Err": ""
    })

@app.route('/NetworkDriver.DeleteEndpoint', methods=['POST'])
def delete_endpoint():
    data = request.get_json()
    endpoint_id = data.get('EndpointID')
    
    if endpoint_id in endpoints:
        veth_host = endpoints[endpoint_id]['VethHost']
        subprocess.run(['ip', 'link', 'del', veth_host])
        del endpoints[endpoint_id]
    
    return jsonify({"Err": ""})

@app.route('/NetworkDriver.Join', methods=['POST'])
def join():
    data = request.get_json()
    endpoint_id = data.get('EndpointID')
    
    veth_container = endpoints[endpoint_id]['VethContainer']
    
    return jsonify({
        "InterfaceName": {
            "SrcName": veth_container,
            "DstPrefix": "eth"
        },
        "Gateway": "",
        "GatewayIPv6": "",
        "Err": ""
    })

@app.route('/NetworkDriver.Leave', methods=['POST'])
def leave():
    return jsonify({"Err": ""})

if __name__ == '__main__':
    socket_path = "/run/docker/plugins/mynet.sock"
    
    if os.path.exists(socket_path):
        os.remove(socket_path)
    
    os.makedirs(os.path.dirname(socket_path), exist_ok=True)
    
    from werkzeug.serving import run_simple
    import socket as py_socket
    
    sock = py_socket.socket(py_socket.AF_UNIX, py_socket.SOCK_STREAM)
    sock.bind(socket_path)
    os.chmod(socket_path, 0o660)
    
    print(f"Network plugin listening on {socket_path}")
    
    run_simple('unix://' + socket_path, 0, app, threaded=True)
```

---

## 🔧 실습 5: Logging Plugin

### Step 1: 간단한 로그 플러그인

```python
# simple_logging_plugin.py
from flask import Flask, request, jsonify
import os
import time

app = Flask(__name__)

LOG_DIR = "/tmp/myplugin/logs"
os.makedirs(LOG_DIR, exist_ok=True)

streams = {}

@app.route('/Plugin.Activate', methods=['POST'])
def activate():
    return jsonify({
        "Implements": ["LogDriver"]
    })

@app.route('/LogDriver.StartLogging', methods=['POST'])
def start_logging():
    data = request.get_json()
    file_path = data.get('File')  # Fifo path
    container_id = data.get('Info', {}).get('ContainerID', 'unknown')
    
    print(f"Starting logging for {container_id[:12]}")
    
    # 로그 파일 경로
    log_file = os.path.join(LOG_DIR, f"{container_id}.log")
    
    # Fifo에서 읽어서 로그 파일에 쓰기 (백그라운드)
    import threading
    
    def log_reader():
        with open(file_path, 'rb') as fifo:
            with open(log_file, 'ab') as log:
                while True:
                    chunk = fifo.read(1024)
                    if not chunk:
                        break
                    timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
                    log.write(f"[{timestamp}] ".encode() + chunk)
                    log.flush()
    
    thread = threading.Thread(target=log_reader, daemon=True)
    thread.start()
    
    streams[container_id] = {
        "LogFile": log_file,
        "Thread": thread
    }
    
    return jsonify({"Err": ""})

@app.route('/LogDriver.StopLogging', methods=['POST'])
def stop_logging():
    data = request.get_json()
    file_path = data.get('File')
    
    print("Stopping logging")
    
    return jsonify({"Err": ""})

@app.route('/LogDriver.Capabilities', methods=['POST'])
def capabilities():
    return jsonify({
        "Cap": {
            "ReadLogs": False
        }
    })

@app.route('/LogDriver.ReadLogs', methods=['POST'])
def read_logs():
    # 로그 읽기 구현 (선택)
    return jsonify({"Err": "Not implemented"})

if __name__ == '__main__':
    socket_path = "/run/docker/plugins/mylog.sock"
    
    if os.path.exists(socket_path):
        os.remove(socket_path)
    
    os.makedirs(os.path.dirname(socket_path), exist_ok=True)
    
    from werkzeug.serving import run_simple
    import socket as py_socket
    
    sock = py_socket.socket(py_socket.AF_UNIX, py_socket.SOCK_STREAM)
    sock.bind(socket_path)
    os.chmod(socket_path, 0o660)
    
    print(f"Logging plugin listening on {socket_path}")
    
    run_simple('unix://' + socket_path, 0, app, threaded=True)
```

---

## 💡 주요 명령어 정리

```bash
# ========== Plugin 관리 ==========
docker plugin ls                        # Plugin 목록
docker plugin install <plugin>          # Plugin 설치
docker plugin enable <plugin>           # Plugin 활성화
docker plugin disable <plugin>          # Plugin 비활성화
docker plugin rm <plugin>               # Plugin 제거
docker plugin inspect <plugin>          # Plugin 상세 정보
docker plugin upgrade <old> <new>       # Plugin 업그레이드

# ========== Plugin 개발 ==========
docker plugin create <name> <rootfs>    # Plugin 생성
docker plugin push <name>               # Plugin 배포 (레지스트리)
docker plugin set <name> <key>=<value>  # Plugin 설정 변경

# ========== Plugin 사용 ==========
# Volume Plugin
docker volume create --driver=<plugin> <name>

# Network Plugin
docker network create --driver=<plugin> <name>

# Logging Plugin
docker run --log-driver=<plugin> <image>

# Authorization Plugin
# /etc/docker/daemon.json에 설정
{
  "authorization-plugins": ["<plugin>"]
}
```

---

## 🎓 연습 문제

### 문제 1: Volume Plugin의 Mount/Unmount가 여러 번 호출되는 이유는?

<details>
<summary>정답 보기</summary>

Volume Plugin의 Mount는 **컨테이너마다** 호출됩니다.

```
시나리오: 1개 볼륨을 2개 컨테이너가 사용

docker volume create --driver=myplugin shared-vol
  → POST /VolumeDriver.Create {"Name": "shared-vol"}

docker run -v shared-vol:/data container1
  → POST /VolumeDriver.Mount {"Name": "shared-vol", "ID": "container1-id"}
  → RefCount = 1

docker run -v shared-vol:/data container2
  → POST /VolumeDriver.Mount {"Name": "shared-vol", "ID": "container2-id"}
  → RefCount = 2

docker stop container1
  → POST /VolumeDriver.Unmount {"Name": "shared-vol", "ID": "container1-id"}
  → RefCount = 1

docker stop container2
  → POST /VolumeDriver.Unmount {"Name": "shared-vol", "ID": "container2-id"}
  → RefCount = 0
```

**RefCount 관리가 중요한 이유:**
- RefCount > 0: 볼륨 삭제 불가 (사용 중)
- RefCount = 0: 볼륨 삭제 가능

**구현 예:**
```python
volumes = {
    "shared-vol": {
        "Name": "shared-vol",
        "Mountpoint": "/mnt/volumes/shared-vol",
        "RefCount": 0,  # 현재 사용 중인 컨테이너 수
        "Mounts": []    # 마운트된 컨테이너 ID 목록
    }
}

def mount(name, container_id):
    volumes[name]['RefCount'] += 1
    volumes[name]['Mounts'].append(container_id)

def unmount(name, container_id):
    volumes[name]['RefCount'] -= 1
    volumes[name]['Mounts'].remove(container_id)

def remove(name):
    if volumes[name]['RefCount'] > 0:
        return {"Err": "Volume is in use"}
    # 실제 삭제
```

</details>

### 문제 2: Authorization Plugin의 AuthZReq와 AuthZRes의 차이는?

<details>
<summary>정답 보기</summary>

**AuthZReq (요청 전 검증):**
- Docker 명령 실행 **전**에 호출
- 사용자가 해당 작업을 **수행할 권한**이 있는지 확인
- Allow=false → Docker 명령 차단

**AuthZRes (응답 후 검증):**
- Docker 명령 실행 **후**에 호출
- 응답에 **민감한 정보**가 있는지 확인하고 필터링
- Allow=false → 응답 차단

**실제 예:**

```go
// AuthZReq: 요청 전 검증
func authZReq(req AuthZRequest) AuthZResponse {
    // 예: alice는 privileged 컨테이너 생성 불가
    if req.User == "alice" && 
       req.RequestURI == "/containers/create" {
        
        // RequestBody 파싱
        if body, ok := req.RequestBody.(map[string]interface{}); ok {
            if hostConfig, ok := body["HostConfig"].(map[string]interface{}); ok {
                if privileged, ok := hostConfig["Privileged"].(bool); ok && privileged {
                    return AuthZResponse{
                        Allow: false,
                        Msg: "User alice cannot create privileged containers"
                    }
                }
            }
        }
    }
    
    return AuthZResponse{Allow: true}
}

// AuthZRes: 응답 후 검증
func authZRes(req AuthZRequest) AuthZResponse {
    // 예: 환경 변수에서 민감한 정보 필터링
    if req.RequestURI == "/containers/{id}/json" {
        // ResponseBody에서 Env 필터링
        // PASSWORD, TOKEN 등 제거
    }
    
    return AuthZResponse{Allow: true}
}
```

**사용 시나리오:**
1. **AuthZReq**: RBAC, 권한 제어, 시간/IP 제한
2. **AuthZRes**: 민감 정보 마스킹, 데이터 유출 방지

</details>

### 문제 3: Plugin을 Managed Plugin vs Legacy Plugin 중 어떤 방식으로 배포해야 하는가?

<details>
<summary>정답 보기</summary>

**Managed Plugin (권장):**
```bash
# 장점
✅ docker plugin 명령어로 관리
✅ 레지스트리에 Push/Pull 가능
✅ 버전 관리 용이
✅ 의존성 포함 (rootfs)
✅ 자동 업데이트

# 단점
❌ 패키징 과정 필요
❌ 디버깅 어려움

# 사용
docker plugin install myregistry.com/myplugin:v1.0
docker plugin enable myplugin:v1.0
```

**Legacy Plugin (개발/테스트):**
```bash
# 장점
✅ 간단한 배포 (Socket만)
✅ 쉬운 디버깅
✅ 빠른 개발 사이클

# 단점
❌ 수동 관리 필요
❌ 레지스트리 배포 불가
❌ 의존성 별도 설치

# 사용
# 1. Socket 생성
/run/docker/plugins/myplugin.sock

# 2. 또는 .spec 파일
/etc/docker/plugins/myplugin.spec
{
  "Addr": "unix:///var/run/myplugin.sock"
}
```

**선택 기준:**
```
개발 단계: Legacy Plugin
- 빠른 수정/테스트
- 로그 직접 확인

프로덕션: Managed Plugin
- 안정적 배포
- 버전 관리
- 자동 업데이트
```

**마이그레이션:**
```bash
# Legacy → Managed 전환

# 1. rootfs 준비
mkdir -p plugin-rootfs/usr/bin
cp myplugin.py plugin-rootfs/usr/bin/myplugin
chmod +x plugin-rootfs/usr/bin/myplugin

# 2. config.json 작성
cat > plugin-rootfs/config.json << EOF
{
  "description": "My Plugin",
  "interface": {
    "types": ["docker.volumedriver/1.0"],
    "socket": "myplugin.sock"
  },
  "entrypoint": ["/usr/bin/myplugin"]
}
EOF

# 3. Plugin 생성
docker plugin create myplugin:v1.0 plugin-rootfs/

# 4. Push (레지스트리)
docker plugin push myplugin:v1.0

# 5. 다른 호스트에서 설치
docker plugin install myplugin:v1.0
```

</details>

---

## 📌 핵심 요약

```
┌──────────────────┬────────────────────────────────────┐
│ Plugin 타입       │ API                                │
├──────────────────┼────────────────────────────────────┤
│ Volume           │ Create, Mount, Unmount, Remove     │
│                  │ Get, List, Path, Capabilities      │
├──────────────────┼────────────────────────────────────┤
│ Network          │ CreateNetwork, DeleteNetwork       │
│                  │ CreateEndpoint, Join, Leave        │
├──────────────────┼────────────────────────────────────┤
│ Authorization    │ AuthZReq (요청 전)                   │
│                  │ AuthZRes (응답 후)                   │
├──────────────────┼────────────────────────────────────┤
│ Logging          │ StartLogging, StopLogging          │
│                  │ ReadLogs (선택)                     │
└──────────────────┴────────────────────────────────────┘

Plugin 통신: Unix Socket + JSON-RPC
배포: Managed Plugin (docker plugin) vs Legacy (Socket)
```

---

## 📚 참고 자료

- [Docker Plugin API](https://docs.docker.com/engine/extend/plugin_api/)
- [Volume Plugin API](https://docs.docker.com/engine/extend/plugins_volume/)
- [Network Plugin API](https://docs.docker.com/engine/extend/plugins_network/)
- [Authorization Plugin](https://docs.docker.com/engine/extend/plugins_authorization/)
- [Managed Plugins](https://docs.docker.com/engine/extend/)

---

## 🤔 생각해볼 문제

1. Volume Plugin을 개발하는 대신 Docker의 기본 local driver를 사용하는 것과 비교했을 때의 장단점은?
2. Authorization Plugin이 모든 API 요청을 검증하면 성능에 영향을 주지 않는가? 어떻게 최적화할 수 있는가?
3. Network Plugin을 직접 개발하는 대신 기존 CNI 플러그인을 사용하는 것이 나은 경우는?

> 💡 **답변**:
>
> **1) Volume Plugin vs Local Driver:**
>
> **Local Driver (기본):**
> ```bash
> docker volume create myvolume
> # /var/lib/docker/volumes/myvolume/_data
> 
> 장점:
> ✅ 간단함 (설정 불필요)
> ✅ 빠름 (로컬 디스크)
> ✅ 안정적 (Docker 내장)
> 
> 단점:
> ❌ 단일 호스트에만 사용
> ❌ 백업/복제 수동
> ❌ 성능 제어 제한적
> ❌ 외부 스토리지 연동 불가
> ```
>
> **Custom Volume Plugin:**
> ```bash
> docker volume create --driver=nfs \
>   --opt device=:/share \
>   --opt addr=192.168.1.100 \
>   nfs-volume
> 
> 장점:
> ✅ 외부 스토리지 (NFS, S3, Ceph)
> ✅ 멀티 호스트 공유
> ✅ 자동 백업/스냅샷
> ✅ 엔터프라이즈 기능 (암호화, 압축)
> ✅ 클라우드 통합 (AWS EBS, Azure Disk)
> 
> 단점:
> ❌ 복잡한 구현
> ❌ 네트워크 오버헤드 (원격 스토리지)
> ❌ 추가 의존성
> ❌ 디버깅 어려움
> ```
>
> **선택 기준:**
> ```
> Local Driver 사용:
> - 단일 호스트 개발/테스트
> - 임시 데이터
> - 간단한 워크로드
> 
> Volume Plugin 개발:
> - 프로덕션 환경
> - 데이터 영속성 중요
> - 멀티 호스트 (Swarm, Kubernetes)
> - 기존 스토리지 인프라 활용
> - 규정 준수 (암호화, 감사)
> ```
>
> **실제 사례:**
> - **Netflix**: AWS EBS 볼륨 플러그인 (상태 저장 서비스)
> - **Dropbox**: 자체 분산 스토리지 플러그인
> - **일반 기업**: NFS/GlusterFS 플러그인 (기존 스토리지 활용)
>
> **2) Authorization Plugin 성능 최적화:**
>
> **성능 영향:**
> ```
> 모든 API 요청 흐름:
> 
> 사용자 → Docker API → AuthZ Plugin → Docker → AuthZ Plugin → 사용자
>          (1)          (2) AuthZReq     (3)      (4) AuthZRes    (5)
> 
> 오버헤드:
> - 요청당 2번 플러그인 호출
> - Network/Socket 통신
> - 정책 평가 시간
> 
> 측정 예:
> docker run nginx (without AuthZ): ~100ms
> docker run nginx (with AuthZ):    ~120ms (+20%)
> ```
>
> **최적화 전략:**
>
> **1. 캐싱:**
> ```python
> class AuthZPlugin:
>     def __init__(self):
>         self.cache = {}  # {(user, uri): (allow, expires)}
>     
>     def check_policy(self, user, uri):
>         cache_key = (user, uri)
>         
>         # 캐시 확인 (5분 TTL)
>         if cache_key in self.cache:
>             result, expires = self.cache[cache_key]
>             if time.time() < expires:
>                 return result
>         
>         # 정책 평가
>         result = self._evaluate_policy(user, uri)
>         
>         # 캐시 저장
>         self.cache[cache_key] = (result, time.time() + 300)
>         return result
> ```
>
> **2. 빠른 경로 (Fast Path):**
> ```python
> def authz_req(req):
>     # GET 요청은 대부분 허용 (읽기)
>     if req.method == "GET":
>         return {"Allow": True}
>     
>     # Admin 사용자는 즉시 허용
>     if req.user in ADMIN_USERS:
>         return {"Allow": True}
>     
>     # 복잡한 정책은 필요시만
>     return check_complex_policy(req)
> ```
>
> **3. 비동기 처리:**
> ```python
> async def authz_req(req):
>     # 정책 평가를 비동기로
>     result = await asyncio.gather(
>         check_rbac(req.user, req.uri),
>         check_time_restriction(req.user),
>         check_ip_whitelist(req.ip)
>     )
>     return {"Allow": all(result)}
> ```
>
> **4. AuthZRes 선택적 사용:**
> ```python
> # AuthZReq: 항상 검증
> # AuthZRes: 민감 데이터만 검증
> 
> def authz_res(req):
>     # 민감한 엔드포인트만 검증
>     if req.uri not in SENSITIVE_ENDPOINTS:
>         return {"Allow": True}
>     
>     # 응답 필터링
>     return filter_sensitive_data(req)
> ```
>
> **5. 정책 DB 최적화:**
> ```python
> # ❌ 느림: 매번 LDAP 쿼리
> def check_policy(user):
>     groups = ldap.get_user_groups(user)  # 100ms
>     return "admin" in groups
> 
> # ✅ 빠름: 로컬 캐시
> @lru_cache(maxsize=1000)
> def get_user_groups(user):
>     return ldap.get_user_groups(user)
> ```
>
> **벤치마크:**
> ```
> Without AuthZ:  100 req/s
> With AuthZ (naive): 50 req/s (-50%)
> With AuthZ (optimized): 90 req/s (-10%)
> 
> 최적화 효과:
> - 캐싱: +30%
> - Fast Path: +15%
> - 비동기: +10%
> ```
>
> **3) Custom Network Plugin vs CNI:**
>
> **CNI (Container Network Interface):**
> ```bash
> # 표준 인터페이스
> # Kubernetes, Podman 등에서 사용
> 
> 주요 CNI 플러그인:
> - Calico: L3 네트워킹, 정책
> - Weave: Overlay 네트워크
> - Cilium: eBPF 기반, 고성능
> - Flannel: 간단한 overlay
> ```
>
> **Custom Plugin 개발이 나은 경우:**
>
> **1. Docker 전용 기능:**
> ```python
> # Docker Swarm 최적화
> # Docker-specific metadata 사용
> # Docker Desktop 통합
> 
> class DockerNetworkPlugin:
>     def create_endpoint(self, req):
>         # Docker Labels 활용
>         labels = req.get('Labels', {})
>         
>         # Swarm Service 정보
>         service_name = labels.get('com.docker.swarm.service.name')
> ```
>
> **2. 기존 네트워크 인프라:**
> ```python
> # 회사 자체 SDN 시스템
> # 레거시 VLAN 관리
> # 특수 하드웨어 (F5, Cisco ACI)
> 
> def create_network(req):
>     # 기존 VLAN 시스템과 통합
>     vlan_id = allocate_vlan_from_legacy_system()
>     configure_hardware_switch(vlan_id)
> ```
>
> **3. 단순한 요구사항:**
> ```python
> # CNI는 과한 경우
> # 예: 단순 bridge만 필요
> 
> def create_network():
>     # 간단한 Linux bridge만
>     subprocess.run(['ip', 'link', 'add', 'br0', 'type', 'bridge'])
> ```
>
> **CNI 사용이 나은 경우:**
>
> **1. Kubernetes 통합:**
> ```yaml
> # Kubernetes + CNI
> # 표준 인터페이스
> 
> apiVersion: v1
> kind: Pod
> metadata:
>   annotations:
>     k8s.v1.cni.cncf.io/networks: macvlan-conf
> ```
>
> **2. 고급 네트워킹 기능:**
> ```bash
> # Calico: 네트워크 정책
> kubectl apply -f - <<EOF
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: deny-all
> spec:
>   podSelector: {}
>   policyTypes:
>   - Ingress
>   - Egress
> EOF
> ```
>
> **3. 검증된 솔루션:**
> ```
> CNI 플러그인:
> ✅ 대규모 프로덕션 검증
> ✅ 활발한 커뮤니티
> ✅ 정기 보안 패치
> ✅ 성능 최적화
> ✅ 문서화
> 
> Custom Plugin:
> ❌ 자체 유지보수
> ❌ 보안 검증 필요
> ❌ 성능 테스트 필요
> ```
>
> **결론:**
> ```
> Custom Network Plugin 개발:
> - Docker 전용 환경
> - 기존 인프라와 깊은 통합
> - 특수한 요구사항
> - 단순한 네트워킹
> 
> CNI 사용:
> - Kubernetes 사용
> - 멀티 플랫폼 (Docker + Podman + K8s)
> - 고급 네트워킹 (정책, 암호화)
> - 검증된 솔루션 선호
> ```
>
> **마이그레이션 경로:**
> 많은 조직이 Docker Network Plugin → CNI로 전환 중 (Kubernetes 표준화)

---

<div align="center">

**[⬅️ 이전: Docker SDK](./06-Docker-SDK.md)** | **[다음: Docker Extensions ➡️](./08-Docker-Extensions.md)**

</div>
