# 07. Docker Engine - 엔진의 내부 동작

## 🎯 이 챕터에서 배울 것

- Docker Engine의 **내부 구조**와 컴포넌트
- **이벤트 시스템**과 실시간 모니터링
- **플러그인 아키텍처**와 확장성
- Engine API와 **프로그래밍 방식** 제어

## 📌 왜 중요한가?

**"Docker를 완전히 이해하려면 엔진을 알아야 합니다."**

```
지금까지 배운 것들:
✅ 컨테이너 vs VM
✅ 아키텍처 (dockerd, containerd, runc)
✅ 이미지 레이어
✅ Union Filesystem
✅ Namespaces (격리)
✅ Cgroups (리소스 제한)

이제 이 모든 것을 통합하는 Docker Engine!
```

**실무 영향:**
- 자동화: API로 Docker 완전 제어
- 모니터링: 이벤트 스트림 활용
- 확장: 커스텀 플러그인 개발
- 디버깅: 내부 동작 이해로 문제 해결

---

## 🔬 Deep Dive

### 1. Docker Engine 전체 구조

#### 아키텍처 리뷰

```
┌─────────────────────────────────────────────┐
│         Docker Engine (dockerd)             │
│                                             │
│  ┌────────────────────────────────────┐   │
│  │        API Server                  │   │
│  │  - REST API 엔드포인트             │   │
│  │  - gRPC 서버                       │   │
│  │  - 인증/권한                        │   │
│  └────────────────────────────────────┘   │
│                                             │
│  ┌────────────────────────────────────┐   │
│  │     Image Manager                  │   │
│  │  - 이미지 빌드 (BuildKit)          │   │
│  │  - 이미지 저장소                   │   │
│  │  - 레이어 캐시                     │   │
│  │  - Registry 통신                   │   │
│  └────────────────────────────────────┘   │
│                                             │
│  ┌────────────────────────────────────┐   │
│  │    Container Manager               │   │
│  │  - 컨테이너 생명주기               │   │
│  │  - 상태 추적                       │   │
│  │  - Health Check                    │   │
│  └────────────────────────────────────┘   │
│                                             │
│  ┌────────────────────────────────────┐   │
│  │    Network Manager                 │   │
│  │  - 네트워크 생성/삭제               │   │
│  │  - IP 할당 (IPAM)                  │   │
│  │  - 드라이버 관리                   │   │
│  └────────────────────────────────────┘   │
│                                             │
│  ┌────────────────────────────────────┐   │
│  │    Volume Manager                  │   │
│  │  - 볼륨 생성/마운트                 │   │
│  │  - 드라이버 관리                   │   │
│  └────────────────────────────────────┘   │
│                                             │
│  ┌────────────────────────────────────┐   │
│  │    Event System                    │   │
│  │  - 이벤트 생성/전파                 │   │
│  │  - 이벤트 구독                      │   │
│  └────────────────────────────────────┘   │
│                                             │
│  ┌────────────────────────────────────┐   │
│  │    Plugin System                   │   │
│  │  - 플러그인 로드/관리               │   │
│  │  - 확장 포인트                      │   │
│  └────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
         ↓ gRPC
┌─────────────────────────────────────────────┐
│              containerd                     │
└─────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────┐
│               runc / OCI                    │
└─────────────────────────────────────────────┘
```

---

### 2. Image Manager

#### 이미지 빌드 (BuildKit)

```
BuildKit = 차세대 빌드 엔진

기존 빌드와의 차이:
├─ 병렬 빌드 단계 실행
├─ 중간 결과 캐싱
├─ 분산 캐시 지원
└─ Secrets 안전 처리

활성화:
export DOCKER_BUILDKIT=1
docker build -t myimage .
```

#### 이미지 저장 구조

```
/var/lib/docker/
├─ image/
│  └─ overlay2/
│     ├─ imagedb/          ← 이미지 메타데이터
│     │  └─ content/
│     │     └─ sha256/
│     │        └─ abc123... (이미지 JSON)
│     ├─ layerdb/          ← 레이어 정보
│     │  └─ sha256/
│     │     └─ def456... (레이어 메타)
│     └─ repositories.json  ← 태그 매핑
└─ overlay2/               ← 실제 레이어 데이터
   ├─ abc123.../
   └─ def456.../
```

#### 레이어 관리

```
레이어 캐시 메커니즘:

1. Content-Addressable Storage
   ├─ 레이어 ID = SHA256(내용)
   ├─ 같은 내용 = 같은 ID
   └─ 자동 중복 제거

2. 레이어 공유
   ├─ 여러 이미지가 레이어 공유
   ├─ 디스크 공간 절약
   └─ Pull 시간 단축

3. Garbage Collection
   ├─ 사용하지 않는 레이어 정리
   └─ docker system prune
```

---

### 3. Container Manager

#### 생명주기 관리

```
컨테이너 상태 전이:

Created → Running → Paused → Running → Stopped → Removed
  ↓         ↓                    ↓         ↓
Start     Pause                Unpause   Start
          Stop                          Remove

각 상태별 작업:
├─ Created: OCI 번들 준비
├─ Running: runc 실행, PID 추적
├─ Paused: SIGSTOP (cgroup freezer)
├─ Stopped: SIGTERM → SIGKILL
└─ Removed: 레이어 정리
```

#### Health Check

```
Health Check = 컨테이너 건강 상태 모니터링

설정:
├─ HEALTHCHECK --interval=30s --timeout=3s CMD curl http://localhost

상태:
├─ starting: 초기 상태
├─ healthy: 체크 성공
└─ unhealthy: 연속 실패

동작:
1. interval마다 CMD 실행
2. timeout 내에 완료 안 되면 실패
3. retries 횟수만큼 연속 실패 → unhealthy
4. unhealthy → 재시작 (--restart=on-unhealthy)
```

#### Restart Policy

```
재시작 정책:

no (기본):
└─ 재시작 안 함

always:
└─ 항상 재시작 (수동 중지 제외)

unless-stopped:
└─ 수동 중지 전까지 재시작

on-failure[:max-retries]:
└─ Exit code ≠ 0일 때만 재시작
```

---

### 4. Event System

#### 이벤트 종류

```
Docker는 모든 작업을 이벤트로 발행:

Container Events:
├─ create, start, stop, die, destroy
├─ pause, unpause, restart
├─ kill, oom, attach, detach
└─ exec_create, exec_start, exec_die

Image Events:
├─ pull, push, tag, untag
├─ delete, import, load, save
└─ build (BuildKit)

Network Events:
├─ create, connect, disconnect
└─ destroy, remove

Volume Events:
├─ create, mount, unmount
└─ destroy

Plugin Events:
├─ install, enable, disable
└─ remove
```

#### 이벤트 구조

```json
{
  "Type": "container",
  "Action": "start",
  "Actor": {
    "ID": "abc123...",
    "Attributes": {
      "image": "nginx:latest",
      "name": "my-nginx"
    }
  },
  "time": 1640000000,
  "timeNano": 1640000000000000000
}
```

#### 실시간 이벤트 스트림

```bash
# 모든 이벤트 구독
docker events

# 필터링
docker events --filter 'type=container'
docker events --filter 'event=start'
docker events --filter 'container=my-nginx'

# 시간 범위
docker events --since '2024-01-01T00:00:00'
docker events --until '2024-01-02T00:00:00'

# 포맷팅
docker events --format '{{.Time}} {{.Action}} {{.Actor.Attributes.name}}'
```

---

### 5. Plugin System

#### 플러그인 종류

```
Docker 플러그인 타입:

Volume Plugins:
├─ 커스텀 스토리지 백엔드
├─ 예: NFS, GlusterFS, Ceph
└─ /var/lib/docker/plugins

Network Plugins:
├─ 커스텀 네트워크 드라이버
├─ 예: Calico, Weave, Flannel
└─ CNI/CNM 인터페이스

Authorization Plugins:
├─ API 요청 인증/권한
└─ 정책 기반 접근 제어

Log Plugins:
├─ 커스텀 로그 드라이버
└─ 예: Fluentd, Splunk, Graylog
```

#### 플러그인 구조

```
플러그인 = HTTP 서버

Docker Engine ←(HTTP/JSON-RPC)→ Plugin

요청:
POST /Plugin.Activate
POST /VolumeDriver.Create
POST /VolumeDriver.Mount
...

응답:
{
  "Implements": ["VolumeDriver"],
  "Err": ""
}
```

---

### 6. API Server

#### REST API

```
Docker Engine API = Docker의 모든 기능 제어

기본 엔드포인트:
unix:///var/run/docker.sock (로컬)
tcp://localhost:2375 (원격, TLS 권장)

API 버전:
├─ /v1.41/containers/json
├─ /v1.42/images/json
└─ 현재 버전: docker version --format '{{.Server.APIVersion}}'
```

#### 주요 엔드포인트

```bash
# 컨테이너
GET    /containers/json            # 목록
POST   /containers/create          # 생성
POST   /containers/{id}/start      # 시작
POST   /containers/{id}/stop       # 중지
DELETE /containers/{id}            # 삭제
GET    /containers/{id}/logs       # 로그
POST   /containers/{id}/exec       # 명령어 실행

# 이미지
GET    /images/json                # 목록
POST   /images/create?fromImage=   # Pull
POST   /build                      # 빌드
DELETE /images/{id}                # 삭제

# 네트워크
GET    /networks                   # 목록
POST   /networks/create            # 생성
POST   /networks/{id}/connect      # 연결

# 볼륨
GET    /volumes                    # 목록
POST   /volumes/create             # 생성

# 시스템
GET    /info                       # 시스템 정보
GET    /version                    # 버전
GET    /events                     # 이벤트 스트림
```

---

## 💻 실습

### 실습 1: 이벤트 모니터링

```bash
# 1. 터미널 1: 이벤트 스트림
docker events --format '{{json .}}' | jq

# 2. 터미널 2: 컨테이너 작업
docker run -d --name event-test nginx
docker stop event-test
docker start event-test
docker rm -f event-test

# 터미널 1 출력:
# {"status":"create",...,"Action":"create","Type":"container"}
# {"status":"start",...}
# {"status":"stop",...}
# {"status":"start",...}
# {"status":"kill",...}
# {"status":"die",...}
# {"status":"destroy",...}

# 3. 특정 이벤트만 필터링
docker events --filter 'event=start' --filter 'event=stop'

# 4. 컨테이너별 이벤트
docker events --filter 'container=event-test'

# 5. 이벤트 시간 제한
docker events --since '5m' --until '1m'
```

### 실습 2: Engine API 직접 호출

```bash
# 1. Unix Socket으로 API 호출
# 버전 확인
curl --unix-socket /var/run/docker.sock http://localhost/version | jq

# 2. 컨테이너 목록
curl --unix-socket /var/run/docker.sock \
  http://localhost/v1.41/containers/json | jq

# 3. 컨테이너 생성
curl --unix-socket /var/run/docker.sock \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "Image": "alpine",
    "Cmd": ["sleep", "3600"],
    "HostConfig": {
      "Memory": 134217728,
      "CpuShares": 512
    }
  }' \
  http://localhost/v1.41/containers/create?name=api-test | jq

# 응답:
# {
#   "Id": "abc123...",
#   "Warnings": []
# }

# 4. 컨테이너 시작
CONTAINER_ID="abc123..."
curl --unix-socket /var/run/docker.sock \
  -X POST \
  http://localhost/v1.41/containers/$CONTAINER_ID/start

# 5. 컨테이너 로그 (스트림)
curl --unix-socket /var/run/docker.sock \
  -X GET \
  "http://localhost/v1.41/containers/$CONTAINER_ID/logs?stdout=1&follow=1"

# 6. 컨테이너 삭제
curl --unix-socket /var/run/docker.sock \
  -X DELETE \
  "http://localhost/v1.41/containers/$CONTAINER_ID?force=true"
```

### 실습 3: 플러그인 사용

```bash
# 1. 플러그인 설치 (Volume 플러그인 예시)
docker plugin install --alias local-persist \
  ashald/docker-volume-plugin-local-persist:latest

# 2. 플러그인 확인
docker plugin ls
# ID        NAME           ENABLED
# abc123    local-persist  true

# 3. 플러그인 사용
docker volume create -d local-persist \
  --opt mountpoint=/data/persistent \
  my-persistent-volume

# 4. 볼륨 사용
docker run -d -v my-persistent-volume:/data nginx

# 5. 플러그인 비활성화
docker plugin disable local-persist

# 6. 플러그인 제거
docker plugin rm local-persist
```

### 실습 4: Health Check 설정

```bash
# 1. Dockerfile에 Health Check
cat > Dockerfile <<'EOF'
FROM nginx:alpine

HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1
EOF

docker build -t nginx-health .

# 2. 컨테이너 실행
docker run -d --name health-test nginx-health

# 3. Health 상태 확인
docker ps
# STATUS: Up 10 seconds (health: starting)

# 잠시 후:
docker ps
# STATUS: Up 30 seconds (healthy)

# 4. Health 로그 확인
docker inspect health-test | jq '.[0].State.Health'
# {
#   "Status": "healthy",
#   "FailingStreak": 0,
#   "Log": [
#     {
#       "Start": "2024-01-01T12:00:00Z",
#       "End": "2024-01-01T12:00:01Z",
#       "ExitCode": 0,
#       "Output": ""
#     }
#   ]
# }

# 5. nginx 중지 (unhealthy 유발)
docker exec health-test nginx -s stop

# 잠시 후:
docker ps
# STATUS: Up 1 minute (unhealthy)

# 정리
docker rm -f health-test
```

---

## 🔥 실전 적용

### 시나리오 1: 이벤트 기반 자동화

```python
#!/usr/bin/env python3
import docker

client = docker.from_env()

# 이벤트 구독
for event in client.events(decode=True):
    if event['Type'] == 'container':
        action = event['Action']
        name = event['Actor']['Attributes'].get('name', 'unknown')
        
        if action == 'die':
            print(f"Container {name} died! Checking exit code...")
            container = client.containers.get(event['Actor']['ID'])
            exit_code = container.attrs['State']['ExitCode']
            
            if exit_code != 0:
                print(f"Unhealthy exit! Restarting {name}...")
                container.start()
        
        elif action == 'oom':
            print(f"OOM! Container {name} ran out of memory!")
            # 알림 전송, 로그 수집 등
```

### 시나리오 2: API를 통한 동적 스케일링

```python
#!/usr/bin/env python3
import docker
import time

client = docker.from_env()

def get_cpu_usage(container):
    stats = container.stats(stream=False)
    cpu_delta = stats['cpu_stats']['cpu_usage']['total_usage'] - \
                stats['precpu_stats']['cpu_usage']['total_usage']
    system_delta = stats['cpu_stats']['system_cpu_usage'] - \
                   stats['precpu_stats']['system_cpu_usage']
    num_cpus = len(stats['cpu_stats']['cpu_usage']['percpu_usage'])
    cpu_percent = (cpu_delta / system_delta) * num_cpus * 100.0
    return cpu_percent

# 오토스케일링
service_name = 'web'
min_replicas = 2
max_replicas = 10
target_cpu = 70

while True:
    containers = client.containers.list(filters={'label': f'service={service_name}'})
    
    if not containers:
        print("No containers found")
        time.sleep(10)
        continue
    
    avg_cpu = sum(get_cpu_usage(c) for c in containers) / len(containers)
    print(f"Average CPU: {avg_cpu:.2f}%, Replicas: {len(containers)}")
    
    if avg_cpu > target_cpu and len(containers) < max_replicas:
        # Scale up
        print("Scaling up...")
        template = containers[0]
        config = template.attrs['Config']
        client.containers.run(
            config['Image'],
            command=config['Cmd'],
            labels={'service': service_name},
            detach=True
        )
    
    elif avg_cpu < target_cpu / 2 and len(containers) > min_replicas:
        # Scale down
        print("Scaling down...")
        containers[-1].stop()
        containers[-1].remove()
    
    time.sleep(30)
```

### 시나리오 3: 커스텀 모니터링 대시보드

```python
#!/usr/bin/env python3
import docker
from flask import Flask, jsonify
import threading
import time

app = Flask(__name__)
client = docker.from_env()
stats_cache = {}

def update_stats():
    """백그라운드에서 통계 업데이트"""
    while True:
        containers = client.containers.list()
        for container in containers:
            stats = container.stats(stream=False)
            stats_cache[container.name] = {
                'cpu': calculate_cpu_percent(stats),
                'memory': stats['memory_stats']['usage'] / 1024 / 1024,  # MB
                'network_rx': stats['networks']['eth0']['rx_bytes'] / 1024 / 1024,  # MB
                'network_tx': stats['networks']['eth0']['tx_bytes'] / 1024 / 1024,  # MB
            }
        time.sleep(5)

@app.route('/api/stats')
def get_stats():
    return jsonify(stats_cache)

@app.route('/api/containers')
def get_containers():
    containers = client.containers.list()
    return jsonify([{
        'name': c.name,
        'status': c.status,
        'image': c.image.tags[0] if c.image.tags else 'none',
    } for c in containers])

# 백그라운드 스레드 시작
threading.Thread(target=update_stats, daemon=True).start()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### 시나리오 4: 재해 복구 자동화

```python
#!/usr/bin/env python3
import docker
import json
import time

client = docker.from_env()

# 상태 백업
def backup_state(filename='docker-state.json'):
    state = {
        'containers': [],
        'networks': [],
        'volumes': []
    }
    
    for container in client.containers.list():
        state['containers'].append({
            'name': container.name,
            'image': container.image.tags[0] if container.image.tags else None,
            'command': container.attrs['Config']['Cmd'],
            'environment': container.attrs['Config']['Env'],
            'labels': container.attrs['Config']['Labels'],
            'ports': container.attrs['HostConfig']['PortBindings'],
            'volumes': container.attrs['HostConfig']['Binds'],
            'restart_policy': container.attrs['HostConfig']['RestartPolicy'],
        })
    
    for network in client.networks.list(filters={'driver': 'bridge'}):
        if network.name not in ['bridge', 'host', 'none']:
            state['networks'].append({
                'name': network.name,
                'driver': network.attrs['Driver'],
                'options': network.attrs['Options'],
            })
    
    for volume in client.volumes.list():
        state['volumes'].append({
            'name': volume.name,
            'driver': volume.attrs['Driver'],
        })
    
    with open(filename, 'w') as f:
        json.dump(state, f, indent=2)
    
    print(f"State backed up to {filename}")

# 상태 복구
def restore_state(filename='docker-state.json'):
    with open(filename, 'r') as f:
        state = json.load(f)
    
    # 네트워크 복구
    for net in state['networks']:
        try:
            client.networks.create(net['name'], driver=net['driver'])
            print(f"Network created: {net['name']}")
        except Exception as e:
            print(f"Network exists or error: {e}")
    
    # 볼륨 복구
    for vol in state['volumes']:
        try:
            client.volumes.create(vol['name'], driver=vol['driver'])
            print(f"Volume created: {vol['name']}")
        except Exception as e:
            print(f"Volume exists or error: {e}")
    
    # 컨테이너 복구
    for c in state['containers']:
        try:
            client.containers.run(
                image=c['image'],
                name=c['name'],
                command=c['command'],
                environment=c['environment'],
                labels=c['labels'],
                ports=c['ports'],
                volumes=c['volumes'],
                restart_policy=c['restart_policy'],
                detach=True
            )
            print(f"Container restored: {c['name']}")
        except Exception as e:
            print(f"Container restore error: {e}")

# 주기적 백업
if __name__ == '__main__':
    while True:
        backup_state()
        time.sleep(3600)  # 1시간마다
```

---

## ⚡ 성능 최적화

### 1. Image 빌드 최적화

```bash
# BuildKit 활성화
export DOCKER_BUILDKIT=1

# 빌드 캐시 최대 활용
docker build \
  --cache-from myregistry/myapp:latest \
  --tag myregistry/myapp:latest \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  .

# 병렬 빌드
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t myapp .
```

### 2. 네트워크 최적화

```json
// /etc/docker/daemon.json
{
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    }
  ],
  "bip": "172.31.0.1/16",
  "mtu": 1500
}
```

### 3. 로그 관리

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3",
    "compress": "true"
  }
}
```

---

## 🎓 핵심 정리

### Docker Engine 구조

```
핵심 컴포넌트:
├─ API Server: REST/gRPC 인터페이스
├─ Image Manager: 빌드/저장/캐시
├─ Container Manager: 생명주기 관리
├─ Network Manager: 네트워크 관리
├─ Volume Manager: 스토리지 관리
├─ Event System: 이벤트 발행/구독
└─ Plugin System: 확장성
```

### API 활용

```
주요 용도:
├─ 자동화: CI/CD 통합
├─ 모니터링: 메트릭 수집
├─ 오케스트레이션: 동적 관리
└─ 커스텀 도구: 자체 솔루션
```

### 실무 팁

```
✅ 이벤트 모니터링 활용
✅ Health Check 필수 설정
✅ API로 자동화 구축
✅ 플러그인으로 확장
✅ BuildKit으로 빌드 최적화
```

---

## 🔗 다음 단계

Docker Engine을 마스터했습니다! 다음 챕터:

- **[images/](../images/)**: 이미지 최적화 심화
- **[networking/](../networking/)**: 네트워킹 완전 정복
- **[performance/](../performance/)**: 성능 최적화

---

## 📚 참고 자료

- [Docker Engine API](https://docs.docker.com/engine/api/)
- [Docker SDK for Python](https://docker-py.readthedocs.io/)
- [BuildKit](https://github.com/moby/buildkit)
- [Docker Plugins](https://docs.docker.com/engine/extend/)

---

**🤔 생각해볼 문제**

1. Docker API를 직접 호출하는 것과 CLI를 사용하는 것의 차이는?
2. 이벤트 시스템을 활용한 모니터링의 장단점은?
3. Kubernetes는 Docker Engine의 어떤 부분을 사용할까?

> 💡 **답변**: 1) API가 더 직접적, CLI는 편의성 제공, 2) 실시간성(장점) vs 부하(단점), 3) containerd 직접 사용 (Docker Engine 우회)
