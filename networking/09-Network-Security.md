# 09. Network Security - 네트워크 보안

## 🎯 이 챕터에서 배울 것

- **네트워크 정책**으로 트래픽 제어
- **방화벽**과 iptables 규칙
- **암호화**된 통신 (TLS, mTLS)
- 컨테이너 **격리**와 보안 베스트 프랙티스

## 📌 왜 중요한가?

**"컨테이너는 기본적으로 모두 연결되어 있습니다. 보안은 선택이 아닌 필수입니다."**

```
보안 없음:
┌─────────────────────────────────┐
│ All containers can communicate  │
│ ┌───┐  ┌───┐  ┌───┐  ┌───┐      │
│ │ A ├──┤ B ├──┤ C ├──┤ D │      │
│ └───┘  └───┘  └───┘  └───┘      │
│ - 무제한 접근                      │
│ - 측면 이동 가능                   │
│ - 데이터 유출 위험                  │
└─────────────────────────────────┘

보안 적용:
┌─────────────────────────────────┐
│ Network Policies                │
│ ┌───┐       ┌───┐       ┌───┐   │
│ │ A │  ✅   │ B │  ❌   │ D │   │
│ └───┘       └─┬─┘       └───┘   │
│               │ ✅              │
│             ┌─▼─┐               │
│             │ C │               │
│             └───┘               │
│ - 명시적 허용만                    │
│ - 최소 권한 원칙                   │
│ - Zero Trust                    │
└─────────────────────────────────┘

보안 계층:
┌──────────────────────────────────┐
│ Layer 7: Application Security    │
├──────────────────────────────────┤
│ Layer 4: Network Policy          │
├──────────────────────────────────┤
│ Layer 3: Firewall/iptables       │
├──────────────────────────────────┤
│ Layer 2: Network Isolation       │
├──────────────────────────────────┤
│ Layer 1: TLS Encryption          │
└──────────────────────────────────┘
```

**실무 영향:**
- 규정 준수: PCI-DSS, HIPAA, SOC2
- 침해 방지: 공격 표면 최소화
- 데이터 보호: 암호화 및 접근 제어
- 감사: 모든 통신 로깅

---

## 🔬 Deep Dive

### 1. 네트워크 격리

#### 기본 격리

```
Docker 네트워크:
- 각 네트워크는 격리됨
- 같은 네트워크만 통신 가능
- 명시적 연결 필요

예시:
┌─────────────────────────────────┐
│ frontend-net                    │
│ ┌────────┐  ┌────────┐          │
│ │  Web   ├──┤  API   │          │
│ └────────┘  └────────┘          │
└─────────────────────────────────┘
          ❌ (격리됨)
┌─────────────────────────────────┐
│ backend-net                     │
│ ┌────────┐  ┌────────┐          │
│ │   DB   ├──┤ Cache  │          │
│ └────────┘  └────────┘          │
└─────────────────────────────────┘
```

#### 실습

```bash
# 네트워크 2개 생성
docker network create frontend-net
docker network create backend-net

# Frontend
docker run -d --name web \
  --network frontend-net \
  nginx:alpine

# Backend
docker run -d --name db \
  --network backend-net \
  postgres:alpine

# 통신 테스트
docker exec web ping -c 1 db
# ping: bad address 'db'
# ❌ 격리됨!

# API Gateway (양쪽 연결)
docker run -d --name api \
  --network frontend-net \
  nginx:alpine

docker network connect backend-net api

# API → DB
docker exec api ping -c 1 db
# ✅ 성공!

# Web → DB (여전히 불가)
docker exec web ping -c 1 db
# ❌ 격리됨

# 정리
docker rm -f web api db
docker network rm frontend-net backend-net
```

---

### 2. 네트워크 정책 (Kubernetes)

#### 기본 거부 정책

```yaml
# deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

```bash
kubectl apply -f deny-all.yaml

# 모든 트래픽 차단
kubectl exec frontend-pod -- curl backend
# timeout ❌
```

#### 선택적 허용

```yaml
# allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
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
# allow-backend-to-db.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: production
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
  namespace: production
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

---
# allow-external-api.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 443
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

```bash
kubectl apply -f allow-frontend-to-backend.yaml
kubectl apply -f allow-backend-to-db.yaml
kubectl apply -f allow-dns.yaml
kubectl apply -f allow-external-api.yaml

# 테스트
# Frontend → Backend
kubectl exec frontend-pod -- curl backend:8080
# ✅ 허용

# Frontend → Database
kubectl exec frontend-pod -- curl database:5432
# ❌ 차단

# Backend → Database
kubectl exec backend-pod -- curl database:5432
# ✅ 허용

# Payment → External API
kubectl exec payment-pod -- curl https://api.stripe.com
# ✅ 허용
```

---

### 3. iptables 방화벽

#### Docker iptables 규칙

```bash
# Docker가 생성한 iptables 규칙 확인
sudo iptables -t nat -L -n

# DOCKER chain
# Chain DOCKER (2 references)
# target     prot opt source      destination
# DNAT       tcp  --  0.0.0.0/0   0.0.0.0/0    tcp dpt:8080 to:172.17.0.2:80

# 컨테이너 포트 퍼블리시 시 DNAT 규칙 추가
docker run -d -p 8080:80 nginx

# 새 규칙 확인
sudo iptables -t nat -L DOCKER -n
# DNAT tcp -- 0.0.0.0/0 0.0.0.0/0 tcp dpt:8080 to:172.17.0.2:80
```

#### 커스텀 방화벽 규칙

```bash
# 1. 특정 IP만 컨테이너 접근 허용
sudo iptables -I DOCKER-USER -i eth0 \
  ! -s 10.0.0.0/8 -j DROP

# 10.0.0.0/8 외부 IP는 모두 차단

# 2. 컨테이너 간 통신 제한
sudo iptables -I DOCKER-USER -i docker0 -o docker0 \
  -s 172.17.0.2 -d 172.17.0.3 -j DROP

# 특정 컨테이너 간 통신 차단

# 3. DDoS 방어 - Rate Limiting
sudo iptables -I DOCKER-USER -p tcp --dport 80 \
  -m state --state NEW \
  -m recent --set

sudo iptables -I DOCKER-USER -p tcp --dport 80 \
  -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 10 \
  -j DROP

# 1분에 10회 초과 연결 차단

# 4. 로깅
sudo iptables -I DOCKER-USER -j LOG \
  --log-prefix "Docker-Firewall: " \
  --log-level 4

# 모든 트래픽 로깅
```

#### 영구 저장

```bash
# iptables 규칙 저장
sudo iptables-save > /etc/iptables/rules.v4

# 또는 iptables-persistent 사용
sudo apt-get install iptables-persistent
sudo netfilter-persistent save
sudo netfilter-persistent reload

# 재부팅 후에도 유지
```

---

### 4. TLS 암호화

#### 자체 서명 인증서

```bash
# CA 생성
openssl genrsa -out ca-key.pem 4096

openssl req -new -x509 -days 365 -key ca-key.pem \
  -sha256 -out ca.pem \
  -subj "/C=US/ST=CA/L=SF/O=MyOrg/CN=CA"

# 서버 키 생성
openssl genrsa -out server-key.pem 4096

# 서버 CSR
openssl req -new -key server-key.pem \
  -out server.csr \
  -subj "/CN=myserver"

# 서버 인증서
echo "subjectAltName = DNS:myserver,DNS:localhost,IP:127.0.0.1" > extfile.cnf
openssl x509 -req -days 365 -sha256 \
  -in server.csr \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out server-cert.pem \
  -extfile extfile.cnf

# 클라이언트 키 생성
openssl genrsa -out client-key.pem 4096

# 클라이언트 CSR
openssl req -new -key client-key.pem \
  -out client.csr \
  -subj "/CN=client"

# 클라이언트 인증서
echo "extendedKeyUsage = clientAuth" > extfile-client.cnf
openssl x509 -req -days 365 -sha256 \
  -in client.csr \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out client-cert.pem \
  -extfile extfile-client.cnf

# 권한 설정
chmod 0400 ca-key.pem server-key.pem client-key.pem
chmod 0444 ca.pem server-cert.pem client-cert.pem
```

#### Nginx TLS 설정

```nginx
# nginx-tls.conf
server {
    listen 443 ssl http2;
    server_name myserver;

    ssl_certificate /etc/nginx/certs/server-cert.pem;
    ssl_certificate_key /etc/nginx/certs/server-key.pem;
    ssl_client_certificate /etc/nginx/certs/ca.pem;
    ssl_verify_client on;

    # TLS 설정
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
        proxy_pass http://backend;
    }
}
```

```bash
# Docker Compose
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
    volumes:
      - ./nginx-tls.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - secure-net

  backend:
    image: nginx:alpine
    networks:
      - secure-net

networks:
  secure-net:
EOF

docker-compose up -d

# 테스트 (mTLS)
curl --cacert ca.pem \
  --cert client-cert.pem \
  --key client-key.pem \
  https://localhost
# ✅ 성공

# 인증서 없이
curl -k https://localhost
# 400 Bad Request (클라이언트 인증서 필요)
```

---

### 5. 암호화된 Overlay 네트워크

#### Docker Swarm 암호화

```bash
# Swarm 초기화
docker swarm init

# 암호화된 Overlay 네트워크
docker network create \
  --driver overlay \
  --opt encrypted \
  secure-net

# 서비스 배포
docker service create \
  --name web \
  --network secure-net \
  --replicas 3 \
  nginx:alpine

# 패킷 캡처 (암호화 확인)
sudo tcpdump -i eth0 -n -X 'udp port 4789'

# 트래픽 생성
docker service create \
  --name client \
  --network secure-net \
  curlimages/curl sh -c 'while true; do curl web; sleep 1; done'

# tcpdump 출력:
# ESP (Encapsulating Security Payload)
# 데이터 암호화됨!
```

---

## 💻 실습

### 실습 1: Zero Trust 네트워크

#### 아키텍처

```
원칙:
- 기본 거부 (Deny All)
- 명시적 허용만 (Allow Specific)
- 최소 권한 (Least Privilege)
- 지속적 검증 (Continuous Verification)

구조:
Internet
  ↓
┌─────────────────┐
│ Nginx (TLS)     │ ← Public
└────────┬────────┘
         │ ✅
┌────────▼────────┐
│ Frontend        │ ← DMZ
└────────┬────────┘
         │ ✅
┌────────▼────────┐
│ API Gateway     │ ← Application
└────────┬────────┘
         │ ✅
┌────────▼────────┐
│ Backend Service │ ← Internal
└────────┬────────┘
         │ ✅
┌────────▼────────┐
│ Database        │ ← Data (격리)
└─────────────────┘
```

#### Kubernetes 구성

```yaml
# 00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production

---
# 01-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# 02-allow-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

---
# 03-allow-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: nginx-ingress
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: api-gateway
    ports:
    - protocol: TCP
      port: 8080

---
# 04-allow-api-gateway.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-gateway
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api-gateway
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080

---
# 05-allow-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api-gateway
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432

---
# 06-allow-database.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

```bash
# 적용
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-deny-all.yaml
kubectl apply -f 02-allow-dns.yaml
kubectl apply -f 03-allow-frontend.yaml
kubectl apply -f 04-allow-api-gateway.yaml
kubectl apply -f 05-allow-backend.yaml
kubectl apply -f 06-allow-database.yaml

# 검증
kubectl get networkpolicy -n production

# 테스트
kubectl run test -n production --image=curlimages/curl -it --rm -- sh

# 허용된 경로
frontend → api-gateway → backend → database ✅

# 차단된 경로
frontend → backend ❌
frontend → database ❌
api-gateway → database ❌
```

---

### 실습 2: 방화벽 규칙

#### Docker iptables 규칙

```bash
# 1. 기본 정책 설정
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# 2. Established 연결 허용
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# 3. Loopback 허용
sudo iptables -A INPUT -i lo -j ACCEPT

# 4. SSH 허용 (포트 22)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 5. Docker 규칙
# 신뢰할 수 있는 소스만 허용
sudo iptables -I DOCKER-USER -i eth0 -s 10.0.0.0/8 -j ACCEPT
sudo iptables -I DOCKER-USER -i eth0 -s 192.168.0.0/16 -j ACCEPT
sudo iptables -I DOCKER-USER -i eth0 -j DROP

# 6. 컨테이너 포트 노출 제한
# 80, 443만 허용
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 7. Rate Limiting
sudo iptables -A INPUT -p tcp --dport 80 \
  -m state --state NEW \
  -m recent --set

sudo iptables -A INPUT -p tcp --dport 80 \
  -m state --state NEW \
  -m recent --update --seconds 10 --hitcount 20 \
  -j DROP

# 10초에 20회 이상 → 차단

# 8. 로깅
sudo iptables -A INPUT -j LOG --log-prefix "IPT-INPUT-DROP: "
sudo iptables -A FORWARD -j LOG --log-prefix "IPT-FORWARD-DROP: "

# 9. 저장
sudo iptables-save > /etc/iptables/rules.v4
```

#### 스크립트로 관리

```bash
#!/bin/bash
# docker-firewall.sh

# 초기화
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# 기본 정책
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Established/Related
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Loopback
iptables -A INPUT -i lo -j ACCEPT

# SSH
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Docker
iptables -I DOCKER-USER -i eth0 -s 10.0.0.0/8 -j ACCEPT
iptables -I DOCKER-USER -i eth0 -j DROP

# HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 저장
iptables-save > /etc/iptables/rules.v4

echo "Firewall rules applied!"
```

---

### 실습 3: mTLS (Mutual TLS)

#### 인증서 생성 자동화

```bash
#!/bin/bash
# generate-certs.sh

set -e

# CA
openssl genrsa -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem \
  -sha256 -out ca.pem \
  -subj "/C=US/ST=CA/L=SF/O=MyOrg/CN=CA"

# Server
openssl genrsa -out server-key.pem 4096
openssl req -new -key server-key.pem \
  -out server.csr \
  -subj "/CN=myserver"
echo "subjectAltName = DNS:myserver,DNS:localhost,IP:127.0.0.1" > extfile.cnf
openssl x509 -req -days 365 -sha256 \
  -in server.csr \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out server-cert.pem \
  -extfile extfile.cnf

# Client
openssl genrsa -out client-key.pem 4096
openssl req -new -key client-key.pem \
  -out client.csr \
  -subj "/CN=client"
echo "extendedKeyUsage = clientAuth" > extfile-client.cnf
openssl x509 -req -days 365 -sha256 \
  -in client.csr \
  -CA ca.pem -CAkey ca-key.pem -CAcreateserial \
  -out client-cert.pem \
  -extfile extfile-client.cnf

# 권한
chmod 0400 ca-key.pem server-key.pem client-key.pem
chmod 0444 ca.pem server-cert.pem client-cert.pem

# 정리
rm -f *.csr *.cnf *.srl

echo "Certificates generated!"
```

#### Nginx mTLS

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server backend:8080;
    }

    server {
        listen 443 ssl http2;
        server_name myserver;

        # TLS 인증서
        ssl_certificate /etc/nginx/certs/server-cert.pem;
        ssl_certificate_key /etc/nginx/certs/server-key.pem;

        # Client 인증서 검증
        ssl_client_certificate /etc/nginx/certs/ca.pem;
        ssl_verify_client on;
        ssl_verify_depth 2;

        # TLS 설정
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;

        # HSTS
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # 클라이언트 인증서 정보 전달
            proxy_set_header X-SSL-Client-Cert $ssl_client_cert;
            proxy_set_header X-SSL-Client-DN $ssl_client_s_dn;
        }

        location /health {
            access_log off;
            return 200 "OK\n";
        }
    }
}
```

#### 테스트

```bash
# 서비스 시작
docker-compose up -d

# mTLS 연결 (성공)
curl --cacert ca.pem \
  --cert client-cert.pem \
  --key client-key.pem \
  https://localhost

# 인증서 없이 (실패)
curl -k https://localhost
# 400 Bad Request - No required SSL certificate was sent

# 잘못된 인증서 (실패)
curl --cacert ca.pem \
  --cert wrong-cert.pem \
  --key wrong-key.pem \
  https://localhost
# SSL certificate problem: unable to get local issuer certificate
```

---

### 실습 4: 보안 모니터링

#### 네트워크 트래픽 로깅

```bash
# 1. iptables 로깅
sudo iptables -I DOCKER-USER -j LOG \
  --log-prefix "Docker-Traffic: " \
  --log-level 4

# 로그 확인
sudo tail -f /var/log/syslog | grep "Docker-Traffic"

# 2. tcpdump로 패킷 캡처
sudo tcpdump -i docker0 -w capture.pcap

# 분석
sudo tcpdump -r capture.pcap -n

# 3. Wireshark로 상세 분석
# capture.pcap를 Wireshark에서 열기
```

#### Falco (침입 탐지)

```yaml
# falco.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-config
  namespace: falco
data:
  falco.yaml: |
    rules_file:
      - /etc/falco/falco_rules.yaml
      - /etc/falco/falco_rules.local.yaml
    
    json_output: true
    json_include_output_property: true
    
    file_output:
      enabled: true
      filename: /var/log/falco/events.log

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: falco
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      containers:
      - name: falco
        image: falcosecurity/falco:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: docker-socket
          mountPath: /var/run/docker.sock
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: boot
          mountPath: /host/boot
          readOnly: true
        - name: modules
          mountPath: /host/lib/modules
          readOnly: true
        - name: usr
          mountPath: /host/usr
          readOnly: true
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
      - name: dev
        hostPath:
          path: /dev
      - name: proc
        hostPath:
          path: /proc
      - name: boot
        hostPath:
          path: /boot
      - name: modules
        hostPath:
          path: /lib/modules
      - name: usr
        hostPath:
          path: /usr
```

```bash
# Falco 설치
kubectl create namespace falco
kubectl apply -f falco.yaml

# 이벤트 확인
kubectl logs -n falco -l app=falco -f

# 의심스러운 활동 감지:
# - 컨테이너에서 쉘 실행
# - 민감한 파일 접근
# - 예상치 못한 네트워크 연결
# - 권한 상승 시도
```

---

## 🔥 실전 적용

### 시나리오 1: 금융 시스템 보안

**요구사항:**
- PCI-DSS 준수
- 결제 데이터 암호화
- 네트워크 격리
- 모든 접근 로깅

**아키텍처:**

```
┌─────────────────────────────────────┐
│ Public Zone (DMZ)                   │
│ ┌─────────────────────────────────┐ │
│ │ WAF + Rate Limiting             │ │
│ │ (Nginx with ModSecurity)        │ │
│ └──────────────┬──────────────────┘ │
└────────────────┼────────────────────┘
                 │ TLS 1.3
┌────────────────▼────────────────────┐
│ Application Zone                    │
│ ┌──────────────────────────────┐    │
│ │ Frontend (React SPA)         │    │
│ └──────────────┬───────────────┘    │
│                │ mTLS               │
│ ┌──────────────▼───────────────┐    │
│ │ API Gateway (OAuth 2.0)      │    │
│ └──────────────┬───────────────┘    │
└────────────────┼────────────────────┘
                 │ mTLS
┌────────────────▼────────────────────┐
│ Business Logic Zone                 │
│ ┌──────────────────────────────┐    │
│ │ Payment Service (PCI-DSS)    │    │
│ │ - Encrypted at rest          │    │
│ │ - Tokenization               │    │
│ │ - Audit logging              │    │
│ └──────────────┬───────────────┘    │
└────────────────┼────────────────────┘
                 │ mTLS + VPN
┌────────────────▼────────────────────┐
│ Data Zone (Isolated)                │
│ ┌──────────────────────────────┐    │
│ │ PostgreSQL (Encrypted)       │    │
│ │ - TDE (Transparent Enc)      │    │
│ │ - Column-level encryption    │    │
│ │ - Access logs                │    │
│ └──────────────────────────────┘    │
└─────────────────────────────────────┘
```

**구현:**

```yaml
# network-policies.yaml
---
# Frontend → API Gateway만
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: api-gateway
    ports:
    - protocol: TCP
      port: 443

---
# API Gateway → Payment Service만
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-gateway-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api-gateway
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: payment-service
    ports:
    - protocol: TCP
      port: 8443

---
# Payment Service → Database만
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: payment-service
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432

---
# Database는 수신만
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: payment-service
    ports:
    - protocol: TCP
      port: 5432
  egress: []  # 외부 통신 차단
```

---

### 시나리오 2: 의료 시스템 (HIPAA)

**요구사항:**
- PHI (Protected Health Information) 보호
- 모든 통신 암호화
- 접근 감사 로그
- 데이터 격리

**구현:**

```yaml
# hipaa-security.yaml
---
# Namespace 격리
apiVersion: v1
kind: Namespace
metadata:
  name: healthcare
  labels:
    compliance: hipaa

---
# Pod Security Policy
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: hipaa-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  readOnlyRootFilesystem: true

---
# 기본 거부
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: healthcare
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Audit Logging
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    namespaces: ["healthcare"]
    verbs: ["get", "list", "create", "update", "patch", "delete"]
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
      - group: "apps"
        resources: ["deployments", "statefulsets"]
```

---

## ⚡ 보안 체크리스트

### 네트워크 격리

```
□ 사용자 정의 네트워크 사용 (기본 bridge 피하기)
□ 네트워크별 역할 분리 (frontend/backend/data)
□ 내부 네트워크 (--internal) 사용
□ 불필요한 네트워크 연결 제거
□ 네트워크 세그멘테이션
```

### 네트워크 정책

```
□ 기본 거부 정책 (Deny All)
□ 최소 권한 원칙 (Least Privilege)
□ 명시적 허용만
□ Ingress + Egress 모두 제어
□ DNS 예외 처리
```

### 암호화

```
□ TLS 1.2 이상 (TLS 1.3 권장)
□ 강력한 암호화 스위트
□ 인증서 관리 (자동 갱신)
□ mTLS (상호 인증)
□ Overlay 네트워크 암호화
```

### 방화벽

```
□ iptables 규칙 설정
□ 신뢰할 수 있는 소스만 허용
□ Rate Limiting
□ DDoS 방어
□ 로깅 활성화
```

### 모니터링

```
□ 네트워크 트래픽 로깅
□ 침입 탐지 시스템 (Falco)
□ 정기적인 보안 감사
□ 이상 징후 알림
□ 접근 로그 분석
```

---

## 🚫 안티패턴

### 1. 모든 포트 노출

```yaml
# ❌ 모든 포트 퍼블리시
services:
  app:
    ports:
      - "3000:3000"
      - "5432:5432"  # Database!
      - "6379:6379"  # Redis!
# 내부 서비스가 외부 노출

# ✅ 필요한 포트만
services:
  app:
    ports:
      - "3000:3000"
  db:
    # 포트 퍼블리시 안 함 (내부만)
  redis:
    # 포트 퍼블리시 안 함
```

### 2. 암호화 없는 통신

```yaml
# ❌ HTTP (평문)
services:
  nginx:
    ports:
      - "80:80"

# ✅ HTTPS (암호화)
services:
  nginx:
    ports:
      - "443:443"
    volumes:
      - ./certs:/etc/nginx/certs:ro
```

### 3. Root 권한 실행

```dockerfile
# ❌ Root로 실행
FROM nginx
# 기본 user가 root

# ✅ 비특권 사용자
FROM nginx
RUN chown -R nginx:nginx /var/cache/nginx /var/run
USER nginx
```

### 4. 네트워크 정책 없음

```yaml
# ❌ 정책 없음
# 모든 Pod이 자유롭게 통신

# ✅ Zero Trust
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

## 🎓 핵심 정리

### 1. 방어 계층 (Defense in Depth)

```
Layer 7: Application
- Input validation
- Authentication/Authorization
- Rate limiting

Layer 4: Network Policy
- Allow/Deny rules
- Pod-to-Pod control

Layer 3: Firewall
- iptables rules
- IP filtering
- Port control

Layer 2: Network Isolation
- Separate networks
- VLANs
- Segmentation

Layer 1: Encryption
- TLS/mTLS
- Encrypted overlay
- At-rest encryption
```

### 2. Zero Trust 원칙

```
Never Trust, Always Verify:
1. 기본 거부 (Deny All)
2. 명시적 허용 (Explicit Allow)
3. 최소 권한 (Least Privilege)
4. 마이크로 세그멘테이션
5. 지속적 검증
```

### 3. 암호화

```
TLS:
- 전송 중 암호화
- 인증서 기반 신뢰
- Forward Secrecy

mTLS:
- 양방향 인증
- 클라이언트 검증
- Zero Trust

At Rest:
- 디스크 암호화
- Secret 관리
- Key rotation
```

### 4. 핵심 명령어

```bash
# 네트워크 정책
kubectl get networkpolicy
kubectl describe networkpolicy <name>

# iptables
sudo iptables -L -n
sudo iptables-save

# TLS 테스트
openssl s_client -connect host:443
curl --cacert ca.pem --cert cert.pem --key key.pem

# 모니터링
tcpdump -i docker0
kubectl logs -l app=falco
```

---

## 📚 참고 자료

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Docker Security](https://docs.docker.com/engine/security/)
- [OWASP Container Security](https://owasp.org/www-project-container-security/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)

---

## 🤔 생각해볼 문제

1. NetworkPolicy의 Ingress와 Egress를 모두 차단하면 DNS도 막힐까?
2. mTLS가 성능에 미치는 영향은?
3. iptables DOCKER-USER 체인을 사용하는 이유는?

> 💡 **답변**: 1) 예, DNS도 막힘 - DNS는 kube-system 네임스페이스의 kube-dns 서비스로 UDP 53번 포트 Egress 필요, 명시적으로 DNS Egress를 허용해야 함 (`to: namespaceSelector + port: 53/UDP`), 기본 거부 정책 적용 시 DNS 예외를 항상 추가해야 함, 2) TLS 핸드셰이크 오버헤드 약 1-2ms 추가, 암호화/복호화 CPU 사용 5-15% 증가, 처리량 10-20% 감소 가능, 하지만 TLS 1.3 + AES-NI 하드웨어 가속으로 영향 최소화, 보안 이점이 성능 손실보다 훨씬 큼, 3) DOCKER 체인은 Docker가 자동 관리하므로 사용자 규칙이 삭제될 수 있음, DOCKER-USER 체인은 사용자 정의 규칙 전용으로 Docker가 건드리지 않음, 컨테이너 재시작/네트워크 변경 시에도 유지됨, Docker 엔진보다 우선순위 높음 (먼저 평가)

---

<div align="center">

**[⬅️ 이전: Load Balancing](./08-Load-Balancing.md)** | **[다음 섹션: Storage ➡️](../storage/01-Volume-Types.md)**

</div>
