# 05. Common Problems - 자주 발생하는 문제들

## 🎯 이 챕터에서 배울 것

- **컨테이너 시작 실패**: CrashLoopBackOff, ImagePullBackOff
- **권한 문제**: Permission Denied
- **네트워크 문제**: DNS, 연결 실패
- **리소스 부족**: OOM, Disk Full
- **설정 문제**: 환경 변수, 볼륨
- **실전 해결책**: 빠른 문제 해결 가이드

## 📌 왜 중요한가?

**"같은 문제가 반복됩니다. 패턴을 알면 빠르게 해결할 수 있습니다."**

```
Common Problems Top 10:
┌─────────────────────────────────────────────────┐
│ 1. ImagePullBackOff (이미지 Pull 실패)             │
│    - 원인: 잘못된 이미지명, 권한, 레지스트리             │
│    - 해결: 이미지명 확인, 로그인                      │
│                                                 │
│ 2. CrashLoopBackOff (반복 재시작)                  │
│    - 원인: 애플리케이션 크래시                        │
│    - 해결: 로그 확인, 설정 점검                       │
│                                                 │
│ 3. Permission Denied (권한 거부)                  │
│    - 원인: 파일 권한, SELinux                      │
│    - 해결: chmod, chown, 사용자 변경                │
│                                                 │
│ 4. Connection Refused (연결 거부)                 │
│    - 원인: 서비스 미시작, 잘못된 포트                  │
│    - 해결: 서비스 확인, 포트 확인                     │
│                                                 │
│ 5. DNS Resolution Failed (DNS 실패)              │
│    - 원인: DNS 설정, 네트워크 분리                   │
│    - 해결: resolv.conf, 네트워크 확인               │
│                                                 │
│ 6. OOMKilled (메모리 부족)                        │
│    - 원인: 메모리 제한 초과                         │
│    - 해결: 메모리 증가, 메모리 누수 수정               │
│                                                │
│ 7. No Space Left (디스크 부족)                    │
│    - 원인: 로그, 이미지 누적                        │
│    - 해결: 정리, 로그 로테이션                       │
│                                                │
│ 8. Volume Mount Failed (볼륨 마운트 실패)          │
│    - 원인: 경로 없음, 권한                          │
│    - 해결: 경로 생성, 권한 수정                      │
│                                                 │
│ 9. Port Already in Use (포트 충돌)                │
│    - 원인: 포트 중복 사용                           │
│    - 해결: 포트 변경, 기존 프로세스 종료                │
│                                                 │
│ 10. Exec Format Error (아키텍처 불일치)             │
│     - 원인: ARM vs AMD64                         │
│     - 해결: 올바른 플랫폼 이미지                      │
└─────────────────────────────────────────────────┘

Problem-Solving Flow:
┌─────────────────────────────────────────────────┐
│ 1. 증상 확인                                      │
│    "컨테이너가 시작 안 돼요"                          │
│    ↓                                            │
│ 2. 상태 확인                                      │
│    docker ps -a                                 │
│    kubectl get pods                             │
│    ↓                                            │
│ 3. 로그 확인                                      │
│    docker logs                                  │
│    kubectl logs                                 │
│    ↓                                            │
│ 4. 이벤트 확인                                     │
│    kubectl describe pod                         │
│    ↓                                            │
│ 5. 패턴 인식                                      │
│    "아, ImagePullBackOff구나!"                    │
│    ↓                                            │
│ 6. 해결책 적용                                     │
│    이미지명 수정 → 재배포                            │
│    ↓                                            │
│ 7. 검증                                          │
│    정상 동작 확인                                  │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **다운타임 최소화**: 빠른 문제 인식
- **생산성**: 반복 작업 자동화
- **전문성**: 패턴 기반 해결
- **예방**: 사전 점검

---

## 🔬 Deep Dive

### 1. ImagePullBackOff

#### 증상

```bash
# Pod 상태
kubectl get pods
# NAME      READY   STATUS             RESTARTS
# myapp-1   0/1     ImagePullBackOff   0

# 이벤트
kubectl describe pod myapp-1
# Events:
#   Failed to pull image "myapp:v1.0.0": rpc error: code = Unknown desc = Error response from daemon: pull access denied
```

#### 원인 및 해결

```bash
# 원인 1: 이미지 이름 오타
# ❌ myap:v1.0.0
# ✅ myapp:v1.0.0

# 원인 2: 태그 없음
docker pull myregistry.io/myapp:v1.0.0
# Error: manifest unknown

# 해결: 태그 확인
docker images | grep myapp

# 원인 3: Private Registry 권한
# 해결: Docker login
docker login myregistry.io

# Kubernetes Secret 생성
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.io \
  --docker-username=user \
  --docker-password=pass

# Deployment에 추가
spec:
  imagePullSecrets:
  - name: regcred
```

---

### 2. CrashLoopBackOff

#### 증상

```bash
kubectl get pods
# NAME      READY   STATUS              RESTARTS
# myapp-1   0/1     CrashLoopBackOff    5

# 재시작 간격이 점점 길어짐
# 0s → 10s → 20s → 40s → 80s → ...
```

#### 디버깅

```bash
# 로그 확인
kubectl logs myapp-1

# 이전 컨테이너 로그
kubectl logs --previous myapp-1

# 종료 코드 확인
kubectl describe pod myapp-1
# Last State: Terminated
#   Reason: Error
#   Exit Code: 1
```

#### 일반적 원인

```bash
# 1. 애플리케이션 크래시
# 로그: panic, exception, segfault

# 2. 설정 파일 없음
# Error: Config file not found

# 3. 환경 변수 누락
# Error: DATABASE_URL not set

# 4. 헬스체크 실패
# Liveness probe failed

# 5. 권한 문제
# Permission denied
```

---

## 🔧 실습 1: 컨테이너 시작 실패 해결

### 시나리오 1: Missing Config File

```bash
# 증상
docker logs myapp
# Error: /etc/app/config.yaml: No such file or directory

# 해결: ConfigMap 마운트
kubectl create configmap app-config --from-file=config.yaml

# Deployment
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config
      mountPath: /etc/app
  volumes:
  - name: config
    configMap:
      name: app-config
```

### 시나리오 2: Wrong Entry Point

```dockerfile
# Dockerfile
FROM alpine
COPY app.sh /
RUN chmod +x /app.sh
CMD ["/app.sh"]
```

```bash
# 실행 실패
docker logs myapp
# /bin/sh: /app.sh: not found

# 원인: CR/LF 문제 (Windows)
# 해결
dos2unix app.sh

# 또는 ENTRYPOINT 사용
ENTRYPOINT ["/bin/sh", "/app.sh"]
```

---

## 🔧 실습 2: 권한 문제 해결

### 시나리오 1: Permission Denied

```bash
# 증상
docker logs myapp
# Error: Permission denied: '/app/data/db.sqlite'

# 확인
docker exec myapp ls -la /app/data
# -rw-r--r-- 1 root root db.sqlite

# 원인: 컨테이너는 non-root로 실행
docker exec myapp whoami
# appuser (uid: 1000)

# 해결 1: 소유자 변경
docker exec myapp chown -R 1000:1000 /app/data

# 해결 2: Dockerfile에서
RUN mkdir -p /app/data && \
    chown -R 1000:1000 /app/data

USER 1000

# 해결 3: initContainer (Kubernetes)
initContainers:
- name: fix-permissions
  image: busybox
  command: ['sh', '-c', 'chown -R 1000:1000 /data']
  volumeMounts:
  - name: data
    mountPath: /data
```

### 시나리오 2: SELinux

```bash
# 증상 (볼륨 마운트)
docker run -v /data:/app/data myapp
# Permission denied

# 확인: SELinux
getenforce
# Enforcing

# 해결: :z 또는 :Z 옵션
docker run -v /data:/app/data:z myapp

# :z  - 공유 (shared)
# :Z  - 전용 (private)
```

---

## 🔧 실습 3: 네트워크 문제 해결

### 시나리오 1: Service Name Resolution

```bash
# 증상
docker exec app curl http://database:5432
# Could not resolve host: database

# 원인 1: 다른 네트워크
docker network inspect bridge
# app: 있음
# database: 없음!

# 해결
docker network create mynetwork
docker network connect mynetwork app
docker network connect mynetwork database

# 또는 docker-compose
services:
  app:
    networks:
      - mynetwork
  database:
    networks:
      - mynetwork

networks:
  mynetwork:
```

### 시나리오 2: DNS 문제

```bash
# 증상
docker exec app curl https://api.github.com
# Could not resolve host: api.github.com

# 확인
docker exec app cat /etc/resolv.conf
# nameserver 127.0.0.1  # 잘못된 DNS

# 해결: Docker daemon.json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}

sudo systemctl restart docker

# 또는 컨테이너별
docker run --dns 8.8.8.8 myapp
```

---

## 🔧 실습 4: 리소스 부족 해결

### 시나리오 1: OOMKilled

```bash
# 증상
docker inspect myapp | jq '.[0].State'
# "OOMKilled": true

kubectl describe pod myapp
# Last State: Terminated
#   Reason: OOMKilled

# 원인 확인
docker stats myapp --no-stream
# CONTAINER  MEM USAGE / LIMIT
# myapp      512MB / 512MB  # 제한 도달!

# 해결 1: 메모리 증가
docker run --memory=1g myapp

# Kubernetes
resources:
  limits:
    memory: "1Gi"

# 해결 2: 메모리 누수 수정
# (04-Performance-Issues.md 참고)
```

### 시나리오 2: Disk Full

```bash
# 증상
docker build -t myapp .
# ERROR: failed to solve: write /var/lib/docker/...: no space left on device

# 확인
df -h
# /var/lib/docker: 100% (디스크 가득!)

# 해결 1: 미사용 리소스 정리
docker system prune -a

# 해결 2: 이미지 정리
docker images | grep '<none>' | awk '{print $3}' | xargs docker rmi

# 해결 3: 볼륨 정리
docker volume prune

# 해결 4: 로그 정리
truncate -s 0 $(docker inspect --format='{{.LogPath}}' myapp)
```

---

## 🔧 실습 5: 설정 문제 해결

### 시나리오 1: 환경 변수 누락

```bash
# 증상
docker logs myapp
# Error: DATABASE_URL environment variable not set

# 확인
docker exec myapp printenv | grep DATABASE

# 해결: 환경 변수 추가
docker run -e DATABASE_URL=postgres://... myapp

# Kubernetes
env:
- name: DATABASE_URL
  value: "postgres://..."

# 또는 ConfigMap에서
env:
- name: DATABASE_URL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: database.url
```

### 시나리오 2: 볼륨 마운트 실패

```bash
# 증상
docker run -v /data:/app/data myapp
# docker: Error response from daemon: invalid mount config

# 원인 1: 경로 없음
ls -la /data
# No such file or directory

# 해결
sudo mkdir -p /data
docker run -v /data:/app/data myapp

# 원인 2: 상대 경로 (Windows)
# ❌ docker run -v .\data:/app/data
# ✅ docker run -v ${PWD}/data:/app/data
```

---

## 🔧 실습 6: 포트 및 호스트 문제

### 시나리오 1: Port Already in Use

```bash
# 증상
docker run -p 8080:8080 myapp
# Error: Bind for 0.0.0.0:8080 failed: port is already allocated

# 확인
sudo netstat -tlnp | grep 8080
# tcp  0.0.0.0:8080  LISTEN  1234/docker-proxy

# 해결 1: 다른 포트 사용
docker run -p 8081:8080 myapp

# 해결 2: 기존 컨테이너 종료
docker ps | grep 8080
docker stop <container-id>

# 해결 3: 호스트 프로세스 종료
sudo kill 1234
```

### 시나리오 2: Bind to 127.0.0.1

```python
# 문제
app.run(host='127.0.0.1', port=8080)

# 외부에서 접근 불가
curl http://localhost:8080
# Connection refused (컨테이너 외부에서)

# 해결
app.run(host='0.0.0.0', port=8080)
```

---

## 🔧 실습 7: 플랫폼 및 아키텍처 문제

### 시나리오 1: Exec Format Error

```bash
# 증상
docker run myapp
# exec /app/main: exec format error

# 원인: 아키텍처 불일치
uname -m
# aarch64 (ARM)

docker inspect myapp | jq '.[0].Architecture'
# "amd64"

# 해결 1: 올바른 플랫폼 이미지 사용
docker pull --platform linux/arm64 myapp

# 해결 2: Multi-platform 빌드
docker buildx build --platform linux/amd64,linux/arm64 -t myapp .

# 해결 3: QEMU (에뮬레이션)
docker run --platform linux/amd64 myapp
# 느리지만 작동
```

---

## 💡 빠른 문제 해결 체크리스트

```
┌──────────────────────────────────────────────────┐
│ 컨테이너가 시작 안 될 때:                              │
│                                                  │
│ □ docker ps -a (상태 확인)                         │
│ □ docker logs (로그 확인)                          │
│ □ docker inspect (상세 정보)                       │
│ □ 이미지 존재? (docker images)                      │
│ □ 포트 충돌? (netstat -tlnp)                       │
│ □ 볼륨 경로? (ls -la)                              │
│ □ 환경 변수? (docker exec printenv)                │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ Kubernetes Pod 문제:                              │
│                                                  │
│ □ kubectl get pods (상태)                         │
│ □ kubectl describe pod (이벤트)                    │
│ □ kubectl logs (로그)                             │
│ □ kubectl logs --previous (이전)                  │
│ □ kubectl get events (클러스터 이벤트)               │
└──────────────────────────────────────────────────┘
```

---

## 🎓 연습 문제

### 문제 1: "It works on my machine"

<details>
<summary>정답 보기</summary>

**환경 차이 원인:**

**1. 환경 변수**
```bash
# 로컬: .env 파일
# 프로덕션: 없음!

# 해결: 명시적 주입
docker run -e DATABASE_URL=... myapp
```

**2. 볼륨/파일**
```bash
# 로컬: 로컬 파일 마운트
# 프로덕션: 파일 없음

# 해결: 이미지에 포함 또는 ConfigMap
```

**3. 의존성**
```dockerfile
# 로컬: 캐시된 npm packages
# 프로덕션: 네트워크 문제

# 해결: Multi-stage build
FROM node:18 AS builder
RUN npm ci --only=production
```

**4. 플랫폼**
```bash
# 로컬: macOS (ARM)
# 프로덕션: Linux (AMD64)

# 해결: Multi-platform
docker buildx build --platform linux/amd64
```

</details>

### 문제 2: 간헐적 실패

<details>
<summary>정답 보기</summary>

**원인 탐색:**

**1. 경쟁 조건**
```python
# 동시 접근
if not exists(file):
    create(file)  # 동시에 2번 실행 → 에러
```

**2. 리소스 고갈**
```bash
# 연결 풀 부족
# 10개 중 10개 모두 사용 중
# 새 요청 → 대기 → 타임아웃
```

**3. 네트워크 타임아웃**
```bash
# 간헐적 네트워크 지연
ping -c 1000 api | grep loss
# 1% packet loss
```

**해결:**
```python
# Retry 로직
@retry(stop=stop_after_attempt(3), wait=wait_fixed(1))
def api_call():
    ...

# Connection Pool
pool = HTTPConnectionPool(maxsize=50)

# Circuit Breaker
@circuit_breaker(fail_max=5, timeout=60)
def external_api():
    ...
```

</details>

### 문제 3: 프로덕션에서만 발생하는 문제

<details>
<summary>정답 보기</summary>

**차이점 찾기:**

**1. 스케일**
```
로컬: 1 user
프로덕션: 1000 users/sec
→ 동시성 문제
```

**2. 데이터 크기**
```
로컬: 100 rows
프로덕션: 10M rows
→ 쿼리 타임아웃
```

**3. 환경**
```
로컬: 개발 모드 (DEBUG=true)
프로덕션: 프로덕션 모드
→ 다른 코드 경로
```

**디버깅:**
```bash
# 프로덕션 데이터 복사 (익명화)
pg_dump --data-only production | anonymize.py > local.sql

# 프로덕션 환경 복제
docker-compose -f staging.yml up

# 부하 테스트
k6 run load-test.js
```

</details>

---

## 📌 핵심 요약

```
Common Problems 핵심:
1. 패턴 인식 (ImagePullBackOff 등)
2. 로그 우선 확인
3. 체계적 디버깅
4. 근본 원인 찾기
5. 재발 방지

Best Practices:
✅ 로그 먼저
✅ 상태 확인 (docker ps, kubectl get)
✅ 이벤트 확인 (kubectl describe)
✅ 환경 차이 점검
✅ 재현 가능하게
```

---

## 📚 참고 자료

- [Docker Troubleshooting](https://docs.docker.com/config/daemon/)
- [Kubernetes Debugging](https://kubernetes.io/docs/tasks/debug/)
- [Common Errors](https://kubernetes.io/docs/tasks/debug/debug-application/)

---

## 🤔 생각해볼 문제

1. 모든 에러를 미리 예방할 수 있는가?
2. 에러 메시지를 어떻게 개선하는가?
3. 사용자에게 에러를 어떻게 보여주는가?

> 💡 **답변**:
> 
> **1) 에러 예방:**
> 
> ```
> ✅ 가능:
> - Validation (입력 검증)
> - Health Check
> - Resource Limits
> - 테스트
> 
> ❌ 불가능:
> - 외부 서비스 장애
> - 네트워크 문제
> - 하드웨어 실패
> 
> 전략: 예방 + 빠른 복구
> ```
> 
> **2) 좋은 에러 메시지:**
> 
> ```python
> # ❌ 나쁜 예
> raise Exception("Error")
> 
> # ✅ 좋은 예
> raise ValueError(
>     "Invalid email format: 'user@'. "
>     "Expected format: user@domain.com"
> )
> 
> # ✅ 액션 포함
> raise ConnectionError(
>     "Failed to connect to database at postgres:5432. "
>     "Check if database is running: "
>     "docker ps | grep postgres"
> )
> ```
> 
> **3) 사용자 친화적 에러:**
> 
> ```json
> // ❌ 개발자용
> {
>   "error": "NullPointerException at line 42"
> }
> 
> // ✅ 사용자용
> {
>   "error": "죄송합니다. 일시적인 문제가 발생했습니다.",
>   "message": "잠시 후 다시 시도해주세요.",
>   "support": "문제가 계속되면 support@example.com",
>   "error_id": "ERR-2024-001-ABC"  // 추적용
> }
> ```

---

<div align="center">

**[⬅️ 이전: Performance Issues](./04-Performance-Issues.md)** | **[다음: Diagnostic Tools ➡️](./06-Diagnostic-Tools.md)**

</div>
