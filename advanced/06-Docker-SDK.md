# 06. Docker SDK - Python/Go로 Docker 제어

## 🎯 이 챕터에서 배울 것

- **Python SDK (docker-py)**: 가장 인기 있는 Docker SDK
- **Go SDK (docker/docker)**: 공식 Docker 클라이언트 라이브러리
- **Container 자동화**: 생성, 실행, 모니터링, 정리
- **이미지 관리**: Build, push, pull 자동화
- **실시간 모니터링**: Events, stats, logs streaming
- **고급 활용**: CI/CD 통합, 오케스트레이션 도구 개발

## 📌 왜 중요한가?

**"SDK를 사용하면 복잡한 Docker 자동화를 쉽고 안전하게 구현할 수 있습니다."**

```
Docker를 제어하는 3가지 방법:

┌────────────────────────────────────────────────────────┐
│ 1. Docker CLI                                          │
│    docker run nginx                                    │
│                                                        │
│    장점: 간단함                                           │
│    단점: 자동화 어려움, 복잡한 로직 구현 불가                    │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│ 2. Docker API (curl)                                   │
│    curl --unix-socket /var/run/docker.sock \           │
│         -X POST http://localhost/containers/create     │
│                                                        │
│    장점: 완전한 제어                                       │
│    단점: 코드가 장황함, 에러 처리 복잡                         │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│ 3. Docker SDK (Python/Go/...)                          │
│    client.containers.run("nginx")                      │
│                                                        │
│    장점: 간결한 코드, 타입 안전, 에러 처리 자동                  │
│    단점: 언어 의존성                                       │
└────────────────────────────────────────────────────────┘

SDK vs API:
┌──────────────────┬──────────────────────────────────┐
│ 작업              │ API vs SDK                       │
├──────────────────┼──────────────────────────────────┤
│ Container 생성    │ API: 20줄 vs SDK: 1줄             │
├──────────────────┼──────────────────────────────────┤
│ 에러 처리          │ API: 수동 vs SDK: 자동             │
├──────────────────┼──────────────────────────────────┤
│ 타입 안전          │ API: ❌ vs SDK: ✅               │
├──────────────────┼──────────────────────────────────┤
│ Streaming        │ API: 복잡 vs SDK: 간단             │
├──────────────────┼──────────────────────────────────┤
│ 문서화             │ API: 일반적 vs SDK: 언어별          │
└──────────────────┴──────────────────────────────────┘

Python SDK 예시:
import docker

client = docker.from_env()

# Container 실행
container = client.containers.run(
    "nginx:alpine",
    detach=True,
    ports={'80/tcp': 8080},
    remove=True
)

# 로그 스트리밍
for line in container.logs(stream=True):
    print(line.decode())

# 정리
container.stop()

Go SDK 예시:
client, _ := docker.NewClientWithOpts(docker.FromEnv)

// Container 실행
resp, _ := client.ContainerCreate(ctx, &container.Config{
    Image: "nginx:alpine",
}, &container.HostConfig{
    PortBindings: nat.PortMap{
        "80/tcp": []nat.PortBinding{{HostPort: "8080"}},
    },
}, nil, nil, "")

client.ContainerStart(ctx, resp.ID, types.ContainerStartOptions{})

// 정리
client.ContainerStop(ctx, resp.ID, nil)
```

**실무 영향:**
- **CI/CD 파이프라인**: Jenkins, GitLab CI에서 Docker 제어
- **테스트 자동화**: Integration test용 컨테이너 자동 생성/정리
- **모니터링 도구**: 커스텀 메트릭 수집기 개발
- **오케스트레이터**: 간단한 컨테이너 스케줄러 구현

---

## 🔬 Deep Dive

### 1. Python SDK (docker-py)

#### 설치 및 기본 사용

```python
# 설치
pip install docker

# 기본 클라이언트 생성
import docker

# 방법 1: 환경 변수에서 자동 감지
client = docker.from_env()

# 방법 2: Unix Socket 직접 지정
client = docker.DockerClient(base_url='unix://var/run/docker.sock')

# 방법 3: TCP 원격 호스트
client = docker.DockerClient(base_url='tcp://192.168.1.100:2375')

# 방법 4: TLS 인증
tls_config = docker.tls.TLSConfig(
    client_cert=('/path/to/cert.pem', '/path/to/key.pem'),
    ca_cert='/path/to/ca.pem',
    verify=True
)
client = docker.DockerClient(
    base_url='tcp://192.168.1.100:2376',
    tls=tls_config
)

# Docker 정보 확인
info = client.info()
print(f"Docker Version: {info['ServerVersion']}")
print(f"Containers: {info['Containers']}")
print(f"Images: {info['Images']}")

# Ping (연결 확인)
client.ping()  # True = 연결 성공
```

#### Container 관리

```python
import docker

client = docker.from_env()

# ========== Container 실행 ==========
# 간단한 실행 (foreground)
output = client.containers.run("alpine:latest", "echo hello")
print(output)  # b'hello\n'

# Detached 실행 (백그라운드)
container = client.containers.run(
    "nginx:alpine",
    detach=True,
    name="my-nginx",
    ports={'80/tcp': 8080},
    environment={"FOO": "bar"},
    volumes={'/host/path': {'bind': '/container/path', 'mode': 'rw'}},
    restart_policy={"Name": "unless-stopped"},
    mem_limit="512m",
    cpu_shares=1024,
    remove=True  # 종료 시 자동 삭제
)

print(f"Container ID: {container.id[:12]}")
print(f"Container Name: {container.name}")

# ========== Container 조회 ==========
# 모든 컨테이너 목록
all_containers = client.containers.list(all=True)
for c in all_containers:
    print(f"{c.id[:12]} {c.name} {c.status}")

# 실행 중인 컨테이너만
running = client.containers.list(filters={"status": "running"})

# 특정 컨테이너 가져오기
container = client.containers.get("my-nginx")

# ========== Container 상태 확인 ==========
# 전체 정보
attrs = container.attrs
print(f"State: {attrs['State']['Status']}")
print(f"PID: {attrs['State']['Pid']}")
print(f"Image: {attrs['Config']['Image']}")

# 리로드 (최신 상태 동기화)
container.reload()

# ========== Container 제어 ==========
# 정지
container.stop(timeout=10)  # SIGTERM 후 10초 대기

# 시작
container.start()

# 재시작
container.restart(timeout=10)

# 강제 종료
container.kill(signal="SIGKILL")

# 일시 정지 / 재개
container.pause()
container.unpause()

# 삭제
container.remove(force=True, v=True)  # force=강제, v=볼륨 삭제

# ========== 로그 ==========
# 전체 로그
logs = container.logs()
print(logs.decode())

# 실시간 스트리밍
for line in container.logs(stream=True, follow=True):
    print(line.decode(), end='')

# 마지막 10줄
logs = container.logs(tail=10)

# Timestamp 포함
logs = container.logs(timestamps=True)

# ========== Stats (리소스 사용량) ==========
# 단일 스냅샷
stats = container.stats(stream=False)
print(stats)

# 실시간 스트리밍
for stat in container.stats(stream=True):
    cpu_percent = stat['cpu_stats']['cpu_usage']['total_usage']
    memory_usage = stat['memory_stats']['usage'] / 1048576  # MB
    print(f"CPU: {cpu_percent}, Memory: {memory_usage:.2f}MB")

# ========== Exec (컨테이너 내 명령 실행) ==========
# 간단한 실행
exit_code, output = container.exec_run("ls -la /")
print(output.decode())

# TTY 포함 (interactive)
exec_result = container.exec_run(
    "/bin/sh",
    stdin=True,
    tty=True,
    stream=True
)

# ========== Top (프로세스 목록) ==========
processes = container.top()
print(processes)
# {'Processes': [['root', '1', 'nginx']], 'Titles': ['UID', 'PID', 'CMD']}

# ========== 파일 복사 ==========
# 컨테이너 → 호스트
bits, stat = container.get_archive('/etc/nginx/nginx.conf')
with open('nginx.conf.tar', 'wb') as f:
    for chunk in bits:
        f.write(chunk)

# 호스트 → 컨테이너
import tarfile
import io

tar_stream = io.BytesIO()
tar = tarfile.open(fileobj=tar_stream, mode='w')
tar.add('local-file.txt', arcname='container-file.txt')
tar.close()
tar_stream.seek(0)

container.put_archive('/tmp', tar_stream)
```

#### Image 관리

```python
# ========== Image 조회 ==========
# 전체 이미지 목록
images = client.images.list()
for img in images:
    print(f"{img.id[:12]} {img.tags}")

# 특정 이미지 가져오기
image = client.images.get("nginx:alpine")

# Dangling 이미지만
dangling = client.images.list(filters={"dangling": True})

# ========== Image Pull ==========
# Pull
image = client.images.pull("nginx", tag="alpine")
print(f"Pulled: {image.tags}")

# 진행 상황 모니터링
for line in client.api.pull("nginx", tag="alpine", stream=True, decode=True):
    if 'status' in line:
        print(f"{line['status']}: {line.get('progress', '')}")

# ========== Image Build ==========
# Dockerfile에서 빌드
image, build_logs = client.images.build(
    path="/path/to/context",
    dockerfile="Dockerfile",
    tag="myapp:v1.0",
    buildargs={"VERSION": "1.0"},
    nocache=False,
    rm=True  # 중간 컨테이너 삭제
)

# 빌드 로그 출력
for log in build_logs:
    if 'stream' in log:
        print(log['stream'], end='')

# ========== Image Push ==========
# 레지스트리에 Push
client.images.push(
    "myregistry.com/myapp",
    tag="v1.0",
    auth_config={
        "username": "user",
        "password": "pass"
    }
)

# ========== Image 삭제 ==========
client.images.remove("nginx:alpine", force=True)

# ========== Image 검사 ==========
image = client.images.get("nginx:alpine")
print(f"ID: {image.id}")
print(f"Tags: {image.tags}")
print(f"Size: {image.attrs['Size'] / 1048576:.2f}MB")
print(f"Architecture: {image.attrs['Architecture']}")
print(f"OS: {image.attrs['Os']}")

# 레이어 정보
history = image.history()
for layer in history:
    print(f"{layer['Id'][:12]} {layer.get('CreatedBy', '')[:50]}")

# ========== Image 태그 ==========
image.tag("myrepo/nginx", tag="custom")

# ========== Image Export/Import ==========
# Export (tar 파일로)
image = client.images.get("nginx:alpine")
with open("nginx.tar", "wb") as f:
    for chunk in image.save():
        f.write(chunk)

# Import (tar 파일에서)
with open("nginx.tar", "rb") as f:
    client.images.load(f.read())
```

---

### 2. Go SDK (docker/docker)

#### 설치 및 기본 사용

```go
// 설치
// go get github.com/docker/docker/client

package main

import (
    "context"
    "fmt"
    "io"
    "os"
    
    "github.com/docker/docker/api/types"
    "github.com/docker/docker/api/types/container"
    "github.com/docker/docker/client"
)

func main() {
    // 클라이언트 생성 (환경 변수에서 자동)
    cli, err := client.NewClientWithOpts(client.FromEnv)
    if err != nil {
        panic(err)
    }
    defer cli.Close()
    
    // API 버전 협상
    cli.NegotiateAPIVersion(context.Background())
    
    // Docker 정보
    info, err := cli.Info(context.Background())
    if err != nil {
        panic(err)
    }
    fmt.Printf("Server Version: %s\n", info.ServerVersion)
    fmt.Printf("Containers: %d\n", info.Containers)
    
    // Ping
    ping, err := cli.Ping(context.Background())
    fmt.Printf("API Version: %s\n", ping.APIVersion)
}
```

#### Container 관리

```go
package main

import (
    "context"
    "fmt"
    "io"
    "os"
    
    "github.com/docker/docker/api/types"
    "github.com/docker/docker/api/types/container"
    "github.com/docker/docker/client"
    "github.com/docker/go-connections/nat"
)

func main() {
    ctx := context.Background()
    cli, _ := client.NewClientWithOpts(client.FromEnv)
    defer cli.Close()
    
    // ========== Container 생성 ==========
    resp, err := cli.ContainerCreate(
        ctx,
        &container.Config{
            Image: "nginx:alpine",
            ExposedPorts: nat.PortSet{
                "80/tcp": struct{}{},
            },
            Env: []string{"FOO=bar"},
        },
        &container.HostConfig{
            PortBindings: nat.PortMap{
                "80/tcp": []nat.PortBinding{
                    {HostPort: "8080"},
                },
            },
            RestartPolicy: container.RestartPolicy{
                Name: "unless-stopped",
            },
            Resources: container.Resources{
                Memory:   536870912, // 512MB
                NanoCPUs: 1000000000, // 1 CPU
            },
            AutoRemove: true,
        },
        nil,
        nil,
        "my-nginx",
    )
    if err != nil {
        panic(err)
    }
    
    containerID := resp.ID
    fmt.Printf("Container ID: %s\n", containerID[:12])
    
    // ========== Container 시작 ==========
    if err := cli.ContainerStart(ctx, containerID, types.ContainerStartOptions{}); err != nil {
        panic(err)
    }
    
    // ========== Container 목록 ==========
    containers, err := cli.ContainerList(ctx, types.ContainerListOptions{
        All: true,
    })
    if err != nil {
        panic(err)
    }
    
    for _, c := range containers {
        fmt.Printf("%s %s %s\n", c.ID[:12], c.Names[0], c.State)
    }
    
    // ========== Container 상태 확인 ==========
    inspect, err := cli.ContainerInspect(ctx, containerID)
    if err != nil {
        panic(err)
    }
    
    fmt.Printf("State: %s\n", inspect.State.Status)
    fmt.Printf("PID: %d\n", inspect.State.Pid)
    fmt.Printf("Image: %s\n", inspect.Config.Image)
    
    // ========== Container 로그 ==========
    out, err := cli.ContainerLogs(ctx, containerID, types.ContainerLogsOptions{
        ShowStdout: true,
        ShowStderr: true,
        Follow:     false,
        Tail:       "10",
    })
    if err != nil {
        panic(err)
    }
    defer out.Close()
    
    io.Copy(os.Stdout, out)
    
    // ========== Container Stats ==========
    stats, err := cli.ContainerStats(ctx, containerID, false) // stream=false
    if err != nil {
        panic(err)
    }
    defer stats.Body.Close()
    
    // JSON 파싱 필요
    
    // ========== Exec ==========
    execResp, err := cli.ContainerExecCreate(ctx, containerID, types.ExecConfig{
        AttachStdout: true,
        AttachStderr: true,
        Cmd:          []string{"ls", "-la", "/"},
    })
    if err != nil {
        panic(err)
    }
    
    execAttach, err := cli.ContainerExecAttach(ctx, execResp.ID, types.ExecStartCheck{})
    if err != nil {
        panic(err)
    }
    defer execAttach.Close()
    
    io.Copy(os.Stdout, execAttach.Reader)
    
    // ========== Container 정지 ==========
    timeout := 10
    if err := cli.ContainerStop(ctx, containerID, container.StopOptions{
        Timeout: &timeout,
    }); err != nil {
        panic(err)
    }
    
    // ========== Container 삭제 ==========
    if err := cli.ContainerRemove(ctx, containerID, types.ContainerRemoveOptions{
        Force:         true,
        RemoveVolumes: true,
    }); err != nil {
        panic(err)
    }
}
```

#### Image 관리

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "os"
    
    "github.com/docker/docker/api/types"
    "github.com/docker/docker/client"
)

func main() {
    ctx := context.Background()
    cli, _ := client.NewClientWithOpts(client.FromEnv)
    defer cli.Close()
    
    // ========== Image Pull ==========
    out, err := cli.ImagePull(ctx, "nginx:alpine", types.ImagePullOptions{})
    if err != nil {
        panic(err)
    }
    defer out.Close()
    
    // 진행 상황 출력
    decoder := json.NewDecoder(out)
    for {
        var event map[string]interface{}
        if err := decoder.Decode(&event); err != nil {
            if err == io.EOF {
                break
            }
            panic(err)
        }
        fmt.Printf("%s: %s\n", event["status"], event["progress"])
    }
    
    // ========== Image 목록 ==========
    images, err := cli.ImageList(ctx, types.ImageListOptions{
        All: false,
    })
    if err != nil {
        panic(err)
    }
    
    for _, img := range images {
        fmt.Printf("%s %s %d\n", img.ID[:12], img.RepoTags, img.Size)
    }
    
    // ========== Image Inspect ==========
    inspect, _, err := cli.ImageInspectWithRaw(ctx, "nginx:alpine")
    if err != nil {
        panic(err)
    }
    
    fmt.Printf("ID: %s\n", inspect.ID)
    fmt.Printf("Architecture: %s\n", inspect.Architecture)
    fmt.Printf("OS: %s\n", inspect.Os)
    fmt.Printf("Size: %d MB\n", inspect.Size/1048576)
    
    // ========== Image Build ==========
    buildContext, _ := os.Open("context.tar")
    defer buildContext.Close()
    
    buildResp, err := cli.ImageBuild(ctx, buildContext, types.ImageBuildOptions{
        Tags:       []string{"myapp:v1.0"},
        Dockerfile: "Dockerfile",
        Remove:     true,
        NoCache:    false,
    })
    if err != nil {
        panic(err)
    }
    defer buildResp.Body.Close()
    
    // 빌드 로그 출력
    decoder = json.NewDecoder(buildResp.Body)
    for {
        var event map[string]interface{}
        if err := decoder.Decode(&event); err != nil {
            if err == io.EOF {
                break
            }
            panic(err)
        }
        if stream, ok := event["stream"].(string); ok {
            fmt.Print(stream)
        }
    }
    
    // ========== Image Push ==========
    pushResp, err := cli.ImagePush(ctx, "myregistry.com/myapp:v1.0", types.ImagePushOptions{
        RegistryAuth: "base64-encoded-auth",
    })
    if err != nil {
        panic(err)
    }
    defer pushResp.Close()
    
    io.Copy(os.Stdout, pushResp)
    
    // ========== Image 삭제 ==========
    _, err = cli.ImageRemove(ctx, "nginx:alpine", types.ImageRemoveOptions{
        Force:         true,
        PruneChildren: true,
    })
    if err != nil {
        panic(err)
    }
}
```

---

## 🔧 실습 1: Python으로 Integration Test 환경 자동화

### Step 1: 테스트 환경 셋업

```python
# test_environment.py
import docker
import time
import requests

class TestEnvironment:
    def __init__(self):
        self.client = docker.from_env()
        self.containers = []
    
    def setup(self):
        """테스트 환경 구성"""
        print("Setting up test environment...")
        
        # 1. PostgreSQL 데이터베이스
        db = self.client.containers.run(
            "postgres:15-alpine",
            detach=True,
            name="test-db",
            environment={
                "POSTGRES_DB": "testdb",
                "POSTGRES_USER": "testuser",
                "POSTGRES_PASSWORD": "testpass"
            },
            ports={'5432/tcp': 5432},
            remove=True
        )
        self.containers.append(db)
        print(f"✅ Database started: {db.id[:12]}")
        
        # 2. Redis 캐시
        redis = self.client.containers.run(
            "redis:7-alpine",
            detach=True,
            name="test-redis",
            ports={'6379/tcp': 6379},
            remove=True
        )
        self.containers.append(redis)
        print(f"✅ Redis started: {redis.id[:12]}")
        
        # 3. 애플리케이션 서버 (가상)
        app = self.client.containers.run(
            "nginx:alpine",
            detach=True,
            name="test-app",
            ports={'80/tcp': 8080},
            volumes={
                './app': {'bind': '/usr/share/nginx/html', 'mode': 'rw'}
            },
            remove=True
        )
        self.containers.append(app)
        print(f"✅ App started: {app.id[:12]}")
        
        # 대기 (서비스 준비 시간)
        time.sleep(5)
        self.wait_for_healthy()
    
    def wait_for_healthy(self):
        """모든 서비스가 준비될 때까지 대기"""
        print("Waiting for services to be ready...")
        
        for container in self.containers:
            container.reload()
            if container.status != 'running':
                raise Exception(f"Container {container.name} not running")
        
        # 애플리케이션 헬스체크
        max_retries = 30
        for i in range(max_retries):
            try:
                resp = requests.get("http://localhost:8080")
                if resp.status_code == 200:
                    print("✅ All services ready")
                    return
            except requests.exceptions.ConnectionError:
                time.sleep(1)
        
        raise Exception("Services failed to start")
    
    def get_logs(self):
        """모든 컨테이너 로그 수집"""
        logs = {}
        for container in self.containers:
            logs[container.name] = container.logs().decode()
        return logs
    
    def teardown(self):
        """테스트 환경 정리"""
        print("Tearing down test environment...")
        
        for container in self.containers:
            try:
                print(f"Stopping {container.name}...")
                container.stop(timeout=5)
                print(f"✅ {container.name} stopped")
            except Exception as e:
                print(f"❌ Error stopping {container.name}: {e}")
        
        self.containers = []
        print("Cleanup completed")

# 사용 예
if __name__ == "__main__":
    env = TestEnvironment()
    
    try:
        env.setup()
        
        # 테스트 실행
        print("\nRunning tests...")
        time.sleep(3)
        print("Tests completed")
        
        # 로그 확인
        logs = env.get_logs()
        for name, log in logs.items():
            print(f"\n=== {name} logs ===")
            print(log[:200])  # 처음 200자만
        
    finally:
        env.teardown()
```

---

## 🔧 실습 2: Python으로 실시간 모니터링 대시보드

### Step 1: Container Stats 수집

```python
# monitor.py
import docker
import time
from datetime import datetime

class ContainerMonitor:
    def __init__(self):
        self.client = docker.from_env()
    
    def get_container_stats(self, container_id):
        """컨테이너 리소스 사용량 조회"""
        container = self.client.containers.get(container_id)
        
        stats = container.stats(stream=False)
        
        # CPU 사용률 계산
        cpu_delta = stats['cpu_stats']['cpu_usage']['total_usage'] - \
                    stats['precpu_stats']['cpu_usage']['total_usage']
        system_delta = stats['cpu_stats']['system_cpu_usage'] - \
                       stats['precpu_stats']['system_cpu_usage']
        cpu_percent = (cpu_delta / system_delta) * 100.0 if system_delta > 0 else 0.0
        
        # 메모리 사용량
        memory_usage = stats['memory_stats']['usage'] / 1048576  # MB
        memory_limit = stats['memory_stats']['limit'] / 1048576  # MB
        memory_percent = (memory_usage / memory_limit) * 100.0
        
        # 네트워크 I/O
        networks = stats.get('networks', {})
        network_rx = sum(net['rx_bytes'] for net in networks.values())
        network_tx = sum(net['tx_bytes'] for net in networks.values())
        
        return {
            'name': container.name,
            'cpu_percent': round(cpu_percent, 2),
            'memory_usage_mb': round(memory_usage, 2),
            'memory_limit_mb': round(memory_limit, 2),
            'memory_percent': round(memory_percent, 2),
            'network_rx_mb': round(network_rx / 1048576, 2),
            'network_tx_mb': round(network_tx / 1048576, 2),
        }
    
    def monitor_all(self, interval=5):
        """모든 실행 중인 컨테이너 모니터링"""
        print("Container Monitoring Dashboard")
        print("=" * 80)
        
        try:
            while True:
                containers = self.client.containers.list()
                
                print(f"\n[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}]")
                print(f"{'Container':<20} {'CPU %':<10} {'Memory':<20} {'Network I/O'}")
                print("-" * 80)
                
                for container in containers:
                    try:
                        stats = self.get_container_stats(container.id)
                        
                        print(f"{stats['name']:<20} "
                              f"{stats['cpu_percent']:<10.2f} "
                              f"{stats['memory_usage_mb']:.0f}MB / {stats['memory_limit_mb']:.0f}MB "
                              f"({stats['memory_percent']:.1f}%)  "
                              f"↓{stats['network_rx_mb']:.2f} ↑{stats['network_tx_mb']:.2f}")
                    
                    except Exception as e:
                        print(f"{container.name:<20} Error: {e}")
                
                time.sleep(interval)
        
        except KeyboardInterrupt:
            print("\n\nMonitoring stopped")

# 사용 예
if __name__ == "__main__":
    monitor = ContainerMonitor()
    monitor.monitor_all(interval=3)
```

---

## 🔧 실습 3: Go로 Image 빌드 자동화

### Step 1: 멀티 스테이지 빌드 자동화

```go
// build_automation.go
package main

import (
    "archive/tar"
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "os"
    "path/filepath"
    
    "github.com/docker/docker/api/types"
    "github.com/docker/docker/client"
)

// BuildContext를 tar로 압축
func createBuildContext(contextPath string) (io.Reader, error) {
    buf := new(bytes.Buffer)
    tw := tar.NewWriter(buf)
    defer tw.Close()
    
    // Dockerfile과 모든 파일 추가
    err := filepath.Walk(contextPath, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        
        // 디렉토리는 스킵
        if info.IsDir() {
            return nil
        }
        
        // 상대 경로
        relPath, err := filepath.Rel(contextPath, path)
        if err != nil {
            return err
        }
        
        // tar 헤더 생성
        header, err := tar.FileInfoHeader(info, info.Name())
        if err != nil {
            return err
        }
        header.Name = relPath
        
        if err := tw.WriteHeader(header); err != nil {
            return err
        }
        
        // 파일 내용 쓰기
        file, err := os.Open(path)
        if err != nil {
            return err
        }
        defer file.Close()
        
        _, err = io.Copy(tw, file)
        return err
    })
    
    return buf, err
}

// 이미지 빌드 및 푸시
func buildAndPush(contextPath, tag string) error {
    ctx := context.Background()
    cli, err := client.NewClientWithOpts(client.FromEnv)
    if err != nil {
        return err
    }
    defer cli.Close()
    
    // Build context 생성
    fmt.Println("Creating build context...")
    buildContext, err := createBuildContext(contextPath)
    if err != nil {
        return err
    }
    
    // 이미지 빌드
    fmt.Printf("Building image: %s\n", tag)
    buildResp, err := cli.ImageBuild(ctx, buildContext, types.ImageBuildOptions{
        Tags:       []string{tag},
        Dockerfile: "Dockerfile",
        Remove:     true,
        NoCache:    false,
        BuildArgs:  map[string]*string{
            "VERSION": stringPtr("1.0.0"),
        },
    })
    if err != nil {
        return err
    }
    defer buildResp.Body.Close()
    
    // 빌드 로그 출력
    decoder := json.NewDecoder(buildResp.Body)
    for {
        var event map[string]interface{}
        if err := decoder.Decode(&event); err != nil {
            if err == io.EOF {
                break
            }
            return err
        }
        
        if stream, ok := event["stream"].(string); ok {
            fmt.Print(stream)
        }
        if errorMsg, ok := event["error"].(string); ok {
            return fmt.Errorf("build error: %s", errorMsg)
        }
    }
    
    fmt.Println("✅ Build completed")
    
    // 이미지 푸시 (레지스트리 설정 필요)
    // pushResp, err := cli.ImagePush(ctx, tag, types.ImagePushOptions{})
    // ...
    
    return nil
}

func stringPtr(s string) *string {
    return &s
}

func main() {
    if err := buildAndPush("./myapp", "myregistry.com/myapp:v1.0"); err != nil {
        fmt.Fprintf(os.Stderr, "Error: %v\n", err)
        os.Exit(1)
    }
}
```

---

## 🔧 실습 4: Python으로 Events 기반 자동화

### Step 1: Event Listener 구현

```python
# event_handler.py
import docker
from datetime import datetime

class EventHandler:
    def __init__(self):
        self.client = docker.from_env()
        self.handlers = {
            'die': self.handle_die,
            'oom': self.handle_oom,
            'start': self.handle_start,
            'stop': self.handle_stop,
        }
    
    def handle_die(self, event):
        """컨테이너 종료 이벤트"""
        container_id = event['Actor']['ID'][:12]
        container_name = event['Actor']['Attributes'].get('name', 'unknown')
        exit_code = event['Actor']['Attributes'].get('exitCode', '0')
        
        print(f"🔴 Container died: {container_name} ({container_id})")
        print(f"   Exit Code: {exit_code}")
        
        # Exit code가 0이 아니면 경고
        if exit_code != '0':
            print(f"   ⚠️  Abnormal exit!")
            # Slack 알림, 로그 수집 등
    
    def handle_oom(self, event):
        """OOM Killed 이벤트"""
        container_id = event['Actor']['ID'][:12]
        container_name = event['Actor']['Attributes'].get('name', 'unknown')
        
        print(f"💀 OOMKilled: {container_name} ({container_id})")
        
        # 컨테이너 정보 조회
        try:
            container = self.client.containers.get(container_id)
            memory_limit = container.attrs['HostConfig']['Memory']
            print(f"   Memory Limit: {memory_limit / 1048576:.0f}MB")
            
            # 자동 재시작 (메모리 증가)
            # new_limit = memory_limit * 1.5
            # ...
        except:
            pass
    
    def handle_start(self, event):
        """컨테이너 시작 이벤트"""
        container_id = event['Actor']['ID'][:12]
        container_name = event['Actor']['Attributes'].get('name', 'unknown')
        
        print(f"✅ Container started: {container_name} ({container_id})")
    
    def handle_stop(self, event):
        """컨테이너 정지 이벤트"""
        container_id = event['Actor']['ID'][:12]
        container_name = event['Actor']['Attributes'].get('name', 'unknown')
        
        print(f"🛑 Container stopped: {container_name} ({container_id})")
    
    def listen(self):
        """이벤트 스트림 청취"""
        print("Event Listener Started")
        print("=" * 60)
        
        # 컨테이너 이벤트만 필터링
        filters = {
            'type': 'container',
            'event': list(self.handlers.keys())
        }
        
        try:
            for event in self.client.events(decode=True, filters=filters):
                timestamp = datetime.fromtimestamp(event['time'])
                print(f"\n[{timestamp.strftime('%Y-%m-%d %H:%M:%S')}]")
                
                action = event['Action']
                if action in self.handlers:
                    self.handlers[action](event)
        
        except KeyboardInterrupt:
            print("\n\nEvent listener stopped")

# 사용 예
if __name__ == "__main__":
    handler = EventHandler()
    handler.listen()
```

---

## 🔧 실습 5: Go로 컨테이너 헬스체크 시스템

### Step 1: 헬스체크 모니터 구현

```go
// health_monitor.go
package main

import (
    "context"
    "fmt"
    "time"
    
    "github.com/docker/docker/api/types"
    "github.com/docker/docker/client"
)

type HealthMonitor struct {
    client *client.Client
}

func NewHealthMonitor() (*HealthMonitor, error) {
    cli, err := client.NewClientWithOpts(client.FromEnv)
    if err != nil {
        return nil, err
    }
    return &HealthMonitor{client: cli}, nil
}

func (hm *HealthMonitor) CheckHealth(containerID string) (string, error) {
    ctx := context.Background()
    
    inspect, err := hm.client.ContainerInspect(ctx, containerID)
    if err != nil {
        return "", err
    }
    
    // State 확인
    if !inspect.State.Running {
        return "stopped", nil
    }
    
    // Health 확인 (Healthcheck 정의된 경우)
    if inspect.State.Health != nil {
        return inspect.State.Health.Status, nil
    }
    
    return "no-healthcheck", nil
}

func (hm *HealthMonitor) MonitorAll(interval time.Duration) {
    ctx := context.Background()
    ticker := time.NewTicker(interval)
    defer ticker.Stop()
    
    fmt.Println("Health Monitor Started")
    fmt.Println(strings.Repeat("=", 80))
    
    for range ticker.C {
        containers, err := hm.client.ContainerList(ctx, types.ContainerListOptions{})
        if err != nil {
            fmt.Printf("Error listing containers: %v\n", err)
            continue
        }
        
        fmt.Printf("\n[%s]\n", time.Now().Format("2006-01-02 15:04:05"))
        fmt.Printf("%-20s %-15s %-20s\n", "Container", "Status", "Health")
        fmt.Println(strings.Repeat("-", 80))
        
        for _, container := range containers {
            name := container.Names[0][1:] // Remove leading /
            status := container.State
            
            health, err := hm.CheckHealth(container.ID)
            if err != nil {
                health = "error"
            }
            
            // Health에 따라 이모지 표시
            healthIcon := "❓"
            switch health {
            case "healthy":
                healthIcon = "✅"
            case "unhealthy":
                healthIcon = "❌"
            case "starting":
                healthIcon = "⏳"
            case "stopped":
                healthIcon = "🛑"
            }
            
            fmt.Printf("%-20s %-15s %s %-20s\n", 
                name, status, healthIcon, health)
            
            // Unhealthy 컨테이너 처리
            if health == "unhealthy" {
                fmt.Printf("  ⚠️  Restarting unhealthy container...\n")
                timeout := 10
                if err := hm.client.ContainerRestart(ctx, container.ID, 
                    &timeout); err != nil {
                    fmt.Printf("  ❌ Restart failed: %v\n", err)
                } else {
                    fmt.Printf("  ✅ Restarted successfully\n")
                }
            }
        }
    }
}

func main() {
    monitor, err := NewHealthMonitor()
    if err != nil {
        panic(err)
    }
    
    monitor.MonitorAll(10 * time.Second)
}
```

---

## 💡 주요 SDK 기능 정리

```python
# ========== Python SDK (docker-py) ==========
import docker

client = docker.from_env()

# Container
client.containers.run()
client.containers.list()
client.containers.get()
container.start() / .stop() / .restart() / .remove()
container.logs(stream=True)
container.stats(stream=True)
container.exec_run()

# Image
client.images.pull()
client.images.build()
client.images.push()
client.images.list()
client.images.get()
image.tag() / .save() / .remove()

# Network
client.networks.create()
client.networks.list()
network.connect() / .disconnect()

# Volume
client.volumes.create()
client.volumes.list()
volume.remove()

# Events
client.events(decode=True, filters={...})

# System
client.info()
client.ping()
client.version()
```

```go
// ========== Go SDK (docker/docker) ==========
import (
    "github.com/docker/docker/client"
    "github.com/docker/docker/api/types"
)

cli, _ := client.NewClientWithOpts(client.FromEnv)

// Container
cli.ContainerCreate()
cli.ContainerStart()
cli.ContainerStop()
cli.ContainerList()
cli.ContainerInspect()
cli.ContainerLogs()
cli.ContainerStats()
cli.ContainerExecCreate()

// Image
cli.ImagePull()
cli.ImageBuild()
cli.ImagePush()
cli.ImageList()
cli.ImageInspect()
cli.ImageRemove()

// Network
cli.NetworkCreate()
cli.NetworkList()
cli.NetworkConnect()

// Volume
cli.VolumeCreate()
cli.VolumeList()
cli.VolumeRemove()

// Events
cli.Events()

// System
cli.Info()
cli.Ping()
cli.ServerVersion()
```

---

## 🎓 연습 문제

### 문제 1: Python SDK로 "모든 정지된 컨테이너를 삭제하는 함수"를 작성하시오.

<details>
<summary>정답 보기</summary>

```python
import docker

def cleanup_stopped_containers():
    """정지된 모든 컨테이너 삭제"""
    client = docker.from_env()
    
    # 정지된 컨테이너 필터링
    stopped = client.containers.list(
        all=True,
        filters={"status": "exited"}
    )
    
    print(f"Found {len(stopped)} stopped containers")
    
    for container in stopped:
        try:
            print(f"Removing {container.name} ({container.id[:12]})...")
            container.remove(v=True)  # v=True: 볼륨도 삭제
            print(f"  ✅ Removed")
        except Exception as e:
            print(f"  ❌ Error: {e}")
    
    print("Cleanup completed")

# 또는 prune 사용 (더 간단)
def cleanup_with_prune():
    client = docker.from_env()
    result = client.containers.prune()
    
    print(f"Deleted {len(result['ContainersDeleted'])} containers")
    print(f"Reclaimed {result['SpaceReclaimed']} bytes")

# 실행
cleanup_stopped_containers()
# cleanup_with_prune()
```

**prune의 장점:**
- 한 번의 API 호출
- 자동으로 모든 정리 수행
- Dangling 볼륨도 함께 정리 가능

</details>

### 문제 2: Go SDK로 "특정 이미지의 모든 레이어를 출력하는 함수"를 작성하시오.

<details>
<summary>정답 보기</summary>

```go
package main

import (
    "context"
    "fmt"
    
    "github.com/docker/docker/client"
)

func printImageLayers(imageName string) error {
    ctx := context.Background()
    cli, err := client.NewClientWithOpts(client.FromEnv)
    if err != nil {
        return err
    }
    defer cli.Close()
    
    // Image inspect
    inspect, _, err := cli.ImageInspectWithRaw(ctx, imageName)
    if err != nil {
        return err
    }
    
    fmt.Printf("Image: %s\n", imageName)
    fmt.Printf("ID: %s\n", inspect.ID)
    fmt.Printf("Size: %.2f MB\n\n", float64(inspect.Size)/1048576)
    
    // RootFS 레이어
    fmt.Println("Layers:")
    for i, layer := range inspect.RootFS.Layers {
        fmt.Printf("  [%d] %s\n", i+1, layer)
    }
    
    // History (빌드 단계)
    fmt.Println("\nHistory:")
    history, err := cli.ImageHistory(ctx, imageName)
    if err != nil {
        return err
    }
    
    for i, h := range history {
        createdBy := h.CreatedBy
        if len(createdBy) > 60 {
            createdBy = createdBy[:60] + "..."
        }
        
        fmt.Printf("  [%d] Size: %.2f MB\n", 
            i+1, float64(h.Size)/1048576)
        fmt.Printf("      Command: %s\n", createdBy)
    }
    
    return nil
}

func main() {
    if err := printImageLayers("nginx:alpine"); err != nil {
        panic(err)
    }
}
```

**출력 예:**
```
Image: nginx:alpine
ID: sha256:abc123...
Size: 41.25 MB

Layers:
  [1] sha256:a3ed95caeb02ffe68...
  [2] sha256:9f13e0ac480c5a4b8ab...
  [3] sha256:1b2c3d4e5f6a7b8c9d0...

History:
  [1] Size: 0.00 MB
      Command: /bin/sh -c #(nop) CMD ["nginx" "-g" "daemon off;"]
  [2] Size: 25.31 MB
      Command: /bin/sh -c apk add --no-cache nginx
  [3] Size: 7.34 MB
      Command: /bin/sh -c #(nop) ADD file:abc123... in /
```

</details>

### 문제 3: Python SDK로 "컨테이너 CPU 사용률이 80%를 초과하면 알림을 보내는 모니터"를 작성하시오.

<details>
<summary>정답 보기</summary>

```python
import docker
import time

class CPUMonitor:
    def __init__(self, threshold=80.0):
        self.client = docker.from_env()
        self.threshold = threshold
        self.alerts = {}  # 중복 알림 방지
    
    def calculate_cpu_percent(self, stats):
        """CPU 사용률 계산"""
        cpu_delta = (stats['cpu_stats']['cpu_usage']['total_usage'] -
                     stats['precpu_stats']['cpu_usage']['total_usage'])
        system_delta = (stats['cpu_stats']['system_cpu_usage'] -
                        stats['precpu_stats']['system_cpu_usage'])
        
        if system_delta > 0:
            return (cpu_delta / system_delta) * 100.0
        return 0.0
    
    def send_alert(self, container_name, cpu_percent):
        """알림 전송 (Slack, Email 등)"""
        print(f"\n🚨 ALERT 🚨")
        print(f"Container: {container_name}")
        print(f"CPU Usage: {cpu_percent:.2f}%")
        print(f"Threshold: {self.threshold}%")
        
        # 실제 구현:
        # requests.post("https://hooks.slack.com/...", json={
        #     "text": f"High CPU: {container_name} ({cpu_percent:.2f}%)"
        # })
    
    def monitor(self, interval=5):
        """지속적 모니터링"""
        print(f"CPU Monitor Started (Threshold: {self.threshold}%)")
        print("=" * 60)
        
        try:
            while True:
                containers = self.client.containers.list()
                
                for container in containers:
                    try:
                        stats = container.stats(stream=False)
                        cpu_percent = self.calculate_cpu_percent(stats)
                        
                        print(f"{container.name}: {cpu_percent:.2f}%")
                        
                        # Threshold 초과 시 알림
                        if cpu_percent > self.threshold:
                            # 중복 알림 방지 (5분 간격)
                            alert_key = f"{container.id}_{int(time.time() / 300)}"
                            
                            if alert_key not in self.alerts:
                                self.send_alert(container.name, cpu_percent)
                                self.alerts[alert_key] = True
                        
                    except Exception as e:
                        print(f"{container.name}: Error - {e}")
                
                time.sleep(interval)
        
        except KeyboardInterrupt:
            print("\n\nMonitoring stopped")

# 사용 예
if __name__ == "__main__":
    monitor = CPUMonitor(threshold=80.0)
    monitor.monitor(interval=5)
```

**고급 기능 추가:**
```python
class AdvancedCPUMonitor(CPUMonitor):
    def __init__(self, threshold=80.0, actions=None):
        super().__init__(threshold)
        self.actions = actions or {}
    
    def handle_high_cpu(self, container, cpu_percent):
        """High CPU 처리"""
        action = self.actions.get('high_cpu', 'alert')
        
        if action == 'restart':
            print(f"  → Restarting container...")
            container.restart(timeout=10)
        
        elif action == 'scale':
            # 리소스 증가 (재생성 필요)
            print(f"  → Scaling resources...")
            # 새 컨테이너 생성 로직
        
        elif action == 'alert':
            self.send_alert(container.name, cpu_percent)

# 사용
monitor = AdvancedCPUMonitor(
    threshold=80.0,
    actions={'high_cpu': 'restart'}
)
monitor.monitor()
```

</details>

---

## 📌 핵심 요약

```
┌──────────────────┬────────────────────────────────────┐
│ SDK              │ 특징                                │
├──────────────────┼────────────────────────────────────┤
│ Python (docker)  │ - 가장 인기, 간결한 API                │
│                  │ - Integration test, CI/CD에 최적     │
│                  │ - pip install docker               │
├──────────────────┼────────────────────────────────────┤
│ Go (docker/      │ - 공식 Docker 클라이언트 라이브러리       │
│ docker)          │ - 타입 안전, 고성능                    │
│                  │ - Kubernetes 등에서 사용              │
├──────────────────┼────────────────────────────────────┤
│ 공통 기능          │ - Container/Image/Network/Volume   │
│                  │ - Events, Stats, Logs streaming    │
│                  │ - Exec, Build, Push/Pull           │
└──────────────────┴────────────────────────────────────┘

주요 사용 사례:
- Integration Test 자동화
- CI/CD 파이프라인 통합
- 모니터링 시스템 구축
- 컨테이너 오케스트레이션
- 자동 스케일링
```

---

## 📚 참고 자료

- [Python Docker SDK](https://docker-py.readthedocs.io/)
- [Go Docker SDK](https://pkg.go.dev/github.com/docker/docker/client)
- [Docker SDK for Node.js](https://github.com/apocas/dockerode)
- [Docker SDK for Java](https://github.com/docker-java/docker-java)

---

## 🤔 생각해볼 문제

1. SDK를 사용한 자동화와 Kubernetes를 사용한 오케스트레이션의 차이점은?
2. Python SDK vs Go SDK - 어떤 상황에서 어떤 것을 선택해야 하는가?
3. SDK를 활용해 간단한 컨테이너 스케줄러를 구현한다면 어떤 기능이 필요할까?

> 💡 **답변**:
> 
> **1) SDK 자동화 vs Kubernetes:**
> 
> **SDK 자동화:**
> - 단일 호스트 관리
> - 스크립트 기반 제어
> - 용도: CI/CD, 테스트, 개발 환경
> - 간단하지만 제한적
> 
> **Kubernetes:**
> - 클러스터 관리 (여러 노드)
> - 선언적 설정 (YAML)
> - 용도: 프로덕션 오케스트레이션
> - 복잡하지만 강력
> 
> **비유:**
> - SDK = 수동 기어 자동차 (직접 제어)
> - Kubernetes = 자율주행 자동차 (목적지만 지정)
> 
> **2) Python vs Go SDK 선택:**
> 
> **Python SDK 선택:**
> - 빠른 프로토타이핑
> - 데이터 분석/처리 포함
> - 간단한 자동화 스크립트
> - DevOps/SRE 팀 주 사용 언어가 Python
> 
> **Go SDK 선택:**
> - 고성능 필요 (대규모 시스템)
> - 타입 안전성 중요
> - Docker/Kubernetes 생태계와 통합
> - 프로덕션급 도구 개발
> 
> **3) 간단한 스케줄러 구현:**
> ```
> 필요한 기능:
> 1. 컨테이너 배치
>    - 리소스 요구사항 체크
>    - 노드 선택 알고리즘
> 
> 2. 헬스체크
>    - 주기적 상태 확인
>    - 자동 재시작
> 
> 3. 리소스 관리
>    - CPU/Memory 할당
>    - 제한 설정
> 
> 4. 로드밸런싱
>    - 여러 컨테이너로 트래픽 분산
> 
> 5. 로깅/모니터링
>    - 중앙 로그 수집
>    - 메트릭 집계
> ```

---

<div align="center">

**[⬅️ 이전: Docker API](./05-Docker-API.md)** | **[다음: Custom Plugins ➡️](07-Custom-Plugins.md)**

</div>
