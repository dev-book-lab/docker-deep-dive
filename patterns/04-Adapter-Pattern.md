# 04. Adapter Pattern - 레거시 시스템 통합

## 🎯 이 챕터에서 배울 것

- **Adapter 패턴 개념**: 인터페이스 변환 및 통합
- **프로토콜 변환**: REST ↔ SOAP, HTTP ↔ gRPC
- **데이터 포맷 변환**: JSON ↔ XML, CSV ↔ JSON
- **레거시 통합**: 오래된 시스템과 현대 시스템 연결
- **API 버전 관리**: 구 버전 ↔ 신 버전 변환
- **실전 구현**: 다양한 Adapter 사례

## 📌 왜 중요한가?

**"Adapter 패턴은 호환되지 않는 시스템들을 연결하는 다리 역할을 합니다."**

```
Adapter 패턴의 핵심:

Without Adapter (호환 불가):
┌─────────────────────────────────────────────────┐
│ Modern Microservices (REST/JSON)                │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │Service A │  │Service B │  │Service C │       │
│  │(REST/    │  │(gRPC/    │  │(GraphQL/ │       │
│  │ JSON)    │  │ Protobuf)│  │ JSON)    │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┘
        │             │             │
        │ ❌ 호환 불가  │             │
        ▼             ▼             ▼
┌─────────────────────────────────────────────────┐
│ Legacy Systems                                  │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │Mainframe │  │SOAP/XML  │  │FTP/CSV   │       │
│  │(COBOL)   │  │Service   │  │Files     │       │
│  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────┘

문제점:
❌ 프로토콜 불일치 (REST vs SOAP)
❌ 데이터 포맷 차이 (JSON vs XML)
❌ 인증 방식 차이 (OAuth vs Basic Auth)
❌ 직접 수정 불가 (레거시 소스 코드 없음)
❌ 테스트 어려움

With Adapter Pattern:
┌─────────────────────────────────────────────────┐
│ Modern Microservices                            │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │Service A │  │Service B │  │Service C │       │
│  │(REST)    │  │(REST)    │  │(REST)    │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┘
        │             │             │
        │ REST/JSON   │             │
        ▼             ▼             ▼
┌─────────────────────────────────────────────────┐
│ Adapter Layer (Sidecar Containers)              │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ SOAP     │  │ gRPC     │  │ CSV      │       │
│  │ Adapter  │  │ Adapter  │  │ Adapter  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │             │             │             │
│       │ SOAP/XML    │ gRPC        │ CSV Files   │
└───────┼─────────────┼─────────────┼─────────────┘
        ▼             ▼             ▼
┌─────────────────────────────────────────────────┐
│ Legacy Systems                                  │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │Mainframe │  │SOAP/XML  │  │FTP/CSV   │       │
│  │Service   │  │Service   │  │Files     │       │
│  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────┘

장점:
✅ 레거시 수정 불필요
✅ 표준화된 인터페이스 (REST)
✅ 독립 배포 (Adapter만 업데이트)
✅ 테스트 용이 (Adapter 모킹)
✅ 단계적 마이그레이션

Adapter vs Ambassador vs Sidecar:
┌──────────────┬──────────────┬──────────────┐
│ 패턴          │ 목적          │ 예시          │
├──────────────┼──────────────┼──────────────┤
│ Sidecar      │ 보조 기능      │ 로깅, 모니터링   │
├──────────────┼──────────────┼──────────────┤
│ Ambassador   │ 외부 통신      │ DB 프록시      │
├──────────────┼──────────────┼──────────────┤
│ Adapter      │ 인터페이스      │ REST→SOAP    │
│              │ 변환          │ 변환          │
└──────────────┴──────────────┴──────────────┘
```

**실무 영향:**
- **레거시 통합**: 기존 시스템 유지하며 현대화
- **마이그레이션**: 단계적 전환 가능
- **표준화**: 일관된 API 제공
- **유지보수**: 변환 로직 중앙 관리

---

## 🔬 Deep Dive

### 1. Adapter 패턴 유형

#### 프로토콜 어댑터

```
1. REST → SOAP Adapter:

┌──────────────┐                    ┌──────────────┐
│ Modern App   │                    │ Legacy SOAP  │
│              │                    │ Service      │
│ POST /api/   │                    │ <soapenv:    │
│ users        │                    │  Envelope>   │
│              │                    │  <CreateUser>│
│ {            │                    │  </CreateUser│
│   "name":    │                    │  </Envelope> │
│   "email":   │                    │              │
│ }            │                    │              │
└──────┬───────┘                    └───────▲──────┘
       │                                    │
       │ REST Request                       │ SOAP Request
       ▼                                    │
┌───────────────────────────────────────────┴────┐
│ SOAP Adapter                                   │
│                                                │
│ 1. REST Request 받기                            │
│    POST /api/users                             │
│    {"name": "Alice", "email": "a@example.com"} │
│                                                │
│ 2. SOAP Envelope 생성                           │
│    <soapenv:Envelope>                          │
│      <soapenv:Body>                            │
│        <CreateUser>                            │
│          <name>Alice</name>                    │
│          <email>a@example.com</email>          │
│        </CreateUser>                           │
│      </soapenv:Body>                           │
│    </soapenv:Envelope>                         │
│                                                │
│ 3. SOAP 응답을 JSON으로 변환                        │
│    <CreateUserResponse>                        │
│      <userId>123</userId>                      │
│    </CreateUserResponse>                       │
│    ↓                                           │
│    {"userId": 123}                             │
└────────────────────────────────────────────────┘

2. HTTP → gRPC Adapter:

JSON Request → Protobuf → gRPC Service
JSON Response ← Protobuf ← gRPC Service

3. REST → GraphQL Adapter:

Multiple REST Calls → Single GraphQL Query
Overfetching 방지 → Precise Field Selection
```

#### 데이터 포맷 어댑터

```
1. JSON ↔ XML:

JSON:                    XML:
{                        <user>
  "name": "Alice",         <name>Alice</name>
  "age": 30                <age>30</age>
}                        </user>

2. CSV ↔ JSON:

CSV:                     JSON:
name,age                 [
Alice,30                   {"name":"Alice","age":30},
Bob,25                     {"name":"Bob","age":25}
                         ]

3. Protobuf ↔ JSON:

Binary Protobuf ↔ Human-readable JSON
```

#### API 버전 어댑터

```
v1 → v2 변환:

v1 API:                  v2 API:
GET /users/123           GET /v2/users/123

Response v1:             Response v2:
{                        {
  "id": 123,               "userId": 123,
  "name": "Alice"          "profile": {
}                            "displayName": "Alice",
                             "email": "alice@example.com"
                           }
                         }

Adapter 역할:
- v1 요청 → v2 요청 변환
- v2 응답 → v1 응답 변환
- 하위 호환성 유지
```

---

### 2. 주요 변환 패턴

#### Request/Response 변환

```python
class Adapter:
    def adapt_request(self, modern_request):
        """현대 API → 레거시 API"""
        return {
            'legacy_format': self.transform(modern_request)
        }
    
    def adapt_response(self, legacy_response):
        """레거시 API → 현대 API"""
        return {
            'modern_format': self.transform(legacy_response)
        }
```

---

## 🔧 실습 1: REST → SOAP Adapter

### Step 1: SOAP Adapter 구현

```python
# soap-adapter/adapter.py
from flask import Flask, request, jsonify
import requests
from zeep import Client
from zeep.transports import Transport

app = Flask(__name__)

# SOAP 클라이언트 설정
SOAP_WSDL = "http://legacy-soap-service/service?wsdl"
transport = Transport(timeout=10)
soap_client = Client(SOAP_WSDL, transport=transport)

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

@app.route('/api/users', methods=['POST'])
def create_user():
    """REST → SOAP 변환"""
    # 1. REST Request (JSON)
    data = request.get_json()
    name = data.get('name')
    email = data.get('email')
    
    print(f"[Adapter] Received REST request: {data}")
    
    # 2. SOAP 호출
    try:
        # SOAP CreateUser 메서드 호출
        soap_response = soap_client.service.CreateUser(
            name=name,
            email=email
        )
        
        print(f"[Adapter] SOAP response: {soap_response}")
        
        # 3. SOAP Response → JSON
        return jsonify({
            'userId': soap_response['userId'],
            'name': soap_response['name'],
            'email': soap_response['email'],
            'createdAt': str(soap_response.get('createdAt'))
        }), 201
    
    except Exception as e:
        print(f"[Adapter] Error: {e}")
        return jsonify({'error': str(e)}), 500

@app.route('/api/users/<user_id>', methods=['GET'])
def get_user(user_id):
    """REST → SOAP 조회"""
    try:
        # SOAP GetUser 메서드 호출
        soap_response = soap_client.service.GetUser(userId=int(user_id))
        
        # SOAP Response → JSON
        return jsonify({
            'userId': soap_response['userId'],
            'name': soap_response['name'],
            'email': soap_response['email']
        })
    
    except Exception as e:
        return jsonify({'error': str(e)}), 404

@app.route('/api/users/<user_id>', methods=['PUT'])
def update_user(user_id):
    """REST → SOAP 업데이트"""
    data = request.get_json()
    
    try:
        soap_response = soap_client.service.UpdateUser(
            userId=int(user_id),
            name=data.get('name'),
            email=data.get('email')
        )
        
        return jsonify({
            'userId': soap_response['userId'],
            'name': soap_response['name'],
            'email': soap_response['email']
        })
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

```txt
# soap-adapter/requirements.txt
flask==2.3.0
zeep==4.2.1
requests==2.31.0
```

### Step 2: Legacy SOAP Service (시뮬레이션)

```python
# legacy-soap/service.py
from spyne import Application, rpc, ServiceBase, Unicode, Integer
from spyne.protocol.soap import Soap11
from spyne.server.wsgi import WsgiApplication
from datetime import datetime

# 간단한 인메모리 DB
users_db = {}
user_id_counter = 1

class UserService(ServiceBase):
    @rpc(Unicode, Unicode, _returns=Unicode)
    def CreateUser(ctx, name, email):
        """사용자 생성"""
        global user_id_counter
        
        user_id = user_id_counter
        user_id_counter += 1
        
        users_db[user_id] = {
            'userId': user_id,
            'name': name,
            'email': email,
            'createdAt': datetime.now()
        }
        
        # SOAP 응답 (XML)
        return f"""
        <CreateUserResponse>
            <userId>{user_id}</userId>
            <name>{name}</name>
            <email>{email}</email>
            <createdAt>{users_db[user_id]['createdAt']}</createdAt>
        </CreateUserResponse>
        """
    
    @rpc(Integer, _returns=Unicode)
    def GetUser(ctx, userId):
        """사용자 조회"""
        if userId not in users_db:
            return "<Error>User not found</Error>"
        
        user = users_db[userId]
        return f"""
        <GetUserResponse>
            <userId>{user['userId']}</userId>
            <name>{user['name']}</name>
            <email>{user['email']}</email>
        </GetUserResponse>
        """

application = Application(
    [UserService],
    tns='legacy.soap.service',
    in_protocol=Soap11(validator='lxml'),
    out_protocol=Soap11()
)

wsgi_app = WsgiApplication(application)

if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    
    server = make_server('0.0.0.0', 8081, wsgi_app)
    print("Legacy SOAP service running on port 8081")
    server.serve_forever()
```

### Step 3: Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Legacy SOAP Service
  legacy-soap:
    build: ./legacy-soap
    ports:
      - "8081:8081"
    networks:
      - backend

  # SOAP Adapter
  soap-adapter:
    build: ./soap-adapter
    ports:
      - "8080:8080"
    environment:
      - SOAP_WSDL=http://legacy-soap:8081/?wsdl
    depends_on:
      - legacy-soap
    networks:
      - backend

networks:
  backend:
    driver: bridge
```

### Step 4: 테스트

```bash
# 1. 서비스 시작
docker-compose up -d

# 2. REST API로 사용자 생성 (JSON)
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice",
    "email": "alice@example.com"
  }'

# Response (JSON):
# {
#   "userId": 1,
#   "name": "Alice",
#   "email": "alice@example.com",
#   "createdAt": "2024-01-15 10:30:00"
# }

# 3. 사용자 조회
curl http://localhost:8080/api/users/1

# 4. 사용자 업데이트
curl -X PUT http://localhost:8080/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Updated",
    "email": "alice.new@example.com"
  }'

# 5. Legacy SOAP Service 직접 호출 (비교)
curl -X POST http://localhost:8081/ \
  -H "Content-Type: text/xml" \
  -d '<?xml version="1.0"?>
<soapenv:Envelope xmlns:soapenv="...">
  <soapenv:Body>
    <GetUser>
      <userId>1</userId>
    </GetUser>
  </soapenv:Body>
</soapenv:Envelope>'

# Response (XML):
# <GetUserResponse>
#   <userId>1</userId>
#   <name>Alice Updated</name>
# </GetUserResponse>
```

---

## 🔧 실습 2: CSV → JSON Adapter

### Step 1: CSV Adapter 구현

```python
# csv-adapter/adapter.py
from flask import Flask, jsonify, request, send_file
import csv
import io
import os

app = Flask(__name__)

CSV_DIR = '/data/csv'
os.makedirs(CSV_DIR, exist_ok=True)

@app.route('/api/data', methods=['GET'])
def get_data():
    """CSV 파일을 JSON으로 변환"""
    filename = request.args.get('file', 'users.csv')
    filepath = os.path.join(CSV_DIR, filename)
    
    if not os.path.exists(filepath):
        return jsonify({'error': 'File not found'}), 404
    
    try:
        with open(filepath, 'r') as f:
            reader = csv.DictReader(f)
            data = list(reader)
        
        return jsonify({
            'filename': filename,
            'count': len(data),
            'data': data
        })
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/api/data', methods=['POST'])
def create_data():
    """JSON을 CSV 파일로 저장"""
    data = request.get_json()
    filename = data.get('filename', 'output.csv')
    records = data.get('records', [])
    
    filepath = os.path.join(CSV_DIR, filename)
    
    try:
        if not records:
            return jsonify({'error': 'No records provided'}), 400
        
        # CSV 작성
        with open(filepath, 'w', newline='') as f:
            fieldnames = records[0].keys()
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            
            writer.writeheader()
            writer.writerows(records)
        
        return jsonify({
            'filename': filename,
            'count': len(records),
            'path': filepath
        }), 201
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/api/data/download', methods=['GET'])
def download_csv():
    """CSV 파일 다운로드"""
    filename = request.args.get('file', 'users.csv')
    filepath = os.path.join(CSV_DIR, filename)
    
    if not os.path.exists(filepath):
        return jsonify({'error': 'File not found'}), 404
    
    return send_file(filepath, as_attachment=True)

@app.route('/api/data/append', methods=['POST'])
def append_data():
    """기존 CSV에 데이터 추가"""
    data = request.get_json()
    filename = data.get('filename', 'users.csv')
    record = data.get('record', {})
    
    filepath = os.path.join(CSV_DIR, filename)
    
    try:
        # 파일이 없으면 생성
        file_exists = os.path.exists(filepath)
        
        with open(filepath, 'a', newline='') as f:
            fieldnames = record.keys()
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            
            if not file_exists:
                writer.writeheader()
            
            writer.writerow(record)
        
        return jsonify({
            'filename': filename,
            'record': record,
            'appended': True
        })
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Step 2: 테스트 데이터 준비

```csv
# data/users.csv
id,name,email,age
1,Alice,alice@example.com,30
2,Bob,bob@example.com,25
3,Charlie,charlie@example.com,35
```

### Step 3: 테스트

```bash
# 1. CSV → JSON 조회
curl http://localhost:8080/api/data?file=users.csv

# Response:
# {
#   "filename": "users.csv",
#   "count": 3,
#   "data": [
#     {"id": "1", "name": "Alice", "email": "alice@example.com", "age": "30"},
#     {"id": "2", "name": "Bob", "email": "bob@example.com", "age": "25"},
#     {"id": "3", "name": "Charlie", "email": "charlie@example.com", "age": "35"}
#   ]
# }

# 2. JSON → CSV 생성
curl -X POST http://localhost:8080/api/data \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "new_users.csv",
    "records": [
      {"id": "4", "name": "David", "email": "david@example.com", "age": "28"},
      {"id": "5", "name": "Eve", "email": "eve@example.com", "age": "32"}
    ]
  }'

# 3. CSV 파일 다운로드
curl -O http://localhost:8080/api/data/download?file=new_users.csv

# 4. 데이터 추가
curl -X POST http://localhost:8080/api/data/append \
  -H "Content-Type: application/json" \
  -d '{
    "filename": "users.csv",
    "record": {"id": "4", "name": "David", "email": "david@example.com", "age": "28"}
  }'
```

---

## 🔧 실습 3: API Version Adapter (v1 → v2)

### Step 1: Version Adapter 구현

```python
# version-adapter/adapter.py
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

# V2 API 엔드포인트
V2_API_BASE = "http://api-v2:8082"

@app.route('/v1/users', methods=['POST'])
def create_user_v1():
    """v1 API → v2 API 변환"""
    # v1 Request
    data = request.get_json()
    
    # v1 → v2 변환
    v2_payload = {
        'profile': {
            'displayName': data.get('name'),
            'email': data.get('email')
        },
        'metadata': {
            'source': 'v1-adapter'
        }
    }
    
    # v2 API 호출
    try:
        response = requests.post(
            f'{V2_API_BASE}/v2/users',
            json=v2_payload,
            timeout=10
        )
        
        v2_data = response.json()
        
        # v2 → v1 변환
        v1_response = {
            'id': v2_data['userId'],
            'name': v2_data['profile']['displayName'],
            'email': v2_data['profile']['email']
        }
        
        return jsonify(v1_response), 201
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/v1/users/<user_id>', methods=['GET'])
def get_user_v1(user_id):
    """v1 조회"""
    try:
        response = requests.get(
            f'{V2_API_BASE}/v2/users/{user_id}',
            timeout=10
        )
        
        v2_data = response.json()
        
        # v2 → v1 변환
        v1_response = {
            'id': v2_data['userId'],
            'name': v2_data['profile']['displayName'],
            'email': v2_data['profile']['email']
        }
        
        return jsonify(v1_response)
    
    except Exception as e:
        return jsonify({'error': str(e)}), 404

@app.route('/v1/users/<user_id>', methods=['PUT'])
def update_user_v1(user_id):
    """v1 업데이트"""
    data = request.get_json()
    
    # v1 → v2 변환
    v2_payload = {
        'profile': {
            'displayName': data.get('name'),
            'email': data.get('email')
        }
    }
    
    try:
        response = requests.put(
            f'{V2_API_BASE}/v2/users/{user_id}',
            json=v2_payload,
            timeout=10
        )
        
        v2_data = response.json()
        
        # v2 → v1 변환
        v1_response = {
            'id': v2_data['userId'],
            'name': v2_data['profile']['displayName'],
            'email': v2_data['profile']['email']
        }
        
        return jsonify(v1_response)
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8081)
```

### Step 2: V2 API (새 버전)

```python
# api-v2/server.py
from flask import Flask, request, jsonify

app = Flask(__name__)

users_db = {}
user_id_counter = 1

@app.route('/v2/users', methods=['POST'])
def create_user_v2():
    """v2 API - 새로운 구조"""
    global user_id_counter
    
    data = request.get_json()
    
    user = {
        'userId': user_id_counter,
        'profile': data['profile'],
        'metadata': data.get('metadata', {}),
        'createdAt': '2024-01-15T10:30:00Z'
    }
    
    users_db[user_id_counter] = user
    user_id_counter += 1
    
    return jsonify(user), 201

@app.route('/v2/users/<int:user_id>', methods=['GET'])
def get_user_v2(user_id):
    """v2 조회"""
    if user_id not in users_db:
        return jsonify({'error': 'User not found'}), 404
    
    return jsonify(users_db[user_id])

@app.route('/v2/users/<int:user_id>', methods=['PUT'])
def update_user_v2(user_id):
    """v2 업데이트"""
    if user_id not in users_db:
        return jsonify({'error': 'User not found'}), 404
    
    data = request.get_json()
    users_db[user_id]['profile'] = data['profile']
    
    return jsonify(users_db[user_id])

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8082)
```

### Step 3: 테스트

```bash
# 1. v1 API로 사용자 생성 (구 형식)
curl -X POST http://localhost:8081/v1/users \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice",
    "email": "alice@example.com"
  }'

# v1 Response (구 형식):
# {
#   "id": 1,
#   "name": "Alice",
#   "email": "alice@example.com"
# }

# 2. v2 API로 직접 생성 (새 형식)
curl -X POST http://localhost:8082/v2/users \
  -H "Content-Type: application/json" \
  -d '{
    "profile": {
      "displayName": "Bob",
      "email": "bob@example.com"
    }
  }'

# v2 Response (새 형식):
# {
#   "userId": 2,
#   "profile": {
#     "displayName": "Bob",
#     "email": "bob@example.com"
#   },
#   "metadata": {},
#   "createdAt": "2024-01-15T10:30:00Z"
# }

# 3. v1 API로 조회 (자동 변환)
curl http://localhost:8081/v1/users/1

# v1 Response (구 형식으로 변환됨):
# {
#   "id": 1,
#   "name": "Alice",
#   "email": "alice@example.com"
# }
```

---

## 🔧 실습 4: Protocol Buffer → JSON Adapter

### Step 1: Protobuf 정의

```protobuf
// user.proto
syntax = "proto3";

package user;

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
}

message UserList {
    repeated User users = 1;
}
```

### Step 2: Protobuf Adapter

```python
# protobuf-adapter/adapter.py
from flask import Flask, request, jsonify
import user_pb2
import requests

app = Flask(__name__)

GRPC_SERVICE = "http://grpc-service:50051"

@app.route('/api/users', methods=['POST'])
def create_user():
    """JSON → Protobuf → gRPC"""
    # 1. JSON 받기
    data = request.get_json()
    
    # 2. Protobuf 메시지 생성
    user = user_pb2.User()
    user.name = data['name']
    user.email = data['email']
    user.age = data.get('age', 0)
    
    # 3. Protobuf 직렬화
    serialized = user.SerializeToString()
    
    # 4. gRPC 서비스 호출 (시뮬레이션: HTTP)
    try:
        response = requests.post(
            f'{GRPC_SERVICE}/create',
            data=serialized,
            headers={'Content-Type': 'application/protobuf'},
            timeout=10
        )
        
        # 5. Protobuf 응답 역직렬화
        response_user = user_pb2.User()
        response_user.ParseFromString(response.content)
        
        # 6. JSON으로 변환
        return jsonify({
            'id': response_user.id,
            'name': response_user.name,
            'email': response_user.email,
            'age': response_user.age
        }), 201
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/api/users', methods=['GET'])
def list_users():
    """Protobuf → JSON 목록 조회"""
    try:
        response = requests.get(
            f'{GRPC_SERVICE}/list',
            timeout=10
        )
        
        # Protobuf 응답 파싱
        user_list = user_pb2.UserList()
        user_list.ParseFromString(response.content)
        
        # JSON으로 변환
        users = [
            {
                'id': user.id,
                'name': user.name,
                'email': user.email,
                'age': user.age
            }
            for user in user_list.users
        ]
        
        return jsonify({'users': users})
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

---

## 🔧 실습 5: Kubernetes에서 Adapter Pattern

### Step 1: Deployment with Adapter

```yaml
# deployment-with-adapter.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      # 메인 애플리케이션 (REST API만 사용)
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: LEGACY_API
          value: "http://localhost:8081/api"  # Adapter 사용

      # SOAP Adapter Sidecar
      - name: soap-adapter
        image: soap-adapter:latest
        ports:
        - containerPort: 8081
        env:
        - name: SOAP_ENDPOINT
          value: "http://legacy-soap-service:8082"

---
apiVersion: v1
kind: Service
metadata:
  name: legacy-soap-service
spec:
  selector:
    app: legacy-soap
  ports:
  - port: 8082
    targetPort: 8082
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ Adapter 타입          │ 용도                        │
├──────────────────────┼────────────────────────────┤
│ Protocol Adapter     │ REST ↔ SOAP, HTTP ↔ gRPC   │
├──────────────────────┼────────────────────────────┤
│ Format Adapter       │ JSON ↔ XML, CSV ↔ JSON     │
├──────────────────────┼────────────────────────────┤
│ Version Adapter      │ API v1 ↔ API v2            │
├──────────────────────┼────────────────────────────┤
│ Auth Adapter         │ OAuth ↔ Basic Auth         │
├──────────────────────┼────────────────────────────┤
│ Data Adapter         │ SQL ↔ NoSQL 쿼리 변환        │
└──────────────────────┴────────────────────────────┘
```

---

## 🎓 연습 문제

### 문제 1: Adapter가 성능 병목이 될 수 있는가? 어떻게 최적화하는가?

<details>
<summary>정답 보기</summary>

**병목 발생 원인:**
```
Request → Adapter → Legacy
         (변환 오버헤드)
         (네트워크 홉)
```

**최적화 방법:**

**1. 캐싱:**
```python
from functools import lru_cache

@lru_cache(maxsize=1000)
def transform_data(data_hash):
    # 동일한 변환 결과 캐싱
    return expensive_transformation(data)
```

**2. 비동기 처리:**
```python
import asyncio

async def adapt_request(request):
    # I/O 병렬 처리
    tasks = [
        fetch_from_legacy(request),
        validate_data(request)
    ]
    results = await asyncio.gather(*tasks)
    return combine_results(results)
```

**3. Connection Pool:**
```python
# HTTP Connection Pool
session = requests.Session()
adapter = HTTPAdapter(pool_connections=10, pool_maxsize=20)
session.mount('http://', adapter)
```

**4. 배치 처리:**
```python
# 여러 요청을 모아서 한 번에
def batch_adapt(requests):
    # 10개씩 모아서 처리
    return [adapt(r) for r in requests]
```

</details>

### 문제 2: Adapter 계층이 장애나면 전체 시스템이 중단되는가?

<details>
<summary>정답 보기</summary>

**Yes - SPOF 문제:**
```
App → Adapter (단일) → Legacy
      (장애 시 중단)
```

**해결 방법:**

**1. Sidecar로 배포 (권장):**
```yaml
# 각 Pod에 Adapter
spec:
  containers:
  - name: app
  - name: adapter  # Pod당 1개
```

**2. Fallback 로직:**
```python
def get_data():
    try:
        return adapter.get()
    except:
        # Fallback: 캐시된 데이터
        return cached_data
```

**3. Circuit Breaker:**
```python
@circuit_breaker
def call_adapter():
    # 장애 시 빠른 실패
    return adapter.call()
```

**4. 복제:**
```yaml
# Adapter를 Service로
apiVersion: v1
kind: Service
metadata:
  name: adapter-service
spec:
  replicas: 3  # 3개 복제
```

</details>

### 문제 3: 여러 버전의 API를 동시에 지원해야 한다면?

<details>
<summary>정답 보기</summary>

**멀티 버전 Adapter:**

```python
class VersionRouter:
    def __init__(self):
        self.adapters = {
            'v1': V1Adapter(),
            'v2': V2Adapter(),
            'v3': V3Adapter()
        }
    
    def route(self, version, request):
        adapter = self.adapters.get(version)
        if not adapter:
            return {'error': 'Unsupported version'}
        
        return adapter.process(request)

# 사용
router = VersionRouter()

@app.route('/<version>/users', methods=['POST'])
def create_user(version):
    return router.route(version, request.get_json())
```

**버전별 변환 체인:**
```python
# v1 → v2 → v3 변환 체인
def upgrade_v1_to_v3(data):
    v2_data = v1_to_v2(data)
    v3_data = v2_to_v3(v2_data)
    return v3_data

def v1_to_v2(data):
    return {'profile': {'name': data['name']}}

def v2_to_v3(data):
    return {'user': data['profile'], 'metadata': {}}
```

</details>

---

## 📌 핵심 요약

```
Adapter 패턴 핵심:
1. 인터페이스 변환 (프로토콜, 포맷)
2. 레거시 통합 (코드 수정 없이)
3. 독립 배포 (Adapter만 업데이트)
4. 단계적 마이그레이션
5. 표준화 (일관된 API)

Best Practices:
✅ Sidecar로 배포
✅ 캐싱으로 성능 최적화
✅ 에러 처리 및 Fallback
✅ 버전 관리
✅ 메트릭 수집
```

---

## 📚 참고 자료

- [Adapter Pattern - Gang of Four](https://refactoring.guru/design-patterns/adapter)
- [API Versioning Best Practices](https://www.freecodecamp.org/news/rest-api-best-practices-rest-endpoint-design-examples/)
- [Protocol Buffers](https://developers.google.com/protocol-buffers)

---

## 🤔 생각해볼 문제

1. Adapter를 언제까지 유지해야 하는가? 언제 제거할 수 있는가?
2. Adapter vs API Gateway - 어떤 차이가 있는가?
3. 레거시 시스템을 현대화하는 전략은? (Strangler Fig Pattern)

> 💡 **답변**:
> 
> **1) Adapter 제거 시점:**
> 
> ```
> 단계적 마이그레이션:
> 
> Phase 1: Adapter 도입
> Old Clients → Adapter → Legacy
> 
> Phase 2: 클라이언트 마이그레이션
> Some Clients → New API
> Old Clients → Adapter → Legacy
> 
> Phase 3: 레거시 폐기
> All Clients → New API
> (Adapter 제거 가능)
> 
> 제거 조건:
> ✅ 모든 클라이언트가 새 API 사용
> ✅ 레거시 시스템 폐기 완료
> ✅ 충분한 테스트 완료
> ```
> 
> **2) Adapter vs API Gateway:**
> 
> ```
> API Gateway:
> - 모든 API의 단일 진입점
> - 인증, Rate Limiting
> - 라우팅
> - 모든 클라이언트 대상
> 
> Adapter:
> - 특정 시스템 간 변환
> - 프로토콜/포맷 변환
> - 레거시 통합
> - 내부 사용
> 
> 함께 사용:
> Client → API Gateway → Adapter → Legacy
> ```
> 
> **3) Strangler Fig Pattern:**
> 
> ```
> 레거시 현대화 전략:
> 
> 1. Facade 추가
>    Client → Facade → Legacy
> 
> 2. 기능별 이전
>    Client → Facade → New Service (User)
>                    → Legacy (Order, Product)
> 
> 3. 점진적 확대
>    Client → Facade → New Service (User, Order)
>                    → Legacy (Product)
> 
> 4. 레거시 제거
>    Client → Facade → New Services (모두)
> 
> Adapter의 역할:
> - Facade 구현
> - 신/구 시스템 브리지
> ```

---

<div align="center">

**[⬅️ 이전: Ambassador Pattern](./03-Ambassador-Pattern.md)** | **[다음: Init Containers ➡️](./05-Init-Containers.md)**

</div>
