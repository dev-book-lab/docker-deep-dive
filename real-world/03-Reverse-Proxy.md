# 03. Reverse Proxy - Nginx/Traefik 설정

## 🎯 이 챕터에서 배울 것

- **Nginx 기본**: Reverse Proxy 설정
- **Nginx 고급**: Load Balancing, SSL
- **Traefik**: 자동 SSL, 서비스 디스커버리
- **Let's Encrypt**: 자동 SSL 인증서
- **실전 구성**: Production-ready 설정
- **성능 최적화**: Caching, Compression

## 📌 왜 중요한가?

**"Reverse Proxy는 모든 트래픽의 진입점이며, 보안과 성능의 핵심입니다."**

```
Reverse Proxy의 역할:

Without Reverse Proxy:
┌─────────────────────────────────────────────────┐
│ User → http://api:8080                          │
│                                                 │
│ 문제:                                           │
│ ❌ 포트 노출 (8080, 3000, 5000...)              │
│ ❌ HTTPS 없음                                    │
│ ❌ 도메인 라우팅 불가                            │
│ ❌ Load Balancing 없음                           │
└─────────────────────────────────────────────────┘

With Reverse Proxy (Nginx):
┌─────────────────────────────────────────────────┐
│ User → https://myapp.com                        │
│         ↓                                       │
│    ┌─────────┐                                  │
│    │  Nginx  │ (Reverse Proxy)                  │
│    │  :80    │                                  │
│    │  :443   │                                  │
│    └────┬────┘                                  │
│         │                                       │
│         ├─→ /         → Frontend:3000           │
│         ├─→ /api      → Backend:8080            │
│         └─→ /admin    → Admin:5000              │
│                                                 │
│ 장점:                                           │
│ ✅ 단일 진입점 (80/443)                          │
│ ✅ HTTPS 자동 (Let's Encrypt)                    │
│ ✅ 도메인 기반 라우팅                            │
│ ✅ Load Balancing                                │
│ ✅ SSL Termination                               │
└─────────────────────────────────────────────────┘

Architecture:
┌─────────────────────────────────────────────────┐
│                                                 │
│  Internet                                       │
│     ↓                                           │
│  ┌──────────────────┐                           │
│  │  Reverse Proxy   │                           │
│  │  (Nginx/Traefik) │                           │
│  └────────┬─────────┘                           │
│           │                                     │
│   ┌───────┼───────┬─────────┐                  │
│   ▼       ▼       ▼         ▼                  │
│ ┌────┐ ┌────┐ ┌────┐   ┌────────┐             │
│ │ F1 │ │ F2 │ │API │   │Database│             │
│ └────┘ └────┘ └────┘   └────────┘             │
│                                                 │
│ Features:                                       │
│  - SSL/TLS Termination                          │
│  - Load Balancing                               │
│  - Caching                                      │
│  - Compression (gzip)                           │
│  - Rate Limiting                                │
│  - Access Control                               │
└─────────────────────────────────────────────────┘

Nginx vs Traefik:
┌────────────────┬────────────┬────────────┐
│ 특성            │ Nginx      │ Traefik    │
├────────────────┼────────────┼────────────┤
│ 설정            │ 파일 기반   │ 자동 감지   │
├────────────────┼────────────┼────────────┤
│ SSL             │ 수동 설정   │ 자동 (LE)  │
├────────────────┼────────────┼────────────┤
│ 학습 곡선       │ 높음        │ 낮음       │
├────────────────┼────────────┼────────────┤
│ 성능            │ 매우 높음   │ 높음       │
├────────────────┼────────────┼────────────┤
│ 사용 사례       │ 정적 설정   │ 동적 환경  │
└────────────────┴────────────┴────────────┘
```

**실무 영향:**
- **보안**: HTTPS 강제, 인증서 관리
- **성능**: Caching, Compression
- **확장성**: Load Balancing
- **관리**: 중앙 집중식 라우팅

---

## 🔧 실습 1: Nginx 기본 Reverse Proxy

### Step 1: 간단한 Nginx 설정

```nginx
# nginx/nginx.conf
server {
    listen 80;
    server_name localhost;

    # Frontend (React)
    location / {
        proxy_pass http://frontend:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Backend API
    location /api {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - frontend
      - backend
    networks:
      - app-network

  # Frontend
  frontend:
    build: ./frontend
    container_name: frontend
    # 포트 외부 노출 안 함 (Nginx를 통해서만)
    expose:
      - "3000"
    networks:
      - app-network

  # Backend
  backend:
    build: ./backend
    container_name: backend
    expose:
      - "8080"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

```bash
# 실행
docker-compose up -d

# 확인
curl http://localhost/api/health
curl http://localhost/

# Nginx 로그
docker logs nginx -f
```

---

## 🔧 실습 2: Nginx 프로덕션 설정

### Step 1: 완전한 Nginx 설정

```nginx
# nginx/nginx.conf
# Upstream 정의 (Backend 서버들)
upstream backend {
    least_conn;  # Load balancing 알고리즘
    server backend1:8080 max_fails=3 fail_timeout=30s;
    server backend2:8080 max_fails=3 fail_timeout=30s;
}

# Rate Limiting
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

# Cache 설정
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m use_temp_path=off;

server {
    listen 80;
    server_name myapp.com www.myapp.com;

    # HTTP → HTTPS 리다이렉트
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.com www.myapp.com;

    # SSL 인증서
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    
    # SSL 설정
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json application/javascript;

    # Frontend (Static Files)
    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
        
        # Cache static files
        location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    # API (Backend)
    location /api {
        # Rate Limiting
        limit_req zone=api_limit burst=20 nodelay;
        
        proxy_pass http://backend;
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffering
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        
        # Caching (GET only)
        proxy_cache my_cache;
        proxy_cache_valid 200 60m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        add_header X-Cache-Status $upstream_cache_status;
    }

    # WebSocket
    location /ws {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;
    }

    # Health Check
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Admin (IP 제한)
    location /admin {
        allow 10.0.0.0/8;    # 내부망만
        allow 172.16.0.0/12;
        deny all;
        
        proxy_pass http://backend;
        proxy_set_header Host $host;
    }
}
```

### Step 2: Docker Compose (Load Balancing)

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./frontend/build:/usr/share/nginx/html:ro
      - nginx_cache:/var/cache/nginx
    depends_on:
      - backend1
      - backend2
    networks:
      - app-network

  # Backend 인스턴스 1
  backend1:
    build: ./backend
    container_name: backend1
    restart: always
    expose:
      - "8080"
    environment:
      - INSTANCE_ID=1
    networks:
      - app-network

  # Backend 인스턴스 2
  backend2:
    build: ./backend
    container_name: backend2
    restart: always
    expose:
      - "8080"
    environment:
      - INSTANCE_ID=2
    networks:
      - app-network

  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    restart: always
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  nginx_cache:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

---

## 🔧 실습 3: Let's Encrypt SSL (Certbot)

### Step 1: Certbot 설정

```yaml
# docker-compose.yml (SSL 추가)
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - certbot_conf:/etc/letsencrypt
      - certbot_www:/var/www/certbot
    command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
    networks:
      - app-network

  # Certbot
  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot_conf:/etc/letsencrypt
      - certbot_www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  certbot_conf:
  certbot_www:

networks:
  app-network:
```

```nginx
# nginx/nginx.conf (Let's Encrypt 지원)
server {
    listen 80;
    server_name myapp.com www.myapp.com;

    # Let's Encrypt validation
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # HTTP → HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name myapp.com www.myapp.com;

    # SSL (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;
    
    # ... (나머지 설정)
}
```

```bash
# SSL 인증서 발급
docker-compose up -d nginx

docker-compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email admin@myapp.com \
  --agree-tos \
  --no-eff-email \
  -d myapp.com \
  -d www.myapp.com

# Nginx 재시작
docker-compose restart nginx
```

---

## 🔧 실습 4: Traefik (자동 SSL)

### Step 1: Traefik 기본 설정

```yaml
# docker-compose.traefik.yml
version: '3.8'

services:
  # Traefik
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: always
    command:
      # API
      - "--api.insecure=true"
      - "--api.dashboard=true"
      
      # Docker Provider
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      
      # Entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      
      # Let's Encrypt
      - "--certificatesresolvers.letsencrypt.acme.email=admin@myapp.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"  # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_letsencrypt:/letsencrypt
    networks:
      - app-network

  # Frontend
  frontend:
    build: ./frontend
    container_name: frontend
    labels:
      - "traefik.enable=true"
      
      # HTTP
      - "traefik.http.routers.frontend.rule=Host(`myapp.com`) || Host(`www.myapp.com`)"
      - "traefik.http.routers.frontend.entrypoints=web"
      
      # HTTPS
      - "traefik.http.routers.frontend-secure.rule=Host(`myapp.com`) || Host(`www.myapp.com`)"
      - "traefik.http.routers.frontend-secure.entrypoints=websecure"
      - "traefik.http.routers.frontend-secure.tls.certresolver=letsencrypt"
      
      # HTTP → HTTPS Redirect
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      - "traefik.http.routers.frontend.middlewares=redirect-to-https"
      
      - "traefik.http.services.frontend.loadbalancer.server.port=3000"
    networks:
      - app-network

  # Backend
  backend:
    build: ./backend
    container_name: backend
    labels:
      - "traefik.enable=true"
      
      # API Routes
      - "traefik.http.routers.backend.rule=Host(`myapp.com`) && PathPrefix(`/api`)"
      - "traefik.http.routers.backend.entrypoints=web"
      
      - "traefik.http.routers.backend-secure.rule=Host(`myapp.com`) && PathPrefix(`/api`)"
      - "traefik.http.routers.backend-secure.entrypoints=websecure"
      - "traefik.http.routers.backend-secure.tls.certresolver=letsencrypt"
      
      - "traefik.http.routers.backend.middlewares=redirect-to-https"
      
      - "traefik.http.services.backend.loadbalancer.server.port=8080"
    networks:
      - app-network

volumes:
  traefik_letsencrypt:

networks:
  app-network:
    driver: bridge
```

### Step 2: Traefik 고급 설정

```yaml
# docker-compose.traefik-advanced.yml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: always
    command:
      # API
      - "--api.dashboard=true"
      
      # Docker
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      
      # Entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      
      # HTTP → HTTPS Redirect
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      
      # Let's Encrypt
      - "--certificatesresolvers.letsencrypt.acme.email=admin@myapp.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      
      # Logging
      - "--log.level=INFO"
      - "--accesslog=true"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_letsencrypt:/letsencrypt
    labels:
      # Dashboard
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.myapp.com`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dashboard.service=api@internal"
      
      # Dashboard Auth
      - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$hash..."
      - "traefik.http.routers.dashboard.middlewares=auth"
    networks:
      - app-network

  backend:
    build: ./backend
    deploy:
      replicas: 3  # 3개 인스턴스 (Load Balancing)
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.backend.rule=Host(`api.myapp.com`)"
      - "traefik.http.routers.backend.entrypoints=websecure"
      - "traefik.http.routers.backend.tls.certresolver=letsencrypt"
      
      # Rate Limiting
      - "traefik.http.middlewares.rate-limit.ratelimit.average=100"
      - "traefik.http.middlewares.rate-limit.ratelimit.burst=50"
      
      # Compression
      - "traefik.http.middlewares.compress.compress=true"
      
      # Apply Middlewares
      - "traefik.http.routers.backend.middlewares=rate-limit,compress"
      
      - "traefik.http.services.backend.loadbalancer.server.port=8080"
    networks:
      - app-network

volumes:
  traefik_letsencrypt:

networks:
  app-network:
```

---

## 🔧 실습 5: 멀티 도메인 및 서브도메인

### Step 1: 여러 도메인 설정

```nginx
# nginx/sites/myapp.conf
server {
    listen 443 ssl http2;
    server_name myapp.com www.myapp.com;
    
    ssl_certificate /etc/nginx/ssl/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/myapp.com/privkey.pem;
    
    location / {
        proxy_pass http://frontend:3000;
    }
    
    location /api {
        proxy_pass http://backend:8080;
    }
}

# nginx/sites/admin.conf
server {
    listen 443 ssl http2;
    server_name admin.myapp.com;
    
    ssl_certificate /etc/nginx/ssl/admin.myapp.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/admin.myapp.com/privkey.pem;
    
    # IP 제한
    allow 10.0.0.0/8;
    deny all;
    
    location / {
        proxy_pass http://admin:5000;
    }
}

# nginx/sites/api.conf
server {
    listen 443 ssl http2;
    server_name api.myapp.com;
    
    ssl_certificate /etc/nginx/ssl/api.myapp.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/api.myapp.com/privkey.pem;
    
    location / {
        proxy_pass http://backend:8080;
    }
}
```

### Step 2: Traefik 멀티 도메인

```yaml
# docker-compose.yml
services:
  # Main App
  frontend:
    labels:
      - "traefik.http.routers.frontend.rule=Host(`myapp.com`, `www.myapp.com`)"
  
  # Admin
  admin:
    labels:
      - "traefik.http.routers.admin.rule=Host(`admin.myapp.com`)"
      
      # IP Whitelist
      - "traefik.http.middlewares.admin-whitelist.ipwhitelist.sourcerange=10.0.0.0/8,172.16.0.0/12"
      - "traefik.http.routers.admin.middlewares=admin-whitelist"
  
  # API
  api:
    labels:
      - "traefik.http.routers.api.rule=Host(`api.myapp.com`)"
```

---

## 💡 성능 최적화 패턴

```nginx
# Cache 설정
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=cache:10m max_size=1g;

# Static Files Caching
location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# Gzip Compression
gzip on;
gzip_types text/plain text/css application/json application/javascript;

# HTTP/2
listen 443 ssl http2;

# Connection Pooling
upstream backend {
    keepalive 32;
}

# Rate Limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req zone=api burst=20 nodelay;
```

---

## 📌 핵심 요약

```
Reverse Proxy 핵심:
1. 단일 진입점 (80/443)
2. HTTPS 자동 (Let's Encrypt)
3. Load Balancing
4. Caching & Compression
5. Security Headers

Nginx vs Traefik:
- Nginx: 정적 설정, 높은 성능
- Traefik: 동적 설정, 자동 SSL

Best Practices:
✅ HTTPS 강제
✅ Security Headers
✅ Gzip Compression
✅ Caching
✅ Rate Limiting
```

---

<div align="center">

**[⬅️ 이전: Database Setup](./02-Database-Setup.md)** | **[다음: Monitoring Stack ➡️](./04-Monitoring-Stack.md)**

</div>
