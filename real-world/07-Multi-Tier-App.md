# 07. Multi-Tier App - 다층 아키텍처 완전 구현

## 🎯 이 챕터에서 배울 것

- **Multi-tier Architecture**: Frontend, Backend, Database
- **통합 구성**: 모든 컴포넌트 통합
- **Production Setup**: 실전 배포 구성
- **Scaling**: 수평적 확장
- **전체 워크플로우**: 개발 → 배포 → 운영
- **실전 프로젝트**: E-commerce 애플리케이션

## 📌 왜 중요한가?

**"실제 프로덕션 애플리케이션은 여러 계층과 서비스의 조화입니다."**

```
Multi-Tier Architecture:

Simple (Single Container):
┌─────────────────────────────────────────────────┐
│  ┌────────────────────┐                         │
│  │  Monolithic App    │                         │
│  │  (All-in-one)      │                         │
│  └────────────────────┘                         │
│                                                 │
│  문제:                                          │
│  ❌ 확장 어려움                                  │
│  ❌ 장애 전파                                    │
│  ❌ 배포 복잡                                    │
└─────────────────────────────────────────────────┘

Multi-Tier (Production):
┌─────────────────────────────────────────────────┐
│                                                 │
│         Internet                                │
│            ↓                                    │
│    ┌───────────────┐                            │
│    │     Nginx     │ ← Reverse Proxy, SSL       │
│    │ Load Balancer │                            │
│    └───────┬───────┘                            │
│            │                                    │
│    ┌───────┴────────┐                           │
│    │                │                           │
│    ▼                ▼                           │
│ ┌──────┐        ┌──────┐                        │
│ │React │        │React │ ← Frontend (Replica)   │
│ │  x2  │        │  x2  │                        │
│ └──┬───┘        └──┬───┘                        │
│    │               │                            │
│    └───────┬───────┘                            │
│            ▼                                    │
│    ┌───────────────┐                            │
│    │  API Gateway  │ ← Rate Limiting, Auth      │
│    └───────┬───────┘                            │
│            │                                    │
│    ┌───────┴────────┬────────────┐              │
│    ▼                ▼            ▼              │
│ ┌─────┐         ┌─────┐      ┌─────┐           │
│ │Node │         │Node │      │Node │           │
│ │ x3  │         │ x3  │      │ x3  │           │
│ └──┬──┘         └──┬──┘      └──┬──┘           │
│    │               │            │              │
│    └───────┬───────┴────────────┘              │
│            ▼                                    │
│    ┌───────────────┐                            │
│    │  PostgreSQL   │ ← Database                 │
│    │  (Primary)    │                            │
│    └───────┬───────┘                            │
│            │                                    │
│    ┌───────┴───────┐                            │
│    ▼               ▼                            │
│ ┌─────┐         ┌─────┐                         │
│ │Redis│         │S3   │                         │
│ │Cache│         │Store│                         │
│ └─────┘         └─────┘                         │
│                                                 │
│  장점:                                          │
│  ✅ 독립적 확장                                  │
│  ✅ 장애 격리                                    │
│  ✅ 유연한 배포                                  │
└─────────────────────────────────────────────────┘

Complete Stack:
┌─────────────────────────────────────────────────┐
│ 1. Infrastructure Layer                         │
│    - Nginx (Reverse Proxy)                      │
│    - Traefik (Load Balancer)                    │
│    - Let's Encrypt (SSL)                        │
│                                                 │
│ 2. Application Layer                            │
│    - Frontend (React)                           │
│    - Backend API (Node.js)                      │
│    - Microservices                              │
│                                                 │
│ 3. Data Layer                                   │
│    - PostgreSQL (Primary DB)                    │
│    - Redis (Cache)                              │
│    - MongoDB (Documents)                        │
│                                                 │
│ 4. Monitoring Layer                             │
│    - Prometheus (Metrics)                       │
│    - Grafana (Visualization)                    │
│    - ELK (Logs)                                 │
│                                                 │
│ 5. Support Services                             │
│    - Backup (Automated)                         │
│    - CI/CD (GitHub Actions)                     │
│    - Alerting (Slack)                           │
└─────────────────────────────────────────────────┘

Deployment Flow:
┌─────────────────────────────────────────────────┐
│ Developer                                       │
│   ↓                                             │
│ Git Push                                        │
│   ↓                                             │
│ CI/CD Pipeline                                  │
│   ├─ Build Images                               │
│   ├─ Run Tests                                  │
│   ├─ Security Scan                              │
│   └─ Push to Registry                           │
│   ↓                                             │
│ Production Deployment                           │
│   ├─ Rolling Update                             │
│   ├─ Health Check                               │
│   └─ Rollback (if failed)                       │
│   ↓                                             │
│ Monitoring                                      │
│   ├─ Metrics                                    │
│   ├─ Logs                                       │
│   └─ Alerts                                     │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **확장성**: 트래픽 증가에 대응
- **안정성**: 장애 격리 및 복구
- **유지보수**: 독립적 배포 가능
- **성능**: 최적화된 아키텍처

---

## 🔧 실습 1: E-commerce 프로젝트 구조

### Step 1: 전체 프로젝트 구조

```
ecommerce-app/
├── frontend/                    # React Frontend
│   ├── Dockerfile
│   ├── Dockerfile.prod
│   ├── package.json
│   └── src/
│
├── backend/                     # Node.js API
│   ├── api/                     # API Services
│   ├── auth/                    # Auth Service
│   ├── products/                # Products Service
│   ├── orders/                  # Orders Service
│   ├── Dockerfile
│   └── package.json
│
├── nginx/                       # Reverse Proxy
│   ├── nginx.conf
│   └── ssl/
│
├── database/                    # Database Setup
│   ├── init-scripts/
│   │   ├── 01-schema.sql
│   │   └── 02-seed.sql
│   └── backups/
│
├── monitoring/                  # Monitoring Stack
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── rules/
│   ├── grafana/
│   │   └── dashboards/
│   └── alertmanager/
│
├── logging/                     # Logging Stack
│   ├── logstash/
│   ├── elasticsearch/
│   └── kibana/
│
├── scripts/                     # Utility Scripts
│   ├── backup.sh
│   ├── deploy.sh
│   └── health-check.sh
│
├── docker-compose.yml           # Development
├── docker-compose.prod.yml      # Production
├── docker-compose.monitoring.yml
└── .env
```

---

## 🔧 실습 2: Backend Services (Microservices)

### Step 1: API Gateway

```javascript
// backend/api-gateway/server.js
const express = require('express');
const httpProxy = require('http-proxy');
const rateLimit = require('express-rate-limit');
const logger = require('./logger');

const app = express();
const proxy = httpProxy.createProxyServer();

// Rate Limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

app.use(limiter);
app.use(express.json());

// Health Check
app.get('/health', (req, res) => {
  res.json({ status: 'OK' });
});

// Proxy to Services
const services = {
  auth: 'http://auth-service:3001',
  products: 'http://products-service:3002',
  orders: 'http://orders-service:3003'
};

// Auth Routes
app.all('/api/auth/*', (req, res) => {
  logger.info('Proxying to auth service', { path: req.path });
  proxy.web(req, res, { target: services.auth });
});

// Products Routes
app.all('/api/products/*', (req, res) => {
  logger.info('Proxying to products service', { path: req.path });
  proxy.web(req, res, { target: services.products });
});

// Orders Routes
app.all('/api/orders/*', (req, res) => {
  logger.info('Proxying to orders service', { path: req.path });
  proxy.web(req, res, { target: services.orders });
});

// Error Handling
proxy.on('error', (err, req, res) => {
  logger.error('Proxy error', { error: err.message });
  res.status(502).json({ error: 'Bad Gateway' });
});

app.listen(8080, () => {
  logger.info('API Gateway running on port 8080');
});
```

### Step 2: Products Service

```javascript
// backend/products-service/server.js
const express = require('express');
const { pool } = require('./db');
const logger = require('./logger');
const redis = require('./redis');

const app = express();
app.use(express.json());

// Get all products (with caching)
app.get('/api/products', async (req, res) => {
  try {
    // Check cache
    const cached = await redis.get('products:all');
    if (cached) {
      logger.info('Cache hit: products');
      return res.json(JSON.parse(cached));
    }

    // Query database
    const result = await pool.query(`
      SELECT id, name, description, price, stock, image_url
      FROM products
      WHERE active = true
      ORDER BY created_at DESC
    `);

    // Cache result (5 minutes)
    await redis.setEx('products:all', 300, JSON.stringify(result.rows));

    logger.info('Products fetched', { count: result.rows.length });
    res.json(result.rows);
  } catch (err) {
    logger.error('Failed to fetch products', { error: err.message });
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Get product by ID
app.get('/api/products/:id', async (req, res) => {
  try {
    const { id } = req.params;

    const result = await pool.query(
      'SELECT * FROM products WHERE id = $1 AND active = true',
      [id]
    );

    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'Product not found' });
    }

    logger.info('Product fetched', { product_id: id });
    res.json(result.rows[0]);
  } catch (err) {
    logger.error('Failed to fetch product', { error: err.message });
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Create product (Admin only)
app.post('/api/products', async (req, res) => {
  try {
    const { name, description, price, stock } = req.body;

    const result = await pool.query(
      'INSERT INTO products (name, description, price, stock) VALUES ($1, $2, $3, $4) RETURNING *',
      [name, description, price, stock]
    );

    // Invalidate cache
    await redis.del('products:all');

    logger.info('Product created', { product_id: result.rows[0].id });
    res.status(201).json(result.rows[0]);
  } catch (err) {
    logger.error('Failed to create product', { error: err.message });
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.listen(3002, () => {
  logger.info('Products service running on port 3002');
});
```

### Step 3: Orders Service

```javascript
// backend/orders-service/server.js
const express = require('express');
const { pool } = require('./db');
const logger = require('./logger');

const app = express();
app.use(express.json());

// Create order
app.post('/api/orders', async (req, res) => {
  const client = await pool.connect();

  try {
    const { user_id, items } = req.body;

    await client.query('BEGIN');

    // Create order
    const orderResult = await client.query(
      'INSERT INTO orders (user_id, status) VALUES ($1, $2) RETURNING *',
      [user_id, 'pending']
    );

    const orderId = orderResult.rows[0].id;

    // Add order items
    for (const item of items) {
      await client.query(
        'INSERT INTO order_items (order_id, product_id, quantity, price) VALUES ($1, $2, $3, $4)',
        [orderId, item.product_id, item.quantity, item.price]
      );

      // Update stock
      await client.query(
        'UPDATE products SET stock = stock - $1 WHERE id = $2',
        [item.quantity, item.product_id]
      );
    }

    await client.query('COMMIT');

    logger.info('Order created', { order_id: orderId, user_id });
    res.status(201).json(orderResult.rows[0]);
  } catch (err) {
    await client.query('ROLLBACK');
    logger.error('Failed to create order', { error: err.message });
    res.status(500).json({ error: 'Failed to create order' });
  } finally {
    client.release();
  }
});

// Get user orders
app.get('/api/orders/user/:userId', async (req, res) => {
  try {
    const { userId } = req.params;

    const result = await pool.query(`
      SELECT o.*, 
             json_agg(json_build_object(
               'product_id', oi.product_id,
               'quantity', oi.quantity,
               'price', oi.price
             )) as items
      FROM orders o
      LEFT JOIN order_items oi ON o.id = oi.order_id
      WHERE o.user_id = $1
      GROUP BY o.id
      ORDER BY o.created_at DESC
    `, [userId]);

    res.json(result.rows);
  } catch (err) {
    logger.error('Failed to fetch orders', { error: err.message });
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.listen(3003, () => {
  logger.info('Orders service running on port 3003');
});
```

---

## 🔧 실습 3: Frontend (React)

### Step 1: Product List Component

```javascript
// frontend/src/components/ProductList.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:8080';

function ProductList() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchProducts();
  }, []);

  const fetchProducts = async () => {
    try {
      setLoading(true);
      const response = await axios.get(`${API_URL}/api/products`);
      setProducts(response.data);
      setError(null);
    } catch (err) {
      setError('Failed to load products');
      console.error(err);
    } finally {
      setLoading(false);
    }
  };

  const addToCart = async (product) => {
    try {
      await axios.post(`${API_URL}/api/cart`, {
        product_id: product.id,
        quantity: 1
      });
      alert('Added to cart!');
    } catch (err) {
      alert('Failed to add to cart');
    }
  };

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div className="product-list">
      <h2>Products</h2>
      <div className="products-grid">
        {products.map(product => (
          <div key={product.id} className="product-card">
            <img src={product.image_url} alt={product.name} />
            <h3>{product.name}</h3>
            <p>{product.description}</p>
            <p className="price">${product.price}</p>
            <p className="stock">Stock: {product.stock}</p>
            <button onClick={() => addToCart(product)}>
              Add to Cart
            </button>
          </div>
        ))}
      </div>
    </div>
  );
}

export default ProductList;
```

---

## 🔧 실습 4: 완전한 Production Docker Compose

### Step 1: Production 구성

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  # ============ Infrastructure ============
  
  # Nginx Reverse Proxy
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
      - certbot_www:/var/www/certbot
    depends_on:
      - frontend
      - api-gateway
    networks:
      - frontend
      - backend

  # ============ Application ============
  
  # Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.prod
    restart: always
    expose:
      - "80"
    networks:
      - frontend

  # API Gateway
  api-gateway:
    build: ./backend/api-gateway
    restart: always
    expose:
      - "8080"
    environment:
      - NODE_ENV=production
    depends_on:
      - auth-service
      - products-service
      - orders-service
    networks:
      - backend

  # Auth Service
  auth-service:
    build: ./backend/auth-service
    restart: always
    expose:
      - "3001"
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis
    networks:
      - backend

  # Products Service (3 replicas)
  products-service:
    build: ./backend/products-service
    restart: always
    deploy:
      replicas: 3
    expose:
      - "3002"
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis
    networks:
      - backend

  # Orders Service
  orders-service:
    build: ./backend/orders-service
    restart: always
    expose:
      - "3003"
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
    depends_on:
      - postgres
    networks:
      - backend

  # ============ Data Layer ============
  
  # PostgreSQL
  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  # Redis Cache
  redis:
    image: redis:7-alpine
    restart: always
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - backend

  # ============ Monitoring ============
  
  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    restart: always
    volumes:
      - ./monitoring/prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  # Grafana
  grafana:
    image: grafana/grafana:latest
    restart: always
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - monitoring

  # Node Exporter
  node-exporter:
    image: prom/node-exporter:latest
    restart: always
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring

  # cAdvisor
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    restart: always
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    privileged: true
    networks:
      - monitoring

  # ============ Logging ============
  
  # Elasticsearch
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    restart: always
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    networks:
      - logging

  # Kibana
  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    restart: always
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - logging

  # ============ Backup ============
  
  # Backup Service
  backup:
    image: postgres:15-alpine
    restart: always
    volumes:
      - ./scripts:/scripts
      - ./backups:/backups
    environment:
      - DB_HOST=postgres
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
    command: >
      sh -c "
      while true; do
        echo 'Running backup at' \$(date)
        /scripts/backup.sh
        sleep 86400
      done
      "
    depends_on:
      - postgres
    networks:
      - backend

volumes:
  postgres_data:
  redis_data:
  prometheus_data:
  grafana_data:
  elasticsearch_data:
  certbot_www:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
  monitoring:
    driver: bridge
  logging:
    driver: bridge
```

---

## 🔧 실습 5: Database Schema

### Step 1: Schema SQL

```sql
-- database/init-scripts/01-schema.sql

-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INTEGER DEFAULT 0,
    image_url VARCHAR(500),
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    status VARCHAR(50) DEFAULT 'pending',
    total_amount DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Order items table
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER NOT NULL,
    price DECIMAL(10, 2) NOT NULL
);

-- Indexes
CREATE INDEX idx_products_active ON products(active);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

```sql
-- database/init-scripts/02-seed.sql

-- Sample users
INSERT INTO users (email, password_hash, name) VALUES
    ('admin@example.com', '$2b$10$hash...', 'Admin User'),
    ('user1@example.com', '$2b$10$hash...', 'John Doe'),
    ('user2@example.com', '$2b$10$hash...', 'Jane Smith');

-- Sample products
INSERT INTO products (name, description, price, stock, image_url) VALUES
    ('Laptop', 'High-performance laptop', 1299.99, 50, '/images/laptop.jpg'),
    ('Mouse', 'Wireless gaming mouse', 49.99, 100, '/images/mouse.jpg'),
    ('Keyboard', 'Mechanical keyboard', 129.99, 75, '/images/keyboard.jpg'),
    ('Monitor', '27-inch 4K monitor', 399.99, 30, '/images/monitor.jpg'),
    ('Headset', 'Noise-canceling headset', 79.99, 60, '/images/headset.jpg');
```

---

## 🔧 실습 6: 배포 스크립트

### Step 1: Deploy Script

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

echo "=== Starting Deployment ==="

# 1. Pull latest code
echo "📥 Pulling latest code..."
git pull origin main

# 2. Build images
echo "🔨 Building Docker images..."
docker-compose -f docker-compose.prod.yml build

# 3. Run database migrations
echo "🗄️  Running migrations..."
docker-compose -f docker-compose.prod.yml run --rm \
  api-gateway npm run migrate

# 4. Backup database
echo "💾 Backing up database..."
./scripts/backup.sh

# 5. Deploy with zero downtime
echo "🚀 Deploying services..."
docker-compose -f docker-compose.prod.yml up -d --no-deps --build

# 6. Health check
echo "🏥 Running health checks..."
sleep 10
./scripts/health-check.sh

if [ $? -eq 0 ]; then
  echo "✅ Deployment successful!"
else
  echo "❌ Deployment failed! Rolling back..."
  docker-compose -f docker-compose.prod.yml down
  exit 1
fi

# 7. Cleanup
echo "🧹 Cleaning up..."
docker system prune -f

echo "=== Deployment Complete ==="
```

### Step 2: Health Check Script

```bash
#!/bin/bash
# scripts/health-check.sh

SERVICES=(
  "http://localhost/health"
  "http://localhost/api/health"
  "http://localhost:9090/-/healthy"
  "http://localhost:3000/api/health"
)

for service in "${SERVICES[@]}"; do
  echo "Checking $service..."
  
  response=$(curl -s -o /dev/null -w "%{http_code}" $service)
  
  if [ $response -eq 200 ]; then
    echo "✅ $service is healthy"
  else
    echo "❌ $service is unhealthy (HTTP $response)"
    exit 1
  fi
done

echo "✅ All services are healthy"
```

---

## 💡 확장 전략

```
Horizontal Scaling:
┌────────────────────────────────────────────────┐
│ docker-compose up --scale products-service=5   │
│                                                │
│ Load Balancer                                  │
│       ├─→ products-service-1                   │
│       ├─→ products-service-2                   │
│       ├─→ products-service-3                   │
│       ├─→ products-service-4                   │
│       └─→ products-service-5                   │
└────────────────────────────────────────────────┘

Database Scaling:
┌────────────────────────────────────────────────┐
│ Primary (Write)                                │
│       ↓                                        │
│ ┌─────┴──────┬──────────┐                     │
│ ▼            ▼          ▼                      │
│ Replica-1  Replica-2  Replica-3 (Read)        │
└────────────────────────────────────────────────┘

Caching Strategy:
┌────────────────────────────────────────────────┐
│ Request → Check Redis                          │
│    ├─ Hit → Return cached                      │
│    └─ Miss → Query DB → Cache → Return         │
└────────────────────────────────────────────────┘
```

---

## 📌 핵심 요약

```
Multi-Tier App 핵심:
1. 계층 분리 (Frontend, Backend, Data)
2. 마이크로서비스 아키텍처
3. Load Balancing
4. Caching (Redis)
5. 모니터링 및 로깅
6. 자동 백업
7. CI/CD 파이프라인

Best Practices:
✅ 서비스 독립성
✅ Health Check 필수
✅ Rolling Update
✅ 자동 복구
✅ 수평적 확장
✅ 중앙 집중식 모니터링
✅ 정기 백업
```

---

<div align="center">

**[⬅️ 이전: Backup System](./06-Backup-System.md)** | **[🏠 Real World 섹션 완료!](./README.md)**

</div>
