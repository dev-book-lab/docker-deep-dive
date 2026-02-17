# 01. Web Application - Full-stack 앱 Docker화

## 🎯 이 챕터에서 배울 것

- **Frontend 컨테이너화**: React, Vue, Angular
- **Backend 컨테이너화**: Node.js, Python, Go
- **Multi-container 구성**: Docker Compose
- **개발 환경**: Hot Reload, Volume Mount
- **프로덕션 환경**: Multi-stage Build, 최적화
- **실전 프로젝트**: 완전한 Full-stack 앱

## 📌 왜 중요한가?

**"실제 웹 애플리케이션을 Docker로 완벽하게 구성하는 것이 실무의 시작입니다."**

```
Web Application Architecture:

Traditional (Without Docker):
┌─────────────────────────────────────────────────┐
│ Development Machine                             │
│                                                 │
│ Frontend (localhost:3000)                       │
│  - npm install                                  │
│  - npm start                                    │
│                                                 │
│ Backend (localhost:8080)                        │
│  - pip install                                  │
│  - python app.py                                │
│                                                 │
│ Database (localhost:5432)                       │
│  - PostgreSQL 설치                              │
│                                                 │
│ 문제:                                           │
│ ❌ 환경 설정 복잡 (Node, Python, PostgreSQL)     │
│ ❌ "내 컴퓨터에선 되는데?"                       │
│ ❌ 팀원마다 다른 환경                            │
└─────────────────────────────────────────────────┘

With Docker:
┌─────────────────────────────────────────────────┐
│ docker-compose up                               │
│                                                 │
│  ┌──────────────┐  ┌──────────────┐            │
│  │  Frontend    │  │   Backend    │            │
│  │  (React)     │─→│  (Node.js)   │            │
│  │  :3000       │  │  :8080       │            │
│  └──────────────┘  └──────┬───────┘            │
│                           │                     │
│                           ▼                     │
│                    ┌──────────────┐             │
│                    │  PostgreSQL  │             │
│                    │  :5432       │             │
│                    └──────────────┘             │
│                                                 │
│ 장점:                                           │
│ ✅ 일관된 환경 (모든 개발자 동일)                │
│ ✅ 간단한 설정 (docker-compose up)              │
│ ✅ 격리된 서비스                                 │
└─────────────────────────────────────────────────┘

Full-stack Structure:
┌─────────────────────────────────────────────────┐
│ myapp/                                          │
│ ├── frontend/                                   │
│ │   ├── Dockerfile                              │
│ │   ├── Dockerfile.prod                         │
│ │   ├── package.json                            │
│ │   └── src/                                    │
│ │                                               │
│ ├── backend/                                    │
│ │   ├── Dockerfile                              │
│ │   ├── Dockerfile.prod                         │
│ │   ├── requirements.txt                        │
│ │   └── app.py                                  │
│ │                                               │
│ ├── docker-compose.yml                          │
│ ├── docker-compose.prod.yml                     │
│ └── .env                                        │
└─────────────────────────────────────────────────┘

Development vs Production:
┌─────────────────────────────────────────────────┐
│ Development:                                    │
│  - Hot Reload (코드 변경 즉시 반영)              │
│  - Source Map (디버깅 쉬움)                      │
│  - 볼륨 마운트 (./src:/app/src)                 │
│  - 개발 서버 (webpack-dev-server)               │
│                                                 │
│ Production:                                     │
│  - Minified/Optimized                           │
│  - Multi-stage Build                            │
│  - Static Files (Nginx)                         │
│  - Health Check                                 │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **생산성**: 환경 설정 시간 제로
- **일관성**: 모든 개발자 동일 환경
- **배포**: 개발 → 프로덕션 동일한 방식
- **확장성**: 서비스 추가 쉬움

---

## 🔧 실습 1: React + Node.js Full-stack (완전한 프로젝트)

### Step 1: 프로젝트 구조 생성

```bash
# 프로젝트 생성
mkdir fullstack-app && cd fullstack-app
mkdir frontend backend

# Frontend (React)
cd frontend
npx create-react-app .

# Backend (Node.js)
cd ../backend
npm init -y
npm install express cors body-parser
npm install --save-dev nodemon
```

### Step 2: Backend 구현

```javascript
// backend/server.js
const express = require('express');
const cors = require('cors');
const bodyParser = require('body-parser');

const app = express();
const PORT = process.env.PORT || 8080;

// Middleware
app.use(cors());
app.use(bodyParser.json());

// In-memory database
let items = [
  { id: 1, name: 'Item 1', description: 'First item', createdAt: new Date() },
  { id: 2, name: 'Item 2', description: 'Second item', createdAt: new Date() },
];
let nextId = 3;

// Routes
app.get('/api/health', (req, res) => {
  res.json({
    status: 'OK',
    timestamp: new Date(),
    uptime: process.uptime()
  });
});

app.get('/api/items', (req, res) => {
  res.json(items);
});

app.get('/api/items/:id', (req, res) => {
  const item = items.find(i => i.id === parseInt(req.params.id));
  if (!item) {
    return res.status(404).json({ error: 'Item not found' });
  }
  res.json(item);
});

app.post('/api/items', (req, res) => {
  const { name, description } = req.body;
  
  if (!name || !description) {
    return res.status(400).json({ error: 'Name and description required' });
  }

  const newItem = {
    id: nextId++,
    name,
    description,
    createdAt: new Date()
  };
  
  items.push(newItem);
  console.log('Created item:', newItem);
  res.status(201).json(newItem);
});

app.put('/api/items/:id', (req, res) => {
  const item = items.find(i => i.id === parseInt(req.params.id));
  
  if (!item) {
    return res.status(404).json({ error: 'Item not found' });
  }

  const { name, description } = req.body;
  if (name) item.name = name;
  if (description) item.description = description;
  
  res.json(item);
});

app.delete('/api/items/:id', (req, res) => {
  const index = items.findIndex(i => i.id === parseInt(req.params.id));
  
  if (index === -1) {
    return res.status(404).json({ error: 'Item not found' });
  }

  items.splice(index, 1);
  res.status(204).send();
});

app.listen(PORT, () => {
  console.log(`🚀 Server running on http://localhost:${PORT}`);
  console.log(`📊 Health check: http://localhost:${PORT}/api/health`);
});
```

```json
// backend/package.json
{
  "name": "backend",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "body-parser": "^1.20.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

```dockerfile
# backend/Dockerfile (개발용)
FROM node:18-alpine

WORKDIR /app

# 의존성 설치
COPY package*.json ./
RUN npm ci

# 소스 복사
COPY . .

# 포트
EXPOSE 8080

# 개발 서버 (nodemon으로 Hot Reload)
CMD ["npm", "run", "dev"]
```

### Step 3: Frontend 구현

```javascript
// frontend/src/App.js
import React, { useState, useEffect } from 'react';
import './App.css';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:8080';

function App() {
  const [items, setItems] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [newItem, setNewItem] = useState({ name: '', description: '' });
  const [editingId, setEditingId] = useState(null);

  // Fetch items
  useEffect(() => {
    fetchItems();
  }, []);

  const fetchItems = async () => {
    try {
      setLoading(true);
      const response = await fetch(`${API_URL}/api/items`);
      if (!response.ok) throw new Error('Failed to fetch items');
      const data = await response.json();
      setItems(data);
      setError(null);
    } catch (err) {
      setError(err.message);
      console.error('Error fetching items:', err);
    } finally {
      setLoading(false);
    }
  };

  // Create item
  const handleSubmit = async (e) => {
    e.preventDefault();
    
    if (!newItem.name.trim() || !newItem.description.trim()) {
      alert('Please fill in all fields');
      return;
    }

    try {
      const response = await fetch(`${API_URL}/api/items`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newItem)
      });

      if (!response.ok) throw new Error('Failed to create item');
      
      const data = await response.json();
      setItems([...items, data]);
      setNewItem({ name: '', description: '' });
    } catch (err) {
      console.error('Error creating item:', err);
      alert('Failed to create item');
    }
  };

  // Delete item
  const handleDelete = async (id) => {
    if (!window.confirm('Are you sure?')) return;

    try {
      const response = await fetch(`${API_URL}/api/items/${id}`, {
        method: 'DELETE'
      });

      if (!response.ok) throw new Error('Failed to delete item');
      
      setItems(items.filter(item => item.id !== id));
    } catch (err) {
      console.error('Error deleting item:', err);
      alert('Failed to delete item');
    }
  };

  if (loading) return <div className="loading">Loading...</div>;
  if (error) return <div className="error">Error: {error}</div>;

  return (
    <div className="App">
      <header className="App-header">
        <h1>📦 Full-stack Docker App</h1>
        <p>React + Node.js + Docker</p>
      </header>

      <main className="container">
        <section className="form-section">
          <h2>Add New Item</h2>
          <form onSubmit={handleSubmit} className="item-form">
            <input
              type="text"
              placeholder="Name"
              value={newItem.name}
              onChange={e => setNewItem({ ...newItem, name: e.target.value })}
              className="form-input"
            />
            <input
              type="text"
              placeholder="Description"
              value={newItem.description}
              onChange={e => setNewItem({ ...newItem, description: e.target.value })}
              className="form-input"
            />
            <button type="submit" className="btn btn-primary">
              ➕ Add Item
            </button>
          </form>
        </section>

        <section className="items-section">
          <h2>Items ({items.length})</h2>
          <div className="items-grid">
            {items.map(item => (
              <div key={item.id} className="item-card">
                <h3>{item.name}</h3>
                <p>{item.description}</p>
                <small>{new Date(item.createdAt).toLocaleString()}</small>
                <button
                  onClick={() => handleDelete(item.id)}
                  className="btn btn-danger"
                >
                  🗑️ Delete
                </button>
              </div>
            ))}
          </div>
        </section>
      </main>
    </div>
  );
}

export default App;
```

```css
/* frontend/src/App.css */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
}

.App {
  min-height: 100vh;
}

.App-header {
  background: white;
  padding: 2rem;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  text-align: center;
}

.App-header h1 {
  color: #667eea;
  margin-bottom: 0.5rem;
}

.container {
  max-width: 1200px;
  margin: 2rem auto;
  padding: 0 1rem;
}

.form-section, .items-section {
  background: white;
  padding: 2rem;
  border-radius: 10px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  margin-bottom: 2rem;
}

.item-form {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
}

.form-input {
  flex: 1;
  min-width: 200px;
  padding: 0.75rem;
  border: 2px solid #e0e0e0;
  border-radius: 5px;
  font-size: 1rem;
}

.form-input:focus {
  outline: none;
  border-color: #667eea;
}

.btn {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 5px;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s;
}

.btn-primary {
  background: #667eea;
  color: white;
}

.btn-primary:hover {
  background: #5568d3;
  transform: translateY(-2px);
}

.btn-danger {
  background: #e74c3c;
  color: white;
  font-size: 0.9rem;
}

.btn-danger:hover {
  background: #c0392b;
}

.items-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.5rem;
  margin-top: 1rem;
}

.item-card {
  border: 2px solid #e0e0e0;
  padding: 1.5rem;
  border-radius: 10px;
  transition: all 0.3s;
}

.item-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 16px rgba(0,0,0,0.1);
}

.item-card h3 {
  color: #667eea;
  margin-bottom: 0.5rem;
}

.item-card p {
  color: #666;
  margin-bottom: 0.75rem;
}

.item-card small {
  color: #999;
  display: block;
  margin-bottom: 1rem;
}

.loading, .error {
  text-align: center;
  padding: 3rem;
  color: white;
  font-size: 1.5rem;
}

.error {
  color: #e74c3c;
  background: white;
  border-radius: 10px;
  margin: 2rem;
}
```

```dockerfile
# frontend/Dockerfile (개발용)
FROM node:18-alpine

WORKDIR /app

# 의존성 설치
COPY package*.json ./
RUN npm ci

# 소스 복사
COPY . .

# 포트
EXPOSE 3000

# 개발 서버
CMD ["npm", "start"]
```

### Step 4: Docker Compose (개발 환경)

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Backend API
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: backend
    ports:
      - "8080:8080"
    volumes:
      # Hot reload - 소스 코드 마운트
      - ./backend:/app
      - /app/node_modules  # node_modules는 컨테이너 것 사용
    environment:
      - NODE_ENV=development
      - PORT=8080
    command: npm run dev
    restart: unless-stopped
  
  # Frontend React App
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: frontend
    ports:
      - "3000:3000"
    volumes:
      # Hot reload
      - ./frontend:/app
      - /app/node_modules
    environment:
      - REACT_APP_API_URL=http://localhost:8080
      - CHOKIDAR_USEPOLLING=true  # Hot reload (Windows/Mac 필요)
      - WDS_SOCKET_PORT=3000       # Webpack Dev Server
    depends_on:
      - backend
    stdin_open: true  # React 개발 서버
    tty: true
    restart: unless-stopped
```

```env
# .env (선택사항)
# Backend
PORT=8080
NODE_ENV=development

# Frontend
REACT_APP_API_URL=http://localhost:8080
```

### Step 5: 실행 및 테스트

```bash
# 1. 빌드 및 실행
docker-compose up --build

# 출력:
# backend  | Server running on http://localhost:8080
# frontend | webpack compiled successfully

# 2. 브라우저에서 확인
# http://localhost:3000

# 3. API 테스트
curl http://localhost:8080/api/health
curl http://localhost:8080/api/items

# 4. Hot Reload 테스트
# backend/server.js 또는 frontend/src/App.js 수정
# → 자동으로 재시작/반영됨

# 5. 로그 확인
docker-compose logs -f backend
docker-compose logs -f frontend

# 6. 중지
docker-compose down

# 7. 정리 (볼륨 포함)
docker-compose down -v
```

---

## 🔧 실습 2: Production Build

### Step 1: Frontend Production Dockerfile

```dockerfile
# frontend/Dockerfile.prod
# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# 의존성 설치
COPY package*.json ./
RUN npm ci --only=production

# 소스 복사 및 빌드
COPY . .
RUN npm run build

# Stage 2: Nginx
FROM nginx:alpine

# 빌드 결과 복사
COPY --from=builder /app/build /usr/share/nginx/html

# Nginx 설정
COPY nginx.conf /etc/nginx/conf.d/default.conf

# 포트
EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# frontend/nginx.conf
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip 압축
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript 
               application/x-javascript application/xml+rss 
               application/json application/javascript;

    # React Router (SPA) - 모든 요청을 index.html로
    location / {
        try_files $uri $uri/ /index.html;
        
        # Cache control
        add_header Cache-Control "no-cache, must-revalidate";
    }

    # Static 파일 캐싱
    location /static/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # API 프록시 (Backend로 전달)
    location /api {
        proxy_pass http://backend:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeout 설정
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### Step 2: Backend Production Dockerfile

```dockerfile
# backend/Dockerfile.prod
# Stage 1: Dependencies
FROM node:18-alpine AS dependencies

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Production
FROM node:18-alpine

WORKDIR /app

# 보안: non-root 사용자 생성
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# 의존성 복사
COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules

# 소스 복사
COPY --chown=nodejs:nodejs . .

# 사용자 전환
USER nodejs

# 포트
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8080/api/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# 실행
CMD ["node", "server.js"]
```

### Step 3: Production Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.prod
    container_name: backend-prod
    restart: always
    environment:
      - NODE_ENV=production
      - PORT=8080
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - app-network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.prod
    container_name: frontend-prod
    restart: always
    ports:
      - "80:80"
    depends_on:
      backend:
        condition: service_healthy
    networks:
      - app-network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

networks:
  app-network:
    driver: bridge
```

### Step 4: 프로덕션 배포

```bash
# 1. 빌드
docker-compose -f docker-compose.prod.yml build

# 2. 실행
docker-compose -f docker-compose.prod.yml up -d

# 3. 상태 확인
docker-compose -f docker-compose.prod.yml ps

# 4. Health check 확인
curl http://localhost/health
curl http://localhost/api/health

# 5. 로그 확인
docker-compose -f docker-compose.prod.yml logs -f

# 6. 이미지 크기 확인
docker images | grep -E "backend|frontend"

# 7. 중지 및 제거
docker-compose -f docker-compose.prod.yml down
```

---

## 🔧 실습 3: Python Flask + Vue.js

### Step 1: Backend (Flask)

```python
# backend/app.py
from flask import Flask, jsonify, request
from flask_cors import CORS
from datetime import datetime

app = Flask(__name__)
CORS(app)

# In-memory database
tasks = [
    {"id": 1, "title": "Learn Docker", "completed": False, "created_at": datetime.now().isoformat()},
    {"id": 2, "title": "Build Full-stack App", "completed": False, "created_at": datetime.now().isoformat()},
]
next_id = 3

@app.route('/api/health')
def health():
    return jsonify({
        "status": "OK",
        "timestamp": datetime.now().isoformat()
    })

@app.route('/api/tasks', methods=['GET'])
def get_tasks():
    return jsonify(tasks)

@app.route('/api/tasks', methods=['POST'])
def create_task():
    data = request.json
    
    if not data.get('title'):
        return jsonify({"error": "Title required"}), 400
    
    global next_id
    new_task = {
        "id": next_id,
        "title": data['title'],
        "completed": False,
        "created_at": datetime.now().isoformat()
    }
    next_id += 1
    tasks.append(new_task)
    
    return jsonify(new_task), 201

@app.route('/api/tasks/<int:task_id>', methods=['PUT'])
def update_task(task_id):
    task = next((t for t in tasks if t['id'] == task_id), None)
    
    if not task:
        return jsonify({"error": "Task not found"}), 404
    
    data = request.json
    if 'title' in data:
        task['title'] = data['title']
    if 'completed' in data:
        task['completed'] = data['completed']
    
    return jsonify(task)

@app.route('/api/tasks/<int:task_id>', methods=['DELETE'])
def delete_task(task_id):
    global tasks
    tasks = [t for t in tasks if t['id'] != task_id]
    return '', 204

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=5000)
```

```txt
# backend/requirements.txt
Flask==3.0.0
Flask-CORS==4.0.0
gunicorn==21.2.0
```

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

# 개발 환경
CMD ["flask", "run", "--host=0.0.0.0"]
```

```dockerfile
# backend/Dockerfile.prod
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# non-root user
RUN adduser --disabled-password --gecos '' appuser && \
    chown -R appuser:appuser /app
USER appuser

EXPOSE 5000

# 프로덕션 환경 (gunicorn)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

### Step 2: Frontend (Vue.js)

```vue
<!-- frontend/src/App.vue -->
<template>
  <div id="app">
    <div class="container">
      <header>
        <h1>✅ Todo List</h1>
        <p>Vue.js + Flask + Docker</p>
      </header>

      <main>
        <form @submit.prevent="addTask" class="add-form">
          <input
            v-model="newTask"
            placeholder="What needs to be done?"
            class="task-input"
          />
          <button type="submit" class="btn btn-primary">Add</button>
        </form>

        <div v-if="loading" class="loading">Loading...</div>
        <div v-else-if="error" class="error">{{ error }}</div>
        <ul v-else class="task-list">
          <li
            v-for="task in tasks"
            :key="task.id"
            class="task-item"
            :class="{ completed: task.completed }"
          >
            <input
              type="checkbox"
              :checked="task.completed"
              @change="toggleTask(task)"
            />
            <span class="task-title">{{ task.title }}</span>
            <button @click="deleteTask(task.id)" class="btn btn-delete">
              🗑️
            </button>
          </li>
        </ul>

        <div class="stats">
          <span>{{ incompleteTasks }} tasks left</span>
          <span>{{ tasks.length }} total</span>
        </div>
      </main>
    </div>
  </div>
</template>

<script>
const API_URL = process.env.VUE_APP_API_URL || 'http://localhost:5000';

export default {
  name: 'App',
  data() {
    return {
      tasks: [],
      newTask: '',
      loading: true,
      error: null
    };
  },
  computed: {
    incompleteTasks() {
      return this.tasks.filter(t => !t.completed).length;
    }
  },
  mounted() {
    this.fetchTasks();
  },
  methods: {
    async fetchTasks() {
      try {
        this.loading = true;
        const response = await fetch(`${API_URL}/api/tasks`);
        if (!response.ok) throw new Error('Failed to fetch tasks');
        this.tasks = await response.json();
        this.error = null;
      } catch (err) {
        this.error = err.message;
        console.error('Error:', err);
      } finally {
        this.loading = false;
      }
    },
    async addTask() {
      if (!this.newTask.trim()) return;
      
      try {
        const response = await fetch(`${API_URL}/api/tasks`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ title: this.newTask })
        });
        
        if (!response.ok) throw new Error('Failed to add task');
        
        const task = await response.json();
        this.tasks.push(task);
        this.newTask = '';
      } catch (err) {
        alert('Failed to add task');
        console.error('Error:', err);
      }
    },
    async toggleTask(task) {
      try {
        const response = await fetch(`${API_URL}/api/tasks/${task.id}`, {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ completed: !task.completed })
        });
        
        if (!response.ok) throw new Error('Failed to update task');
        
        const updated = await response.json();
        task.completed = updated.completed;
      } catch (err) {
        alert('Failed to update task');
        console.error('Error:', err);
      }
    },
    async deleteTask(id) {
      if (!confirm('Delete this task?')) return;
      
      try {
        const response = await fetch(`${API_URL}/api/tasks/${id}`, {
          method: 'DELETE'
        });
        
        if (!response.ok) throw new Error('Failed to delete task');
        
        this.tasks = this.tasks.filter(t => t.id !== id);
      } catch (err) {
        alert('Failed to delete task');
        console.error('Error:', err);
      }
    }
  }
};
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
}

#app {
  min-height: 100vh;
  padding: 2rem;
}

.container {
  max-width: 600px;
  margin: 0 auto;
  background: white;
  border-radius: 10px;
  box-shadow: 0 10px 40px rgba(0,0,0,0.2);
  overflow: hidden;
}

header {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  padding: 2rem;
  text-align: center;
}

header h1 {
  margin-bottom: 0.5rem;
}

main {
  padding: 2rem;
}

.add-form {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 2rem;
}

.task-input {
  flex: 1;
  padding: 0.75rem;
  border: 2px solid #e0e0e0;
  border-radius: 5px;
  font-size: 1rem;
}

.task-input:focus {
  outline: none;
  border-color: #667eea;
}

.btn {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.3s;
}

.btn-primary {
  background: #667eea;
  color: white;
}

.btn-primary:hover {
  background: #5568d3;
}

.btn-delete {
  background: transparent;
  padding: 0.5rem;
}

.btn-delete:hover {
  background: #ffe0e0;
}

.task-list {
  list-style: none;
}

.task-item {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 1rem;
  border-bottom: 1px solid #e0e0e0;
  transition: background 0.3s;
}

.task-item:hover {
  background: #f9f9f9;
}

.task-item.completed .task-title {
  text-decoration: line-through;
  color: #999;
}

.task-title {
  flex: 1;
}

.stats {
  display: flex;
  justify-content: space-between;
  margin-top: 2rem;
  padding: 1rem;
  background: #f0f0f0;
  border-radius: 5px;
  color: #666;
}

.loading, .error {
  text-align: center;
  padding: 2rem;
}

.error {
  color: #e74c3c;
}
</style>
```

```dockerfile
# frontend/Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

EXPOSE 8080

CMD ["npm", "run", "serve"]
```

```yaml
# docker-compose.yml (Flask + Vue)
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    volumes:
      - ./backend:/app
    environment:
      - FLASK_ENV=development
      - FLASK_APP=app.py
  
  frontend:
    build: ./frontend
    ports:
      - "8080:8080"
    volumes:
      - ./frontend:/app
      - /app/node_modules
    environment:
      - VUE_APP_API_URL=http://localhost:5000
    depends_on:
      - backend
```

---

## 💡 핵심 패턴 정리

```
개발 환경:
✅ Volume Mount (Hot Reload)
✅ 개발 서버 (webpack-dev-server, nodemon)
✅ Source Map
✅ 디버깅 도구

프로덕션 환경:
✅ Multi-stage Build
✅ Nginx (Static Serving)
✅ Minify/Optimize
✅ Health Check
✅ Non-root User
✅ Logging
```

---

## 📌 핵심 요약

```
Full-stack Docker 핵심:
1. Frontend + Backend 분리
2. Docker Compose로 통합
3. 개발: Hot Reload
4. 프로덕션: Multi-stage
5. Nginx로 Reverse Proxy

Best Practices:
✅ Multi-stage Build
✅ .dockerignore
✅ Health Check
✅ Non-root User
✅ Logging 설정
```

---

<div align="center">

**[다음: Database Setup ➡️](./02-Database-Setup.md)**

</div>
