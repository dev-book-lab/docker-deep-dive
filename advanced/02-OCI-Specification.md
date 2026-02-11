# 02. OCI Specification - OCI 표준 명세

## 🎯 이 챕터에서 배울 것

- **OCI Image Spec**: 이미지 매니페스트, 설정, 레이어 구조
- **OCI Runtime Spec**: 런타임 설정과 생명주기 상세
- **OCI Distribution Spec**: 레지스트리 API와 이미지 배포
- Image Spec **v1 vs v2** 차이와 **멀티 아키텍처** 이미지
- OCI 아티팩트를 직접 **분해하고 조립**하는 방법

## 📌 왜 중요한가?

**"OCI Spec을 이해하면 Docker 이미지가 '마법의 블랙박스'에서 '투명한 파일 묶음'이 됩니다."**

```
Docker 이미지의 실체:

당신이 보는 것:                    실제로 존재하는 것:
┌──────────────┐                 ┌──────────────────────────┐
│ nginx:latest │                 │ OCI Image                │
│              │                 │                          │
│  (블랙박스?)   │       →         │ ┌──────────────────────┐ │
│              │                 │ │ Index (선택)          │ │
└──────────────┘                 │ │ - amd64 → Manifest A │ │
                                 │ │ - arm64 → Manifest B │ │
                                 │ └──────────┬───────────┘ │
                                 │            │             │
                                 │ ┌──────────▼───────────┐ │
                                 │ │ Manifest             │ │
                                 │ │ - Config (1개)        │ │
                                 │ │ - Layers (N개)        │ │
                                 │ └──┬──────────┬────────┘ │
                                 │    │          │          │
                                 │ ┌──▼───┐  ┌──▼────────┐  │
                                 │ │Config│  │ Layer 1   │  │
                                 │ │.json │  │ Layer 2   │  │
                                 │ │      │  │ Layer 3   │  │
                                 │ └──────┘  └───────────┘  │
                                 └──────────────────────────┘

OCI 3대 표준:
┌───────────────────────────────────────────────────────────┐
│                                                           │
│  Image Spec          Runtime Spec        Distribution Spec│
│  ┌───────────┐       ┌───────────┐       ┌───────────┐    │
│  │ 이미지를    │       │ 컨테이너를   │       │ 이미지를    │    │
│  │ 어떻게      │       │ 어떻게     │       │ 어떻게      │    │
│  │ 패키징?     │       │ 실행?     │        │ 배포?      │    │
│  │           │       │           │       │           │    │
│  │ manifest  │       │config.json│       │ Registry  │    │
│  │ config    │  ───→ │ rootfs    │  ←─── │ API       │    │
│  │ layers    │       │ lifecycle │       │ Pull/Push │    │
│  └───────────┘       └───────────┘       └───────────┘    │
│                                                           │
│  "무엇을 담을까"    "어떻게 실행할까"    "어떻게 전달할까"           │
└───────────────────────────────────────────────────────────┘

왜 알아야 하는가:
❌ 모르면: 이미지 문제 발생 시 원인 파악 불가
❌ 모르면: 멀티 아키텍처 빌드 이해 불가
❌ 모르면: 레지스트리 문제 디버깅 불가

✅ 알면: 이미지 내부를 직접 분석/수정 가능
✅ 알면: 커스텀 OCI 아티팩트 생성 가능
✅ 알면: 레지스트리 API 직접 활용 가능
```

**실무 영향:**
- 이미지 크기 최적화 전략 수립
- 멀티 아키텍처 이미지 빌드 이해
- 레지스트리 마이그레이션/트러블슈팅
- Helm Charts, WASM 등 OCI 아티팩트 활용

---

## 🔬 Deep Dive

### 1. OCI Image Specification

#### 이미지 구성 요소

```
OCI Image 구조 (Content-Addressable Storage):

모든 것은 SHA256 해시로 참조됨:

Index (Fat Manifest) ─── 선택적, 멀티 아키텍처용
│
├── Manifest (amd64/linux)
│   ├── Config Blob (sha256:abc123...)
│   │   └── Image Configuration JSON
│   │       ├── architecture: "amd64"
│   │       ├── os: "linux"
│   │       ├── config (Env, Cmd, Entrypoint)
│   │       ├── rootfs.diff_ids (레이어 해시)
│   │       └── history (빌드 히스토리)
│   │
│   └── Layer Blobs
│       ├── sha256:layer1... (base OS)
│       ├── sha256:layer2... (apt install)
│       └── sha256:layer3... (app copy)
│
└── Manifest (arm64/linux)
    ├── Config Blob (sha256:def456...)
    └── Layer Blobs
        ├── sha256:layer4...
        ├── sha256:layer5...
        └── sha256:layer6...

Content-Addressable:
- 내용이 같으면 해시가 같음
- 해시가 같으면 같은 레이어 → 중복 저장 없음
- 레이어 공유 가능 (여러 이미지가 같은 base 사용)
```

#### Media Types

```
OCI Image Spec Media Types:

┌──────────────────────────────────────────┬─────────────────────┐
│ Media Type                               │ 용도                 │
├──────────────────────────────────────────┼─────────────────────┤
│ application/vnd.oci.image.index.v1+json  │ Image Index         │
│                                          │ (멀티 아키텍처 목록)    │
├──────────────────────────────────────────┼─────────────────────┤
│ application/vnd.oci.image.manifest.v1    │ Image Manifest      │
│ +json                                    │ (단일 이미지 설명)      │
├──────────────────────────────────────────┼─────────────────────┤
│ application/vnd.oci.image.config.v1+json │ Image Configuration │
│                                          │ (실행 설정)           │
├──────────────────────────────────────────┼─────────────────────┤
│ application/vnd.oci.image.layer.v1.tar   │ Layer (비압축)        │
│ +gzip                                    │ Layer (gzip 압축)    │
│ +zstd                                    │ Layer (zstd 압축)    │
└──────────────────────────────────────────┴─────────────────────┘

Docker 호환 Media Types:
┌──────────────────────────────────────────┬─────────────────────┐
│ Docker Media Type                        │ OCI 대응             │
├──────────────────────────────────────────┼─────────────────────┤
│ application/vnd.docker.distribution.     │ OCI Image Index     │
│ manifest.list.v2+json                    │                     │
├──────────────────────────────────────────┼─────────────────────┤
│ application/vnd.docker.distribution.     │ OCI Image Manifest  │
│ manifest.v2+json                         │                     │
├──────────────────────────────────────────┼─────────────────────┤
│ application/vnd.docker.container.image.  │ OCI Image Config    │
│ v1+json                                  │                     │
└──────────────────────────────────────────┴─────────────────────┘
```

---

### 2. Manifest 상세

#### Image Manifest 구조

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "size": 7023
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4",
      "size": 32654
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:9f13e0ac480c5a4b8ab45e6e9c8b3e7e0f7a8c0d3e5b2a1f4c6d8e0a2b4c6d8",
      "size": 16724
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2",
      "size": 73109
    }
  ],
  "annotations": {
    "org.opencontainers.image.created": "2024-01-15T10:00:00Z",
    "org.opencontainers.image.authors": "example@example.com"
  }
}
```

```
Manifest가 연결하는 것:

Manifest
├── config (1개)
│   └── sha256:e3b0c4... → Image Config JSON
│       (아키텍처, OS, Env, Cmd, 레이어 순서)
│
└── layers (N개, 순서 중요!)
    ├── [0] sha256:a3ed95... → Base OS 레이어 (alpine:3.18)
    ├── [1] sha256:9f13e0... → apt install 레이어
    └── [2] sha256:1b2c3d... → COPY app 레이어

    레이어 적용 순서:
    [0] ─────────── (base)
    [1] ──── (중간)
    [2] ─ (최상위, 가장 마지막에 적용)
```

#### Image Index (Fat Manifest)

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:aaa111...",
      "size": 7143,
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:bbb222...",
      "size": 7682,
      "platform": {
        "architecture": "arm64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:ccc333...",
      "size": 7025,
      "platform": {
        "architecture": "amd64",
        "os": "windows",
        "os.version": "10.0.17763.1234"
      }
    }
  ]
}
```

```
Image Index (멀티 아키텍처) 동작:

docker pull nginx:latest
        │
        ▼
┌── Image Index ──────────────────────┐
│                                     │
│  내 플랫폼: linux/amd64               │
│                                     │
│  ┌─ amd64/linux → Manifest A ──┐    │ ← 이것을 선택!
│  │                             │    │
│  ├─ arm64/linux → Manifest B   │    │
│  │                             │    │
│  └─ amd64/windows → Manifest C │    │
│                                     │
└─────────────────────────────────────┘
        │
        ▼
  Manifest A의 레이어들 다운로드

같은 태그(nginx:latest)로:
- x86 서버 → amd64 레이어 다운로드
- Apple M1  → arm64 레이어 다운로드
- Windows   → windows 레이어 다운로드
```

---

### 3. Image Configuration 상세

```json
{
  "architecture": "amd64",
  "os": "linux",
  "config": {
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "NGINX_VERSION=1.25.3"
    ],
    "Cmd": ["nginx", "-g", "daemon off;"],
    "ExposedPorts": {
      "80/tcp": {}
    },
    "WorkingDir": "/",
    "Labels": {
      "maintainer": "NGINX Docker Maintainers"
    },
    "StopSignal": "SIGQUIT"
  },
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:layer1_uncompressed_hash...",
      "sha256:layer2_uncompressed_hash...",
      "sha256:layer3_uncompressed_hash..."
    ]
  },
  "history": [
    {
      "created": "2024-01-10T00:00:00Z",
      "created_by": "/bin/sh -c #(nop) ADD file:... in / ",
      "comment": "base layer"
    },
    {
      "created": "2024-01-10T00:01:00Z",
      "created_by": "/bin/sh -c apt-get update && apt-get install -y nginx",
      "empty_layer": false
    },
    {
      "created": "2024-01-10T00:02:00Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"nginx\" \"-g\" \"daemon off;\"]",
      "empty_layer": true
    }
  ]
}
```

```
Config → Dockerfile 역매핑:

Image Config                     Dockerfile
─────────────────────────────────────────────────────
config.Env                   ←   ENV
config.Cmd                   ←   CMD
config.Entrypoint            ←   ENTRYPOINT
config.ExposedPorts          ←   EXPOSE
config.WorkingDir            ←   WORKDIR
config.Labels                ←   LABEL
config.User                  ←   USER
config.Volumes               ←   VOLUME
config.StopSignal            ←   STOPSIGNAL

rootfs.diff_ids              ←   각 레이어 (비압축 해시)
history[].created_by         ←   각 Dockerfile 명령어

empty_layer 주의:
- CMD, ENV, LABEL 등 메타데이터 변경 → 레이어 생성 안 함
- RUN, COPY, ADD → 실제 레이어 생성
- history 항목 수 ≥ 레이어 수 (empty_layer 때문)
```

---

### 4. OCI Distribution Specification

```
레지스트리 API 흐름:

docker pull nginx:latest 의 내부 동작:

Client                              Registry
  │                                    │
  │  1. GET /v2/                       │
  │───────────────────────────────────→│  인증 확인
  │  ← 200 OK                          │
  │                                    │
  │  2. GET /v2/nginx/manifests/latest │
  │───────────────────────────────────→│  매니페스트 요청
  │  ← Index JSON (멀티 아키텍처)         │
  │                                    │
  │  3. GET /v2/nginx/manifests/sha256:│
  │     aaa111...                      │  플랫폼별 매니페스트
  │───────────────────────────────────→│
  │  ← Manifest JSON                   │
  │                                    │
  │  4. GET /v2/nginx/blobs/sha256:... │
  │───────────────────────────────────→│  Config 다운로드
  │  ← Config JSON                     │
  │                                    │
  │  5. HEAD /v2/nginx/blobs/sha256:...│
  │───────────────────────────────────→│  레이어 존재 확인
  │  ← 200 (있음) / 404 (없음)           │
  │                                    │
  │  6. GET /v2/nginx/blobs/sha256:... │
  │───────────────────────────────────→│  레이어 다운로드
  │  ← Layer tar+gzip                  │  (없는 것만)
  │                                    │

docker push 의 내부 동작:

Client                              Registry
  │                                    │
  │  1. POST /v2/myapp/blobs/uploads/  │
  │───────────────────────────────────→│  업로드 세션 시작
  │  ← 202 + Location header           │
  │                                    │
  │  2. PATCH /v2/myapp/blobs/uploads/ │
  │     {uuid}                         │  레이어 업로드
  │───────────────────────────────────→│  (청크 또는 전체)
  │  ← 202 Accepted                    │
  │                                    │
  │  3. PUT /v2/myapp/blobs/uploads/   │
  │     {uuid}?digest=sha256:...       │  업로드 완료
  │───────────────────────────────────→│
  │  ← 201 Created                     │
  │                                    │
  │  4. PUT /v2/myapp/manifests/v1.0   │
  │───────────────────────────────────→│  매니페스트 등록
  │  ← 201 Created                     │
  │                                    │

주요 API 엔드포인트:
┌────────────────────────────────────────┬────────────┐
│ Endpoint                               │ 용도        │
├────────────────────────────────────────┼────────────┤
│ GET /v2/                               │ API 버전    │
│ GET /v2/_catalog                       │ 저장소 목록   │
│ GET /v2/{name}/tags/list               │ 태그 목록    │
│ GET /v2/{name}/manifests/{ref}         │ 매니페스트    │
│ GET /v2/{name}/blobs/{digest}          │ Blob 조회   │
│ POST /v2/{name}/blobs/uploads/         │ 업로드 시작   │
│ DELETE /v2/{name}/manifests/{ref}      │ 이미지 삭제   │
└────────────────────────────────────────┴────────────┘
```

---

## 🔧 실습 1: 이미지 매니페스트 분석

### Step 1: Docker CLI로 매니페스트 확인

```bash
# 이미지 매니페스트 확인 (Docker Hub)
docker manifest inspect nginx:latest
# {
#   "schemaVersion": 2,
#   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
#   "manifests": [
#     { "platform": {"architecture": "amd64", "os": "linux"}, ... },
#     { "platform": {"architecture": "arm64", "os": "linux"}, ... },
#     ...
#   ]
# }

# 특정 플랫폼 매니페스트
docker manifest inspect --verbose nginx:latest | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for m in data:
    p = m.get('Descriptor', {}).get('platform', m.get('Platform', {}))
    print(f\"{p.get('architecture','?')}/{p.get('os','?')}: {m.get('Descriptor',{}).get('digest','')[:30]}...\")
"
# amd64/linux: sha256:abc123...
# arm/linux: sha256:def456...
# arm64/linux: sha256:ghi789...
# ...

# 현재 플랫폼의 이미지 상세
docker inspect nginx:latest --format '{{.RepoDigests}}'
# [nginx@sha256:...]
```

### Step 2: 레지스트리 API 직접 호출

```bash
# Docker Hub 인증 토큰 획득
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:library/alpine:pull" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# 매니페스트 조회 (Index)
curl -s -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.oci.image.index.v1+json,application/vnd.docker.distribution.manifest.list.v2+json" \
  "https://registry-1.docker.io/v2/library/alpine/manifests/3.18" \
  | python3 -m json.tool | head -30

# 특정 플랫폼 매니페스트 digest 추출
AMD64_DIGEST=$(curl -s -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.docker.distribution.manifest.list.v2+json" \
  "https://registry-1.docker.io/v2/library/alpine/manifests/3.18" \
  | python3 -c "
import sys, json
data = json.load(sys.stdin)
for m in data.get('manifests', []):
    if m.get('platform', {}).get('architecture') == 'amd64':
        print(m['digest'])
        break
")

echo "amd64 manifest digest: $AMD64_DIGEST"

# 해당 매니페스트 조회
curl -s -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  "https://registry-1.docker.io/v2/library/alpine/manifests/${AMD64_DIGEST}" \
  | python3 -m json.tool

# 출력 예시:
# {
#   "schemaVersion": 2,
#   "config": {
#     "digest": "sha256:config_hash...",
#     "size": 1471
#   },
#   "layers": [
#     {
#       "digest": "sha256:layer_hash...",
#       "size": 3401613
#     }
#   ]
# }
```

---

## 🔧 실습 2: 이미지 레이어 분해

### Step 1: 이미지를 OCI 형식으로 저장

```bash
# 이미지 pull
docker pull alpine:3.18

# OCI 형식으로 저장
mkdir -p /tmp/oci-analysis
docker save alpine:3.18 -o /tmp/oci-analysis/alpine.tar

# tar 파일 분석
cd /tmp/oci-analysis
mkdir alpine-extracted
tar xf alpine.tar -C alpine-extracted

# 구조 확인
find alpine-extracted -type f | sort
# alpine-extracted/blobs/sha256/...  (레이어, config)
# alpine-extracted/index.json
# alpine-extracted/manifest.json
# alpine-extracted/oci-layout
# (또는 Docker 형식에 따라 다를 수 있음)

# manifest.json 확인
cat alpine-extracted/manifest.json | python3 -m json.tool
```

### Step 2: 각 구성 요소 분석

```bash
cd /tmp/oci-analysis/alpine-extracted

# Docker save 형식 (Docker manifest)
cat manifest.json | python3 -m json.tool
# [
#   {
#     "Config": "sha256:xxxxx.json",
#     "RepoTags": ["alpine:3.18"],
#     "Layers": ["sha256:yyyyy/layer.tar"]
#   }
# ]

# Config JSON 분석
CONFIG_FILE=$(cat manifest.json | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['Config'])")
echo "=== Image Config ==="
cat "$CONFIG_FILE" | python3 -m json.tool | head -40

# 아키텍처, OS 확인
cat "$CONFIG_FILE" | python3 -c "
import sys, json
c = json.load(sys.stdin)
print(f\"Architecture: {c['architecture']}\")
print(f\"OS: {c['os']}\")
print(f\"Cmd: {c.get('config', {}).get('Cmd', 'N/A')}\")
print(f\"Env: {c.get('config', {}).get('Env', [])}\")
print(f\"Layers: {len(c['rootfs']['diff_ids'])}\")
"

# 레이어 분석
LAYER_DIR=$(cat manifest.json | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['Layers'][0])")
echo "=== Layer Contents (top 20) ==="
tar tf "$LAYER_DIR" | head -20
# bin/
# bin/arch
# bin/ash
# bin/base64
# ...

# 레이어 크기
echo "Layer size: $(du -h "$LAYER_DIR" | cut -f1)"

# 정리
cd /tmp && rm -rf /tmp/oci-analysis
```

---

## 🔧 실습 3: 레지스트리 API로 이미지 조작

### Step 1: 로컬 레지스트리 설정

```bash
# 로컬 레지스트리 시작
docker run -d -p 5000:5000 --name local-registry registry:2

# 테스트 이미지 푸시
docker pull alpine:3.18
docker tag alpine:3.18 localhost:5000/my-alpine:v1
docker push localhost:5000/my-alpine:v1

# 레지스트리 API 확인
curl -s http://localhost:5000/v2/
# {}

# 카탈로그 (저장소 목록)
curl -s http://localhost:5000/v2/_catalog
# {"repositories":["my-alpine"]}

# 태그 목록
curl -s http://localhost:5000/v2/my-alpine/tags/list
# {"name":"my-alpine","tags":["v1"]}
```

### Step 2: 매니페스트 조회 및 분석

```bash
# 매니페스트 조회
curl -s -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://localhost:5000/v2/my-alpine/manifests/v1 | python3 -m json.tool

# 매니페스트에서 Config digest 추출
CONFIG_DIGEST=$(curl -s \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://localhost:5000/v2/my-alpine/manifests/v1 \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['config']['digest'])")

echo "Config digest: $CONFIG_DIGEST"

# Config blob 다운로드
curl -s http://localhost:5000/v2/my-alpine/blobs/${CONFIG_DIGEST} | python3 -m json.tool
# {
#   "architecture": "amd64",
#   "config": { "Cmd": ["/bin/sh"], ... },
#   "rootfs": { "diff_ids": [...] },
#   ...
# }

# 레이어 digest 추출
LAYER_DIGEST=$(curl -s \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
  http://localhost:5000/v2/my-alpine/manifests/v1 \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['layers'][0]['digest'])")

echo "Layer digest: $LAYER_DIGEST"

# 레이어 blob 다운로드 (tar+gzip)
curl -s http://localhost:5000/v2/my-alpine/blobs/${LAYER_DIGEST} \
  | tar tz | head -10
# bin/
# bin/arch
# bin/ash
# ...
```

### Step 3: 태그 관리

```bash
# 같은 이미지에 여러 태그
docker tag alpine:3.18 localhost:5000/my-alpine:latest
docker tag alpine:3.18 localhost:5000/my-alpine:stable
docker push localhost:5000/my-alpine:latest
docker push localhost:5000/my-alpine:stable

# 태그 목록 확인
curl -s http://localhost:5000/v2/my-alpine/tags/list
# {"name":"my-alpine","tags":["latest","stable","v1"]}

# 모든 태그가 같은 매니페스트를 가리킴 확인
for TAG in v1 latest stable; do
  DIGEST=$(curl -s -I \
    -H "Accept: application/vnd.docker.distribution.manifest.v2+json" \
    http://localhost:5000/v2/my-alpine/manifests/${TAG} \
    | grep Docker-Content-Digest | tr -d '\r')
  echo "${TAG}: ${DIGEST}"
done
# v1: Docker-Content-Digest: sha256:xxx
# latest: Docker-Content-Digest: sha256:xxx
# stable: Docker-Content-Digest: sha256:xxx
# ✅ 모두 같은 digest → 같은 이미지!

# 정리
docker rm -f local-registry
```

---

## 🔧 실습 4: 멀티 아키텍처 이미지 분석

### Step 1: 멀티 아키텍처 이미지 구조 확인

```bash
# nginx의 멀티 아키텍처 지원 확인
docker manifest inspect nginx:alpine 2>/dev/null | python3 -c "
import sys, json
data = json.load(sys.stdin)
if 'manifests' in data:
    print('=== Multi-Architecture Image ===')
    for m in data['manifests']:
        p = m.get('platform', {})
        arch = p.get('architecture', '?')
        os = p.get('os', '?')
        variant = p.get('variant', '')
        size = m.get('size', 0)
        digest = m.get('digest', '')[:20]
        print(f'  {os}/{arch}{\"v\"+variant if variant else \"\"}: {digest}... ({size} bytes)')
else:
    print('Single architecture image')
"

# 출력 예시:
# === Multi-Architecture Image ===
#   linux/amd64: sha256:a3ed95caeb02f... (1234 bytes)
#   linux/arm/v6: sha256:9f13e0ac480c5... (1234 bytes)
#   linux/arm/v7: sha256:1b2c3d4e5f6a7... (1234 bytes)
#   linux/arm64: sha256:4a5b6c7d8e9f0... (1234 bytes)
#   linux/386: sha256:2c3d4e5f6a7b8... (1234 bytes)
#   linux/ppc64le: sha256:5d6e7f8a9b0c1... (1234 bytes)
#   linux/s390x: sha256:8a9b0c1d2e3f4... (1234 bytes)
```

### Step 2: buildx로 멀티 아키텍처 이미지 빌드

```bash
# buildx 빌더 생성
docker buildx create --name multiarch --driver docker-container --use
docker buildx inspect --bootstrap

# 간단한 Dockerfile
mkdir -p /tmp/multiarch-test && cd /tmp/multiarch-test

cat > Dockerfile << 'EOF'
FROM alpine:3.18
RUN echo "Built for $(uname -m)" > /arch.txt
CMD ["cat", "/arch.txt"]
EOF

# 멀티 아키텍처 빌드 (로컬 확인만)
docker buildx build --platform linux/amd64,linux/arm64 -t multiarch-demo . --load 2>/dev/null || \
  echo "(--load는 단일 플랫폼만 지원, --push 사용 권장)"

# 단일 플랫폼으로 확인
docker buildx build --platform linux/amd64 -t multiarch-demo:amd64 --load .
docker run --rm multiarch-demo:amd64
# Built for x86_64

# 빌드 컨텍스트에서 플랫폼 정보 확인
cat > Dockerfile << 'EOF'
FROM --platform=$BUILDPLATFORM alpine:3.18 AS builder
ARG TARGETPLATFORM
ARG TARGETARCH
ARG TARGETOS
RUN echo "Building on $BUILDPLATFORM for $TARGETPLATFORM" && \
    echo "Target: ${TARGETOS}/${TARGETARCH}" > /info.txt

FROM alpine:3.18
COPY --from=builder /info.txt /info.txt
CMD ["cat", "/info.txt"]
EOF

docker buildx build --platform linux/amd64 -t multiarch-demo:info --load .
docker run --rm multiarch-demo:info
# Target: linux/amd64

# 정리
cd /tmp && rm -rf /tmp/multiarch-test
docker buildx rm multiarch
```

---

## 🔧 실습 5: OCI 아티팩트 이해

### Step 1: skopeo로 이미지 조작

```bash
# skopeo 설치
sudo apt-get update && sudo apt-get install -y skopeo

# Docker Hub에서 OCI 형식으로 이미지 복사
mkdir -p /tmp/oci-artifact
skopeo copy docker://alpine:3.18 oci:/tmp/oci-artifact/alpine:3.18

# OCI 레이아웃 구조 확인
find /tmp/oci-artifact/alpine -type f | sort
# /tmp/oci-artifact/alpine/blobs/sha256/abc123...
# /tmp/oci-artifact/alpine/blobs/sha256/def456...
# /tmp/oci-artifact/alpine/blobs/sha256/ghi789...
# /tmp/oci-artifact/alpine/index.json
# /tmp/oci-artifact/alpine/oci-layout

# oci-layout 확인
cat /tmp/oci-artifact/alpine/oci-layout
# {"imageLayoutVersion": "1.0.0"}

# index.json 확인
cat /tmp/oci-artifact/alpine/index.json | python3 -m json.tool
# {
#   "schemaVersion": 2,
#   "manifests": [
#     {
#       "mediaType": "application/vnd.oci.image.manifest.v1+json",
#       "digest": "sha256:...",
#       "size": ...,
#       "annotations": {
#         "org.opencontainers.image.ref.name": "3.18"
#       }
#     }
#   ]
# }

# Manifest blob 확인
MANIFEST_DIGEST=$(cat /tmp/oci-artifact/alpine/index.json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['manifests'][0]['digest'].split(':')[1])")

cat /tmp/oci-artifact/alpine/blobs/sha256/${MANIFEST_DIGEST} | python3 -m json.tool
```

### Step 2: 레지스트리 간 이미지 복사

```bash
# 로컬 레지스트리 시작
docker run -d -p 5000:5000 --name registry registry:2

# Docker Hub → 로컬 레지스트리 (Docker 없이!)
skopeo copy \
  docker://alpine:3.18 \
  docker://localhost:5000/alpine:3.18 \
  --dest-tls-verify=false

# 확인
curl -s http://localhost:5000/v2/_catalog
# {"repositories":["alpine"]}

# 이미지 정보 조회 (pull 없이)
skopeo inspect docker://localhost:5000/alpine:3.18 --tls-verify=false
# {
#   "Name": "localhost:5000/alpine",
#   "Tag": "3.18",
#   "Digest": "sha256:...",
#   "Architecture": "amd64",
#   ...
# }

# 이미지 삭제
skopeo delete docker://localhost:5000/alpine:3.18 --tls-verify=false

# 정리
docker rm -f registry
rm -rf /tmp/oci-artifact
```

---

## 🚫 안티패턴

### 1. 매니페스트 Media Type 무시

```bash
# ❌ Accept 헤더 없이 매니페스트 요청
curl http://registry/v2/myapp/manifests/latest
# Docker Schema v1 반환 (레거시, 비효율적)

# ✅ OCI 또는 Docker v2 Media Type 명시
curl -H "Accept: application/vnd.oci.image.manifest.v1+json" \
  http://registry/v2/myapp/manifests/latest
# 올바른 v2 매니페스트 반환
```

### 2. 단일 아키텍처만 빌드

```bash
# ❌ amd64만 빌드
docker build -t myapp .
# ARM 서버나 Apple Silicon에서 실행 불가

# ✅ 멀티 아키텍처 빌드
docker buildx build --platform linux/amd64,linux/arm64 -t myapp .
# 어디서든 실행 가능
```

### 3. Digest 대신 태그만 사용

```bash
# ❌ 가변적인 태그로 배포
docker pull myapp:latest
# latest가 가리키는 이미지가 변경될 수 있음!

# ✅ 불변 Digest로 배포 (프로덕션)
docker pull myapp@sha256:e3b0c44298fc1c149afbf4c8996fb924...
# 항상 동일한 이미지 보장
```

---

## 🎓 연습 문제

### 문제 1: Image Index와 Image Manifest의 차이는?

<details>
<summary>정답 보기</summary>

| 구분 | Image Index | Image Manifest |
|------|-------------|----------------|
| 별칭 | Fat Manifest, Manifest List | Manifest |
| 역할 | 플랫폼별 Manifest를 묶는 상위 목록 | 단일 플랫폼의 이미지 설명 |
| 포함 | Manifest 목록 + platform 정보 | Config + Layers 목록 |
| 필수 | 선택 (멀티 아키텍처 시 필요) | 필수 |
| 사용 | `docker pull` 시 플랫폼 선택 | 레이어 다운로드에 사용 |

```
Tag: nginx:latest
      ↓
Image Index (optional)
├── linux/amd64 → Manifest A → Config A + Layers A
├── linux/arm64 → Manifest B → Config B + Layers B
└── windows     → Manifest C → Config C + Layers C
```

단일 아키텍처 이미지는 Index 없이 Manifest만 존재할 수 있습니다.

</details>

### 문제 2: diff_id와 layer digest가 다른 이유는?

<details>
<summary>정답 보기</summary>

**diff_id**: 비압축 레이어의 SHA256 해시 (Image Config에 기록)
**digest**: 압축된 레이어의 SHA256 해시 (Manifest에 기록)

```
Layer (비압축 tar)
│ SHA256 → diff_id: sha256:aaa...
│
└── gzip 압축
    │
    Layer.tar.gz (압축 tar)
    │ SHA256 → digest: sha256:bbb...
```

- 같은 내용이라도 diff_id ≠ digest (압축 전후 해시가 다름)
- Manifest는 **네트워크 전송용 (압축)** digest 사용
- Config의 rootfs는 **로컬 식별용 (비압축)** diff_id 사용
- 이 분리 덕분에 압축 알고리즘(gzip/zstd)을 바꿔도 diff_id로 레이어 동일성 확인 가능

</details>

### 문제 3: `docker pull`이 내부적으로 레지스트리에 보내는 API 요청 순서는?

<details>
<summary>정답 보기</summary>

1. **`GET /v2/`** — API 버전 확인 및 인증
2. **`GET /v2/{name}/manifests/{tag}`** — 매니페스트 요청 (Index 또는 Manifest)
3. (Index인 경우) **`GET /v2/{name}/manifests/{digest}`** — 플랫폼별 Manifest 요청
4. **`GET /v2/{name}/blobs/{config_digest}`** — Image Config 다운로드
5. (각 레이어별) **`HEAD /v2/{name}/blobs/{layer_digest}`** — 로컬에 이미 있는지 확인
6. (없는 레이어만) **`GET /v2/{name}/blobs/{layer_digest}`** — 레이어 다운로드

최적화 포인트:
- HEAD 요청으로 이미 있는 레이어는 스킵 → 재다운로드 방지
- 레이어 공유: 여러 이미지가 같은 base 레이어를 쓰면 한 번만 다운로드
- 병렬 다운로드: 여러 레이어를 동시에 다운로드

</details>

---

## 📌 핵심 요약

```
┌─────────────────────┬──────────────────────────────────────┐
│ 개념                 │ 설명                                  │
├─────────────────────┼──────────────────────────────────────┤
│ OCI Image Spec      │ 이미지 패키징 표준                       │
├─────────────────────┼──────────────────────────────────────┤
│ Image Index         │ 멀티 아키텍처 Manifest 목록              │
├─────────────────────┼──────────────────────────────────────┤
│ Image Manifest      │ Config + Layers 참조                  │
├─────────────────────┼──────────────────────────────────────┤
│ Image Config        │ 아키텍처, Cmd, Env, 레이어 diff_ids      │
├─────────────────────┼──────────────────────────────────────┤
│ Layer Blob          │ tar+gzip/zstd 압축 파일시스템 변경분       │
├─────────────────────┼──────────────────────────────────────┤
│ Content-Addressable │ SHA256 해시로 모든 것을 참조              │
├─────────────────────┼──────────────────────────────────────┤
│ Distribution Spec   │ 레지스트리 Pull/Push API 표준            │
├─────────────────────┼──────────────────────────────────────┤
│ Media Type          │ 각 구성 요소의 MIME 타입                 │
├─────────────────────┼──────────────────────────────────────┤
│ Digest vs Tag       │ 불변(digest) vs 가변(tag) 참조          │
├─────────────────────┼──────────────────────────────────────┤
│ skopeo              │ 이미지 조회/복사/삭제 (Docker 없이)        │
└─────────────────────┴──────────────────────────────────────┘
```

---

## 💡 주요 명령어 정리

```bash
# 매니페스트 확인
docker manifest inspect <image>           # 매니페스트 조회
docker manifest inspect --verbose <image> # 상세 정보

# 이미지 분석
docker inspect <image>                    # 로컬 이미지 상세
docker save <image> -o image.tar          # 이미지 파일로 저장
docker image inspect <image> --format '{{.RootFS.Layers}}'  # 레이어 해시

# skopeo (Docker 없이)
skopeo inspect docker://<image>           # 이미지 정보 조회
skopeo copy docker://<src> docker://<dst> # 레지스트리 간 복사
skopeo copy docker://<img> oci:<dir>      # OCI 형식으로 저장
skopeo delete docker://<image>            # 이미지 삭제

# Registry API
curl /v2/_catalog                         # 저장소 목록
curl /v2/{name}/tags/list                 # 태그 목록
curl /v2/{name}/manifests/{ref}           # 매니페스트
curl /v2/{name}/blobs/{digest}            # Blob (Config/Layer)

# 멀티 아키텍처 빌드
docker buildx create --name builder --use # 빌더 생성
docker buildx build --platform linux/amd64,linux/arm64 -t <tag> .
```

---

## 📚 참고 자료

- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec)
- [Docker Registry HTTP API V2](https://docs.docker.com/registry/spec/api/)
- [skopeo](https://github.com/containers/skopeo)

---

## 🤔 생각해볼 문제

1. Content-Addressable Storage가 레지스트리 저장 효율에 어떤 영향을 주는가?
2. OCI 아티팩트로 이미지 외에 무엇을 저장할 수 있는가? (Helm, WASM 등)
3. Docker Schema v1에서 v2로 전환한 이유는 무엇인가?

> 💡 **답변**: 1) Content-Addressable = 내용이 같으면 해시가 같음 = 중복 저장 없음. 100개 이미지가 같은 alpine base를 쓰면 해당 레이어는 레지스트리에 1번만 저장. push 시에도 HEAD 요청으로 이미 존재하는 레이어는 업로드 스킵. 대규모 레지스트리에서 수십 TB 절약 가능. 2) OCI 아티팩트 확장: Helm Charts (helm push), WASM 모듈 (containerd-wasm), Singularity (과학 컴퓨팅), Cosign 서명 (이미지 서명/검증), SBOM (Software Bill of Materials), Terraform 모듈 등. OCI 레지스트리가 범용 아티팩트 저장소로 진화 중. ORAS (OCI Registry as Storage) 프로젝트가 이를 주도. 3) Schema v1 문제점: 내용 기반 해싱 불가 (무결성 검증 어려움), 멀티 아키텍처 미지원, 비효율적 메타데이터 구조, 서명 방식의 보안 취약점. v2에서 Content-Addressable, Fat Manifest (멀티 아키텍처), 효율적 레이어 공유, OCI 표준과 호환 가능해짐.

---

<div align="center">

**[⬅️ 이전: Container Runtime](./01-Container-Runtime.md)** | **[다음: containerd ➡️](./03-containerd.md)**

</div>
