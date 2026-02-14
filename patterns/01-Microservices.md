# 01. Microservices - 마이크로서비스 아키텍처

## 🎯 이 챕터에서 배울 것

- **마이크로서비스 개념**: Monolith vs Microservices
- **서비스 분리**: 도메인 기반 서비스 설계
- **서비스 간 통신**: REST, gRPC, Message Queue
- **데이터 관리**: Database per Service 패턴
- **서비스 디스커버리**: 동적 서비스 탐색
- **실전 구현**: Docker Compose로 마이크로서비스 구축

## 📌 왜 중요한가?

**"마이크로서비스는 복잡한 애플리케이션을 독립적이고 배포 가능한 작은 서비스로 분해합니다."**

```
Monolith vs Microservices:

┌─────────────────────────────────────────────────────────┐
│ Monolithic Architecture                                 │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Single Application                                │  │
│  │                                                   │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │  │
│  │  │  UI     │  │Business │  │  Data   │            │  │
│  │  │  Layer  │→ │ Logic   │→ │ Layer   │            │  │
│  │  └─────────┘  └─────────┘  └─────────┘            │  │
│  │                                                   │  │
│  │  모든 기능이 하나의 프로세스                             │  │
│  │  단일 데이터베이스                                     │  │
│  │  함께 배포됨                                         │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  장점:                                                   │
│  ✅ 개발 초기 단순함                                        │
│  ✅ 트랜잭션 처리 간단                                       │
│  ✅ 테스트 용이                                            │
│                                                         │
│  단점:                                                   │
│  ❌ 확장성 제한 (전체를 스케일)                               │
│  ❌ 기술 스택 고정                                         │
│  ❌ 배포 위험 (전체 재배포)                                  │
│  ❌ 팀 간 충돌                                            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ Microservices Architecture                              │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ User     │  │ Order    │  │ Product  │               │
│  │ Service  │  │ Service  │  │ Service  │               │
│  │          │  │          │  │          │               │
│  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │               │
│  │ │  DB  │ │  │ │  DB  │ │  │ │  DB  │ │               │
│  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Payment  │  │ Shipping │  │ Notif.   │               │
│  │ Service  │  │ Service  │  │ Service  │               │
│  │          │  │          │  │          │               │
│  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │               │
│  │ │  DB  │ │  │ │  DB  │ │  │ │  DB  │ │               │
│  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                         │
│  각 서비스:                                               │
│  - 독립적 배포                                             │
│  - 독립적 확장                                             │
│  - 전용 데이터베이스                                        │
│  - 자율적 팀 소유                                          │
│                                                         │
│  장점:                                                   │
│  ✅ 독립 배포/확장                                         │
│  ✅ 기술 스택 자유                                         │
│  ✅ 팀 자율성                                             │
│  ✅ 장애 격리                                             │
│                                                         │
│  단점:                                                   │
│  ❌ 분산 시스템 복잡도                                      │
│  ❌ 데이터 일관성                                          │
│  ❌ 네트워크 통신 오버헤드                                   │
│  ❌ 운영 복잡도                                           │
└─────────────────────────────────────────────────────────┘
```

**실무 영향:**
- **확장성**: 트래픽이 많은 서비스만 스케일
- **배포 속도**: 서비스별 독립 배포
- **기술 선택**: 서비스마다 최적 기술 사용
- **팀 생산성**: 팀별 독립적 개발

---

## 🔬 Deep Dive

### 1. 서비스 분리 전략

#### 도메인 기반 분리 (DDD)

```
서비스 분리 기준:

1. 비즈니스 도메인 (Domain-Driven Design)
┌─────────────────────────────────────────────────────┐
│ E-Commerce 도메인                                     │
│                                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐     │
│  │   User     │  │  Product   │  │   Order    │     │
│  │  Context   │  │  Catalog   │  │  Context   │     │
│  │            │  │  Context   │  │            │     │
│  │ - 회원가입   │  │ - 상품관리   │  │ - 주문생성    │     │
│  │ - 로그인     │  │ - 재고관리   │  │ - 결제      │     │
│  │ - 프로필     │  │ - 카테고리   │  │ - 배송추적   │     │
│  └────────────┘  └────────────┘  └────────────┘     │
│                                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐     │
│  │  Payment   │  │  Shipping  │  │   Review   │     │
│  │  Context   │  │  Context   │  │  Context   │     │
│  │            │  │            │  │            │     │
│  │ - 결제처리   │  │ - 배송관리   │  │ - 리뷰작성    │     │
│  │ - 환불      │  │ - 추적      │  │ - 평점       │    │
│  └────────────┘  └────────────┘  └────────────┘     │
└─────────────────────────────────────────────────────┘

2. 비즈니스 기능
- 각 Bounded Context = 하나의 마이크로서비스
- 높은 응집도, 낮은 결합도
- 팀이 이해하고 관리 가능한 크기

3. 데이터 소유권
- 각 서비스가 자신의 데이터 소유
- 다른 서비스는 API를 통해서만 접근
- Database per Service 패턴

4. 변경 빈도
- 자주 변경되는 기능 분리
- 독립적 배포 가능

5. 확장 요구사항
- 트래픽이 많은 기능 분리
- 독립적 스케일링
```

#### 잘못된 분리 (Anti-patterns)

```
❌ 나쁜 분리:

1. 기술 레이어로 분리
┌──────────┐  ┌──────────┐  ┌──────────┐
│   UI     │  │ Business │  │   Data   │
│ Service  │→ │ Service  │→ │ Service  │
└──────────┘  └──────────┘  └──────────┘
문제: 간단한 기능도 3개 서비스 거침

2. 너무 작게 분리 (Nano-services)
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│getUserById   getOrders...  (50개+)
└────┘ └────┘ └────┘ └────┘ └────┘
문제: 관리 불가능, 통신 오버헤드

3. 공유 데이터베이스
┌──────────┐  ┌──────────┐
│ Service1 │  │ Service2 │
└────┬─────┘  └────┬─────┘
     │             │
     └─────┬───────┘
           ▼
      ┌─────────┐
      │Shared DB│
      └─────────┘
문제: 강한 결합, 독립 배포 불가

✅ 좋은 분리:

1. 비즈니스 도메인 기반
┌──────────────┐  ┌──────────────┐
│ User Service │  │Order Service │
│ - Users DB   │  │ - Orders DB  │
└──────────────┘  └──────────────┘

2. 적절한 크기 (2-pizza team)
- 5-10명 팀이 관리 가능
- 1-2주 내 재작성 가능

3. 독립적 데이터
- 각 서비스가 자신의 DB 소유
- API를 통한 데이터 접근
```

---

### 2. 서비스 간 통신

#### 동기 vs 비동기 통신

```
1. 동기 통신 (Synchronous):

REST API:
┌──────────┐      HTTP       ┌──────────┐
│ Service1 │ ─────────────→  │ Service2 │
│          │ ←─────────────  │          │
└──────────┘    Response     └──────────┘

장점:
✅ 단순함
✅ 즉시 응답
✅ 디버깅 쉬움

단점:
❌ 강한 결합
❌ Service2 장애 시 Service1 영향
❌ 응답 시간 누적

gRPC:
┌──────────┐    Protobuf     ┌──────────┐
│ Service1 │ ─────────────→  │ Service2 │
│          │ ←─────────────  │          │
└──────────┘                 └──────────┘

장점:
✅ 빠른 성능
✅ 타입 안전
✅ 양방향 스트리밍

2. 비동기 통신 (Asynchronous):

Message Queue:
┌──────────┐                 ┌──────────┐
│ Service1 │                 │ Service2 │
└────┬─────┘                 └────▲─────┘
     │ Publish                    │ Subscribe
     ▼                            │
┌─────────────────────────────────┴─────┐
│ Message Queue (RabbitMQ, Kafka)       │
└───────────────────────────────────────┘

장점:
✅ 느슨한 결합
✅ 비동기 처리
✅ 탄력성 (buffering)
✅ 이벤트 기반

단점:
❌ 복잡도 증가
❌ 데이터 일관성
❌ 디버깅 어려움

Event-Driven:
┌──────────┐                 ┌──────────┐
│Publisher │ ─── Event ────→ │Subscriber│
└──────────┘                 └──────────┘
                             ┌──────────┐
                    Event ──→│Subscriber│
                             └──────────┘

장점:
✅ 완전한 분리
✅ 확장 용이
✅ 실시간 처리
```

#### 통신 패턴 선택

```
┌────────────────┬──────────────┬──────────────┐
│ 시나리오         │ REST/gRPC    │ Message Queue│
├────────────────┼──────────────┼──────────────┤
│ 조회 (Query)    │ ✅ REST      │ ❌           │
├────────────────┼──────────────┼──────────────┤
│ 실시간 필요       │ ✅ gRPC     │ ❌           │
├────────────────┼──────────────┼──────────────┤
│ 장시간 작업       │ ❌           │ ✅ MQ        │
├────────────────┼──────────────┼──────────────┤
│ 이벤트 알림       │ ❌           │ ✅ MQ        │
├────────────────┼──────────────┼──────────────┤
│ 일괄 처리        │ ❌           │ ✅ MQ        │
├────────────────┼──────────────┼──────────────┤
│ 트랜잭션 필요     │ ✅ REST       │ ❌ (Saga)   │
└────────────────┴──────────────┴──────────────┘

예시:

사용자 주문:
1. Frontend → Order Service (REST)
   - 즉시 응답 필요
   
2. Order Service → Payment Service (REST)
   - 결제 결과 즉시 확인
   
3. Order Service → Shipping Service (MQ)
   - 비동기 처리 가능
   
4. Order Service → Email Service (MQ)
   - 이벤트 기반 알림
```

---

### 3. 데이터 관리

#### Database per Service

```
Database per Service 패턴:

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ User Service │  │Order Service │  │Product Svc   │
├──────────────┤  ├──────────────┤  ├──────────────┤
│   User API   │  │  Order API   │  │ Product API  │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       ▼                 ▼                 ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Users DB   │  │  Orders DB  │  │ Products DB │
│  (Postgres) │  │   (MySQL)   │  │  (MongoDB)  │
└─────────────┘  └─────────────┘  └─────────────┘

각 서비스:
✅ 자신의 데이터 완전 소유
✅ 다른 서비스의 DB 직접 접근 금지
✅ API를 통해서만 데이터 교환
✅ 독립적 기술 스택 선택 가능

데이터 일관성:

문제: 분산 트랜잭션
┌──────────────┐      ┌──────────────┐
│ Order Service│      │Payment Svc   │
│              │      │              │
│ 1. 주문 생성   │  →   │ 2. 결제 처리   │
│              │      │              │
│ 3. 실패 시?    │  ←   │ (실패)        │
└──────────────┘      └──────────────┘

해결: Saga 패턴

Choreography Saga:
┌──────────────┐                ┌──────────────┐
│ Order Service│                │Payment Svc   │
│              │                │              │
│ OrderCreated │ ──── Event ──→ │ ProcessPmt   │
│              │                │              │
│ CompensateOrd│ ←── Event ──── │ PmtFailed    │
└──────────────┘                └──────────────┘

Orchestration Saga:
┌──────────────────────────────────┐
│     Saga Orchestrator            │
│                                  │
│  1. CreateOrder                  │
│  2. ProcessPayment               │
│  3. IF failed → CancelOrder      │
│  4. UpdateInventory              │
│  5. IF failed → RefundPayment    │
└──────────────────────────────────┘

CQRS (Command Query Responsibility Segregation):
┌──────────────┐                ┌──────────────┐
│ Write Model  │                │  Read Model  │
│  (Commands)  │                │  (Queries)   │
│              │                │              │
│  Order DB    │ ── Events ──→  │ Denormalized │
│  Product DB  │                │   View DB    │
└──────────────┘                └──────────────┘
```

---

## 🔧 실습 1: 기본 마이크로서비스 구축

### Step 1: 프로젝트 구조

```bash
# 프로젝트 디렉토리 생성
mkdir microservices-demo
cd microservices-demo

# 서비스별 디렉토리
mkdir -p user-service order-service product-service api-gateway

# 파일 구조
microservices-demo/
├── user-service/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
├── order-service/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
├── product-service/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
├── api-gateway/
│   ├── nginx.conf
│   └── Dockerfile
└── docker-compose.yml
```

### Step 2: User Service 구현

```python
# user-service/app.py
from flask import Flask, jsonify, request
import psycopg2
import os

app = Flask(__name__)

# Database 연결
def get_db():
    return psycopg2.connect(
        host=os.getenv('DB_HOST', 'user-db'),
        database=os.getenv('DB_NAME', 'users'),
        user=os.getenv('DB_USER', 'postgres'),
        password=os.getenv('DB_PASSWORD', 'password')
    )

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'service': 'user-service'})

@app.route('/users', methods=['GET'])
def get_users():
    conn = get_db()
    cur = conn.cursor()
    cur.execute('SELECT id, name, email FROM users')
    users = [{'id': row[0], 'name': row[1], 'email': row[2]} 
             for row in cur.fetchall()]
    cur.close()
    conn.close()
    return jsonify(users)

@app.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    conn = get_db()
    cur = conn.cursor()
    cur.execute('SELECT id, name, email FROM users WHERE id = %s', (user_id,))
    row = cur.fetchone()
    cur.close()
    conn.close()
    
    if row:
        return jsonify({'id': row[0], 'name': row[1], 'email': row[2]})
    return jsonify({'error': 'User not found'}), 404

@app.route('/users', methods=['POST'])
def create_user():
    data = request.get_json()
    conn = get_db()
    cur = conn.cursor()
    cur.execute(
        'INSERT INTO users (name, email) VALUES (%s, %s) RETURNING id',
        (data['name'], data['email'])
    )
    user_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    
    return jsonify({'id': user_id, 'name': data['name'], 'email': data['email']}), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```dockerfile
# user-service/Dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY ../../Downloads/app.py .

CMD ["python", "app.py"]
```

```txt
# user-service/requirements.txt
flask==2.3.0
psycopg2-binary==2.9.6
```

### Step 3: Order Service 구현

```python
# order-service/app.py
from flask import Flask, jsonify, request
import requests
import os

app = Flask(__name__)

USER_SERVICE = os.getenv('USER_SERVICE_URL', 'http://user-service:5000')
PRODUCT_SERVICE = os.getenv('PRODUCT_SERVICE_URL', 'http://product-service:5000')

# In-memory storage (실제론 DB 사용)
orders = []

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'service': 'order-service'})

@app.route('/orders', methods=['GET'])
def get_orders():
    return jsonify(orders)

@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    user_id = data['user_id']
    product_id = data['product_id']
    quantity = data['quantity']
    
    # User Service 호출
    try:
        user_response = requests.get(f'{USER_SERVICE}/users/{user_id}')
        if user_response.status_code != 200:
            return jsonify({'error': 'User not found'}), 404
        user = user_response.json()
    except Exception as e:
        return jsonify({'error': f'User service error: {str(e)}'}), 500
    
    # Product Service 호출
    try:
        product_response = requests.get(f'{PRODUCT_SERVICE}/products/{product_id}')
        if product_response.status_code != 200:
            return jsonify({'error': 'Product not found'}), 404
        product = product_response.json()
    except Exception as e:
        return jsonify({'error': f'Product service error: {str(e)}'}), 500
    
    # 주문 생성
    order = {
        'id': len(orders) + 1,
        'user': user,
        'product': product,
        'quantity': quantity,
        'total': product['price'] * quantity,
        'status': 'pending'
    }
    orders.append(order)
    
    return jsonify(order), 201

@app.route('/orders/<int:order_id>', methods=['GET'])
def get_order(order_id):
    order = next((o for o in orders if o['id'] == order_id), None)
    if order:
        return jsonify(order)
    return jsonify({'error': 'Order not found'}), 404

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Step 4: Product Service 구현

```python
# product-service/app.py
from flask import Flask, jsonify, request

app = Flask(__name__)

# In-memory storage
products = [
    {'id': 1, 'name': 'Laptop', 'price': 999.99, 'stock': 10},
    {'id': 2, 'name': 'Mouse', 'price': 29.99, 'stock': 50},
    {'id': 3, 'name': 'Keyboard', 'price': 79.99, 'stock': 30},
]

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'service': 'product-service'})

@app.route('/products', methods=['GET'])
def get_products():
    return jsonify(products)

@app.route('/products/<int:product_id>', methods=['GET'])
def get_product(product_id):
    product = next((p for p in products if p['id'] == product_id), None)
    if product:
        return jsonify(product)
    return jsonify({'error': 'Product not found'}), 404

@app.route('/products', methods=['POST'])
def create_product():
    data = request.get_json()
    product = {
        'id': len(products) + 1,
        'name': data['name'],
        'price': data['price'],
        'stock': data['stock']
    }
    products.append(product)
    return jsonify(product), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Step 5: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # User Service
  user-service:
    build: ./user-service
    ports:
      - "5001:5000"
    environment:
      - DB_HOST=user-db
      - DB_NAME=users
      - DB_USER=postgres
      - DB_PASSWORD=password
    depends_on:
      - user-db
    networks:
      - microservices

  user-db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=users
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - user-data:/var/lib/postgresql/data
      - ./user-service/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - microservices

  # Order Service
  order-service:
    build: ./order-service
    ports:
      - "5002:5000"
    environment:
      - USER_SERVICE_URL=http://user-service:5000
      - PRODUCT_SERVICE_URL=http://product-service:5000
    depends_on:
      - user-service
      - product-service
    networks:
      - microservices

  # Product Service
  product-service:
    build: ./product-service
    ports:
      - "5003:5000"
    networks:
      - microservices

  # API Gateway (Nginx)
  api-gateway:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./api-gateway/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - user-service
      - order-service
      - product-service
    networks:
      - microservices

networks:
  microservices:
    driver: bridge

volumes:
  user-data:
```

### Step 6: API Gateway 설정

```nginx
# api-gateway/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream user-service {
        server user-service:5000;
    }

    upstream order-service {
        server order-service:5000;
    }

    upstream product-service {
        server product-service:5000;
    }

    server {
        listen 80;

        # User Service
        location /api/users {
            rewrite ^/api/users(.*)$ /users$1 break;
            proxy_pass http://user-service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Order Service
        location /api/orders {
            rewrite ^/api/orders(.*)$ /orders$1 break;
            proxy_pass http://order-service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Product Service
        location /api/products {
            rewrite ^/api/products(.*)$ /products$1 break;
            proxy_pass http://product-service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Health check
        location /health {
            return 200 'API Gateway is healthy';
            add_header Content-Type text/plain;
        }
    }
}
```

### Step 7: 실행 및 테스트

```bash
# 1. 서비스 시작
docker-compose up -d

# 2. 로그 확인
docker-compose logs -f

# 3. User 생성
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe",
    "email": "john@example.com"
  }'

# 4. User 조회
curl http://localhost:8080/api/users/1

# 5. Product 조회
curl http://localhost:8080/api/products

# 6. Order 생성
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": 1,
    "product_id": 1,
    "quantity": 2
  }'

# 7. Order 조회
curl http://localhost:8080/api/orders

# 8. Health Check
curl http://localhost:8080/api/users/health
curl http://localhost:8080/api/orders/health
curl http://localhost:8080/api/products/health

# 9. 정리
docker-compose down -v
```

---

## 🔧 실습 2: 서비스 간 비동기 통신 (RabbitMQ)

### Step 1: RabbitMQ 추가

```yaml
# docker-compose.yml에 추가
services:
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=password
    networks:
      - microservices
```

### Step 2: Email Service 추가

```python
# email-service/app.py
from flask import Flask
import pika
import json
import time

app = Flask(__name__)

def connect_rabbitmq():
    credentials = pika.PlainCredentials('admin', 'password')
    parameters = pika.ConnectionParameters(
        'rabbitmq',
        5672,
        '/',
        credentials
    )
    return pika.BlockingConnection(parameters)

def send_email(order_data):
    """이메일 발송 시뮬레이션"""
    print(f"📧 Sending email for order {order_data['id']}")
    print(f"   To: {order_data['user']['email']}")
    print(f"   Subject: Order #{order_data['id']} confirmed")
    time.sleep(1)  # 이메일 발송 시뮬레이션
    print(f"✅ Email sent!")

def callback(ch, method, properties, body):
    order_data = json.loads(body)
    print(f"📨 Received order: {order_data['id']}")
    send_email(order_data)
    ch.basic_ack(delivery_tag=method.delivery_tag)

def start_consumer():
    connection = connect_rabbitmq()
    channel = connection.channel()
    
    # Queue 선언
    channel.queue_declare(queue='order_created', durable=True)
    
    # Consumer 설정
    channel.basic_qos(prefetch_count=1)
    channel.basic_consume(
        queue='order_created',
        on_message_callback=callback
    )
    
    print('📬 Email Service waiting for messages...')
    channel.start_consuming()

if __name__ == '__main__':
    # RabbitMQ 연결 대기
    while True:
        try:
            start_consumer()
        except Exception as e:
            print(f'Error: {e}. Retrying in 5 seconds...')
            time.sleep(5)
```

### Step 3: Order Service 수정 (이벤트 발행)

```python
# order-service/app.py에 추가
import pika
import json

def publish_order_created(order):
    """주문 생성 이벤트 발행"""
    try:
        credentials = pika.PlainCredentials('admin', 'password')
        parameters = pika.ConnectionParameters(
            'rabbitmq',
            5672,
            '/',
            credentials
        )
        connection = pika.BlockingConnection(parameters)
        channel = connection.channel()
        
        # Queue 선언
        channel.queue_declare(queue='order_created', durable=True)
        
        # 메시지 발행
        channel.basic_publish(
            exchange='',
            routing_key='order_created',
            body=json.dumps(order),
            properties=pika.BasicProperties(
                delivery_mode=2,  # persistent
            )
        )
        
        connection.close()
        print(f"📤 Published order_created event: {order['id']}")
    except Exception as e:
        print(f"❌ Failed to publish event: {e}")

@app.route('/orders', methods=['POST'])
def create_order():
    # ... (기존 코드)
    
    # 주문 생성
    order = {
        'id': len(orders) + 1,
        'user': user,
        'product': product,
        'quantity': quantity,
        'total': product['price'] * quantity,
        'status': 'pending'
    }
    orders.append(order)
    
    # 이벤트 발행
    publish_order_created(order)
    
    return jsonify(order), 201
```

---

## 🔧 실습 3: 서비스 디스커버리 (Consul)

### Step 1: Consul 추가

```yaml
# docker-compose.yml에 추가
services:
  consul:
    image: consul:latest
    ports:
      - "8500:8500"
      - "8600:8600/udp"
    command: agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0
    networks:
      - microservices
```

### Step 2: 서비스 등록

```python
# user-service/app.py에 추가
import requests
import socket

def register_service():
    """Consul에 서비스 등록"""
    service_name = "user-service"
    service_port = 5000
    service_id = f"{service_name}-{socket.gethostname()}"
    
    registration = {
        "ID": service_id,
        "Name": service_name,
        "Address": socket.gethostname(),
        "Port": service_port,
        "Check": {
            "HTTP": f"http://{socket.gethostname()}:{service_port}/health",
            "Interval": "10s"
        }
    }
    
    try:
        response = requests.put(
            "http://consul:8500/v1/agent/service/register",
            json=registration
        )
        if response.status_code == 200:
            print(f"✅ Registered {service_name} with Consul")
        else:
            print(f"❌ Failed to register: {response.text}")
    except Exception as e:
        print(f"❌ Consul registration error: {e}")

if __name__ == '__main__':
    register_service()
    app.run(host='0.0.0.0', port=5000)
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 패턴                  │ 설명                        │
├──────────────────────┼────────────────────────────┤
│ API Gateway          │ 단일 진입점, 라우팅             │
├──────────────────────┼────────────────────────────┤
│ Database per Service │ 서비스별 독립 DB              │
├──────────────────────┼────────────────────────────┤
│ Event-Driven         │ 비동기 이벤트 기반 통신          │
├──────────────────────┼────────────────────────────┤
│ Saga                 │ 분산 트랜잭션 관리              │
├──────────────────────┼────────────────────────────┤
│ CQRS                 │ 읽기/쓰기 모델 분리             │
├──────────────────────┼────────────────────────────┤
│ Service Discovery    │ 동적 서비스 탐색               │
├──────────────────────┼────────────────────────────┤
│ Circuit Breaker      │ 장애 격리 및 복구              │
└──────────────────────┴────────────────────────────┘
```

---

## 🎓 연습 문제

### 문제 1: Order Service에 Circuit Breaker를 추가하시오.

<details>
<summary>정답 보기</summary>

```python
# order-service/app.py
from pybreaker import CircuitBreaker

# Circuit Breaker 설정
user_service_breaker = CircuitBreaker(
    fail_max=5,
    timeout_duration=60
)

product_service_breaker = CircuitBreaker(
    fail_max=5,
    timeout_duration=60
)

@app.route('/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    
    # Circuit Breaker로 User Service 호출
    try:
        @user_service_breaker
        def get_user():
            response = requests.get(
                f'{USER_SERVICE}/users/{data["user_id"]}',
                timeout=3
            )
            response.raise_for_status()
            return response.json()
        
        user = get_user()
    except Exception as e:
        return jsonify({'error': f'User service unavailable: {str(e)}'}), 503
    
    # Circuit Breaker로 Product Service 호출
    try:
        @product_service_breaker
        def get_product():
            response = requests.get(
                f'{PRODUCT_SERVICE}/products/{data["product_id"]}',
                timeout=3
            )
            response.raise_for_status()
            return response.json()
        
        product = get_product()
    except Exception as e:
        return jsonify({'error': f'Product service unavailable: {str(e)}'}'), 503
    
    # ... 주문 생성
```

**Circuit Breaker 동작:**
1. **Closed (정상)**: 요청 정상 처리
2. **Open (차단)**: 5번 실패 시 즉시 에러 반환
3. **Half-Open (테스트)**: 60초 후 일부 요청 허용

</details>

### 문제 2: Saga 패턴으로 주문-결제-배송 트랜잭션을 구현하시오.

<details>
<summary>정답 보기</summary>

```python
# saga-orchestrator/app.py
from flask import Flask, jsonify, request
import requests

app = Flask(__name__)

class OrderSaga:
    def __init__(self, order_data):
        self.order_data = order_data
        self.state = []
    
    def execute(self):
        """Saga 실행"""
        try:
            # Step 1: 주문 생성
            order = self.create_order()
            self.state.append(('order', order))
            
            # Step 2: 결제 처리
            payment = self.process_payment(order)
            self.state.append(('payment', payment))
            
            # Step 3: 재고 감소
            inventory = self.update_inventory(order)
            self.state.append(('inventory', inventory))
            
            # Step 4: 배송 시작
            shipping = self.create_shipping(order)
            self.state.append(('shipping', shipping))
            
            return {'success': True, 'order': order}
        
        except Exception as e:
            # 실패 시 Compensate
            self.compensate()
            return {'success': False, 'error': str(e)}
    
    def create_order(self):
        response = requests.post('http://order-service:5000/orders', 
                                json=self.order_data)
        if response.status_code != 201:
            raise Exception('Order creation failed')
        return response.json()
    
    def process_payment(self, order):
        response = requests.post('http://payment-service:5000/payments',
                                json={'order_id': order['id']})
        if response.status_code != 200:
            raise Exception('Payment failed')
        return response.json()
    
    def update_inventory(self, order):
        response = requests.post('http://inventory-service:5000/decrease',
                                json={'product_id': order['product_id']})
        if response.status_code != 200:
            raise Exception('Inventory update failed')
        return response.json()
    
    def create_shipping(self, order):
        response = requests.post('http://shipping-service:5000/shipments',
                                json={'order_id': order['id']})
        if response.status_code != 201:
            raise Exception('Shipping creation failed')
        return response.json()
    
    def compensate(self):
        """보상 트랜잭션"""
        print("🔄 Starting compensation...")
        
        # 역순으로 롤백
        for step_name, step_data in reversed(self.state):
            try:
                if step_name == 'shipping':
                    requests.delete(f'http://shipping-service:5000/shipments/{step_data["id"]}')
                    print("  ✅ Shipping cancelled")
                
                elif step_name == 'inventory':
                    requests.post('http://inventory-service:5000/increase',
                                json={'product_id': step_data['product_id']})
                    print("  ✅ Inventory restored")
                
                elif step_name == 'payment':
                    requests.post(f'http://payment-service:5000/refund',
                                json={'payment_id': step_data['id']})
                    print("  ✅ Payment refunded")
                
                elif step_name == 'order':
                    requests.delete(f'http://order-service:5000/orders/{step_data["id"]}')
                    print("  ✅ Order cancelled")
            
            except Exception as e:
                print(f"  ❌ Compensation failed for {step_name}: {e}")

@app.route('/saga/orders', methods=['POST'])
def create_order_saga():
    saga = OrderSaga(request.get_json())
    result = saga.execute()
    
    if result['success']:
        return jsonify(result['order']), 201
    else:
        return jsonify(result), 400

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

</details>

### 문제 3: API Gateway에 Rate Limiting을 추가하시오.

<details>
<summary>정답 보기</summary>

```nginx
# api-gateway/nginx.conf
http {
    # Rate Limit Zone 정의
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    
    upstream user-service {
        server user-service:5000;
    }

    server {
        listen 80;

        # Rate Limiting 적용
        location /api/ {
            limit_req zone=api_limit burst=20 nodelay;
            limit_req_status 429;
            
            # Error responses
            error_page 429 = @rate_limit_error;
        }

        location @rate_limit_error {
            default_type application/json;
            return 429 '{"error": "Too many requests", "retry_after": 1}';
        }

        location /api/users {
            rewrite ^/api/users(.*)$ /users$1 break;
            proxy_pass http://user-service;
        }
    }
}
```

**또는 Python으로 구현:**

```python
# api-gateway/app.py
from flask import Flask, jsonify
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

app = Flask(__name__)
limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    default_limits=["100 per hour", "10 per minute"]
)

@app.route('/api/users')
@limiter.limit("5 per minute")
def get_users():
    # Proxy to user-service
    pass

@app.errorhandler(429)
def ratelimit_handler(e):
    return jsonify({
        'error': 'Rate limit exceeded',
        'message': str(e.description)
    }), 429
```

</details>

---

## 📌 핵심 요약

```
마이크로서비스 핵심 원칙:
1. 단일 책임: 각 서비스는 하나의 비즈니스 기능
2. 독립 배포: 서비스별 독립적 배포
3. 분산 데이터: Database per Service
4. 느슨한 결합: API를 통한 통신
5. 장애 격리: Circuit Breaker, Bulkhead

주요 패턴:
- API Gateway: 단일 진입점
- Service Discovery: 동적 서비스 탐색
- Event-Driven: 비동기 통신
- Saga: 분산 트랜잭션
- CQRS: 읽기/쓰기 분리
```

---

## 📚 참고 자료

- [Microservices Patterns](https://microservices.io/patterns/)
- [Martin Fowler - Microservices](https://martinfowler.com/articles/microservices.html)
- [Building Microservices by Sam Newman](https://www.oreilly.com/library/view/building-microservices/9781491950340/)
- [Docker Compose for Microservices](https://docs.docker.com/compose/)

---

## 🤔 생각해볼 문제

1. 마이크로서비스가 항상 Monolith보다 나은 선택인가? 언제 Monolith를 선택해야 하는가?
2. 서비스 간 통신에서 동기(REST/gRPC)와 비동기(MQ)를 어떤 기준으로 선택해야 하는가?
3. Database per Service 패턴에서 여러 서비스에 걸친 조회(Join)는 어떻게 처리해야 하는가?

> 💡 **답변**:
> 
> **1) Monolith vs Microservices 선택:**
> 
> **Monolith를 선택해야 하는 경우:**
> - 팀 크기: 10명 미만의 작은 팀
> - 프로젝트 초기: MVP, 빠른 검증 필요
> - 도메인 불명확: 비즈니스 경계가 아직 명확하지 않음
> - 트래픽 적음: 단일 서버로 충분
> - DevOps 경험 부족: 분산 시스템 운영 어려움
> 
> **Microservices를 선택해야 하는 경우:**
> - 팀 크기: 여러 팀 (20명 이상)
> - 확장 필요: 특정 기능만 독립적으로 스케일
> - 기술 다양성: 서비스마다 다른 기술 스택
> - 빈번한 배포: 서비스별 독립 배포
> - 명확한 도메인: Bounded Context가 명확
> 
> **2) 동기 vs 비동기 통신:**
> 
> **동기 통신 (REST/gRPC) 사용:**
> - 즉시 응답 필요: 사용자 인증, 조회
> - 단순한 요청-응답: CRUD 작업
> - 강한 일관성: 트랜잭션 즉시 확인
> 
> **비동기 통신 (Message Queue) 사용:**
> - 장시간 작업: 이미지 처리, 리포트 생성
> - 이벤트 알림: 주문 생성 → 이메일 발송
> - 느슨한 결합: 구독자가 언제든 추가/제거
> - 탄력성: 일시적 장애 허용
> 
> **3) 여러 서비스 조인:**
> 
> **방법 1: API Composition (Application-level Join)**
> ```python
> # Order Service에서 조합
> order = get_order(order_id)
> user = call_user_service(order.user_id)
> product = call_product_service(order.product_id)
> 
> return {
>     'order': order,
>     'user': user,
>     'product': product
> }
> ```
> 단점: N+1 문제, 성능 이슈
> 
> **방법 2: CQRS - Read Model (Denormalization)**
> ```
> Write Model (정규화):
> - User Service: users 테이블
> - Order Service: orders 테이블
> 
> Read Model (비정규화):
> - OrderView DB: orders + users 조인된 뷰
> - Event로 동기화
> ```
> 장점: 빠른 조회
> 단점: Eventual Consistency
> 
> **방법 3: Data Replication**
> ```
> Order Service에 user_name, product_name 복제
> - 조인 불필요
> - 빠른 조회
> - 주기적 동기화
> ```

---

<div align="center">

**[다음: Sidecar Pattern ➡️](./02-Sidecar-Pattern.md)**

</div>
