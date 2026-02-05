<div align="center">

# 🐳 Docker Deep Dive

**단순한 Docker 튜토리얼이 아닌, Docker 내부까지 완전히 이해하는 심화 학습 자료**

<br/>

> *"컨테이너를 실행하는 것을 넘어, Docker가 실제로 어떻게 동작하는지 완전히 이해하기"*

Namespaces, Cgroups, Union Filesystem부터 네트워킹, 보안, 성능까지  
**왜 그렇게 동작하는지** 원리부터 파헤치는 Docker 완전 정복 가이드

<br/>

[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)](https://www.kernel.org)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 프로젝트에 대하여

Docker를 "그냥 쓰는 것"에서 "**완전히 이해하는 것**"으로 도약하기 위한 심화 학습 자료입니다.

### ✨ 특징

| 🔬 **원리 중심** | 💻 **실습 중심** | 🔥 **실전 사례** | 🏗️ **아키텍처 이해** |
|:---:|:---:|:---:|:---:|
| "왜?"라는 질문에<br/>명확한 답 | 모든 개념은<br/>직접 실행 가능 | 실무에서<br/>바로 쓰는 패턴 | 내부 구조까지<br/>완벽 파악 |

- ✅ **80+ 심화 주제** - Fundamentals부터 고급 주제까지 완전 커버
- ✅ **실행 가능한 예제** - 모든 명령어와 코드를 직접 실행 가능
- ✅ **원리 기반 학습** - 단순 사용법이 아닌 동작 원리 이해
- ✅ **실전 시나리오** - 프로덕션 환경의 실제 문제 해결
- ✅ **Before/After 비교** - 최적화 전후를 명확히 비교
- ✅ **성능 벤치마크** - 실제 성능 측정 결과 포함

---

## 📚 목차

> 💡 **각 챕터를 클릭하면 상세한 학습 문서로 이동합니다**

### 🔹 Fundamentals - 핵심 기초

<details>
<summary><b>Docker의 근본 원리를 완전히 이해하기 (7개 챕터)</b></summary>

<br/>

|                            주제                             | 핵심 내용 | 난이도 |
|:-----------------------------------------------------------:|----------|--------|
|       **[01. Container vs VM](./fundamentals/01-Container-vs-VM.md)**        | 컨테이너와 VM의 근본적 차이, 성능 비교 실험 | ⭐ |
|   **[02. Docker Architecture](./fundamentals/02-Docker-Architecture.md)**   | dockerd, containerd, runc 구조, 컴포넌트 통신 흐름 | ⭐⭐ |
|         **[03. Image Layers](./fundamentals/03-Image-Layers.md)**          | 레이어 시스템, Copy-on-Write, 캐싱 전략 | ⭐⭐ |
|       **[04. Union Filesystem](./fundamentals/04-Union-Filesystem.md)**        | OverlayFS 동작 원리, 스토리지 드라이버 비교 | ⭐⭐⭐ |
|      **[05. Namespaces](./fundamentals/05-Namespaces.md)**      | 7가지 Namespace, 격리 메커니즘, 실전 활용 | ⭐⭐⭐ |
|        **[06. Cgroups](./fundamentals/06-Cgroups.md)**       | CPU/메모리/I/O 제한, OOM Killer, 리소스 관리 | ⭐⭐⭐ |
| **[07. Docker Engine](./fundamentals/07-Docker-Engine.md)** | 이벤트 시스템, 플러그인, Engine API | ⭐⭐⭐ |

</details>

<br/>

### 🔹 Images - 이미지 심화

<details>
<summary><b>효율적이고 안전한 이미지 빌드 (7개 챕터)</b></summary>

<br/>

|                      주제                      | 핵심 내용 | 학습 포인트 |
|:----------------------------------------------:|----------|-----------|
|   **[01. Dockerfile Best Practices](./images/01-Dockerfile-Best-Practices.md)**   | 레이어 최적화, 빌드 컨텍스트, 캐시 활용 | 빌드 시간 10배 단축 |
| **[02. Multi-Stage Builds](./images/02-Multi-Stage-Builds.md)** | 빌드/실행 분리, 이미지 크기 최소화 | 크기 5배 감소 |
| **[03. Image Optimization](./images/03-Image-Optimization.md)** | Alpine vs Distroless, 불필요한 파일 제거 | 프로덕션 최적화 |
| **[04. Cache Mechanism](./images/04-Cache-Mechanism.md)** | 빌드 캐시 동작, 무효화 조건, 원격 캐시 | CI/CD 가속화 |
|    **[05. BuildKit](./images/05-BuildKit.md)**    | 병렬 빌드, Secrets, SSH 마운트 | 차세대 빌드 엔진 |
| **[06. Image Security](./images/06-Image-Security.md)** | 취약점 스캔, 서명, 최소 권한 | 보안 강화 |
| **[07. Custom Base Images](./images/07-Custom-Base-Images.md)** | scratch부터 시작, 맞춤형 베이스 제작 | 완전 제어 |

</details>

<br/>

### 🔹 Networking - 네트워킹 완전 정복

<details>
<summary><b>컨테이너 네트워킹의 모든 것 (9개 챕터)</b></summary>

<br/>

|                       주제                        | 핵심 내용 |
|:-------------------------------------------------:|----------|
|   **[01. Network Fundamentals](./networking/01-Network-Fundamentals.md)**   | veth pair, bridge, iptables, 패킷 흐름 |
| **[02. Bridge Network](./networking/02-Bridge-Network.md)** | 기본 네트워크, 사용자 정의 bridge, DNS |
| **[03. Host Network](./networking/03-Host-Network.md)** | Host 모드, 성능 특성, 사용 시나리오 |
| **[04. Overlay Network](./networking/04-Overlay-Network.md)** | 멀티 호스트 네트워킹, VXLAN, Swarm |
| **[05. Macvlan Network](./networking/05-Macvlan-Network.md)** | 물리 네트워크 통합, VLAN 태깅 |
| **[06. Custom Networks](./networking/06-Custom-Networks.md)** | CNI 플러그인, Calico, Weave |
| **[07. DNS Resolution](./networking/07-DNS-Resolution.md)** | 내부 DNS, 서비스 디스커버리 |
| **[08. Load Balancing](./networking/08-Load-Balancing.md)** | 내장 LB, 헬스 체크, 외부 연동 |
| **[09. Network Security](./networking/09-Network-Security.md)** | 네트워크 정책, 방화벽, 암호화 |

</details>
<br/>

### 🔹 Storage - 스토리지 & 데이터 관리

<details>
<summary><b>영속적 데이터 관리 전략 (7개 챕터)</b></summary>

<br/>

|                    주제                     | 핵심 내용 |
|:-------------------------------------------:|----------|
|   **[01. Volume Types](./storage/01-Volume-Types.md)**   | Named Volume, Anonymous Volume 비교 |
| **[02. Bind Mounts](./storage/02-Bind-Mounts.md)** | 호스트 디렉토리 마운트, 개발 환경 |
| **[03. Tmpfs Mounts](./storage/03-Tmpfs-Mounts.md)** | 메모리 기반 스토리지, 임시 데이터 |
| **[04. Volume Drivers](./storage/04-Volume-Drivers.md)** | NFS, GlusterFS, Ceph 통합 |
| **[05. Storage Drivers](./storage/05-Storage-Drivers.md)** | overlay2, btrfs, zfs 상세 비교 |
| **[06. Data Persistence](./storage/06-Data-Persistence.md)** | 데이터베이스 영속성 전략 |
| **[07. Backup & Restore](./storage/07-Backup-Restore.md)** | 백업 자동화, 재해 복구 |

</details>
<br/>

### 🔹 Orchestration - 오케스트레이션

<details>
<summary><b>컨테이너 편성과 관리 (7개 챕터)</b></summary>

<br/>

|                        주제                         | 핵심 내용 |
|:---------------------------------------------------:|----------|
|   **[01. Docker Compose](./orchestration/01-Docker-Compose.md)**   | 멀티 컨테이너 앱, YAML 작성법 |
| **[02. Compose Advanced](./orchestration/02-Compose-Advanced.md)** | extends, profiles, 환경 분리 |
| **[03. Docker Swarm](./orchestration/03-Docker-Swarm.md)** | 클러스터 구성, 매니저/워커 노드 |
| **[04. Swarm Services](./orchestration/04-Swarm-Services.md)** | 서비스 배포, 레플리카, 제약 조건 |
| **[05. Swarm Networking](./orchestration/05-Swarm-Networking.md)** | Ingress 네트워크, 서비스 메시 |
| **[06. Rolling Updates](./orchestration/06-Rolling-Updates.md)** | 무중단 배포, 롤백 전략 |
| **[07. High Availability](./orchestration/07-High-Availability.md)** | 고가용성 아키텍처, 장애 복구 |

</details>
<br/>

### 🔹 Security - 보안 강화

<details>
<summary><b>프로덕션 보안 베스트 프랙티스 (8개 챕터)</b></summary>

<br/>

|                          주제                           | 핵심 내용 |
|:-------------------------------------------------------:|----------|
|   **[01. Security Principles](./security/01-Security-Principles.md)**   | 최소 권한, 심층 방어, 공격 표면 |
| **[02. Image Scanning](./security/02-Image-Scanning.md)** | Trivy, Clair, Anchore 활용 |
| **[03. Runtime Security](./security/03-Runtime-Security.md)** | Seccomp, AppArmor, Capabilities |
| **[04. Secrets Management](./security/04-Secrets-Management.md)** | Docker Secrets, Vault 통합 |
| **[05. AppArmor & SELinux](./security/05-AppArmor-SELinux.md)** | MAC 시스템, 프로파일 작성 |
| **[06. User Namespaces](./security/06-User-Namespaces.md)** | UID 재매핑, rootless 컨테이너 |
| **[07. Security Scanning Tools](./security/07-Security-Scanning-Tools.md)** | 자동화된 보안 스캔 파이프라인 |
| **[08. Compliance](./security/08-Compliance.md)** | CIS 벤치마크, PCI-DSS, HIPAA |

</details>
<br/>

### 🔹 Performance - 성능 최적화

<details>
<summary><b>컨테이너 성능 극대화 (8개 챕터)</b></summary>

<br/>

|                       주제                        | 핵심 내용 |
|:-------------------------------------------------:|----------|
|   **[01. Resource Limits](./performance/01-Resource-Limits.md)**   | CPU/메모리 제한 전략 |
| **[02. CPU Management](./performance/02-CPU-Management.md)** | CPU pinning, NUMA 인식 |
| **[03. Memory Management](./performance/03-Memory-Management.md)** | 메모리 누수 탐지, OOM 대응 |
| **[04. I/O Performance](./performance/04-IO-Performance.md)** | 디스크 I/O 최적화, 벤치마킹 |
| **[05. Monitoring](./performance/05-Monitoring.md)** | Prometheus, cAdvisor 통합 |
| **[06. Logging](./performance/06-Logging.md)** | 중앙화 로깅, ELK 스택 |
| **[07. Profiling](./performance/07-Profiling.md)** | 애플리케이션 프로파일링 |
| **[08. Benchmarking](./performance/08-Benchmarking.md)** | 성능 테스트 방법론 |

</details>
<br/>

### 🔹 Advanced - 고급 주제

<details>
<summary><b>Docker 내부 깊숙이 (8개 챕터)</b></summary>

<br/>

|                     주제                      | 핵심 내용 |
|:---------------------------------------------:|----------|
|   **[01. Container Runtime](./advanced/01-Container-Runtime.md)**   | OCI Runtime Spec 상세 |
| **[02. OCI Specification](./advanced/02-OCI-Specification.md)** | Image/Runtime Spec 분석 |
| **[03. containerd](./advanced/03-containerd.md)** | containerd 독립 사용 |
| **[04. runc](./advanced/04-runc.md)** | runc 직접 제어 |
| **[05. Docker API](./advanced/05-Docker-API.md)** | REST API 완전 정복 |
| **[06. Docker SDK](./advanced/06-Docker-SDK.md)** | Python/Go SDK 활용 |
| **[07. Custom Plugins](./advanced/07-Custom-Plugins.md)** | 플러그인 개발 가이드 |
| **[08. Docker Extensions](./advanced/08-Docker-Extensions.md)** | Docker Desktop 확장 |

</details>
<br/>

### 🔹 Patterns - 실전 패턴

<details>
<summary><b>프로덕션 검증된 디자인 패턴 (8개 챕터)</b></summary>

<br/>

|                         주제                          | 핵심 내용 |
|:-----------------------------------------------------:|----------|
|   **[01. Microservices](./patterns/01-Microservices.md)**   | 마이크로서비스 아키텍처 |
| **[02. Sidecar Pattern](./patterns/02-Sidecar-Pattern.md)** | 사이드카 컨테이너 활용 |
| **[03. Ambassador Pattern](./patterns/03-Ambassador-Pattern.md)** | 프록시 패턴 구현 |
| **[04. Adapter Pattern](./patterns/04-Adapter-Pattern.md)** | 레거시 시스템 통합 |
| **[05. Init Containers](./patterns/05-Init-Containers.md)** | 초기화 컨테이너 패턴 |
| **[06. Health Checks](./patterns/06-Health-Checks.md)** | 헬스 체크 전략 |
| **[07. Graceful Shutdown](./patterns/07-Graceful-Shutdown.md)** | 우아한 종료 처리 |
| **[08. Configuration Management](./patterns/08-Configuration-Management.md)** | 설정 관리 베스트 프랙티스 |

</details>
<br/>

### 🔹 CI/CD - 지속적 통합/배포

<details>
<summary><b>Docker 기반 CI/CD 파이프라인 (7개 챕터)</b></summary>

<br/>

|                        주제                         | 핵심 내용 |
|:---------------------------------------------------:|----------|
|   **[01. Docker in CI](./cicd/01-Docker-in-CI.md)**   | GitHub Actions, GitLab CI |
| **[02. Image Tagging](./cicd/02-Image-Tagging.md)** | 태깅 전략, 버저닝 |
| **[03. Registry Setup](./cicd/03-Registry-Setup.md)** | Private Registry 구축 |
| **[04. Automated Testing](./cicd/04-Automated-Testing.md)** | 컨테이너 기반 테스트 |
| **[05. Security Scanning](./cicd/05-Security-Scanning.md)** | 파이프라인 보안 스캔 |
| **[06. GitOps](./cicd/06-GitOps.md)** | Git 기반 배포 자동화 |
| **[07. Deployment Strategies](./cicd/07-Deployment-Strategies.md)** | Blue/Green, Canary |

</details>
<br/>

### 🔹 Debugging - 디버깅 & 트러블슈팅

<details>
<summary><b>실전 문제 해결 가이드 (6개 챕터)</b></summary>

<br/>

|                         주제                          | 핵심 내용 |
|:-----------------------------------------------------:|----------|
|   **[01. Debugging Techniques](./debugging/01-Debugging-Techniques.md)**   | strace, nsenter, 컨테이너 진입 |
| **[02. Log Analysis](./debugging/02-Log-Analysis.md)** | 로그 분석 기법 |
| **[03. Network Debugging](./debugging/03-Network-Debugging.md)** | tcpdump, 네트워크 추적 |
| **[04. Performance Issues](./debugging/04-Performance-Issues.md)** | 성능 병목 찾기 |
| **[05. Common Problems](./debugging/05-Common-Problems.md)** | 자주 발생하는 문제들 |
| **[06. Diagnostic Tools](./debugging/06-Diagnostic-Tools.md)** | 진단 도구 모음 |

</details>
<br/>

### 🔹 Real World - 실전 프로젝트

<details>
<summary><b>실무 프로젝트 완전 구현 (7개 챕터)</b></summary>

<br/>

|                        주제                         | 핵심 내용 |
|:---------------------------------------------------:|----------|
|   **[01. Web Application](./real-world/01-Web-Application.md)**   | Full-stack 앱 Docker화 |
| **[02. Database Setup](./real-world/02-Database-Setup.md)** | 데이터베이스 컨테이너화 |
| **[03. Reverse Proxy](./real-world/03-Reverse-Proxy.md)** | Nginx/Traefik 설정 |
| **[04. Monitoring Stack](./real-world/04-Monitoring-Stack.md)** | Prometheus + Grafana |
| **[05. Log Aggregation](./real-world/05-Log-Aggregation.md)** | ELK/EFK 스택 구축 |
| **[06. Backup System](./real-world/06-Backup-System.md)** | 자동 백업 시스템 |
| **[07. Multi-Tier App](./real-world/07-Multi-Tier-App.md)** | 다층 아키텍처 구현 |

</details>
<br/>

### 🔹 Kubernetes Bridge - K8s로의 전환

<details>
<summary><b>Docker에서 Kubernetes로 (4개 챕터)</b></summary>

<br/>

|                          주제                           | 핵심 내용 |
|:-------------------------------------------------------:|----------|
|   **[01. Docker to K8s](./kubernetes-bridge/01-Docker-to-K8s.md)**   | 개념 매핑, 차이점 이해 |
| **[02. Pod Concepts](./kubernetes-bridge/02-Pod-Concepts.md)** | Pod vs 컨테이너 |
| **[03. Deployment Patterns](./kubernetes-bridge/03-Deployment-Patterns.md)** | Deployment, StatefulSet |
| **[04. Migration Guide](./kubernetes-bridge/04-Migration-Guide.md)** | 마이그레이션 전략 |

</details>

---

## 🗺️ 학습 로드맵

### 🎯 목적별 학습 경로

<details>
<summary><b>📘 초급자 (Docker 입문)</b></summary>

<br/>

**Week 1-2: Docker 기초 이해**
```
✅ Fundamentals 01-03
   ├─ Container vs VM
   ├─ Docker Architecture
   └─ Image Layers

✅ Images 01-02
   ├─ Dockerfile Best Practices
   └─ Multi-Stage Builds

목표: Docker가 무엇인지, 왜 사용하는지 이해
```

**Week 3-4: 실전 사용**
```
✅ Networking 01-02
   ├─ Network Fundamentals
   └─ Bridge Network

✅ Storage 01-02
   ├─ Volume Types
   └─ Bind Mounts

✅ Orchestration 01
   └─ Docker Compose

목표: 실제 애플리케이션을 Docker로 실행
```

**Week 5-6: 프로젝트 실습**
```
✅ Real World 01-03
   ├─ Web Application
   ├─ Database Setup
   └─ Reverse Proxy

목표: 간단한 웹 앱을 Docker로 완전 배포
```

</details>

<details>
<summary><b>💼 중급자 (실무 활용)</b></summary>

<br/>

**Month 1: 심화 이론**
```
✅ Fundamentals 전체 (01-07)
✅ Images 전체 (01-07)
✅ Networking 01-06

목표: Docker 내부 동작 원리 완전 이해
```

**Month 2: 보안 & 성능**
```
✅ Security 01-06
✅ Performance 01-06
✅ Debugging 01-04

목표: 프로덕션 준비 완료
```

**Month 3: 자동화 & 오케스트레이션**
```
✅ CI/CD 전체
✅ Orchestration 전체
✅ Patterns 01-06

목표: CI/CD 파이프라인 구축
```

</details>

<details>
<summary><b>🏆 고급자 (아키텍트 레벨)</b></summary>

<br/>

**Phase 1: 전체 복습 (2주)**
```
✅ Fundamentals 전체 재학습
✅ 핵심 개념 정리
```

**Phase 2: 고급 주제 (1개월)**
```
✅ Advanced 전체 (01-08)
✅ Performance 전체
✅ Security 전체
```

**Phase 3: 실전 적용 (1개월)**
```
✅ Real World 전체
✅ Patterns 전체
✅ Kubernetes Bridge 전체
```

**Phase 4: 커스터마이징 (진행형)**
```
✅ Custom Plugins 개발
✅ 자체 모니터링 시스템 구축
✅ 오픈소스 기여
```

</details>

<details>
<summary><b>⚡ 빠른 복습 (경력 개발자)</b></summary>

<br/>

**Day 1: Core Concepts**
```
□ Fundamentals 01-04 (4시간)
□ Images 01-03 (2시간)
```

**Day 2: Networking & Security**
```
□ Networking 01-03 (3시간)
□ Security 01-04 (3시간)
```

**Day 3: Advanced & Patterns**
```
□ Advanced 01-04 (3시간)
□ Patterns 01-05 (3시간)
```

</details>

---

## 🎓 학습 방법

```
📖 Read → 💻 Practice → 🤔 Think → 📝 Review → 🔁 Repeat
```

### 1️⃣ 기초부터 차근차근
```
Fundamentals (필수) 
→ Images (이미지 최적화)
→ Networking (네트워크 이해)
→ Storage (데이터 관리)
→ Security (보안 강화)
```

### 2️⃣ 실습 중심 학습
```
1. 문서 읽기 (개념 이해)
2. 명령어 직접 실행 (체득)
3. 결과 분석 (원리 파악)
4. 변형 실험 (응용력 향상)
```

### 3️⃣ 프로젝트 기반 학습
```
이론 학습 → Real World 프로젝트 → 실제 적용
예: Fundamentals 학습 후 → Web Application 프로젝트 구현
```

---

## 📖 문서 구조

각 문서는 다음과 같은 구조로 구성됩니다:

| 섹션 | 설명 |
|------|------|
| 🎯 **학습 목표** | 이 챕터에서 배울 핵심 내용 |
| 📌 **왜 중요한가** | 실무에서의 중요성과 적용 사례 |
| 🔬 **Deep Dive** | 내부 동작 원리 상세 설명 |
| 💻 **실습** | 직접 실행 가능한 예제 |
| 🔥 **실전 적용** | 프로덕션 시나리오 |
| ⚡ **최적화** | 성능/보안 개선 팁 |
| 🚫 **안티패턴** | 피해야 할 실수들 |
| 🎓 **핵심 정리** | 빠른 복습용 요약 |

---

## 💡 학습 팁

### ✅ 효과적인 학습 전략

1. **순서대로 학습**
    - Fundamentals는 반드시 순서대로
    - 다른 섹션은 필요에 따라 선택

2. **실습 필수**
    - 모든 명령어를 직접 실행
    - 결과를 눈으로 확인
    - 변형해서 실험

3. **메모하기**
    - 이해 안 되는 부분 표시
    - 나중에 다시 학습
    - 자신만의 정리 노트 작성

4. **프로젝트 적용**
    - 학습한 내용을 실제 프로젝트에 적용
    - 문제 발생 시 Debugging 섹션 참고

### ⚠️ 주의사항

```
❌ 단순 암기
✅ 원리 이해

❌ 명령어만 복사
✅ 왜 그렇게 동작하는지 이해

❌ 건너뛰기
✅ Fundamentals는 필수

❌ 이론만
✅ 반드시 실습 병행
```

---

## 🔧 실습 환경 구성

### 최소 요구사항

```bash
# OS: Linux (Ubuntu 22.04 권장)
# CPU: 2 cores
# RAM: 4GB
# Disk: 20GB

# Docker 설치
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 사용자 권한 추가
sudo usermod -aG docker $USER
newgrp docker

# 확인
docker --version
docker run hello-world
```

### 권장 환경

```bash
# OS: Linux (Ubuntu 22.04+)
# CPU: 4+ cores
# RAM: 8GB+
# Disk: 50GB+

# Docker Compose 설치
sudo apt-get update
sudo apt-get install docker-compose-plugin

# 확인
docker compose version
```

---

## 🤝 기여하기

더 좋은 예제나 설명이 있다면 언제든 환영합니다!

```bash
# 1. Fork the repository
# 2. Create your feature branch
git checkout -b feature/AmazingContent

# 3. Commit your changes
git commit -m 'Add amazing Docker content'

# 4. Push to the branch
git push origin feature/AmazingContent

# 5. Open a Pull Request
```

### 기여 가이드라인

- ✅ 실행 가능한 예제 코드
- ✅ 명확한 설명과 주석
- ✅ Before/After 비교
- ✅ 실전 시나리오 포함
- ✅ 성능 측정 결과 (가능한 경우)

---

## 📚 추천 학습 순서

### 🥇 1단계: 필수 기초 (2-4주)
```
Fundamentals → Images → Networking
```

### 🥈 2단계: 실전 적용 (2-4주)
```
Storage → Orchestration → Real World
```

### 🥉 3단계: 심화 학습 (1-2개월)
```
Security → Performance → Advanced
```

### 🏆 4단계: 마스터 (진행형)
```
모든 섹션 완료 → 프로덕션 적용 → 오픈소스 기여
```

---

## 🙏 Reference & Resources

### 📚 공식 문서
- [Docker Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [containerd](https://containerd.io/)
- [OCI Specification](https://opencontainers.org/)

### 🎓 추가 학습 자료
- [Docker Curriculum](https://docker-curriculum.com/)
- [Docker Labs](https://github.com/docker/labs)

### 🛠️ 유용한 도구
- [Dive](https://github.com/wagoodman/dive) - 이미지 레이어 분석
- [Hadolint](https://github.com/hadolint/hadolint) - Dockerfile 린터
- [Trivy](https://github.com/aquasecurity/trivy) - 취약점 스캐너
- [Portainer](https://www.portainer.io/) - Docker GUI

---

## ✨ Dev Book Lab

<div align="center">

**AI와 함께 개발 서적을 분석하고 정리하는 연구소**

[📂 다른 프로젝트 보기](https://github.com/dev-book-lab)

</div>

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by Dev Book Lab

<br/>

*"컨테이너를 실행하는 것을 넘어, Docker가 실제로 어떻게 동작하는지 완전히 이해하기"*

</div>
