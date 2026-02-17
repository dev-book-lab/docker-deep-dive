# 08. Docker Extensions - Docker Desktop 확장 개발

## 🎯 이 챕터에서 배울 것

- **Extension 아키텍처**: Docker Desktop 확장 시스템
- **Frontend 개발**: React 기반 UI 구축
- **Backend 통합**: Docker API 및 서비스 연동
- **Extension SDK**: Docker Desktop API 활용
- **배포 및 마켓플레이스**: Extension 패키징 및 배포
- **실전 예제**: 실용적인 Extension 개발

## 📌 왜 중요한가?

**"Docker Extensions를 통해 Docker Desktop을 커스터마이징하고 팀 워크플로우를 개선할 수 있습니다."**

```
Docker Extensions의 위치:

┌─────────────────────────────────────────────────────┐
│ Docker Desktop                                      │
│                                                     │
│  ┌────────────────────────────────────────────┐     │
│  │ Built-in Features                          │     │
│  │ - Containers                               │     │
│  │ - Images                                   │     │
│  │ - Volumes                                  │     │
│  │ - Dev Environments                         │     │
│  └────────────────────────────────────────────┘     │
│                                                     │
│  ┌────────────────────────────────────────────┐     │
│  │ Extensions (추가 기능) ◄─── 이 챕터의 핵심!      │     │
│  │                                            │     │
│  │ ┌──────────┐  ┌──────────┐  ┌──────────┐   │     │
│  │ │ Disk     │  │ Resource │  │ Logs     │   │     │
│  │ │ Usage    │  │ Usage    │  │ Viewer   │   │     │
│  │ └──────────┘  └──────────┘  └──────────┘   │     │
│  │                                            │     │
│  │ ┌──────────┐  ┌──────────┐  ┌──────────┐   │     │
│  │ │ Snyk     │  │ Trivy    │  │ Custom   │   │     │
│  │ │ Security │  │ Scanner  │  │ Tools    │   │     │
│  │ └──────────┘  └──────────┘  └──────────┘   │     │
│  └────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────┘

Extension의 구성:

┌─────────────────────────────────────────────────────┐
│ Docker Extension                                    │
│                                                     │
│  ┌────────────────────────────────────────────┐     │
│  │ Frontend (UI)                              │     │
│  │ - React 컴포넌트                             │     │
│  │ - Docker Desktop UI Kit                    │     │
│  │ - 사용자 인터랙션                              │     │
│  └───────────────┬────────────────────────────┘     │
│                  │                                  │
│                  │ Extension SDK                    │
│                  ▼                                  │
│  ┌────────────────────────────────────────────┐     │
│  │ Backend (선택)                              │     │
│  │ - REST API 서버                             │     │
│  │ - 데이터 처리                                 │     │
│  │ - 외부 서비스 통합                             │     │
│  └───────────────┬────────────────────────────┘     │
│                  │                                  │
│                  │ Docker API                       │
│                  ▼                                  │
│  ┌────────────────────────────────────────────┐     │
│  │ Docker Engine                              │     │
│  │ - Container 관리                            │     │
│  │ - Image 관리                                │     │
│  └────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────┘

Extension의 장점:
✅ Docker Desktop UI에 통합
✅ Docker API 직접 접근
✅ 팀 워크플로우 커스터마이징
✅ Marketplace에 배포 가능

사용 사례:
┌──────────────────┬──────────────────────────────┐
│ 카테고리          │ 예시                          │
├──────────────────┼──────────────────────────────┤
│ 모니터링          │ 리소스 사용량, 로그 뷰어        │
├──────────────────┼──────────────────────────────┤
│ 보안              │ 취약점 스캔, 이미지 서명        │
├──────────────────┼──────────────────────────────┤
│ 개발 도구         │ DB 클라이언트, API 테스터      │
├──────────────────┼──────────────────────────────┤
│ CI/CD 통합        │ Jenkins, GitLab 연동          │
├──────────────────┼──────────────────────────────┤
│ 클라우드 통합     │ AWS, Azure, GCP 배포          │
└──────────────────┴──────────────────────────────┘
```

**실무 영향:**
- **생산성 향상**: 반복 작업 자동화
- **팀 협업**: 공통 도구 제공
- **표준화**: 일관된 워크플로우
- **통합**: 기존 도구와 연동

---

## 🔬 Deep Dive

### 1. Extension 아키텍처

#### Extension 구조

```
Extension 파일 구조:

my-extension/
├── Dockerfile                    ← Extension 빌드 정의
├── metadata.json                 ← Extension 메타데이터
├── docker-compose.yaml           ← Backend 서비스 (선택)
├── ui/                          ← Frontend (React)
│   ├── package.json
│   ├── src/
│   │   ├── App.tsx              ← 메인 컴포넌트
│   │   └── index.tsx
│   └── public/
│       ├── index.html
│       └── icon.svg             ← Extension 아이콘
└── backend/                     ← Backend (선택)
    ├── package.json
    └── src/
        └── server.js

Dockerfile 예:
FROM --platform=$BUILDPLATFORM node:18-alpine AS client-builder
WORKDIR /ui
COPY ui/package*.json ./
RUN npm ci
COPY ui/ ./
RUN npm run build

FROM alpine
LABEL org.opencontainers.image.title="My Extension" \
      org.opencontainers.image.description="Extension description" \
      org.opencontainers.image.vendor="Your Name" \
      com.docker.desktop.extension.api.version=">= 0.3.0" \
      com.docker.extension.screenshots="" \
      com.docker.extension.detailed-description="" \
      com.docker.extension.publisher-url="" \
      com.docker.extension.additional-urls="" \
      com.docker.extension.changelog=""

COPY --from=client-builder /ui/build ui
COPY metadata.json .
COPY docker.svg .

metadata.json:
{
  "icon": "docker.svg",
  "ui": {
    "dashboard-tab": {
      "title": "My Extension",
      "root": "/ui",
      "src": "index.html"
    }
  },
  "vm": {
    "image": "${DESKTOP_PLUGIN_IMAGE}"
  }
}
```

#### Extension SDK

```typescript
// Extension SDK 주요 API

// 1. Docker Desktop API
import { createDockerDesktopClient } from '@docker/extension-api-client';

const ddClient = createDockerDesktopClient();

// Docker Engine API 호출
const containers = await ddClient.docker.listContainers();

// CLI 실행
const result = await ddClient.extension.host.cli.exec('docker', [
  'ps',
  '-a'
]);

// 2. UI Components (Material-UI 기반)
import { 
  Box, 
  Button, 
  Typography,
  Table,
  TableBody,
  TableCell,
  TableRow
} from '@mui/material';

// 3. Navigation
ddClient.desktopUI.navigate.viewContainer(containerId);
ddClient.desktopUI.navigate.viewImage(imageId);

// 4. Toast 알림
ddClient.desktopUI.toast.success('Operation successful');
ddClient.desktopUI.toast.error('Error occurred');

// 5. Dialog
const result = await ddClient.desktopUI.dialog.showOpenDialog({
  properties: ['openFile']
});

// 6. Extension Backend 호출
const response = await ddClient.extension.vm.service.get('/api/data');
```

---

### 2. Frontend 개발

#### React 컴포넌트 구조

```typescript
// src/App.tsx
import React, { useEffect, useState } from 'react';
import { createDockerDesktopClient } from '@docker/extension-api-client';
import {
  Box,
  Button,
  Typography,
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableRow,
  Stack
} from '@mui/material';

const ddClient = createDockerDesktopClient();

interface Container {
  Id: string;
  Names: string[];
  Image: string;
  State: string;
  Status: string;
}

export function App() {
  const [containers, setContainers] = useState<Container[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadContainers();
  }, []);

  const loadContainers = async () => {
    try {
      setLoading(true);
      const result = await ddClient.docker.listContainers();
      setContainers(result);
    } catch (error) {
      ddClient.desktopUI.toast.error(`Error loading containers: ${error}`);
    } finally {
      setLoading(false);
    }
  };

  const handleStartContainer = async (id: string) => {
    try {
      await ddClient.docker.cli.exec('start', [id]);
      ddClient.desktopUI.toast.success('Container started');
      loadContainers();
    } catch (error) {
      ddClient.desktopUI.toast.error(`Error starting container: ${error}`);
    }
  };

  const handleStopContainer = async (id: string) => {
    try {
      await ddClient.docker.cli.exec('stop', [id]);
      ddClient.desktopUI.toast.success('Container stopped');
      loadContainers();
    } catch (error) {
      ddClient.desktopUI.toast.error(`Error stopping container: ${error}`);
    }
  };

  if (loading) {
    return (
      <Box display="flex" justifyContent="center" alignItems="center" height="100vh">
        <Typography>Loading...</Typography>
      </Box>
    );
  }

  return (
    <Box p={3}>
      <Stack direction="row" justifyContent="space-between" alignItems="center" mb={2}>
        <Typography variant="h3">Container Manager</Typography>
        <Button variant="contained" onClick={loadContainers}>
          Refresh
        </Button>
      </Stack>

      <Table>
        <TableHead>
          <TableRow>
            <TableCell>Container ID</TableCell>
            <TableCell>Name</TableCell>
            <TableCell>Image</TableCell>
            <TableCell>State</TableCell>
            <TableCell>Actions</TableCell>
          </TableRow>
        </TableHead>
        <TableBody>
          {containers.map((container) => (
            <TableRow key={container.Id}>
              <TableCell>{container.Id.substring(0, 12)}</TableCell>
              <TableCell>{container.Names[0]?.replace('/', '')}</TableCell>
              <TableCell>{container.Image}</TableCell>
              <TableCell>{container.State}</TableCell>
              <TableCell>
                <Stack direction="row" spacing={1}>
                  {container.State === 'running' ? (
                    <Button 
                      size="small" 
                      onClick={() => handleStopContainer(container.Id)}
                    >
                      Stop
                    </Button>
                  ) : (
                    <Button 
                      size="small" 
                      onClick={() => handleStartContainer(container.Id)}
                    >
                      Start
                    </Button>
                  )}
                  <Button 
                    size="small"
                    onClick={() => ddClient.desktopUI.navigate.viewContainer(container.Id)}
                  >
                    View Details
                  </Button>
                </Stack>
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </Box>
  );
}
```

---

### 3. Backend 서비스

#### Express 서버 예제

```typescript
// backend/src/server.ts
import express from 'express';
import Docker from 'dockerode';

const app = express();
const docker = new Docker({ socketPath: '/var/run/docker.sock' });

app.use(express.json());

// Container 통계
app.get('/api/stats', async (req, res) => {
  try {
    const containers = await docker.listContainers();
    
    const stats = await Promise.all(
      containers.map(async (containerInfo) => {
        const container = docker.getContainer(containerInfo.Id);
        const stats = await container.stats({ stream: false });
        
        // CPU 계산
        const cpuDelta = stats.cpu_stats.cpu_usage.total_usage - 
                        stats.precpu_stats.cpu_usage.total_usage;
        const systemDelta = stats.cpu_stats.system_cpu_usage - 
                           stats.precpu_stats.system_cpu_usage;
        const cpuPercent = (cpuDelta / systemDelta) * 100.0;
        
        // Memory 계산
        const memoryUsage = stats.memory_stats.usage / (1024 * 1024); // MB
        const memoryLimit = stats.memory_stats.limit / (1024 * 1024); // MB
        
        return {
          id: containerInfo.Id.substring(0, 12),
          name: containerInfo.Names[0]?.replace('/', ''),
          cpu: cpuPercent.toFixed(2),
          memory: {
            usage: memoryUsage.toFixed(2),
            limit: memoryLimit.toFixed(2),
            percent: ((memoryUsage / memoryLimit) * 100).toFixed(2)
          }
        };
      })
    );
    
    res.json(stats);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Image 크기 분석
app.get('/api/images/analysis', async (req, res) => {
  try {
    const images = await docker.listImages();
    
    const analysis = images.map(image => ({
      id: image.Id.split(':')[1].substring(0, 12),
      tags: image.RepoTags || ['<none>'],
      size: (image.Size / (1024 * 1024)).toFixed(2), // MB
      created: new Date(image.Created * 1000).toISOString()
    }));
    
    // 크기순 정렬
    analysis.sort((a, b) => parseFloat(b.size) - parseFloat(a.size));
    
    res.json(analysis);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Dockerfile 분석
app.post('/api/dockerfile/analyze', async (req, res) => {
  try {
    const { dockerfile } = req.body;
    
    // 간단한 분석
    const lines = dockerfile.split('\n');
    const analysis = {
      layers: 0,
      instructions: {},
      recommendations: []
    };
    
    lines.forEach(line => {
      const trimmed = line.trim();
      if (!trimmed || trimmed.startsWith('#')) return;
      
      const instruction = trimmed.split(' ')[0].toUpperCase();
      analysis.instructions[instruction] = 
        (analysis.instructions[instruction] || 0) + 1;
      
      // Layer 생성 명령어
      if (['RUN', 'COPY', 'ADD'].includes(instruction)) {
        analysis.layers++;
      }
    });
    
    // 권장사항
    if (analysis.layers > 10) {
      analysis.recommendations.push('Consider reducing layers (currently ' + analysis.layers + ')');
    }
    if (!analysis.instructions['USER']) {
      analysis.recommendations.push('Consider adding USER instruction (security)');
    }
    if (analysis.instructions['RUN'] > 5) {
      analysis.recommendations.push('Consider combining RUN instructions');
    }
    
    res.json(analysis);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

const PORT = 8080;
app.listen(PORT, () => {
  console.log(`Backend listening on port ${PORT}`);
});
```

---

## 🔧 실습 1: 간단한 Container Stats Extension

### Step 1: 프로젝트 초기화

```bash
# 1. Extension 디렉토리 생성
mkdir container-stats-extension
cd container-stats-extension

# 2. Frontend 초기화
npx create-react-app ui --template typescript
cd ui
npm install @docker/extension-api-client @mui/material @emotion/react @emotion/styled
cd ..

# 3. 파일 구조 생성
mkdir -p ui/public
```

### Step 2: Frontend 구현

```typescript
// ui/src/App.tsx
import React, { useEffect, useState } from 'react';
import { createDockerDesktopClient } from '@docker/extension-api-client';
import {
  Box,
  Typography,
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableRow,
  LinearProgress,
  Button,
  Stack
} from '@mui/material';

const ddClient = createDockerDesktopClient();

interface ContainerStats {
  id: string;
  name: string;
  cpu: number;
  memory: {
    usage: number;
    limit: number;
    percent: number;
  };
}

export function App() {
  const [stats, setStats] = useState<ContainerStats[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadStats();
    const interval = setInterval(loadStats, 3000); // 3초마다 갱신
    return () => clearInterval(interval);
  }, []);

  const loadStats = async () => {
    try {
      const containers = await ddClient.docker.listContainers();
      
      const statsData = await Promise.all(
        containers.map(async (c) => {
          const statsStr = await ddClient.docker.cli.exec('stats', [
            '--no-stream',
            '--format',
            '{{json .}}',
            c.Id
          ]);
          
          const stat = JSON.parse(statsStr.stdout);
          
          return {
            id: c.Id.substring(0, 12),
            name: c.Names[0]?.replace('/', '') || 'unknown',
            cpu: parseFloat(stat.CPUPerc.replace('%', '')),
            memory: {
              usage: parseFloat(stat.MemUsage.split('/')[0]),
              limit: parseFloat(stat.MemUsage.split('/')[1]),
              percent: parseFloat(stat.MemPerc.replace('%', ''))
            }
          };
        })
      );
      
      setStats(statsData);
      setLoading(false);
    } catch (error) {
      console.error('Error loading stats:', error);
      setLoading(false);
    }
  };

  if (loading) {
    return (
      <Box p={3}>
        <Typography variant="h4" gutterBottom>
          Container Resource Usage
        </Typography>
        <LinearProgress />
      </Box>
    );
  }

  return (
    <Box p={3}>
      <Stack direction="row" justifyContent="space-between" alignItems="center" mb={2}>
        <Typography variant="h4">
          Container Resource Usage
        </Typography>
        <Button variant="outlined" onClick={loadStats}>
          Refresh
        </Button>
      </Stack>

      {stats.length === 0 ? (
        <Typography>No running containers</Typography>
      ) : (
        <Table>
          <TableHead>
            <TableRow>
              <TableCell>Container</TableCell>
              <TableCell>CPU %</TableCell>
              <TableCell>Memory Usage</TableCell>
              <TableCell>Memory %</TableCell>
            </TableRow>
          </TableHead>
          <TableBody>
            {stats.map((stat) => (
              <TableRow key={stat.id}>
                <TableCell>
                  <Typography variant="body2" fontWeight="bold">
                    {stat.name}
                  </Typography>
                  <Typography variant="caption" color="text.secondary">
                    {stat.id}
                  </Typography>
                </TableCell>
                <TableCell>
                  <Box>
                    <Typography variant="body2">
                      {stat.cpu.toFixed(2)}%
                    </Typography>
                    <LinearProgress 
                      variant="determinate" 
                      value={Math.min(stat.cpu, 100)} 
                      sx={{ mt: 1 }}
                    />
                  </Box>
                </TableCell>
                <TableCell>
                  {stat.memory.usage.toFixed(2)} MB / {stat.memory.limit.toFixed(2)} MB
                </TableCell>
                <TableCell>
                  <Box>
                    <Typography variant="body2">
                      {stat.memory.percent.toFixed(2)}%
                    </Typography>
                    <LinearProgress 
                      variant="determinate" 
                      value={stat.memory.percent} 
                      color={stat.memory.percent > 80 ? 'error' : 'primary'}
                      sx={{ mt: 1 }}
                    />
                  </Box>
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      )}
    </Box>
  );
}
```

### Step 3: Dockerfile 및 메타데이터

```dockerfile
# Dockerfile
FROM --platform=$BUILDPLATFORM node:18-alpine AS client-builder
WORKDIR /ui
COPY ui/package*.json ./
RUN npm ci
COPY ui/ ./
RUN npm run build

FROM alpine
LABEL org.opencontainers.image.title="Container Stats" \
      org.opencontainers.image.description="Real-time container resource monitoring" \
      org.opencontainers.image.vendor="Your Name" \
      com.docker.desktop.extension.api.version=">= 0.3.0"

COPY --from=client-builder /ui/build ui
COPY metadata.json .
COPY docker.svg .
```

```json
// metadata.json
{
  "icon": "docker.svg",
  "ui": {
    "dashboard-tab": {
      "title": "Container Stats",
      "root": "/ui",
      "src": "index.html"
    }
  }
}
```

### Step 4: 빌드 및 설치

```bash
# 1. Extension 빌드
docker build -t container-stats:latest .

# 2. Extension 설치
docker extension install container-stats:latest

# 3. Docker Desktop에서 확인
# Extensions → Container Stats 탭 열기

# 4. 업데이트
docker extension update container-stats:latest

# 5. 제거
docker extension rm container-stats:latest
```

---

## 🔧 실습 2: Image Analyzer Extension (Backend 포함)

### Step 1: Backend 구현

```typescript
// backend/src/server.ts
import express from 'express';
import Docker from 'dockerode';

const app = express();
const docker = new Docker({ socketPath: '/var/run/docker.sock' });

app.use(express.json());
app.use((req, res, next) => {
  res.header('Access-Control-Allow-Origin', '*');
  res.header('Access-Control-Allow-Headers', 'Origin, X-Requested-With, Content-Type, Accept');
  next();
});

// 이미지 분석
app.get('/analyze', async (req, res) => {
  try {
    const images = await docker.listImages({ all: false });
    
    const analysis = images.map(image => {
      const sizeMB = image.Size / (1024 * 1024);
      const age = Math.floor((Date.now() - image.Created * 1000) / (1000 * 60 * 60 * 24));
      
      return {
        id: image.Id.split(':')[1]?.substring(0, 12) || image.Id,
        tags: image.RepoTags || ['<none>'],
        size: sizeMB.toFixed(2),
        created: new Date(image.Created * 1000).toISOString(),
        age: age,
        recommendation: getRecommendation(sizeMB, age)
      };
    });
    
    res.json({
      total: images.length,
      totalSize: (images.reduce((sum, img) => sum + img.Size, 0) / (1024 * 1024 * 1024)).toFixed(2),
      images: analysis
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

function getRecommendation(sizeMB: number, ageDays: number): string {
  const recommendations = [];
  
  if (sizeMB > 1000) {
    recommendations.push('Large image (>1GB)');
  }
  if (ageDays > 90) {
    recommendations.push('Old image (>90 days)');
  }
  if (sizeMB > 500 && ageDays > 30) {
    recommendations.push('Consider cleanup');
  }
  
  return recommendations.join(', ') || 'OK';
}

app.listen(8080, () => {
  console.log('Backend running on port 8080');
});
```

### Step 2: docker-compose.yaml

```yaml
# docker-compose.yaml
services:
  backend:
    image: ${DESKTOP_PLUGIN_IMAGE}
    command: node /backend/server.js
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

### Step 3: Frontend (Backend 호출)

```typescript
// ui/src/App.tsx
import React, { useEffect, useState } from 'react';
import { createDockerDesktopClient } from '@docker/extension-api-client';
import {
  Box,
  Typography,
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableRow,
  Chip,
  Card,
  CardContent,
  Grid
} from '@mui/material';

const ddClient = createDockerDesktopClient();

interface ImageAnalysis {
  total: number;
  totalSize: string;
  images: Array<{
    id: string;
    tags: string[];
    size: string;
    age: number;
    recommendation: string;
  }>;
}

export function App() {
  const [analysis, setAnalysis] = useState<ImageAnalysis | null>(null);

  useEffect(() => {
    loadAnalysis();
  }, []);

  const loadAnalysis = async () => {
    try {
      const result = await ddClient.extension.vm?.service.get('/analyze');
      setAnalysis(result);
    } catch (error) {
      console.error('Error:', error);
    }
  };

  if (!analysis) {
    return <Typography>Loading...</Typography>;
  }

  return (
    <Box p={3}>
      <Typography variant="h3" gutterBottom>
        Image Analyzer
      </Typography>

      <Grid container spacing={2} mb={3}>
        <Grid item xs={6}>
          <Card>
            <CardContent>
              <Typography variant="h6">Total Images</Typography>
              <Typography variant="h4">{analysis.total}</Typography>
            </CardContent>
          </Card>
        </Grid>
        <Grid item xs={6}>
          <Card>
            <CardContent>
              <Typography variant="h6">Total Size</Typography>
              <Typography variant="h4">{analysis.totalSize} GB</Typography>
            </CardContent>
          </Card>
        </Grid>
      </Grid>

      <Table>
        <TableHead>
          <TableRow>
            <TableCell>Image</TableCell>
            <TableCell>Size (MB)</TableCell>
            <TableCell>Age (days)</TableCell>
            <TableCell>Recommendation</TableCell>
          </TableRow>
        </TableHead>
        <TableBody>
          {analysis.images.map((image) => (
            <TableRow key={image.id}>
              <TableCell>
                {image.tags.map(tag => (
                  <Chip key={tag} label={tag} size="small" sx={{ mr: 1 }} />
                ))}
              </TableCell>
              <TableCell>{image.size}</TableCell>
              <TableCell>{image.age}</TableCell>
              <TableCell>
                {image.recommendation !== 'OK' && (
                  <Chip label={image.recommendation} color="warning" size="small" />
                )}
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </Box>
  );
}
```

---

## 🔧 실습 3: Dockerfile Linter Extension

### Step 1: Frontend (Dockerfile 분석)

```typescript
// ui/src/App.tsx
import React, { useState } from 'react';
import { createDockerDesktopClient } from '@docker/extension-api-client';
import {
  Box,
  Typography,
  TextField,
  Button,
  List,
  ListItem,
  ListItemIcon,
  ListItemText,
  Alert,
  Stack
} from '@mui/material';
import ErrorIcon from '@mui/icons-material/Error';
import WarningIcon from '@mui/icons-material/Warning';
import CheckCircleIcon from '@mui/icons-material/CheckCircle';

const ddClient = createDockerDesktopClient();

interface LintResult {
  level: 'error' | 'warning' | 'info';
  message: string;
  line?: number;
}

export function App() {
  const [dockerfile, setDockerfile] = useState(`FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y nginx
COPY app /app
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]`);
  const [results, setResults] = useState<LintResult[]>([]);

  const analyzeDockerfile = () => {
    const lines = dockerfile.split('\n');
    const lintResults: LintResult[] = [];

    // 분석 규칙
    let hasUser = false;
    let runCommands = 0;
    let hasHealthcheck = false;

    lines.forEach((line, index) => {
      const trimmed = line.trim();
      
      // FROM 최신 태그 확인
      if (trimmed.startsWith('FROM') && trimmed.includes(':latest')) {
        lintResults.push({
          level: 'warning',
          message: 'Avoid using :latest tag for reproducibility',
          line: index + 1
        });
      }

      // USER 지시어 확인
      if (trimmed.startsWith('USER')) {
        hasUser = true;
      }

      // RUN 명령 개수
      if (trimmed.startsWith('RUN')) {
        runCommands++;
      }

      // HEALTHCHECK 확인
      if (trimmed.startsWith('HEALTHCHECK')) {
        hasHealthcheck = true;
      }

      // apt-get update && install 패턴
      if (trimmed.includes('apt-get update') && !trimmed.includes('&&')) {
        lintResults.push({
          level: 'warning',
          message: 'Combine apt-get update and install in single RUN',
          line: index + 1
        });
      }

      // COPY vs ADD
      if (trimmed.startsWith('ADD') && !trimmed.includes('.tar')) {
        lintResults.push({
          level: 'info',
          message: 'Prefer COPY over ADD unless extracting archives',
          line: index + 1
        });
      }
    });

    // 전체 체크
    if (!hasUser) {
      lintResults.push({
        level: 'warning',
        message: 'No USER instruction found (security risk)'
      });
    }

    if (runCommands > 5) {
      lintResults.push({
        level: 'info',
        message: `Consider reducing RUN instructions (current: ${runCommands})`
      });
    }

    if (!hasHealthcheck) {
      lintResults.push({
        level: 'info',
        message: 'No HEALTHCHECK instruction found'
      });
    }

    setResults(lintResults);
  };

  const getIcon = (level: string) => {
    switch (level) {
      case 'error': return <ErrorIcon color="error" />;
      case 'warning': return <WarningIcon color="warning" />;
      default: return <CheckCircleIcon color="info" />;
    }
  };

  return (
    <Box p={3}>
      <Typography variant="h3" gutterBottom>
        Dockerfile Linter
      </Typography>

      <Stack spacing={3}>
        <TextField
          multiline
          rows={15}
          fullWidth
          label="Dockerfile"
          value={dockerfile}
          onChange={(e) => setDockerfile(e.target.value)}
          variant="outlined"
          sx={{ fontFamily: 'monospace' }}
        />

        <Button 
          variant="contained" 
          onClick={analyzeDockerfile}
          size="large"
        >
          Analyze Dockerfile
        </Button>

        {results.length > 0 && (
          <Box>
            <Typography variant="h5" gutterBottom>
              Analysis Results ({results.length} issues)
            </Typography>
            <List>
              {results.map((result, index) => (
                <ListItem key={index}>
                  <ListItemIcon>
                    {getIcon(result.level)}
                  </ListItemIcon>
                  <ListItemText
                    primary={result.message}
                    secondary={result.line ? `Line ${result.line}` : undefined}
                  />
                </ListItem>
              ))}
            </List>
          </Box>
        )}

        {results.length === 0 && dockerfile && (
          <Alert severity="success">
            No issues found! Your Dockerfile looks good.
          </Alert>
        )}
      </Stack>
    </Box>
  );
}
```

---

## 💡 주요 명령어 정리

```bash
# ========== Extension 개발 ==========
# Extension 초기화
docker extension init <n>

# Extension 빌드
docker build -t <n>:latest .

# Extension 설치
docker extension install <n>:latest

# Extension 업데이트
docker extension update <n>:latest

# Extension 제거
docker extension rm <n>

# ========== Extension 관리 ==========
# Extension 목록
docker extension ls

# Extension 활성화/비활성화
docker extension enable <n>
docker extension disable <n>

# Extension 검증
docker extension validate <n>:latest

# ========== 개발 모드 ==========
# 개발 중 빌드 (hot reload)
docker extension dev ui-source <n> http://localhost:3000

# Extension 디버그
docker extension dev debug <n>

# Extension 리셋
docker extension dev reset <n>

# ========== 배포 ==========
# Marketplace에 제출
docker extension publish <n>:latest
```

---

## 🎓 연습 문제

### 문제 1: Extension에서 Docker API와 CLI를 각각 언제 사용해야 하는가?

<details>
<summary>정답 보기</summary>

**Docker API 사용:**
```typescript
// 구조화된 데이터 필요
const containers = await ddClient.docker.listContainers();
// → JSON 객체 배열 반환

// 여러 API 조합
const container = await ddClient.docker.getContainer(id);
const stats = await container.stats({ stream: false });
```

**장점:**
- 타입 안전
- 파싱 불필요
- 에러 처리 명확

**Docker CLI 사용:**
```typescript
// CLI 전용 기능
const result = await ddClient.docker.cli.exec('buildx', [
  'build',
  '--platform', 'linux/amd64,linux/arm64',
  '.'
]);

// 복잡한 출력 포맷
const output = await ddClient.docker.cli.exec('ps', [
  '--format', '{{.Names}}: {{.Status}}'
]);
```

**장점:**
- 최신 기능 즉시 사용
- 복잡한 명령어 실행
- 커스텀 포맷

**선택 기준:**
```
API 사용:
✅ Container/Image 조회
✅ Stats, Logs
✅ 타입 안전 필요

CLI 사용:
✅ buildx, compose 등
✅ 복잡한 필터링
✅ API 미지원 기능
```

</details>

### 문제 2: Extension Backend를 언제 추가해야 하는가?

<details>
<summary>정답 보기</summary>

**Backend 불필요한 경우:**
```typescript
// 단순 Docker API 호출만
const containers = await ddClient.docker.listContainers();
// Frontend에서 충분

// UI만 필요
<Button onClick={() => ddClient.docker.cli.exec('start', [id])}>
  Start
</Button>
```

**Backend 필요한 경우:**

1. **복잡한 데이터 처리:**
```typescript
// Backend: 여러 컨테이너 통계 집계
app.get('/api/aggregate-stats', async (req, res) => {
  const containers = await docker.listContainers();
  
  // 복잡한 계산
  const aggregated = containers.reduce((acc, c) => {
    // ...
  }, {});
  
  res.json(aggregated);
});
```

2. **외부 서비스 통합:**
```typescript
// Slack, GitHub, AWS 등 외부 API 호출
app.post('/api/notify', async (req, res) => {
  await fetch('https://hooks.slack.com/...', {
    method: 'POST',
    body: JSON.stringify({ text: 'Container failed' })
  });
});
```

3. **데이터베이스:**
```typescript
// 로그, 메트릭 저장
app.get('/api/history', async (req, res) => {
  const history = await db.query('SELECT * FROM metrics');
  res.json(history);
});
```

4. **인증/권한:**
```typescript
// API 키 보안 저장
app.use((req, res, next) => {
  const token = req.headers.authorization;
  if (!validateToken(token)) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
});
```

**선택 기준:**
```
Frontend만:
✅ Docker API 직접 호출
✅ 간단한 UI 로직
✅ 빠른 개발

Frontend + Backend:
✅ 복잡한 처리
✅ 외부 통합
✅ 데이터 저장
✅ 인증 필요
```

</details>

### 문제 3: Extension을 Marketplace에 배포하려면 어떤 준비가 필요한가?

<details>
<summary>정답 보기</summary>

**1. 메타데이터 완성:**
```dockerfile
# Dockerfile Labels
LABEL org.opencontainers.image.title="My Extension" \
      org.opencontainers.image.description="Detailed description" \
      org.opencontainers.image.vendor="Company Name" \
      org.opencontainers.image.licenses="MIT" \
      com.docker.desktop.extension.api.version=">= 0.3.0" \
      com.docker.extension.screenshots='[
        {"alt":"Screenshot 1","url":"https://example.com/screenshot1.png"},
        {"alt":"Screenshot 2","url":"https://example.com/screenshot2.png"}
      ]' \
      com.docker.extension.detailed-description="Long description with markdown" \
      com.docker.extension.publisher-url="https://example.com" \
      com.docker.extension.additional-urls='[
        {"title":"Documentation","url":"https://docs.example.com"},
        {"title":"Support","url":"https://support.example.com"}
      ]' \
      com.docker.extension.changelog="
        # v1.0.0
        - Initial release
        - Feature A
        - Feature B
      "
```

**2. 문서 작성:**
- README.md (설치/사용법)
- CHANGELOG.md (버전 히스토리)
- LICENSE (라이선스)

**3. 아이콘 및 스크린샷:**
```
icons/
├── icon.svg          # Extension 아이콘 (필수)
└── screenshot.png    # 스크린샷 (권장)

요구사항:
- 아이콘: SVG, 최소 256x256
- 스크린샷: PNG/JPG, 1280x720 권장
```

**4. 테스트:**
```bash
# 여러 플랫폼에서 테스트
docker build --platform linux/amd64 -t myext:amd64 .
docker build --platform linux/arm64 -t myext:arm64 .

# 검증
docker extension validate myext:latest
```

**5. 레지스트리 Push:**
```bash
# Docker Hub 또는 다른 레지스트리
docker tag myext:latest myorg/myext:1.0.0
docker push myorg/myext:1.0.0
```

**6. Marketplace 제출:**
```
1. Docker Hub 계정 필요
2. Extension 페이지 작성
3. 검토 대기 (Docker 팀)
4. 승인 후 게시
```

**체크리스트:**
```
✅ 메타데이터 완성 (title, description, vendor)
✅ 스크린샷 3개 이상
✅ 상세 설명 (markdown)
✅ CHANGELOG 작성
✅ LICENSE 파일
✅ README.md (사용법)
✅ 여러 플랫폼 테스트 (amd64, arm64)
✅ 검증 통과 (docker extension validate)
✅ 레지스트리에 Push
✅ 버전 태그 (semantic versioning)
```

</details>

---

## 📌 핵심 요약

```
┌──────────────────┬────────────────────────────────────┐
│ 구성 요소          │ 설명                                │
├──────────────────┼────────────────────────────────────┤
│ Frontend         │ React + Material-UI                │
│                  │ Extension SDK 사용                  │
├──────────────────┼────────────────────────────────────┤
│ Backend (선택)    │ Express/FastAPI 등                  │
│                  │ Docker API 접근                     │
├──────────────────┼────────────────────────────────────┤
│ Dockerfile       │ Multi-stage build                  │
│                  │ Labels (메타데이터)                   │
├──────────────────┼────────────────────────────────────┤
│ metadata.json    │ UI 위치, 아이콘 정의                   │
└──────────────────┴────────────────────────────────────┘

Extension SDK 주요 API:
- ddClient.docker (Docker API)
- ddClient.docker.cli (CLI 실행)
- ddClient.extension.vm.service (Backend 호출)
- ddClient.desktopUI (Toast, Dialog, Navigation)
```

---

## 📚 참고 자료

- [Docker Extensions Documentation](https://docs.docker.com/desktop/extensions/)
- [Extension SDK Reference](https://docs.docker.com/desktop/extensions-sdk/)
- [Extensions Marketplace](https://hub.docker.com/search?q=&type=extension)
- [Extension Samples](https://github.com/docker/extensions-sdk/tree/main/samples)

---

## 🤔 생각해볼 문제

1. Docker Extension과 Docker Plugin의 차이점은 무엇이며, 각각 어떤 상황에서 사용해야 하는가?
2. Extension Backend에서 Docker Socket에 직접 접근하는 것의 보안 위험은? 어떻게 완화할 수 있는가?
3. Extension을 여러 플랫폼(Windows, Mac, Linux)에서 동작하게 하려면 어떤 점을 고려해야 하는가?

> 💡 **답변**:
>
> **1) Extension vs Plugin:**
>
> **Docker Extension (Docker Desktop):**
> ```
> 위치: Docker Desktop UI 내부
> 용도: 사용자 인터페이스 확장
> 
> ┌─────────────────────────────┐
> │ Docker Desktop              │
> │ ┌─────────────────────────┐ │
> │ │ Extensions Tab          │ │
> │ │ ┌─────────────────────┐ │ │
> │ │ │ My Extension        │ │ │
> │ │ │ - React UI          │ │ │
> │ │ │ - Charts/Graphs     │ │ │
> │ │ │ - Interactive Tools │ │ │
> │ │ └─────────────────────┘ │ │
> │ └─────────────────────────┘ │
> └─────────────────────────────┘
> 
> 특징:
> ✅ UI 중심 (React 컴포넌트)
> ✅ Docker Desktop에만 동작
> ✅ 개발자 도구, 모니터링, 분석
> ✅ Marketplace에서 설치
> ✅ 사용자 친화적
> 
> 사용 사례:
> - 리소스 모니터링 대시보드
> - 취약점 스캔 UI
> - 로그 뷰어
> - DB 클라이언트
> - CI/CD 통합 UI
> ```
>
> **Docker Plugin (Docker Engine):**
> ```
> 위치: Docker Engine 내부
> 용도: Docker 핵심 기능 확장
> 
> ┌─────────────────────────────┐
> │ Docker Engine               │
> │ ┌─────────────────────────┐ │
> │ │ Plugin System           │ │
> │ │ - Volume Driver         │ │
> │ │ - Network Driver        │ │
> │ │ - Authorization         │ │
> │ │ - Logging Driver        │ │
> │ └─────────────────────────┘ │
> └─────────────────────────────┘
> 
> 특징:
> ✅ 기능 확장 (런타임 레벨)
> ✅ CLI/API에서 투명하게 동작
> ✅ Docker Desktop + Docker Engine 모두
> ✅ UI 없음 (백엔드만)
> ✅ 시스템 레벨 통합
> 
> 사용 사례:
> - NFS 볼륨 드라이버
> - Calico 네트워크
> - LDAP 인증
> - Splunk 로깅
> - 커스텀 스토리지
> ```
>
> **비교표:**
> ```
> ┌──────────────┬──────────────┬──────────────┐
> │ 기준          │ Extension    │ Plugin       │
> ├──────────────┼──────────────┼──────────────┤
> │ UI           │ ✅ React     │ ❌ 없음       │
> ├──────────────┼──────────────┼──────────────┤
> │ 플랫폼         │ Desktop 전용  │ Engine 전용   │
> ├──────────────┼──────────────┼──────────────┤
> │ 설치          │ Marketplace  │ CLI/API      │
> ├──────────────┼──────────────┼──────────────┤
> │ 목적          │ 개발자 경험     │ 런타임 기능     │
> ├──────────────┼──────────────┼──────────────┤
> │ 사용자         │ 개발자        │ DevOps/인프라  │
> └──────────────┴──────────────┴──────────────┘
> ```
>
> **선택 기준:**
> ```
> Extension 개발:
> - 개발자 도구 (모니터링, 분석)
> - UI가 필요함
> - Docker Desktop 사용자 대상
> - 팀 내부 도구
> - 빠른 프로토타이핑
> 
> Plugin 개발:
> - 인프라 통합 (스토리지, 네트워크)
> - UI 불필요
> - Docker Engine 사용자 대상
> - 프로덕션 환경
> - 시스템 레벨 요구사항
> ```
>
> **실제 예:**
> ```
> Extension: Snyk (취약점 스캔 UI)
> - Docker Desktop에서 이미지 스캔
> - 결과를 UI에 표시
> - 개발자가 직접 확인
> 
> Plugin: Portworx (스토리지)
> - Docker Engine 레벨 통합
> - 볼륨 드라이버로 동작
> - UI 없이 투명하게 동작
> - docker volume create --driver=portworx
> ```
>
> **함께 사용:**
> ```
> Extension + Plugin 조합 가능:
> 
> 1. Plugin으로 기능 구현
>    (예: 커스텀 볼륨 드라이버)
> 
> 2. Extension으로 관리 UI 제공
>    (예: 볼륨 사용량 시각화)
> 
> docker volume create --driver=myplugin vol1
>   ↓
> Extension UI에서 볼륨 상태 모니터링
> ```
>
> **2) Extension Backend 보안:**
>
> **위험:**
> ```yaml
> # docker-compose.yaml
> services:
>   backend:
>     image: ${DESKTOP_PLUGIN_IMAGE}
>     volumes:
>       - /var/run/docker.sock:/var/run/docker.sock  # ⚠️ 위험!
> 
> 보안 문제:
> ❌ 컨테이너 탈출 가능
> ❌ 호스트 전체 제어 가능
> ❌ 다른 컨테이너 조작 가능
> ❌ 이미지 조작 가능
> ❌ 민감한 데이터 접근
> ```
>
> **공격 시나리오:**
> ```python
> # Extension Backend가 악의적이라면:
> 
> import docker
> client = docker.from_env()
> 
> # 1. 호스트 파일시스템 접근
> container = client.containers.run(
>     "alpine",
>     command="cat /host/etc/shadow",
>     volumes={'/': {'bind': '/host', 'mode': 'ro'}},
>     privileged=True
> )
> 
> # 2. 다른 컨테이너 조작
> for c in client.containers.list():
>     c.stop()  # 모든 컨테이너 정지!
> 
> # 3. 이미지 변조
> client.images.pull("malicious/backdoor")
> ```
>
> **완화 방법:**
>
> **1. Read-Only Socket:**
> ```yaml
> # Docker Desktop Extension은 자동으로 제한됨
> # 사용자가 승인한 작업만 가능
> 
> services:
>   backend:
>     # Docker Desktop이 자동으로 제한된 socket 제공
>     # 모든 작업이 사용자 권한으로 실행
> ```
>
> **2. Extension SDK 사용 (권장):**
> ```typescript
> // Frontend에서 Docker API 호출
> const ddClient = createDockerDesktopClient();
> 
> // 사용자가 명시적으로 승인한 작업만
> const containers = await ddClient.docker.listContainers();
> 
> // Backend에서 직접 Socket 접근 대신
> // Frontend → Extension SDK → Docker
> ```
>
> **3. 최소 권한 원칙:**
> ```typescript
> // Backend에서 필요한 작업만 노출
> app.get('/api/stats', async (req, res) => {
>   // ✅ 읽기 전용 작업만
>   const stats = await getContainerStats();
>   res.json(stats);
> });
> 
> // ❌ 위험한 작업 노출 금지
> app.post('/api/exec', async (req, res) => {
>   // 임의 명령 실행 - 매우 위험!
> });
> ```
>
> **4. 입력 검증:**
> ```typescript
> app.get('/api/container/:id', async (req, res) => {
>   const { id } = req.params;
>   
>   // ✅ ID 검증
>   if (!/^[a-f0-9]{12,64}$/.test(id)) {
>     return res.status(400).json({ error: 'Invalid ID' });
>   }
>   
>   // ✅ 존재 여부 확인
>   const container = await docker.getContainer(id);
>   const info = await container.inspect();
>   
>   // ✅ 민감 정보 필터링
>   const safe = {
>     id: info.Id,
>     name: info.Name,
>     state: info.State.Status
>     // Env, Binds 등 민감 정보 제외
>   };
>   
>   res.json(safe);
> });
> ```
>
> **5. 코드 서명 및 검증:**
> ```bash
> # Extension 배포 시 서명
> docker extension sign myext:v1.0
> 
> # Docker Desktop이 서명 검증
> # 변조된 Extension은 설치 거부
> ```
>
> **6. 샌드박싱:**
> ```yaml
> # Backend를 격리된 환경에서 실행
> services:
>   backend:
>     security_opt:
>       - no-new-privileges:true
>     cap_drop:
>       - ALL
>     cap_add:
>       - NET_BIND_SERVICE  # 필요한 것만
>     read_only: true
>     tmpfs:
>       - /tmp
> ```
>
> **Best Practices:**
> ```
> ✅ Frontend에서 Extension SDK 사용 (Backend 최소화)
> ✅ Backend는 계산/집계만
> ✅ 모든 입력 검증
> ✅ 최소 권한 요청
> ✅ 민감 정보 필터링
> ✅ 감사 로그
> ✅ 정기 보안 스캔
> ```
>
> **3) 멀티 플랫폼 Extension:**
>
> **플랫폼별 차이:**
> ```
> Windows:
> - Docker Desktop for Windows
> - WSL2 백엔드
> - Windows Container (선택)
> 
> macOS:
> - Docker Desktop for Mac
> - VM 기반 (Apple Silicon / Intel)
> - Unix Socket
> 
> Linux:
> - Docker Desktop for Linux
> - Native Docker Engine
> - Unix Socket
> ```
>
> **고려사항:**
>
> **1. Docker Socket 경로:**
> ```typescript
> // ❌ 하드코딩
> const socket = '/var/run/docker.sock';
> 
> // ✅ 환경에 따라 자동 감지
> import Docker from 'dockerode';
> const docker = new Docker(); // 자동으로 올바른 경로 사용
> 
> // 또는 환경 변수
> const socket = process.env.DOCKER_HOST || '/var/run/docker.sock';
> ```
>
> **2. 파일 경로:**
> ```typescript
> // ❌ Unix 스타일 하드코딩
> const path = '/home/user/data';
> 
> // ✅ 플랫폼 독립적
> import path from 'path';
> const dataPath = path.join(process.env.HOME, 'data');
> ```
>
> **3. 아키텍처:**
> ```dockerfile
> # Multi-platform 빌드
> FROM --platform=$BUILDPLATFORM node:18-alpine AS builder
> ARG TARGETPLATFORM
> ARG BUILDPLATFORM
> 
> # 플랫폼별 의존성
> RUN case "$TARGETPLATFORM" in \
>     "linux/amd64") apk add --no-cache some-amd64-package ;; \
>     "linux/arm64") apk add --no-cache some-arm64-package ;; \
>     esac
> ```
>
> **4. UI 컴포넌트:**
> ```typescript
> // ✅ 플랫폼 독립적 UI
> import { Box, Button } from '@mui/material';
> 
> // Material-UI는 모든 플랫폼에서 동일하게 동작
> <Button onClick={handleClick}>
>   Start Container
> </Button>
> ```
>
> **5. 테스트:**
> ```bash
> # 모든 플랫폼에서 빌드 테스트
> docker buildx build \
>   --platform linux/amd64,linux/arm64 \
>   -t myext:latest .
> 
> # 각 플랫폼에서 설치 테스트
> # - Windows 10/11
> # - macOS Intel
> # - macOS Apple Silicon
> # - Linux (Ubuntu, Fedora)
> ```
>
> **6. Extension Metadata:**
> ```json
> // metadata.json
> {
>   "icon": "icon.svg",
>   "ui": {
>     "dashboard-tab": {
>       "title": "My Extension",
>       "root": "/ui",
>       "src": "index.html"
>     }
>   },
>   "host": {
>     "binaries": [
>       {
>         "darwin": [
>           {
>             "path": "/darwin/amd64/mytool"
>           },
>           {
>             "path": "/darwin/arm64/mytool"
>           }
>         ],
>         "linux": [
>           {
>             "path": "/linux/amd64/mytool"
>           },
>           {
>             "path": "/linux/arm64/mytool"
>           }
>         ],
>         "windows": [
>           {
>             "path": "/windows/amd64/mytool.exe"
>           }
>         ]
>       }
>     ]
>   }
> }
> ```
>
> **7. 플랫폼별 기능 감지:**
> ```typescript
> const platform = process.platform; // 'win32', 'darwin', 'linux'
> 
> if (platform === 'darwin') {
>   // macOS 전용 기능
> } else if (platform === 'win32') {
>   // Windows 전용 기능
> } else {
>   // Linux 기능
> }
> ```
>
> **체크리스트:**
> ```
> ✅ Multi-platform Docker 빌드
> ✅ 경로를 하드코딩하지 않음
> ✅ Docker SDK 사용 (socket 자동 감지)
> ✅ 모든 플랫폼에서 테스트
> ✅ 플랫폼별 바이너리 포함 (필요 시)
> ✅ UI는 플랫폼 독립적
> ✅ 문서에 플랫폼 요구사항 명시
> ```

---

<div align="center">

**[⬅️ 이전: Custom Plugins](./07-Custom-Plugins.md)** | **[홈으로 🏠](../README.md)**

</div>
