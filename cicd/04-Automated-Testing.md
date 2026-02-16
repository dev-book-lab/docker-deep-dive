# 04. Automated Testing - 컨테이너 기반 테스트

## 🎯 이 챕터에서 배울 것

- **테스트 전략**: Unit, Integration, E2E
- **컨테이너 테스트**: Docker 이미지 검증
- **테스트 환경**: Docker Compose로 격리
- **CI/CD 통합**: 자동화된 테스트 파이프라인
- **성능 테스트**: Load Testing in Containers
- **실전 구현**: 프로덕션급 테스트 전략

## 📌 왜 중요한가?

**"자동화된 테스트는 버그를 조기에 발견하고 배포 신뢰도를 높입니다."**

```
Automated Testing의 핵심:

Without Automated Tests (수동 테스트):
┌─────────────────────────────────────────────────┐
│ Manual Testing Process                          │
│                                                 │
│ 1. 개발자가 코드 작성                               │
│    ↓                                            │
│ 2. 로컬에서 수동 테스트                              │
│    - "내 컴퓨터에선 작동해요" 🤷                     │
│    ↓                                            │
│ 3. 커밋 후 배포                                    │
│    ↓                                            │
│ 4. 프로덕션에서 버그 발견 💥                         │
│    - 사용자가 발견                                 │
│    - 긴급 핫픽스                                  │
│    - 롤백                                        │
└─────────────────────────────────────────────────┘

문제점:
❌ 늦은 버그 발견 (프로덕션)
❌ 높은 수정 비용
❌ 사용자 영향
❌ 반복 불가능
❌ 환경 차이

With Automated Tests:
┌─────────────────────────────────────────────────┐
│ Automated Testing Pipeline                      │
│                                                 │
│ git push                                        │
│    ↓                                            │
│ CI/CD Triggered                                 │
│    │                                            │
│    ├─ Unit Tests (5초) ✅                       │
│    │   └─ 함수 레벨 테스트                          │
│    │                                            │
│    ├─ Integration Tests (30초) ✅               │
│    │   └─ DB, API 연동 테스트                      │
│    │                                            │
│    ├─ E2E Tests (2분) ✅                        │
│    │   └─ 사용자 시나리오 테스트                     │
│    │                                            │
│    ├─ Container Tests (1분) ✅                  │
│    │   └─ 이미지 빌드 및 실행 검증                   │
│    │                                            │
│    └─ Security Scan (1분) ✅                    │
│        └─ 취약점 검사                              │
│    ↓                                            │
│ All Passed → Deploy                             │
│ Any Failed → Block Deploy                       │
└─────────────────────────────────────────────────┘

장점:
✅ 조기 버그 발견 (커밋 직후)
✅ 낮은 수정 비용
✅ 배포 신뢰도 향상
✅ 반복 가능
✅ 일관된 환경

Testing Pyramid:
┌─────────────────────────────────────────────────┐
│                                                 │
│              ▲                                  │
│             ╱ ╲                                 │
│            ╱   ╲  E2E Tests                     │
│           ╱     ╲ (느림, 비용 높음)                │
│          ╱───────╲                              │
│         ╱         ╲                             │
│        ╱Integration╲                            │
│       ╱    Tests    ╲ (중간)                     │
│      ╱───────────────╲                          │
│     ╱                 ╲                         │
│    ╱   Unit Tests      ╲ (빠름, 많이)             │
│   ╱─────────────────────╲                       │
│  ▼                       ▼                      │
│                                                 │
│ Unit: 70% (빠르고 많이)                            │
│ Integration: 20% (적당히)                         │
│ E2E: 10% (핵심 시나리오만)                          │
└─────────────────────────────────────────────────┘

Container Testing Layers:
┌─────────────────────────────────────────────────┐
│ 1. Image Build Test                             │
│    - Dockerfile 문법                             │
│    - 빌드 성공                                    │
│    - 이미지 크기                                   │
│                                                 │
│ 2. Image Structure Test                         │
│    - 필요한 파일 존재                               │
│    - 권한 설정                                    │
│    - 환경 변수                                    │
│                                                 │
│ 3. Container Runtime Test                       │
│    - 정상 시작                                    │
│    - Health Check                               │
│    - 포트 노출                                    │
│                                                 │
│ 4. Application Test                             │
│    - API 엔드포인트                                │
│    - 기능 동작                                    │
│    - 성능                                        │
│                                                 │
│ 5. Integration Test                             │
│    - 다른 서비스와 연동                              │
│    - DB, Cache, Queue                           │
│    - 외부 API                                    │
└─────────────────────────────────────────────────┘

Test Environment (Docker Compose):
┌─────────────────────────────────────────────────┐
│ docker-compose -f test.yml up                   │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   App    │  │ Postgres │  │  Redis   │       │
│  │ (Test)   │─→│  (Test)  │  │ (Test)   │       │
│  └──────────┘  └──────────┘  └──────────┘       │
│       │                                         │
│       ▼                                         │
│  Run Tests → Cleanup → Exit                     │
│                                                 │
│ 완전히 격리된 테스트 환경!                            │
└─────────────────────────────────────────────────┘
```

**실무 영향:**
- **품질**: 버그 조기 발견 (80% 이상)
- **속도**: 자동화로 빠른 피드백 (5분)
- **신뢰**: 프로덕션 배포 자신감
- **비용**: 버그 수정 비용 90% 절감

---

## 🔬 Deep Dive

### 1. 테스트 종류

#### Unit Tests

```python
# tests/test_calculator.py
def test_add():
    from calculator import add
    assert add(2, 3) == 5
    assert add(-1, 1) == 0

# 실행
docker run --rm myapp:test pytest tests/test_calculator.py

# 특징:
✅ 빠름 (밀리초)
✅ 격리됨
✅ 많이 작성
```

#### Integration Tests

```python
# tests/test_api.py
def test_create_user():
    response = requests.post('http://api:8080/users', json={
        'name': 'Alice',
        'email': 'alice@example.com'
    })
    assert response.status_code == 201
    assert response.json()['name'] == 'Alice'

# 실행 (DB 필요)
docker-compose -f test.yml up --abort-on-container-exit

# 특징:
✅ 실제 DB 사용
✅ API 통합 검증
⏱️ 느림 (초 단위)
```

#### E2E Tests

```javascript
// tests/e2e/login.spec.js
test('user can login', async ({ page }) => {
  await page.goto('http://localhost:3000/login');
  await page.fill('#email', 'user@example.com');
  await page.fill('#password', 'password123');
  await page.click('button[type="submit"]');
  
  await expect(page).toHaveURL('/dashboard');
});

// 실행 (전체 스택 필요)
docker-compose -f e2e.yml up

# 특징:
✅ 사용자 시나리오
✅ 브라우저 시뮬레이션
⏱️ 매우 느림 (분 단위)
```

---

## 🔧 실습 1: Unit Tests in Container

### Step 1: 테스트용 Dockerfile

```dockerfile
# Dockerfile.test
FROM node:18-alpine AS test

WORKDIR /app

# 의존성 설치
COPY package*.json ./
RUN npm ci

# 소스 코드 복사
COPY . .

# 테스트 실행
CMD ["npm", "test"]
```

### Step 2: 테스트 코드

```javascript
// tests/calculator.test.js
const { add, subtract } = require('../src/calculator');

describe('Calculator', () => {
  test('add two numbers', () => {
    expect(add(2, 3)).toBe(5);
    expect(add(-1, 1)).toBe(0);
  });
  
  test('subtract two numbers', () => {
    expect(subtract(5, 3)).toBe(2);
    expect(subtract(1, 1)).toBe(0);
  });
});
```

### Step 3: GitHub Actions

```yaml
# .github/workflows/test.yml
name: Run Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build test image
        run: docker build -f Dockerfile.test -t myapp:test .
      
      - name: Run unit tests
        run: docker run --rm myapp:test
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
```

---

## 🔧 실습 2: Integration Tests with Docker Compose

### Step 1: 테스트 환경 정의

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  # PostgreSQL (테스트용)
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    tmpfs:
      - /var/lib/postgresql/data  # 메모리에 저장 (빠름)
  
  # Redis (테스트용)
  redis:
    image: redis:7-alpine
  
  # 애플리케이션
  app:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      DATABASE_URL: postgresql://testuser:testpass@postgres:5432/testdb
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis
    command: npm run test:integration
```

### Step 2: Integration Test 코드

```javascript
// tests/integration/api.test.js
const request = require('supertest');
const app = require('../../src/app');
const { sequelize } = require('../../src/models');

beforeAll(async () => {
  // DB 연결 및 마이그레이션
  await sequelize.sync({ force: true });
});

afterAll(async () => {
  await sequelize.close();
});

describe('API Integration Tests', () => {
  test('POST /users - create user', async () => {
    const response = await request(app)
      .post('/users')
      .send({
        name: 'Alice',
        email: 'alice@example.com'
      });
    
    expect(response.status).toBe(201);
    expect(response.body).toHaveProperty('id');
    expect(response.body.name).toBe('Alice');
  });
  
  test('GET /users/:id - get user', async () => {
    // 먼저 사용자 생성
    const createRes = await request(app)
      .post('/users')
      .send({ name: 'Bob', email: 'bob@example.com' });
    
    const userId = createRes.body.id;
    
    // 조회
    const response = await request(app).get(`/users/${userId}`);
    
    expect(response.status).toBe(200);
    expect(response.body.name).toBe('Bob');
  });
  
  test('Redis caching', async () => {
    const redis = require('../../src/redis');
    
    await redis.set('test-key', 'test-value', 'EX', 60);
    const value = await redis.get('test-key');
    
    expect(value).toBe('test-value');
  });
});
```

### Step 3: 실행

```bash
# 1. 테스트 환경 시작 및 테스트 실행
docker-compose -f docker-compose.test.yml up \
  --abort-on-container-exit \
  --exit-code-from app

# 2. 정리
docker-compose -f docker-compose.test.yml down -v

# 3. CI/CD에서
# .github/workflows/integration-test.yml
- name: Run integration tests
  run: |
    docker-compose -f docker-compose.test.yml up \
      --abort-on-container-exit \
      --exit-code-from app
    docker-compose -f docker-compose.test.yml down -v
```

---

## 🔧 실습 3: Container Structure Tests

### Step 1: Container Structure Test 도구

```bash
# 설치
curl -LO https://storage.googleapis.com/container-structure-test/latest/container-structure-test-linux-amd64
chmod +x container-structure-test-linux-amd64
sudo mv container-structure-test-linux-amd64 /usr/local/bin/container-structure-test

# 또는 Docker로
docker pull gcr.io/gcp-runtimes/container-structure-test
```

### Step 2: 테스트 설정

```yaml
# container-structure-test.yaml
schemaVersion: '2.0.0'

# 파일 존재 테스트
fileExistenceTests:
  - name: 'App binary exists'
    path: '/app/main'
    shouldExist: true
    permissions: '-rwxr-xr-x'
  
  - name: 'Config file exists'
    path: '/etc/app/config.yaml'
    shouldExist: true
  
  - name: 'No shell'
    path: '/bin/sh'
    shouldExist: false

# 메타데이터 테스트
metadataTest:
  exposedPorts: ["8080"]
  volumes: ["/data"]
  workdir: "/app"
  env:
    - key: NODE_ENV
      value: production

# 명령 실행 테스트
commandTests:
  - name: "Node version"
    command: "node"
    args: ["--version"]
    expectedOutput: ["v18.*"]
  
  - name: "Health check endpoint"
    command: "curl"
    args: ["-f", "http://localhost:8080/health"]
    expectedOutput: ["healthy"]

# 파일 내용 테스트
fileContentTests:
  - name: "Package.json has correct name"
    path: "/app/package.json"
    expectedContents: ['"name": "myapp"']
```

### Step 3: 실행

```bash
# 로컬 실행
container-structure-test test \
  --image myapp:latest \
  --config container-structure-test.yaml

# CI/CD
# .github/workflows/container-test.yml
- name: Build image
  run: docker build -t myapp:test .

- name: Run container structure tests
  run: |
    container-structure-test test \
      --image myapp:test \
      --config container-structure-test.yaml
```

---

## 🔧 실습 4: E2E Tests with Playwright

### Step 1: Playwright 설정

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],
  
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Step 2: E2E 테스트

```typescript
// tests/e2e/user-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('User Flow', () => {
  test('complete user journey', async ({ page }) => {
    // 1. 홈페이지 방문
    await page.goto('/');
    await expect(page).toHaveTitle(/My App/);
    
    // 2. 회원가입
    await page.click('text=Sign Up');
    await page.fill('#name', 'Test User');
    await page.fill('#email', 'test@example.com');
    await page.fill('#password', 'SecurePass123!');
    await page.click('button[type="submit"]');
    
    // 3. 대시보드 확인
    await expect(page).toHaveURL(/dashboard/);
    await expect(page.locator('h1')).toContainText('Welcome, Test User');
    
    // 4. 데이터 생성
    await page.click('text=New Item');
    await page.fill('#title', 'Test Item');
    await page.click('button:has-text("Create")');
    
    // 5. 생성 확인
    await expect(page.locator('.item-list')).toContainText('Test Item');
    
    // 6. 로그아웃
    await page.click('#user-menu');
    await page.click('text=Logout');
    await expect(page).toHaveURL('/');
  });
});
```

### Step 3: Docker Compose E2E

```yaml
# docker-compose.e2e.yml
version: '3.8'

services:
  # Frontend
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      API_URL: http://backend:8080
    depends_on:
      - backend
  
  # Backend
  backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/app
    depends_on:
      - postgres
  
  # Database
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: app
  
  # Playwright Tests
  playwright:
    image: mcr.microsoft.com/playwright:v1.40.0
    working_dir: /app
    volumes:
      - ./tests:/app/tests
      - ./playwright.config.ts:/app/playwright.config.ts
    command: npx playwright test
    depends_on:
      - frontend
    environment:
      BASE_URL: http://frontend:3000
```

```bash
# 실행
docker-compose -f docker-compose.e2e.yml up \
  --abort-on-container-exit \
  --exit-code-from playwright

# 정리
docker-compose -f docker-compose.e2e.yml down -v
```

---

## 🔧 실습 5: Performance Testing in Containers

### Step 1: Load Testing with k6

```javascript
// tests/load/script.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 10 },   // Ramp-up
    { duration: '3m', target: 100 },  // Stay at 100 users
    { duration: '1m', target: 0 },    // Ramp-down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% under 500ms
    http_req_failed: ['rate<0.01'],   // Error rate < 1%
  },
};

export default function () {
  // GET request
  const res = http.get('http://api:8080/users');
  
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  
  // POST request
  const payload = JSON.stringify({
    name: 'Load Test User',
    email: `user${Math.random()}@example.com`,
  });
  
  const params = {
    headers: { 'Content-Type': 'application/json' },
  };
  
  const createRes = http.post('http://api:8080/users', payload, params);
  
  check(createRes, {
    'create status is 201': (r) => r.status === 201,
  });
  
  sleep(1);
}
```

### Step 2: Docker Compose Load Test

```yaml
# docker-compose.load.yml
version: '3.8'

services:
  # API (테스트 대상)
  api:
    build: .
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/app
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis
  
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: app
  
  redis:
    image: redis:7-alpine
  
  # k6 Load Testing
  k6:
    image: grafana/k6:latest
    volumes:
      - ./tests/load:/scripts
    command: run /scripts/script.js
    depends_on:
      - api
```

```bash
# 실행
docker-compose -f docker-compose.load.yml up \
  --abort-on-container-exit \
  --exit-code-from k6

# 결과 예시:
# ✓ status is 200
# ✓ response time < 500ms
# 
# checks.........................: 100.00% ✓ 12000 ✗ 0
# data_received..................: 2.4 MB  40 kB/s
# http_req_duration..............: avg=245ms min=120ms med=230ms max=450ms p(95)=380ms
# http_reqs......................: 6000    100/s
```

---

## 🔧 실습 6: Test Reports and Artifacts

### Step 1: Test Report 생성

```yaml
# .github/workflows/test-report.yml
name: Test with Reports

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tests
        run: |
          docker-compose -f docker-compose.test.yml up \
            --abort-on-container-exit
      
      - name: Copy test results
        if: always()
        run: |
          docker cp $(docker ps -aqf "name=app") \
            :/app/test-results ./test-results
      
      - name: Publish Test Report
        uses: mikepenz/action-junit-report@v4
        if: always()
        with:
          report_paths: 'test-results/junit.xml'
          check_name: 'Test Results'
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./test-results/coverage/lcov.info
      
      - name: Upload Artifacts
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: test-results/
```

### Step 2: HTML Report

```javascript
// jest.config.js
module.exports = {
  reporters: [
    'default',
    [
      'jest-html-reporter',
      {
        pageTitle: 'Test Report',
        outputPath: 'test-results/report.html',
        includeFailureMsg: true,
        includeConsoleLog: true,
      },
    ],
  ],
  coverageReporters: ['html', 'lcov', 'text-summary'],
  coverageDirectory: 'test-results/coverage',
};
```

---

## 💡 주요 패턴 정리

```
┌──────────────────────┬────────────────────────────┐
│ 테스트 타입             │ 실행 시간                    │
├──────────────────────┼────────────────────────────┤
│ Unit                 │ < 5초                       │
├──────────────────────┼────────────────────────────┤
│ Integration          │ 30초 - 2분                  │
├──────────────────────┼────────────────────────────┤
│ E2E                  │ 2분 - 10분                  │
├──────────────────────┼────────────────────────────┤
│ Performance          │ 5분 - 30분                  │
└──────────────────────┴────────────────────────────┘

Best Practices:
1. Pyramid 구조 유지
2. 격리된 테스트 환경
3. 빠른 피드백
4. 자동 정리 (cleanup)
5. CI/CD 통합
```

---

## 🎓 연습 문제

### 문제 1: 테스트가 로컬에서는 성공하는데 CI에서 실패한다면?

<details>
<summary>정답 보기</summary>

**원인:**
```bash
# 환경 차이
- 로컬: macOS, 8GB RAM
- CI: Linux, 2GB RAM

# 타이밍 이슈
- 로컬: 빠른 네트워크
- CI: 느린 네트워크
```

**해결:**
```yaml
# 1. 동일한 환경 (Docker)
test:
  image: node:18-alpine
  # 로컬과 CI 모두 동일한 이미지

# 2. 충분한 타임아웃
jest.setTimeout(10000);  # 10초

# 3. 비동기 대기
await waitFor(() => {
  expect(element).toBeInTheDocument();
}, { timeout: 5000 });

# 4. Retry 설정
retries: process.env.CI ? 2 : 0
```

**디버깅:**
```yaml
- name: Debug
  if: failure()
  run: |
    docker logs app
    docker exec app cat /app/test.log
```

</details>

### 문제 2: 테스트 데이터베이스를 어떻게 격리하는가?

<details>
<summary>정답 보기</summary>

**방법 1: 트랜잭션 롤백**
```javascript
beforeEach(async () => {
  await sequelize.transaction(async (t) => {
    // 테스트 실행
  });
  // 자동 롤백
});
```

**방법 2: DB별 격리**
```yaml
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: test_${TEST_ID}  # 고유 DB
```

**방법 3: tmpfs (메모리)**
```yaml
services:
  postgres:
    image: postgres:15-alpine
    tmpfs:
      - /var/lib/postgresql/data
    # 메모리에만 저장 → 빠르고 자동 정리
```

**방법 4: 테스트 후 정리**
```javascript
afterAll(async () => {
  await sequelize.drop();  # 모든 테이블 삭제
  await sequelize.close();
});
```

</details>

### 문제 3: E2E 테스트가 너무 느리다면?

<details>
<summary>정답 보기</summary>

**최적화 전략:**

**1. 병렬 실행:**
```typescript
// playwright.config.ts
workers: process.env.CI ? 4 : undefined,
fullyParallel: true,
```

**2. 선택적 실행:**
```yaml
# 중요한 것만
on:
  push:
    branches: [main]
  pull_request:
    # E2E는 main PR만
```

**3. 스모크 테스트:**
```typescript
// 핵심 시나리오만
test.describe('Critical Path', () => {
  test('user can login and view dashboard', ...);
  // 나머지는 nightly
});
```

**4. Headless 모드:**
```typescript
use: {
  headless: true,  # UI 없이 (빠름)
}
```

**5. 비디오/스크린샷 최소화:**
```typescript
use: {
  video: 'retain-on-failure',  # 실패 시만
  screenshot: 'only-on-failure',
}
```

</details>

---

## 📌 핵심 요약

```
Automated Testing 핵심:
1. Test Pyramid (70-20-10)
2. 격리된 환경 (Docker)
3. 빠른 피드백 (< 5분)
4. CI/CD 통합
5. 자동 정리

Best Practices:
✅ Unit tests 많이
✅ Docker Compose로 격리
✅ 병렬 실행
✅ 빠른 피드백
✅ Coverage tracking
```

---

## 📚 참고 자료

- [Testing Best Practices](https://testingjavascript.com/)
- [Docker Testing](https://docs.docker.com/language/nodejs/run-tests/)
- [Playwright Documentation](https://playwright.dev/)

---

## 🤔 생각해볼 문제

1. 테스트 커버리지 목표는 몇 %가 적절한가?
2. Flaky Test (간헐적 실패)를 어떻게 처리하는가?
3. 프로덕션 데이터로 테스트해야 하는가?

> 💡 **답변**:
> 
> **1) 커버리지 목표:**
> 
> ```
> 80% 권장 (과도한 목표는 역효과)
> 
> 중요도 기반:
> - Critical Path: 100%
> - Business Logic: 90%
> - Utilities: 70%
> - UI Components: 60%
> 
> ❌ 100% 커버리지 추구 금지
> - 테스트를 위한 테스트
> - 유지보수 부담
> ```
> 
> **2) Flaky Test 처리:**
> 
> ```javascript
> // 1. Retry (단기)
> test.retry(2);
> 
> // 2. Timeout 증가
> test.setTimeout(10000);
> 
> // 3. 대기 개선
> await waitForElement();  # sleep 대신
> 
> // 4. 격리 강화
> beforeEach(() => clearDatabase());
> 
> // 5. 마지막 수단: skip
> test.skip('flaky test', ...);
> ```
> 
> **3) 프로덕션 데이터:**
> 
> ```
> ❌ 절대 프로덕션 DB에서 테스트
> 
> ✅ 대안:
> 1. Anonymized Dump
>    - 프로덕션 스냅샷
>    - 민감 정보 제거
> 
> 2. Synthetic Data
>    - Faker.js로 생성
>    - 실제 패턴 모방
> 
> 3. Staging 환경
>    - 프로덕션과 유사
>    - 별도 데이터
> ```

---

<div align="center">

**[⬅️ 이전: Registry Setup](./03-Registry-Setup.md)** | **[다음: Security Scanning ➡️](./05-Security-Scanning.md)**

</div>
