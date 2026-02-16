# 01. Debugging Techniques - 컨테이너 디버깅 기법

## 🎯 이 챕터에서 배울 것

- **컨테이너 진입**: docker exec, kubectl exec
- **프로세스 추적**: strace, ltrace
- **네임스페이스 진입**: nsenter, docker run --pid
- **파일시스템 접근**: 볼륨 마운트, 복사
- **디버깅 컨테이너**: ephemeral containers
- **실전 기법**: 프로덕션 환경 디버깅

## 📌 왜 중요한가?

**"컨테이너는 격리되어 있어 디버깅이 어렵지만, 적절한 도구로 효과적으로 해결할 수 있습니다."**

```
Container Debugging의 핵심:

Problem: 컨테이너 내부 문제
┌─────────────────────────────────────────────────┐
│ Container (격리됨)                                │
│                                                 │
│  ┌────────────────────────────────────────┐     │
│  │ Application                            │     │
│  │                                        │     │
│  │ ❌ 에러 발생!                            │     │
│  │ ❌ 프로세스 행 hang                      │     │
│  │ ❌ 네트워크 연결 안 됨                     │     │
│  └────────────────────────────────────────┘     │
│                                                 │
│  문제:                                           │
│  - 쉘 접근 불가 (distroless)                       │
│  - 디버깅 도구 없음                                 │
│  - 프로세스가 PID 1                                │
│  - 로그만으로 파악 어려움                             │
└─────────────────────────────────────────────────┘

Solution: 다양한 디버깅 기법
┌─────────────────────────────────────────────────┐
│ Debugging Techniques                            │
│                                                 │
│ 1. docker exec (쉘 진입)                          │
│    docker exec -it container /bin/bash          │
│                                                 │
│ 2. nsenter (네임스페이스 진입)                       │
│    nsenter -t PID -n -p /bin/bash               │
│                                                 │
│ 3. strace (시스템 콜 추적)                         │
│    strace -p PID                                │
│                                                 │
│ 4. 디버깅 컨테이너 (사이드카)                         │
│    kubectl debug pod/myapp -it --image=busybox  │
│                                                 │
│ 5. 파일 복사                                      │
│    docker cp container:/log /tmp/               │
└─────────────────────────────────────────────────┘

Debugging Layers:
┌─────────────────────────────────────────────────┐
│ Level 1: Application Layer                      │
│  - 로그 확인                                      │
│  - 환경 변수 확인                                  │
│  - 프로세스 상태 확인                               │
│                                                 │
│ Level 2: Container Layer                        │
│  - 컨테이너 메타데이터                               │
│  - 리소스 사용률                                   │
│  - 종료 코드                                      │
│                                                 │
│ Level 3: System Call Layer                      │
│  - strace (시스템 콜)                             │
│  - ltrace (라이브러리 호출)                         │
│  - 파일 디스크립터                                  │
│                                                 │
│ Level 4: Kernel Layer                           │
│  - dmesg (커널 로그)                              │
│  - cgroup 설정                                   │
│  - 네임스페이스 상태                                │
└─────────────────────────────────────────────────┘

Common Scenarios:
┌──────────────────────┬────────────────────────────┐
│ 문제                  │ 디버깅 도구                   │
├──────────────────────┼────────────────────────────┤
│ 프로세스 Hang          │ strace, gdb                │
├──────────────────────┼────────────────────────────┤
│ 파일 접근 실패          │ strace, ls -la             │
├──────────────────────┼────────────────────────────┤
│ 네트워크 연결 실패       │ tcpdump, netstat           │
├──────────────────────┼────────────────────────────┤
│ 메모리 누수             │ pmap, /proc/PID/smaps      │
├──────────────────────┼────────────────────────────┤
│ 높은 CPU              │ top, perf, flamegraph      │
└──────────────────────┴────────────────────────────┘
```

**실무 영향:**
- **빠른 해결**: 문제 원인 신속 파악
- **다운타임 최소화**: 효율적 트러블슈팅
- **사전 예방**: 패턴 학습으로 재발 방지
- **전문성**: 심층 분석 능력 향상

---

## 🔬 Deep Dive

### 1. 컨테이너 진입 기법

#### docker exec (기본)

```bash
# 쉘 진입 (bash)
docker exec -it container-name /bin/bash

# 쉘 진입 (sh, alpine 등)
docker exec -it container-name /bin/sh

# 특정 명령 실행
docker exec container-name ls -la /app

# 특정 사용자로 실행
docker exec -u 1000 container-name whoami

# 환경 변수 포함
docker exec -e DEBUG=true container-name printenv
```

#### kubectl exec (Kubernetes)

```bash
# Pod 내 쉘 진입
kubectl exec -it pod-name -- /bin/bash

# 특정 컨테이너 지정 (multi-container)
kubectl exec -it pod-name -c container-name -- /bin/bash

# 명령 실행
kubectl exec pod-name -- ls -la /app

# 파일 복사
kubectl cp pod-name:/app/log.txt ./log.txt
kubectl cp ./config.yaml pod-name:/app/config.yaml
```

---

### 2. strace (시스템 콜 추적)

#### 기본 사용법

```bash
# 프로세스 추적
strace -p PID

# 새 프로세스 시작하며 추적
strace ls -la

# 파일 I/O만 추적
strace -e trace=open,read,write -p PID

# 네트워크만 추적
strace -e trace=network -p PID

# 출력 파일로 저장
strace -o trace.log -p PID
```

---

## 🔧 실습 1: 기본 컨테이너 진입

### Step 1: 실행 중인 컨테이너 디버깅

```bash
# 1. 컨테이너 시작
docker run -d --name myapp nginx

# 2. 프로세스 확인
docker exec myapp ps aux
# PID 1: nginx master
# PID 7: nginx worker

# 3. 쉘 진입
docker exec -it myapp /bin/bash

# 4. 내부에서 디버깅
root@container:/# ps aux
root@container:/# ls -la /etc/nginx/
root@container:/# cat /etc/nginx/nginx.conf
root@container:/# curl localhost
root@container:/# exit

# 5. 특정 명령 실행 (외부에서)
docker exec myapp cat /var/log/nginx/access.log
docker exec myapp nginx -t  # 설정 파일 검증
```

### Step 2: 디버깅 도구가 없는 컨테이너

```bash
# distroless 이미지 (쉘 없음)
docker run -d --name minimal gcr.io/distroless/base

# exec 실패
docker exec -it minimal /bin/sh
# Error: executable file not found

# 해결 1: 디버깅 컨테이너 사용
docker run -it --rm \
  --pid=container:minimal \
  --net=container:minimal \
  --cap-add SYS_PTRACE \
  busybox sh

# 내부에서 프로세스 확인
/ # ps aux  # minimal 컨테이너 프로세스 보임

# 해결 2: 파일시스템 마운트
docker run -it --rm \
  -v /var/lib/docker:/docker \
  busybox sh

# 컨테이너 파일 시스템 찾기
/ # find /docker -name "*minimal*"
```

---

## 🔧 실습 2: strace를 이용한 시스템 콜 추적

### Step 1: 파일 I/O 문제 디버깅

```bash
# 문제: 애플리케이션이 설정 파일을 못 찾음
docker run -d --name app myapp:latest

# 로그 확인
docker logs app
# Error: Config file not found

# strace로 추적
docker exec app strace -e trace=open,openat -f -p 1

# 출력 분석:
# openat(AT_FDCWD, "/app/config.yaml", O_RDONLY) = -1 ENOENT
# → /app/config.yaml 파일이 없음!

# 실제 경로 확인
docker exec app find / -name "*.yaml"
# /etc/app/config.yaml 발견

# 문제: 경로가 다름
# 해결: 심볼릭 링크 또는 환경 변수 수정
```

### Step 2: 네트워크 연결 문제

```python
# app.py (Python)
import requests

response = requests.get('http://api.example.com')
print(response.text)
```

```bash
# 컨테이너 실행
docker run -d --name app myapp

# 로그: 연결 실패
docker logs app
# ConnectionError: Failed to establish connection

# strace로 네트워크 추적
docker exec app strace -e trace=socket,connect,sendto,recvfrom -p 1

# 출력:
# socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
# connect(3, {sa_family=AF_INET, sin_port=htons(80), 
#            sin_addr=inet_addr("192.168.1.100")}, 16) = -1 ETIMEDOUT
# → 192.168.1.100으로 연결 시도 → 타임아웃

# DNS 확인
docker exec app cat /etc/resolv.conf
docker exec app nslookup api.example.com

# 문제: DNS 해석 실패 또는 네트워크 규칙
```

---

## 🔧 실습 3: nsenter를 이용한 네임스페이스 진입

### Step 1: nsenter 기본 사용

```bash
# 컨테이너 시작
docker run -d --name myapp nginx

# 컨테이너의 PID 확인
PID=$(docker inspect -f '{{.State.Pid}}' myapp)
echo $PID  # 예: 12345

# 네임스페이스 진입 (모든 네임스페이스)
sudo nsenter -t $PID -m -u -i -n -p /bin/bash

# 내부에서 확인
root@host:/# hostname  # 컨테이너 hostname
root@host:/# ip addr   # 컨테이너 네트워크
root@host:/# ps aux    # 컨테이너 프로세스

# 특정 네임스페이스만 진입
# Network namespace만
sudo nsenter -t $PID -n ip addr

# Mount namespace만
sudo nsenter -t $PID -m ls /
```

### Step 2: 네트워크 디버깅

```bash
# 문제: 컨테이너에서 외부 연결 안 됨
docker run -d --name app myapp

# nsenter로 네트워크 진입
PID=$(docker inspect -f '{{.State.Pid}}' app)
sudo nsenter -t $PID -n bash

# 네트워크 상태 확인
ip addr
ip route
iptables -L
netstat -tlnp

# 연결 테스트
ping 8.8.8.8
curl google.com
```

---

## 🔧 실습 4: Kubernetes Ephemeral Containers

### Step 1: 디버깅 컨테이너 추가

```bash
# Pod에 디버깅 컨테이너 추가
kubectl debug -it pod/myapp --image=busybox --target=myapp

# 또는 디버깅 전용 컨테이너
kubectl debug pod/myapp -it --image=nicolaka/netshoot --share-processes

# 생성된 디버깅 컨테이너에서
/ # ps aux  # 모든 프로세스 보임 (shared PID namespace)
/ # netstat -tlnp
/ # tcpdump -i any port 8080
```

### Step 2: 노드 디버깅

```bash
# 노드 쉘 접근
kubectl debug node/worker-1 -it --image=ubuntu

# 노드 파일시스템 마운트됨
root@node:/# ls /host  # 노드 루트

# 노드의 Docker 소켓 사용
root@node:/# docker -H unix:///host/var/run/docker.sock ps

# 노드 리소스 확인
root@node:/# cat /host/proc/meminfo
root@node:/# df -h /host
```

---

## 🔧 실습 5: 파일 및 로그 분석

### Step 1: 파일 복사

```bash
# 컨테이너 → 호스트
docker cp myapp:/var/log/app.log ./app.log

# 호스트 → 컨테이너
docker cp ./config.yaml myapp:/etc/app/config.yaml

# 디렉토리 전체 복사
docker cp myapp:/app/logs ./logs/

# Kubernetes
kubectl cp myapp:/var/log/app.log ./app.log
kubectl cp ./config.yaml myapp:/etc/app/
```

### Step 2: 실시간 로그 분석

```bash
# 실시간 로그 (tail -f)
docker logs -f myapp

# 최근 100줄
docker logs --tail 100 myapp

# 타임스탬프 포함
docker logs -t myapp

# 특정 시간 이후
docker logs --since 2024-01-15T10:00:00 myapp

# 에러만 필터링
docker logs myapp 2>&1 | grep ERROR

# Kubernetes
kubectl logs -f pod/myapp
kubectl logs --previous pod/myapp  # 이전 컨테이너 로그
kubectl logs pod/myapp -c sidecar  # 특정 컨테이너
```

---

## 🔧 실습 6: 프로세스 및 리소스 분석

### Step 1: 프로세스 상태

```bash
# 컨테이너 내부 프로세스
docker exec myapp ps aux

# 프로세스 트리
docker exec myapp ps auxf

# 특정 프로세스 상세
docker exec myapp cat /proc/1/status

# 파일 디스크립터
docker exec myapp ls -la /proc/1/fd

# 환경 변수
docker exec myapp cat /proc/1/environ | tr '\0' '\n'
```

### Step 2: 리소스 사용률

```bash
# 실시간 리소스 모니터링
docker stats myapp

# 한 번만 출력
docker stats --no-stream myapp

# 메모리 상세
docker exec myapp cat /proc/meminfo

# CPU 상세
docker exec myapp cat /proc/cpuinfo

# 디스크 사용
docker exec myapp df -h

# 프로세스별 메모리
docker exec myapp cat /proc/1/smaps
```

---

## 🔧 실습 7: 고급 디버깅 - GDB

### Step 1: GDB로 프로세스 디버깅

```bash
# GDB 설치된 컨테이너 필요
docker run -d --name app --cap-add=SYS_PTRACE myapp

# GDB 실행
docker exec -it app gdb -p 1

# GDB 명령어
(gdb) backtrace      # 스택 추적
(gdb) info threads   # 스레드 목록
(gdb) thread 2       # 스레드 전환
(gdb) print variable # 변수 값 확인
(gdb) continue       # 계속 실행
(gdb) quit           # 종료
```

### Step 2: Core Dump 분석

```bash
# Core dump 활성화
docker run -d --name app \
  --ulimit core=-1 \
  -v /tmp/cores:/cores \
  myapp

# 컨테이너 내부 설정
docker exec app sh -c 'echo "/cores/core.%e.%p" > /proc/sys/kernel/core_pattern'

# 크래시 발생 시 core dump 생성됨
# /tmp/cores/core.myapp.1234

# GDB로 분석
gdb /path/to/binary /tmp/cores/core.myapp.1234
(gdb) backtrace
(gdb) info registers
```

---

## 💡 주요 명령어 정리

```
┌──────────────────────┬────────────────────────────┐
│ 도구                  │ 용도                        │
├──────────────────────┼────────────────────────────┤
│ docker exec          │ 컨테이너 진입                 │
├──────────────────────┼────────────────────────────┤
│ strace               │ 시스템 콜 추적                │
├──────────────────────┼────────────────────────────┤
│ nsenter              │ 네임스페이스 진입              │
├──────────────────────┼────────────────────────────┤
│ kubectl debug        │ 임시 디버깅 컨테이너            │
├──────────────────────┼────────────────────────────┤
│ docker cp            │ 파일 복사                    │
├──────────────────────┼────────────────────────────┤
│ gdb                  │ 프로세스 디버깅                │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. 최소 권한으로 디버깅
2. 프로덕션은 read-only
3. 디버깅 후 정리
4. 보안 고려
5. 로그 우선 확인
```

---

## 🎓 연습 문제

### 문제 1: 컨테이너에 쉘이 없다면?

<details>
<summary>정답 보기</summary>

**방법 1: 디버깅 컨테이너**
```bash
# 같은 네임스페이스 공유
docker run -it --rm \
  --pid=container:myapp \
  --net=container:myapp \
  busybox sh
```

**방법 2: nsenter**
```bash
PID=$(docker inspect -f '{{.State.Pid}}' myapp)
sudo nsenter -t $PID -m -p /bin/sh
```

**방법 3: 파일 복사**
```bash
# 로그나 설정 파일만 복사
docker cp myapp:/app/log.txt ./
```

**방법 4: Docker commit**
```bash
# 디버깅 도구 추가한 새 이미지
docker commit myapp myapp:debug
docker run -it myapp:debug /bin/sh
```

</details>

### 문제 2: 프로세스가 hang된 원인을 찾으려면?

<details>
<summary>정답 보기</summary>

**1. strace로 시스템 콜 확인**
```bash
docker exec myapp strace -p 1

# Hang 중이면:
# futex(...) = ?  # 대기 중
# read(3,  # I/O 대기
```

**2. 스택 추적**
```bash
docker exec myapp kill -QUIT 1  # SIGQUIT
docker logs myapp  # 스택 출력 (Java, Go)
```

**3. GDB 사용**
```bash
docker exec -it myapp gdb -p 1
(gdb) thread apply all bt  # 모든 스레드 스택
```

**4. /proc 확인**
```bash
docker exec myapp cat /proc/1/stack
docker exec myapp cat /proc/1/wchan  # 대기 채널
```

**일반적 원인:**
- Deadlock
- 네트워크 I/O 대기
- 파일 I/O 대기 (NFS)
- Mutex 대기

</details>

### 문제 3: 메모리 사용량이 계속 증가한다면?

<details>
<summary>정답 보기</summary>

**1. 메모리 사용 확인**
```bash
# 실시간 모니터링
docker stats myapp

# 상세 정보
docker exec myapp cat /proc/1/smaps | grep -A 15 heap
```

**2. 메모리 프로파일링**
```bash
# Go: pprof
docker exec myapp curl http://localhost:6060/debug/pprof/heap > heap.prof

# Python: memory_profiler
docker exec myapp python -m memory_profiler script.py

# Node.js: heapdump
docker exec myapp kill -USR2 $(pidof node)
```

**3. 객체 수 확인**
```bash
# Java: jmap
docker exec myapp jmap -histo 1

# Python: objgraph
import objgraph
objgraph.show_most_common_types()
```

**4. 시스템 툴**
```bash
# pmap
docker exec myapp pmap 1

# /proc/meminfo
docker exec myapp cat /proc/meminfo
```

</details>

---

## 📌 핵심 요약

```
Debugging Techniques 핵심:
1. docker exec (기본 진입)
2. strace (시스템 콜 추적)
3. nsenter (네임스페이스 진입)
4. kubectl debug (K8s)
5. 파일 복사 및 분석

Best Practices:
✅ 로그 우선 확인
✅ 최소 권한 사용
✅ 프로덕션은 read-only
✅ 디버깅 후 정리
✅ 재현 가능한 환경
```

---

## 📚 참고 자료

- [strace Manual](https://man7.org/linux/man-pages/man1/strace.1.html)
- [nsenter Manual](https://man7.org/linux/man-pages/man1/nsenter.1.html)
- [Kubernetes Debugging](https://kubernetes.io/docs/tasks/debug/)

---

## 🤔 생각해볼 문제

1. 프로덕션 환경에서 strace를 사용해도 되는가?
2. 디버깅 컨테이너를 사이드카로 항상 실행하는 것은?
3. 컨테이너 재시작 시 디버깅 정보는?

> 💡 **답변**:
> 
> **1) 프로덕션 strace:**
> 
> ```
> ⚠️ 주의사항:
> - 성능 영향: 30-50% 느려짐
> - CPU 사용 증가
> 
> 권장:
> - 짧은 시간만 (1-2분)
> - 특정 시스템 콜만
>   strace -e trace=open,network -p PID
> - Off-peak 시간
> - 복제본이 여러 개일 때
> 
> 대안:
> - eBPF (성능 영향 적음)
> - 메트릭 + 로그 먼저
> ```
> 
> **2) 디버깅 사이드카:**
> 
> ```
> ❌ 항상 실행: 비효율
> - 리소스 낭비
> - 공격 표면 증가
> 
> ✅ 필요 시만:
> - Ephemeral Containers
> - kubectl debug
> - 일시적 배포
> ```
> 
> **3) 재시작 시:**
> 
> ```
> 문제: 정보 손실
> - 메모리 내용
> - 실행 중인 프로세스
> - 네트워크 연결
> 
> 보존 방법:
> 1. 영구 볼륨
>    - 로그 파일
>    - Core dump
> 
> 2. 외부 수집
>    - Prometheus 메트릭
>    - 중앙 로그 (ELK)
> 
> 3. --restart=no
>    - 일시적으로 재시작 방지
> ```

---

<div align="center">

**[다음: Log Analysis ➡️](02-Log-Analysis.md)**

</div>
