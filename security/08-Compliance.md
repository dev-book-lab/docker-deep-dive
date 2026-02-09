# 08. Compliance - 규정 준수

## 🎯 이 챕터에서 배울 것

- **CIS Docker Benchmark** - 업계 표준 보안 가이드
- **PCI-DSS** - 결제 카드 데이터 보안
- **HIPAA** - 의료 정보 보안
- **SOC 2** - 서비스 조직 통제
- **자동화된 컴플라이언스** - 지속적 준수 검증

## 📌 왜 중요한가?

**"컴플라이언스는 법적 요구사항이자 고객 신뢰의 기반입니다."**

```
컴플라이언스 없이 vs 컴플라이언스 준수:

컴플라이언스 무시:
┌─────────────────────────────────────┐
│ 1. 보안 사고 발생                      │
│    - 데이터 유출                      │
│    - 고객 정보 탈취                    │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. 법적 제재                          │
│    - GDPR: 최대 €20M 또는 4% 매출      │
│    - PCI-DSS: 카드 처리 정지           │
│    - HIPAA: 최대 $1.5M 벌금           │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 비즈니스 영향                       │
│    - 고객 신뢰 상실                    │
│    - 주가 하락                        │
│    - 계약 해지                        │
│    - 파산 위험                        │
└─────────────────────────────────────┘

실제 사례:
┌─────────────────────────────────────┐
│ Equifax (2017)                      │
│ - 1억 4천만 명 정보 유출                │
│ - 벌금: $700M                        │
│ - CEO 사임                           │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ British Airways (2018)              │
│ - GDPR 위반                          │
│ - 벌금: £20M                         │
│ - 브랜드 이미지 손상                    │
└─────────────────────────────────────┘

컴플라이언스 준수:
┌─────────────────────────────────────┐
│ 1. 사전 예방                          │
│    - CIS Benchmark 적용              │
│    - 자동화된 검증                     │
│    - 지속적 모니터링                    │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. 인증 획득                          │
│    - PCI-DSS Level 1                │
│    - SOC 2 Type II                  │
│    - ISO 27001                      │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 비즈니스 가치                       │
│    - 엔터프라이즈 고객 확보              │
│    - 보험료 절감                      │
│    - 경쟁 우위                        │
│    - 투자 유치                        │
└─────────────────────────────────────┘

주요 규정 개요:

CIS Docker Benchmark:
┌──────────────────────────────────────┐
│ 230+ 보안 권장사항                      │
├──────────────────────────────────────┤
│ 1. Host Configuration (호스트 설정)     │
│    - 파티션 분리                        │
│    - 커널 강화                         │
│    - auditd 활성화                     │
│                                      │
│ 2. Docker Daemon (데몬 설정)           │
│    - TLS 인증                         │
│    - User namespace                  │
│    - 로깅 설정                         │
│                                      │
│ 3. Daemon Files (데몬 파일)            │
│    - /etc/docker 권한                 │
│    - daemon.json 보호                 │
│                                      │
│ 4. Container Images (이미지)           │
│    - 신뢰할 수 있는 레지스트리             │
│    - Content Trust                   │
│    - 이미지 스캔                        │
│                                      │
│ 5. Container Runtime (런타임)          │
│    - 최소 권한                         │
│    - Read-only 파일시스템               │
│    - Capabilities 제한                │
│                                      │
│ 6. Security Operations (보안 운영)     │
│    - 감사 로그                         │
│    - 모니터링                          │
│    - 인시던트 대응                      │
└──────────────────────────────────────┘

PCI-DSS (Payment Card Industry):
┌──────────────────────────────────────┐
│ 신용카드 데이터 보호                      │
├──────────────────────────────────────┤
│ 1. 방화벽 설정                          │
│    - 네트워크 분리                      │
│    - DMZ 구성                         │
│                                      │
│ 2. 암호화                              │
│    - 전송 중: TLS 1.2+                 │
│    - 저장 시: AES-256                  │
│                                      │
│ 3. 접근 제어                           │
│    - MFA 필수                         │
│    - 최소 권한                         │
│                                      │
│ 4. 모니터링                            │
│    - 로그 보관 (1년)                    │
│    - 실시간 감사                        │
│                                      │
│ 5. 취약점 관리                          │
│    - 정기 스캔                         │
│    - 패치 관리                         │
│                                      │
│ 6. 보안 정책                           │
│    - 문서화                           │
│    - 정기 검토                         │
└──────────────────────────────────────┘

HIPAA (Healthcare):
┌──────────────────────────────────────┐
│ 의료 정보 보호                          │
├──────────────────────────────────────┤
│ 1. 암호화                              │
│    - PHI 데이터 암호화                  │
│    - 백업 암호화                        │
│                                      │
│ 2. 접근 제어                           │
│    - 역할 기반 접근 (RBAC)              │
│    - 감사 추적                         │
│                                      │
│ 3. 데이터 무결성                        │
│    - 변경 감지                         │
│    - 버전 관리                         │
│                                      │
│ 4. 재해 복구                           │
│    - 백업 계획                         │
│    - 복구 테스트                        │
│                                      │
│ 5. 인증                               │
│    - 강력한 암호                        │
│    - 세션 타임아웃                      │
└──────────────────────────────────────┘

SOC 2 (Service Organization Control):
┌──────────────────────────────────────┐
│ 서비스 조직 통제                         │
├──────────────────────────────────────┤
│ Trust Service Criteria:              │
│                                      │
│ 1. Security (보안)                    │
│    - 접근 제어                         │
│    - 변경 관리                         │
│                                      │
│ 2. Availability (가용성)              │
│    - 모니터링                          │
│    - 백업                             │
│                                      │
│ 3. Processing Integrity (무결성)       │
│    - 데이터 검증                        │
│    - 오류 처리                         │
│                                      │
│ 4. Confidentiality (기밀성)            │
│    - 암호화                            │
│    - 데이터 분류                        │
│                                      │
│ 5. Privacy (프라이버시)                 │
│    - 동의 관리                         │
│    - 데이터 보존                        │
└──────────────────────────────────────┘

컴플라이언스 자동화:

Traditional Manual Audit:
┌─────────────────────────────────────┐
│ 1. 감사 준비 (1개월)                   │
│    - 문서 수집                        │
│    - 증거 준비                        │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. 감사 실시 (1-2주)                   │
│    - 감사관 방문                       │
│    - 샘플 테스트                       │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 보고서 (2주)                       │
│    - 부적합 사항 발견                   │
│    - 시정 조치 요구                    │
└─────────────────────────────────────┘
Total: 2-3개월, 높은 비용

Automated Continuous Compliance:
┌─────────────────────────────────────┐
│ 1. 자동 검증 (매일)                    │
│    - CIS Benchmark 스캔              │
│    - 정책 준수 확인                    │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 2. 실시간 리포트                       │
│    - 대시보드                         │
│    - 알림                            │
└────────────┬────────────────────────┘
             │
┌────────────▼────────────────────────┐
│ 3. 자동 수정                          │
│    - 정책 위반 → 자동 롤백              │
│    - 증거 수집                        │
└─────────────────────────────────────┘
Total: 실시간, 낮은 비용
```

**실무 영향:**
- 법적 리스크 제거 → 벌금 회피
- 고객 신뢰 → 매출 증대
- 보험료 절감 → 비용 감소
- 시장 접근성 → 엔터프라이즈 확장

---

## 🔧 실습 1: CIS Docker Benchmark

### Step 1: Docker Bench Security 설치

```bash
# Docker Bench Security 다운로드
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security

# 실행
sudo sh docker-bench-security.sh

# 또는 컨테이너로 실행
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /etc:/etc:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --label docker_bench_security \
  docker/docker-bench-security

# 출력 예시:
# ...
# [INFO] 1 - Host Configuration
# [PASS] 1.1.1 - Ensure a separate partition for containers has been created
# [WARN] 1.1.2 - Ensure only trusted users are allowed to control Docker daemon
# ...
# [INFO] 2 - Docker daemon configuration
# [PASS] 2.1 - Ensure network traffic is restricted between containers
# [WARN] 2.2 - Ensure the logging level is set to 'info'
# ...
```

### Step 2: CIS Benchmark 주요 권장사항 적용

```bash
# 1. Host Configuration

# 1.1 별도 파티션 생성 (초기 설치 시)
# /var/lib/docker를 별도 파티션에 마운트

# 1.2 Docker 그룹 멤버 최소화
getent group docker
sudo gpasswd -d unnecessary_user docker

# 1.3 Auditd 설정
sudo apt-get install -y auditd
sudo systemctl enable auditd
sudo systemctl start auditd

# Audit 규칙 추가
sudo tee -a /etc/audit/rules.d/docker.rules <<EOF
-w /usr/bin/dockerd -k docker
-w /var/lib/docker -k docker
-w /etc/docker -k docker
-w /usr/lib/systemd/system/docker.service -k docker
-w /usr/lib/systemd/system/docker.socket -k docker
-w /etc/default/docker -k docker
-w /etc/docker/daemon.json -k docker
-w /usr/bin/containerd -k docker
-w /usr/bin/runc -k docker
EOF

sudo systemctl restart auditd

# 2. Docker Daemon Configuration

# 2.1 daemon.json 최적 설정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "icc": false,
  "userns-remap": "default",
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "tls": true,
  "tlsverify": true,
  "tlscacert": "/etc/docker/certs/ca.pem",
  "tlscert": "/etc/docker/certs/server-cert.pem",
  "tlskey": "/etc/docker/certs/server-key.pem"
}
EOF

sudo systemctl restart docker

# 3. Daemon Files 권한

# 3.1 /etc/docker 권한
sudo chown root:root /etc/docker
sudo chmod 755 /etc/docker

# 3.2 daemon.json 권한
sudo chown root:root /etc/docker/daemon.json
sudo chmod 644 /etc/docker/daemon.json

# 3.3 TLS 인증서 권한
sudo chmod 0400 /etc/docker/certs/*.pem

# 3.4 Docker socket 권한
sudo chown root:docker /var/run/docker.sock
sudo chmod 660 /var/run/docker.sock

# 4. Container Images

# 4.1 Content Trust 활성화
export DOCKER_CONTENT_TRUST=1
echo 'export DOCKER_CONTENT_TRUST=1' >> ~/.bashrc

# 5. Container Runtime

# 컨테이너 실행 시 보안 옵션 적용 (예시)
docker run -d \
  --name secure-app \
  --user 1000:1000 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --read-only \
  --tmpfs /tmp \
  --security-opt=no-new-privileges:true \
  --security-opt apparmor=docker-default \
  --memory=512m \
  --cpus=0.5 \
  --pids-limit=100 \
  myapp:latest
```

### Step 3: 자동화된 CIS 검증

```bash
# InSpec 설치 (Chef)
curl https://omnitruck.chef.io/install.sh | sudo bash -s -- -P inspec

# CIS Docker Benchmark 프로필
git clone https://github.com/dev-sec/cis-docker-benchmark.git
cd cis-docker-benchmark

# 검증 실행
sudo inspec exec . -t ssh://root@localhost

# JSON 리포트
sudo inspec exec . -t ssh://root@localhost --reporter json > cis-report.json

# HTML 리포트
sudo inspec exec . -t ssh://root@localhost --reporter html > cis-report.html
```

### Step 4: CI/CD 통합

```yaml
# .gitlab-ci.yml
cis-benchmark:
  stage: compliance
  image: docker:24-dind
  services:
    - docker:24-dind
  script:
    - apk add --no-cache git
    - git clone https://github.com/docker/docker-bench-security.git
    - cd docker-bench-security
    - sh docker-bench-security.sh -l /tmp/bench.log
    - |
      # WARN/FAIL이 있으면 실패
      if grep -qE '\[WARN\]|\[FAIL\]' /tmp/bench.log; then
        cat /tmp/bench.log
        exit 1
      fi
  artifacts:
    paths:
      - docker-bench-security/bench.log
    expire_in: 30 days
  allow_failure: false
  only:
    - main
```

---

## 🔧 실습 2: PCI-DSS 준수

### Step 1: 네트워크 분리 (PCI Requirement 1)

```bash
# Docker 네트워크 생성
docker network create \
  --driver overlay \
  --subnet=10.0.1.0/24 \
  --opt encrypted \
  pci-dmz

docker network create \
  --driver overlay \
  --subnet=10.0.2.0/24 \
  --opt encrypted \
  --internal \
  pci-cardholder-data

# 방화벽 규칙 (iptables)
sudo tee /etc/iptables/pci-rules.v4 <<'EOF'
*filter
# Default policies
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

# Allow established connections
-A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH (restricted)
-A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Allow HTTPS only
-A INPUT -p tcp --dport 443 -j ACCEPT

# Drop all other inbound
-A INPUT -j DROP

# Logging
-A INPUT -j LOG --log-prefix "PCI-INPUT-DROP: " --log-level 7
-A FORWARD -j LOG --log-prefix "PCI-FORWARD-DROP: " --log-level 7

COMMIT
EOF

sudo iptables-restore < /etc/iptables/pci-rules.v4
```

### Step 2: 암호화 (PCI Requirement 2 & 4)

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    image: nginx:alpine
    networks:
      - pci-dmz
    ports:
      - "443:443"
    volumes:
      - tls-certs:/etc/nginx/certs:ro
    environment:
      - TLS_MIN_VERSION=1.2
    deploy:
      replicas: 2

  app:
    image: myapp:latest
    networks:
      - pci-dmz
      - pci-cardholder-data
    secrets:
      - db_encryption_key
    environment:
      - DB_ENCRYPT_AT_REST=true
      - DB_TLS_REQUIRED=true
    deploy:
      replicas: 3

  db:
    image: postgres:15-alpine
    networks:
      - pci-cardholder-data
    volumes:
      - db-data:/var/lib/postgresql/data
    secrets:
      - db_password
      - db_encryption_key
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
      # PAN (Primary Account Number) 암호화
      - PGCRYPTO_ENABLED=true
    command: >
      postgres
      -c ssl=on
      -c ssl_cert_file=/run/secrets/db_cert
      -c ssl_key_file=/run/secrets/db_key
      -c ssl_ca_file=/run/secrets/ca_cert
    deploy:
      placement:
        constraints:
          - node.labels.pci_environment == cardholder_data_environment

networks:
  pci-dmz:
    driver: overlay
    encrypted: true
  pci-cardholder-data:
    driver: overlay
    encrypted: true
    internal: true

volumes:
  tls-certs:
  db-data:
    driver: local
    driver_opts:
      type: none
      o: bind,encryption=aes256
      device: /mnt/encrypted/db-data

secrets:
  db_password:
    external: true
  db_encryption_key:
    external: true
  db_cert:
    external: true
  db_key:
    external: true
  ca_cert:
    external: true
```

### Step 3: 접근 제어 및 감사 (PCI Requirement 7-10)

```bash
# 1. 강력한 인증
cat > enforce-mfa.sh <<'EOF'
#!/bin/bash

# MFA 필수 검증
if ! command -v google-authenticator &> /dev/null; then
    echo "ERROR: MFA not configured"
    exit 1
fi

# 세션 타임아웃 (15분)
export TMOUT=900

# 로그인 기록
logger -p auth.info "Docker access by $USER from $(echo $SSH_CLIENT | awk '{print $1}')"
EOF

# 2. Audit 로깅
sudo tee -a /etc/audit/rules.d/pci.rules <<'EOF'
# PCI-DSS Requirement 10: 로그 및 모니터링

# 네트워크 접근
-a always,exit -F arch=b64 -S connect -k pci_network_access

# 파일 접근 (카드 데이터)
-w /var/lib/docker/volumes/cardholder-data -p wa -k pci_cardholder_access

# 인증 이벤트
-w /var/log/auth.log -p wa -k pci_authentication

# Docker 명령
-w /usr/bin/docker -p x -k pci_docker_execution
EOF

sudo systemctl restart auditd

# 3. 로그 보관 (1년)
sudo tee /etc/logrotate.d/pci-docker <<'EOF'
/var/log/docker/*.log {
    daily
    rotate 365
    compress
    delaycompress
    notifempty
    create 0640 root docker
    sharedscripts
    postrotate
        docker kill -s HUP $(docker ps -q) 2>/dev/null || true
    endscript
}
EOF

# 4. 중앙 집중식 로깅
docker service create \
  --name log-aggregator \
  --mount type=bind,source=/var/log,target=/logs:ro \
  --publish 514:514/udp \
  --replicas 2 \
  --constraint 'node.role==manager' \
  fluent/fluentd:latest
```

### Step 4: PCI 스캔 자동화

```bash
# Nessus/OpenVAS 취약점 스캔 (분기별)
cat > pci-quarterly-scan.sh <<'EOF'
#!/bin/bash

# ASV (Approved Scanning Vendor) 스캔 시뮬레이션
docker run --rm \
  --network host \
  tenable/nessus:latest \
  nessuscli scan --targets="$(hostname -I)" \
  --policy=PCI-DSS \
  --output=/reports/pci-scan-$(date +%Y%m%d).html

# 결과 이메일 발송
if [ $? -eq 0 ]; then
    mail -s "PCI Quarterly Scan Passed" compliance@company.com < /reports/pci-scan-*.html
else
    mail -s "PCI Quarterly Scan FAILED - ACTION REQUIRED" compliance@company.com < /reports/pci-scan-*.html
fi
EOF

# Cron 설정 (분기별 1일)
# 0 2 1 1,4,7,10 * /path/to/pci-quarterly-scan.sh
```

---

## 🔧 실습 3: HIPAA 준수

### Step 1: PHI 데이터 암호화

```bash
# 1. 데이터베이스 암호화
cat > init-hipaa-db.sql <<'EOF'
-- HIPAA PHI 암호화

-- 암호화 확장
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- 암호화 함수
CREATE OR REPLACE FUNCTION encrypt_phi(text)
RETURNS bytea AS $$
BEGIN
    RETURN pgp_sym_encrypt($1, current_setting('app.encryption_key'));
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- 복호화 함수
CREATE OR REPLACE FUNCTION decrypt_phi(bytea)
RETURNS text AS $$
BEGIN
    RETURN pgp_sym_decrypt($1, current_setting('app.encryption_key'));
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- PHI 테이블
CREATE TABLE patient_data (
    patient_id UUID PRIMARY KEY,
    encrypted_ssn BYTEA NOT NULL,
    encrypted_name BYTEA NOT NULL,
    encrypted_diagnosis BYTEA NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    accessed_by VARCHAR(255),  -- 감사 추적
    access_time TIMESTAMP
);

-- 감사 트리거
CREATE OR REPLACE FUNCTION audit_phi_access()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO phi_access_log (
        patient_id,
        user_id,
        action,
        timestamp
    ) VALUES (
        NEW.patient_id,
        current_user,
        TG_OP,
        NOW()
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER phi_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON patient_data
FOR EACH ROW EXECUTE FUNCTION audit_phi_access();
EOF

# 2. 볼륨 암호화 (LUKS)
sudo cryptsetup luksFormat /dev/sdb
sudo cryptsetup luksOpen /dev/sdb hipaa-data
sudo mkfs.ext4 /dev/mapper/hipaa-data
sudo mount /dev/mapper/hipaa-data /mnt/hipaa-data

# 3. Docker 볼륨 설정
docker volume create \
  --driver local \
  --opt type=none \
  --opt o=bind,encryption=aes256 \
  --opt device=/mnt/hipaa-data \
  hipaa-volume
```

### Step 2: 접근 제어 및 감사 추적

```yaml
# docker-compose-hipaa.yml
version: '3.8'

services:
  app:
    image: hipaa-app:latest
    environment:
      - ENABLE_AUDIT_LOG=true
      - SESSION_TIMEOUT=900  # 15분
      - REQUIRE_MFA=true
    volumes:
      - hipaa-volume:/data
    secrets:
      - encryption_key
    deploy:
      labels:
        - "com.example.hipaa=true"
        - "com.example.phi-access=restricted"

  audit-logger:
    image: audit-logger:latest
    volumes:
      - audit-logs:/var/log/audit
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command: >
      --watch-containers=label=com.example.hipaa=true
      --log-phi-access
      --retention=6years  # HIPAA 요구사항

volumes:
  hipaa-volume:
    driver: local
    driver_opts:
      type: none
      o: bind,encryption=aes256
      device: /mnt/hipaa-data
  audit-logs:
    driver: local

secrets:
  encryption_key:
    external: true
```

### Step 3: 백업 및 재해 복구

```bash
# HIPAA 백업 스크립트
cat > hipaa-backup.sh <<'EOF'
#!/bin/bash

BACKUP_DIR="/mnt/encrypted-backups"
DATE=$(date +%Y%m%d_%H%M%S)

# 1. 데이터베이스 백업 (암호화)
docker exec hipaa-db pg_dump -U postgres | \
  gpg --encrypt --recipient backups@company.com > \
  $BACKUP_DIR/db-backup-$DATE.sql.gpg

# 2. 볼륨 백업 (암호화)
tar -czf - /mnt/hipaa-data | \
  gpg --encrypt --recipient backups@company.com > \
  $BACKUP_DIR/volume-backup-$DATE.tar.gz.gpg

# 3. 백업 검증
gpg --decrypt $BACKUP_DIR/db-backup-$DATE.sql.gpg | head -10

# 4. 오프사이트 복사
aws s3 cp $BACKUP_DIR/db-backup-$DATE.sql.gpg \
  s3://hipaa-backups-encrypted/ \
  --sse aws:kms \
  --sse-kms-key-id $KMS_KEY_ID

# 5. 로그
echo "Backup completed: $DATE" >> $BACKUP_DIR/backup.log

# 6. 7년 이상 된 백업 삭제
find $BACKUP_DIR -name "*.gpg" -mtime +2555 -delete  # 7년 = 2555일
EOF

# Cron (매일 새벽 2시)
# 0 2 * * * /path/to/hipaa-backup.sh
```

---

## 🔧 실습 4: SOC 2 준수

### Step 1: 보안 제어 (Security)

```bash
# 1. 변경 관리
cat > change-management.sh <<'EOF'
#!/bin/bash

# 모든 변경사항 기록
CHANGE_LOG="/var/log/change-management.log"

log_change() {
    echo "[$(date)] User: $USER, Action: $1, Target: $2" >> $CHANGE_LOG
}

# Docker 명령 감사
docker() {
    case "$1" in
        run|create|update|rm)
            log_change "docker $@" "$2"
            ;;
    esac
    command docker "$@"
}

export -f docker
EOF

# 2. 접근 로그
docker service create \
  --name access-logger \
  --mode global \
  --mount type=bind,source=/var/log,target=/logs \
  access-logger:latest \
  --enable-soc2-compliance
```

### Step 2: 가용성 (Availability)

```yaml
# High Availability 구성
version: '3.8'

services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      placement:
        max_replicas_per_node: 1
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 40s

  lb:
    image: nginx:alpine
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.role == manager
```

### Step 3: 데이터 무결성 (Processing Integrity)

```bash
# 데이터 무결성 검증
cat > integrity-check.sh <<'EOF'
#!/bin/bash

# 1. 데이터베이스 체크섬
docker exec db-container psql -U postgres -c "
    SELECT table_name, md5(string_agg(row::text, ''))
    FROM (
        SELECT * FROM important_table ORDER BY id
    ) AS row
    GROUP BY table_name;
"

# 2. 파일 무결성 (AIDE)
docker run --rm \
  -v /mnt/data:/data:ro \
  aide:latest \
  --check /data

# 3. 변경 감지 알림
if [ $? -ne 0 ]; then
    curl -X POST https://alerts.company.com/api/alert \
      -d '{"severity":"high","message":"Data integrity violation detected"}'
fi
EOF
```

---

## 💡 주요 명령어 정리

### Docker Bench Security

```bash
# 실행
sh docker-bench-security.sh

# 컨테이너로 실행
docker run --rm --net host --pid host --userns host \
  --cap-add audit_control \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security

# 로그 저장
sh docker-bench-security.sh -l /tmp/bench.log
```

### InSpec

```bash
# CIS 프로필 실행
inspec exec cis-docker-benchmark

# JSON 리포트
inspec exec cis-docker-benchmark --reporter json > report.json

# HTML 리포트
inspec exec cis-docker-benchmark --reporter html > report.html
```

### Audit

```bash
# Audit 규칙 추가
auditctl -w /etc/docker -p wa -k docker

# 로그 검색
ausearch -k docker

# 리포트
aureport -k
```

---

## 🎓 연습 문제

### 문제 1: CIS Benchmark 적용

Docker Bench Security를 실행하고 WARN/FAIL 항목을 모두 수정하세요.

<details>
<summary>정답 보기</summary>

```bash
# 1. 실행 및 결과 확인
sh docker-bench-security.sh -l /tmp/bench.log
grep -E 'WARN|FAIL' /tmp/bench.log

# 2. 주요 수정사항
# daemon.json 설정
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "icc": false,
  "userns-remap": "default",
  "live-restore": true,
  "no-new-privileges": true
}
EOF

# 파일 권한
sudo chmod 644 /etc/docker/daemon.json
sudo chmod 660 /var/run/docker.sock

# Auditd
sudo apt-get install auditd
# 규칙 추가 (위 실습 참고)

# 3. 재실행 및 검증
sudo systemctl restart docker
sh docker-bench-security.sh
```

</details>

### 문제 2: PCI-DSS 네트워크 분리

PCI-DSS 요구사항에 따라 DMZ와 카드 데이터 환경을 분리하는 Docker 네트워크를 구성하세요.

<details>
<summary>정답 보기</summary>

위의 "실습 2: PCI-DSS 준수 - Step 1" 참고

</details>

### 문제 3: HIPAA 백업 자동화

HIPAA 요구사항에 맞는 암호화된 백업 스크립트를 작성하고 7년 보관 정책을 구현하세요.

<details>
<summary>정답 보기</summary>

위의 "실습 3: HIPAA 준수 - Step 3" 참고

</details>

---

## 📌 핵심 요약

### 주요 규정 비교

| 규정 | 적용 대상 | 주요 요구사항 | 벌금 |
|-----|----------|-------------|------|
| **CIS Benchmark** | 모든 조직 | 230+ 보안 권장사항 | N/A (권장사항) |
| **PCI-DSS** | 카드 처리 | 암호화, 네트워크 분리 | 카드 처리 정지 |
| **HIPAA** | 의료 | PHI 암호화, 6년 보관 | 최대 $1.5M |
| **SOC 2** | SaaS | 5가지 신뢰 기준 | 고객 신뢰 상실 |
| **GDPR** | EU 데이터 | 개인정보 보호 | €20M/4% 매출 |

### CIS Docker Benchmark 핵심

```
1. Host (20%)
   - 별도 파티션
   - Auditd
   - User namespace

2. Docker Daemon (25%)
   - TLS 인증
   - 로깅
   - icc=false

3. Daemon Files (10%)
   - 파일 권한
   - 소유권

4. Container Images (15%)
   - Content Trust
   - 이미지 스캔
   - 신뢰할 수 있는 레지스트리

5. Container Runtime (25%)
   - 최소 권한
   - Read-only
   - Capabilities

6. Security Operations (5%)
   - 모니터링
   - 감사
```

### 컴플라이언스 자동화

```yaml
# CI/CD 파이프라인
stages:
  - compliance-check
  - security-scan
  - deploy

cis-benchmark:
  script:
    - docker-bench-security.sh
  allow_failure: false

pci-network-scan:
  script:
    - nessus-scan.sh
  only:
    - schedules  # 분기별

hipaa-backup:
  script:
    - backup.sh
  schedule:
    - cron: "0 2 * * *"  # 매일

soc2-audit:
  script:
    - collect-evidence.sh
  artifacts:
    expire_in: 7 years
```

### 실무 체크리스트

**공통:**
- [ ] CIS Benchmark 90% 이상 준수
- [ ] 자동화된 검증 (CI/CD)
- [ ] 정기 감사 (연 1회)
- [ ] 증거 수집 자동화
- [ ] 인시던트 대응 계획
- [ ] 정기 교육

**PCI-DSS:**
- [ ] 네트워크 분리 (DMZ)
- [ ] TLS 1.2+ 암호화
- [ ] 로그 1년 보관
- [ ] 분기별 ASV 스캔
- [ ] 연간 침투 테스트
- [ ] 카드 데이터 암호화

**HIPAA:**
- [ ] PHI 암호화 (저장/전송)
- [ ] MFA 필수
- [ ] 6년 로그 보관
- [ ] 백업 암호화
- [ ] 접근 감사 추적
- [ ] BAA (Business Associate Agreement)

**SOC 2:**
- [ ] 5가지 신뢰 기준 충족
- [ ] 변경 관리 프로세스
- [ ] 고가용성 구성
- [ ] 데이터 무결성 검증
- [ ] 정기 침투 테스트
- [ ] Type II 보고서 (1년)

### 비용-효과 분석

```
수동 컴플라이언스:
- 초기 감사: $50K-$100K
- 연간 유지: $30K-$50K
- 인력: 2-3 FTE
- 시간: 2-3개월/년

자동화된 컴플라이언스:
- 초기 설정: $20K-$30K
- 연간 유지: $5K-$10K
- 인력: 0.5 FTE
- 시간: 실시간

절감: ~70-80%
```

---

<div align="center">

**[⬅️ 이전: Security Scanning Tools](./07-Security-Scanning-Tools.md)** | **[홈으로 🏠](../README.md)**

</div>
