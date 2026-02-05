# 06. Custom Networks - 커스텀 네트워크

## 🎯 이 챕터에서 배울 것

- **CNI (Container Network Interface)** 플러그인 시스템
- **Calico**, **Weave**, **Flannel** 등 서드파티 네트워크 솔루션
- 커스텀 네트워크 **드라이버 개발** 기초
- 실무 **네트워크 선택 기준**

## 📌 왜 중요한가?

**"Docker 기본 네트워크만으로는 복잡한 요구사항을 충족할 수 없습니다."**

```
기본 네트워크 드라이버의 한계:
┌──────────────┬─────────┬────────────┐
│ 드라이버       │ 멀티호스트 │ 네트워크 정책 │
├──────────────┼─────────┼────────────┤
│ Bridge       │ ❌      │ ❌         │
│ Host         │ ❌      │ ❌         │
│ Overlay      │ ✅      │ ❌         │
│ Macvlan      │ ❌      │ ❌         │
└──────────────┴─────────┴────────────┘

커스텀 네트워크 솔루션:
┌──────────────┬─────────┬────────────┬────────┐
│ 솔루션         │ 멀티호스트│ 네트워크 정책  │ 암호화   │
├──────────────┼─────────┼────────────┼────────┤
│ Calico       │ ✅      │ ✅         │ ✅     │
│ Weave        │ ✅      │ ✅         │ ✅     │
│ Flannel      │ ✅      │ ❌         │ ❌     │
│ Cilium       │ ✅      │ ✅         │ ✅     │
└──────────────┴─────────┴────────────┴────────┘
```

**실무 영향:**
- 보안: 세밀한 네트워크 정책 제어
- 성능: 워크로드에 최적화된 네트워킹
- 운영: Kubernetes 등 오케스트레이터 통합
- 확장: 대규모 클러스터 지원

---

## 🔬 Deep Dive

### 1. CNI (Container Network Interface)

#### CNI란?

```
CNI:
- 컨테이너 네트워킹 표준 인터페이스
- CNCF 프로젝트
- 플러그인 기반 아키텍처
- Kubernetes, Mesos 등에서 사용

구조:
┌──────────────────────────────────────┐
│ Container Runtime (Docker/containerd)│
├──────────────────────────────────────┤
│ CNI Plugin Interface                 │
├──────────────────────────────────────┤
│ ┌────────┬─────────┬────────┬──────┐ │
│ │ Bridge │ Macvlan │ IPVLAN │ ...  │ │
│ └────────┴─────────┴────────┴──────┘ │
├──────────────────────────────────────┤
│ ┌────────┬─────────┬────────┬──────┐ │
│ │ Calico │ Weave   │ Flannel│Cilium│ │
│ └────────┴─────────┴────────┴──────┘ │
└──────────────────────────────────────┘

CNI 플러그인:
1. Main Plugin (네트워크 생성)
   - bridge, macvlan, ipvlan, etc.
2. IPAM Plugin (IP 관리)
   - host-local, dhcp, static
3. Meta Plugin (기능 추가)
   - portmap, bandwidth, firewall
```

#### CNI 설정

```bash
# CNI 설정 디렉토리
ls /etc/cni/net.d/
# 10-mynet.conflist

# CNI 설정 예시
cat > /etc/cni/net.d/10-mynet.conflist << 'EOF'
{
  "cniVersion": "0.4.0",
  "name": "mynet",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "subnet": "10.22.0.0/16",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    },
    {
      "type": "firewall"
    }
  ]
}
EOF

# CNI 바이너리
ls /opt/cni/bin/
# bridge dhcp firewall host-local ipvlan macvlan ...
```

---

### 2. Calico

#### Calico란?

```
Calico:
- Layer 3 네트워크 솔루션
- BGP 기반 라우팅
- 네트워크 정책 (NetworkPolicy)
- eBPF 데이터플레인 지원

특징:
- 순수 L3 라우팅 (오버레이 없음)
- 뛰어난 성능
- 세밀한 보안 정책
- 대규모 클러스터 지원 (1000+ 노드)

구조:
┌──────────────────────────────────────┐
│ Node 1                               │
│  ┌────────────┐  ┌────────────┐      │
│  │ Container  │  │ Container  │      │
│  │ 10.1.1.2   │  │ 10.1.1.3   │      │
│  └─────┬──────┘  └──────┬─────┘      │
│        │                │            │
│    ┌───▼────────────────▼───┐        │
│    │   Felix (Agent)        │        │
│    │   - iptables rules     │        │
│    │   - routing table      │        │
│    └────────────┬───────────┘        │
│                 │                    │
│            ┌────▼────┐               │
│            │ BGP     │               │
│            └────┬────┘               │
└─────────────────┼────────────────────┘
                  │ BGP Peering
┌─────────────────▼─────────────────────┐
│ Node 2                                │
│            ┌────┬────┐                │
│            │ BGP     │                │
│            └────┬────┘                │
│                 │                     │
│    ┌────────────▼───────────┐         │
│    │   Felix (Agent)        │         │
│    └───┬────────────────┬───┘         │
│        │                │             │
│  ┌─────▼──────┐  ┌─────▼──────┐       │
│  │ Container  │  │ Container  │       │
│  │ 10.1.2.2   │  │ 10.1.2.3   │       │
│  └────────────┘  └────────────┘       │
└───────────────────────────────────────┘
```

#### Calico 설치 및 사용

```bash
# Kubernetes에서 Calico 설치
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Calico 상태 확인
kubectl get pods -n kube-system -l k8s-app=calico-node

# 출력:
# NAME                READY   STATUS    RESTARTS   AGE
# calico-node-abc123  1/1     Running   0          2m
# calico-node-def456  1/1     Running   0          2m

# BGP 피어 확인
calicoctl node status

# 출력:
# Calico process is running.
# 
# IPv4 BGP status
# +--------------+-------------------+-------+----------+
# | PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |
# +--------------+-------------------+-------+----------+
# | 10.0.0.2     | node-to-node mesh | up    | 12:34:56 |
# | 10.0.0.3     | node-to-node mesh | up    | 12:34:58 |
# +--------------+-------------------+-------+----------+
```

#### 네트워크 정책

```yaml
# deny-all.yaml - 모든 트래픽 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# allow-frontend.yaml - Frontend만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# allow-egress-dns.yaml - DNS 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-dns
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

```bash
# 정책 적용
kubectl apply -f deny-all.yaml
kubectl apply -f allow-frontend.yaml
kubectl apply -f allow-egress-dns.yaml

# 정책 확인
kubectl get networkpolicy

# 테스트
# Frontend → Backend (허용)
kubectl exec frontend-pod -- curl backend:8080
# 성공 ✅

# Other → Backend (차단)
kubectl exec other-pod -- curl backend:8080
# timeout ❌
```

---

### 3. Weave Net

#### Weave Net이란?

```
Weave Net:
- 간단한 멀티 호스트 네트워킹
- 자동 메시 네트워크
- 암호화 지원
- 서비스 디스커버리

특징:
- 설정 최소화 (제로 컨피그)
- Fast Datapath (OVS 기반)
- 네트워크 정책 지원
- Docker/Kubernetes 플러그인

구조:
┌──────────────────────────────────────┐
│ Node 1                               │
│  ┌────────────┐  ┌────────────┐      │
│  │ Container  │  │ Container  │      │
│  │ 10.32.0.2  │  │ 10.32.0.3  │      │
│  └─────┬──────┘  └──────┬─────┘      │
│        │                │            │
│    ┌───▼────────────────▼───┐        │
│    │   Weave Net Router     │        │
│    │   - Mesh network       │        │
│    │   - VXLAN/UDP          │        │
│    └────────────┬───────────┘        │
└─────────────────┼────────────────────┘
                  │ Mesh Connection
┌─────────────────▼─────────────────────┐
│ Node 2                                │
│    ┌────────────┬───────────┐         │
│    │   Weave Net Router     │         │
│    └───┬────────────────┬───┘         │
│        │                │             │
│  ┌─────▼──────┐  ┌─────▼──────┐       │
│  │ Container  │  │ Container  │       │
│  │ 10.32.0.10 │  │ 10.32.0.11 │       │
│  └────────────┘  └────────────┘       │
└───────────────────────────────────────┘
```

#### Weave Net 설치 및 사용

```bash
# Kubernetes에서 Weave 설치
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Weave 상태 확인
kubectl get pods -n kube-system -l name=weave-net

# 출력:
# NAME              READY   STATUS    RESTARTS   AGE
# weave-net-abc123  2/2     Running   0          1m
# weave-net-def456  2/2     Running   0          1m

# Weave 네트워크 상태
kubectl exec -n kube-system weave-net-abc123 -c weave -- \
  /home/weave/weave --local status

# 출력:
#         Version: 2.8.1
#        Service: router
#       Protocol: weave 1..2
#           Name: 7a:12:34:56:78:90(node1)
#     Encryption: disabled
#   Peer Discovery: enabled
# 
# Connections: 2 (1 established, 1 pending)
# Peers: 3 (with 6 established connections)
```

#### 암호화 활성화

```bash
# 암호화 패스워드 생성
PASSWORD=$(openssl rand -base64 32)

# Secret 생성
kubectl create secret generic weave-passwd \
  -n kube-system \
  --from-literal=weave-password=$PASSWORD

# Weave 재시작 (암호화 활성화)
kubectl delete pods -n kube-system -l name=weave-net

# 암호화 확인
kubectl exec -n kube-system weave-net-abc123 -c weave -- \
  /home/weave/weave --local status | grep Encryption
# Encryption: enabled
```

---

### 4. Flannel

#### Flannel이란?

```
Flannel:
- 간단한 오버레이 네트워크
- CoreOS에서 개발
- 여러 백엔드 지원
- Kubernetes에 최적화

백엔드:
- VXLAN (기본)
- Host-GW (성능)
- UDP (레거시)
- AWS VPC
- GCE

구조:
┌──────────────────────────────────────┐
│ Node 1 (10.0.0.10)                   │
│  ┌────────────┐  ┌────────────┐      │
│  │ Container  │  │ Container  │      │
│  │ 10.244.0.2 │  │ 10.244.0.3 │      │
│  └─────┬──────┘  └──────┬─────┘      │
│        │                │            │
│    ┌───▼────────────────▼───┐        │
│    │   cni0 (bridge)        │        │
│    │   10.244.0.1           │        │
│    └────────────┬───────────┘        │
│                 │                    │
│    ┌────────────▼───────────┐        │
│    │   flannel.1 (VXLAN)    │        │
│    └────────────┬───────────┘        │
│                 │                    │
│            ┌────▼────┐               │
│            │  eth0   │               │
│            └────┬────┘               │
└─────────────────┼────────────────────┘
                  │ VXLAN Tunnel
┌─────────────────▼─────────────────────┐
│ Node 2 (10.0.0.11)                    │
│            ┌────┬────┐                │
│            │  eth0   │                │
│            └────┬────┘                │
│                 │                     │
│    ┌────────────▼───────────┐         │
│    │   flannel.1 (VXLAN)    │         │
│    └────────────┬───────────┘         │
│                 │                     │
│    ┌────────────▼───────────┐         │
│    │   cni0 (bridge)        │         │
│    │   10.244.1.1           │         │
│    └───┬───────────────┬────┘         │
│        │               │              │
│  ┌─────▼──────┐  ┌─────▼──────┐       │
│  │ Container  │  │ Container  │       │
│  │ 10.244.1.2 │  │ 10.244.1.3 │       │
│  └────────────┘  └────────────┘       │
└───────────────────────────────────────┘
```

#### Flannel 설치 및 사용

```bash
# Kubernetes에서 Flannel 설치
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Flannel 상태 확인
kubectl get pods -n kube-system -l app=flannel

# 출력:
# NAME                    READY   STATUS    RESTARTS   AGE
# kube-flannel-ds-abc123  1/1     Running   0          30s
# kube-flannel-ds-def456  1/1     Running   0          30s

# VXLAN 인터페이스 확인
ip -d link show flannel.1

# 출력:
# 4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     vxlan id 1 local 10.0.0.10 dev eth0 port 8472
```

#### Flannel 설정

```yaml
# kube-flannel-cfg ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

---

### 5. Cilium

#### Cilium이란?

```
Cilium:
- eBPF 기반 네트워킹
- API-aware 보안 정책
- L7 로드 밸런싱
- 고성능

특징:
- eBPF (커널 바이패스)
- Identity-based 보안
- HTTP/gRPC 인식
- Hubble (관찰성)

구조:
┌──────────────────────────────────────┐
│ Kernel Space                         │
│  ┌────────────────────────────────┐  │
│  │ eBPF Programs                  │  │
│  │ - XDP                          │  │
│  │ - TC (Traffic Control)         │  │
│  │ - Socket operations            │  │
│  └────────────────────────────────┘  │
├──────────────────────────────────────┤
│ User Space                           │
│  ┌────────────────────────────────┐  │
│  │ Cilium Agent                   │  │
│  │ - Policy enforcement           │  │
│  │ - Identity management          │  │
│  │ - Load balancing               │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

eBPF 장점:
- 커널 컨텍스트 스위칭 없음
- 동적 프로그래밍
- 네트워크 스택 바이패스
- 초고성능
```

#### Cilium 설치

```bash
# Helm으로 Cilium 설치
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.14.0 \
  --namespace kube-system

# Cilium 상태 확인
cilium status

# 출력:
#     /¯¯\
#  /¯¯\__/¯¯\    Cilium:         OK
#  \__/¯¯\__/    Operator:       OK
#  /¯¯\__/¯¯\    Hubble:         OK
#  \__/¯¯\__/    ClusterMesh:    disabled
#     \__/
# 
# DaemonSet         cilium             Desired: 3, Ready: 3/3
# Deployment        cilium-operator    Desired: 1, Ready: 1/1

# 연결성 테스트
cilium connectivity test

# 출력:
# ✅ All tests passed!
```

#### L7 네트워크 정책

```yaml
# http-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-api
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/.*"

---
# rate-limit.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: rate-limit
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - {}
    toPorts:
    - ports:
      - port: "80"
      rules:
        http:
        - method: "*"
          rateLimit:
            requestsPerSecond: 100
```

---

## 💻 실습

### 실습 1: Calico 네트워크 정책

#### 환경 준비

```bash
# Minikube로 Kubernetes 클러스터 생성
minikube start --network-plugin=cni --cni=calico

# Calico 설치 확인
kubectl get pods -n kube-system -l k8s-app=calico-node

# 테스트 애플리케이션 배포
kubectl create deployment frontend --image=nginx
kubectl create deployment backend --image=nginx
kubectl create deployment database --image=postgres:alpine

kubectl expose deployment frontend --port=80
kubectl expose deployment backend --port=80
kubectl expose deployment database --port=5432
```

#### 기본 정책 - 모든 트래픽 차단

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

```bash
kubectl apply -f default-deny.yaml

# 테스트 (차단됨)
kubectl exec deployment/frontend -- curl backend
# timeout ❌

kubectl exec deployment/backend -- curl database:5432
# timeout ❌
```

#### 선택적 허용 정책

```yaml
# allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80

---
# allow-backend-to-database.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432

---
# allow-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

```bash
kubectl apply -f allow-frontend-to-backend.yaml
kubectl apply -f allow-backend-to-database.yaml
kubectl apply -f allow-dns.yaml

# 테스트
# Frontend → Backend (허용)
kubectl exec deployment/frontend -- curl backend
# ✅ 성공!

# Frontend → Database (차단)
kubectl exec deployment/frontend -- curl database:5432
# ❌ timeout

# Backend → Database (허용)
kubectl exec deployment/backend -- curl database:5432
# ✅ 성공!
```

---

### 실습 2: Weave Net 암호화

#### Weave 설치

```bash
# Weave Net 설치
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# 암호화 패스워드 생성
PASSWORD=$(openssl rand -base64 32)
echo "Password: $PASSWORD"

# Secret 생성
kubectl create secret generic weave-passwd \
  -n kube-system \
  --from-literal=weave-password=$PASSWORD

# Weave 재시작
kubectl delete pods -n kube-system -l name=weave-net

# 암호화 확인
kubectl exec -n kube-system $(kubectl get pod -n kube-system -l name=weave-net -o jsonpath='{.items[0].metadata.name}') \
  -c weave -- /home/weave/weave --local status | grep Encryption
# Encryption: enabled
```

#### 트래픽 캡처 (암호화 확인)

```bash
# Pod 2개 생성
kubectl run pod1 --image=nicolaka/netshoot -- sleep 3600
kubectl run pod2 --image=nicolaka/netshoot -- sleep 3600

# Pod IP 확인
POD1_IP=$(kubectl get pod pod1 -o jsonpath='{.status.podIP}')
POD2_IP=$(kubectl get pod pod2 -o jsonpath='{.status.podIP}')

# 노드에서 트래픽 캡처
ssh node1
sudo tcpdump -i any -n -X 'udp port 6783'

# Pod1에서 Pod2로 통신
kubectl exec pod1 -- ping -c 3 $POD2_IP

# tcpdump 출력:
# - ESP (Encapsulating Security Payload)
# - 데이터가 암호화되어 읽을 수 없음
```

---

### 실습 3: Flannel 백엔드 비교

#### VXLAN 백엔드 (기본)

```yaml
# flannel-vxlan.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
data:
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan",
        "Port": 8472
      }
    }
```

```bash
# 적용
kubectl apply -f flannel-vxlan.yaml

# VXLAN 인터페이스 확인
ip -d link show flannel.1

# 성능 테스트
kubectl run iperf-server --image=networkstatic/iperf3 -- -s
kubectl run iperf-client --image=networkstatic/iperf3 -- \
  -c <SERVER_IP> -t 10

# 결과: ~7 Gbps
```

#### Host-GW 백엔드

```yaml
# flannel-hostgw.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
data:
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "host-gw"
      }
    }
```

```bash
# 적용
kubectl apply -f flannel-hostgw.yaml
kubectl delete pods -n kube-system -l app=flannel

# 라우팅 테이블 확인
ip route
# 10.244.1.0/24 via 10.0.0.11 dev eth0

# 성능 테스트
kubectl run iperf-client --image=networkstatic/iperf3 -- \
  -c <SERVER_IP> -t 10

# 결과: ~9 Gbps (+28%)
# Host-GW가 VXLAN보다 빠름 (캡슐화 없음)
```

---

### 실습 4: Cilium L7 정책

#### Cilium 설치

```bash
# Cilium 설치
helm install cilium cilium/cilium --version 1.14.0 \
  --namespace kube-system \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true

# Hubble UI 포트포워딩
kubectl port-forward -n kube-system svc/hubble-ui 12000:80

# 브라우저에서 http://localhost:12000 접속
```

#### L7 HTTP 정책

```yaml
# http-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-policy
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: client
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/.*"
        - method: "POST"
          path: "/api/v1/users"
```

```bash
# 적용
kubectl apply -f http-policy.yaml

# 테스트 애플리케이션
kubectl run api --image=kennethreitz/httpbin --labels=app=api
kubectl expose pod api --port=8080 --target-port=80

kubectl run client --image=curlimages/curl --labels=app=client \
  -- sleep 3600

# 허용된 요청
kubectl exec client -- curl api:8080/api/v1/users
# ✅ 성공

kubectl exec client -- curl -X POST api:8080/api/v1/users
# ✅ 성공

# 차단된 요청
kubectl exec client -- curl api:8080/admin
# ❌ 403 Forbidden

kubectl exec client -- curl -X DELETE api:8080/api/v1/users/1
# ❌ 403 Forbidden

# Hubble에서 플로우 확인
hubble observe --pod client
```

---

## 🔥 실전 적용

### 시나리오 1: 마이크로서비스 보안

**상황:**
- 20개 마이크로서비스
- 서비스 간 통신 제한 필요
- Zero Trust 보안 모델

**솔루션: Calico 네트워크 정책**

```yaml
# 기본 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Frontend → API Gateway만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-gateway
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: gateway
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - port: 8080

---
# API Gateway → Backend Services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gateway-to-services
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: gateway
    ports:
    - port: 8080

---
# Backend → Database만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - port: 5432

---
# 모든 서비스 → DNS 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - port: 53
      protocol: UDP

---
# 외부 API 호출 허용 (특정 서비스만)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - port: 443
      protocol: TCP
```

**효과:**
```
- 공격 표면 최소화
- 측면 이동 차단
- 규정 준수 (PCI-DSS, SOC2)
- 침해 영향 제한
```

---

### 시나리오 2: 멀티 클라우드 네트워킹

**상황:**
- AWS + GCP + 온프레미스
- 클러스터 간 통신
- 통합 네트워크 정책

**솔루션: Cilium ClusterMesh**

```bash
# Cluster 1 (AWS)
helm install cilium cilium/cilium \
  --set cluster.name=aws-cluster \
  --set cluster.id=1 \
  --set ipam.mode=kubernetes

# Cluster 2 (GCP)
helm install cilium cilium/cilium \
  --set cluster.name=gcp-cluster \
  --set cluster.id=2 \
  --set ipam.mode=kubernetes

# ClusterMesh 활성화
cilium clustermesh enable

# Cluster 연결
cilium clustermesh connect \
  --context aws-cluster \
  --destination-context gcp-cluster

# 글로벌 서비스
kubectl annotate service backend \
  io.cilium/global-service="true"

# 이제 AWS 클러스터의 Pod에서 GCP 클러스터의 서비스 접근 가능!
```

---

## ⚡ 네트워크 솔루션 선택 가이드

### 기능 비교

```
┌──────────┬───────┬───────┬─────────┬──────────┐
│ 기능      │Calico │ Weave │ Flannel │ Cilium   │
├──────────┼───────┼───────┼─────────┼──────────┤
│ 성능      │ ⭐⭐⭐│ ⭐⭐ │ ⭐⭐    │ ⭐⭐⭐⭐│
├──────────┼───────┼───────┼─────────┼──────────┤
│ 간편성     │ ⭐⭐ │ ⭐⭐⭐│ ⭐⭐⭐ │ ⭐⭐     │
├──────────┼───────┼───────┼─────────┼──────────┤
│ 정책 기능  │ ⭐⭐⭐│ ⭐⭐ │ ❌      │ ⭐⭐⭐⭐│
├──────────┼───────┼───────┼─────────┼──────────┤
│ 암호화     │ ✅    │ ✅    │ ❌     │ ✅       │
├──────────┼───────┼───────┼─────────┼──────────┤
│ L7 인식   │ ❌    │ ❌    │ ❌      │ ✅       │
├──────────┼───────┼───────┼─────────┼──────────┤
│ 관찰성     │ ⭐⭐ │ ⭐    │ ⭐      │ ⭐⭐⭐⭐│
└──────────┴───────┴───────┴─────────┴──────────┘
```

### 선택 기준

```
Calico:
✅ 대규모 클러스터 (1000+ 노드)
✅ 세밀한 네트워크 정책 필요
✅ BGP 라우팅 인프라 있음
✅ 온프레미스 환경

Weave:
✅ 간단한 설정 원함
✅ 암호화 필요
✅ 중소규모 클러스터
✅ 빠른 프로토타이핑

Flannel:
✅ 최소 설정
✅ 네트워크 정책 불필요
✅ 단순 오버레이만 필요
✅ 레거시 환경

Cilium:
✅ 최고 성능 필요
✅ L7 정책 필요
✅ API 인식 보안
✅ 최신 커널 (4.9+)
```

---

## 🎓 핵심 정리

### 1. CNI 플러그인

```
표준:
- Container Network Interface
- 플러그인 기반
- Runtime 독립적

타입:
- Main (네트워크 생성)
- IPAM (IP 관리)
- Meta (기능 추가)
```

### 2. 주요 솔루션

```
Calico:
- BGP 라우팅
- 네트워크 정책
- 대규모 지원

Weave:
- 메시 네트워크
- 간단한 설정
- 암호화 지원

Flannel:
- 오버레이 네트워크
- 여러 백엔드
- 단순함

Cilium:
- eBPF 기반
- L7 인식
- 최고 성능
```

### 3. 네트워크 정책

```
레벨:
- L3/L4 (IP, Port)
- L7 (HTTP, gRPC)

정책:
- Ingress (수신)
- Egress (송신)
- Default Deny
```

### 4. 핵심 명령어

```bash
# CNI 설정
ls /etc/cni/net.d/

# Calico
calicoctl node status
kubectl get networkpolicy

# Weave
weave status
weave report

# Flannel
ip route
ip -d link show flannel.1

# Cilium
cilium status
hubble observe
```

---

## 📚 참고 자료

- [CNI Specification](https://github.com/containernetworking/cni/blob/master/SPEC.md)
- [Calico Documentation](https://docs.projectcalico.org/)
- [Weave Net Documentation](https://www.weave.works/docs/net/latest/overview/)
- [Flannel Documentation](https://github.com/flannel-io/flannel)
- [Cilium Documentation](https://docs.cilium.io/)

---

## 🤔 생각해볼 문제

1. Calico의 BGP와 Flannel의 VXLAN, 어느 것이 더 효율적일까?
2. 네트워크 정책이 없으면 어떤 보안 위험이 있을까?
3. eBPF 기반 Cilium이 iptables 기반보다 빠른 이유는?

> 💡 **답변**: 1) 환경에 따라 다름 - BGP는 순수 L3 라우팅으로 캡슐화 오버헤드 없지만 네트워크 인프라 지원 필요, VXLAN은 캡슐화 오버헤드 있지만 기존 네트워크와 독립적, 같은 L2 세그먼트에서는 BGP가 20-30% 빠르지만 다른 네트워크 환경에서는 VXLAN이 더 범용적, 2) East-West 공격 (측면 이동), 컨테이너 간 무제한 통신, 손상된 컨테이너가 전체 클러스터 공격 가능, 데이터 유출 위험, 규정 준수 실패 (PCI-DSS, HIPAA), 3) eBPF는 커널 내부에서 직접 패킷 처리 (커널 바이패스), iptables는 각 규칙마다 선형 탐색 (O(n)), eBPF는 해시 기반 룩업 (O(1)), 컨텍스트 스위칭 없음, JIT 컴파일로 네이티브 성능, 실제로 10-100배 빠른 경우도 있음

---

<div align="center">

**[⬅️ 이전: Macvlan Network](./05-Macvlan-Network.md)** | **[다음: DNS Resolution ➡️](./07-DNS-Resolution.md)**

</div>
