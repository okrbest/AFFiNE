# AFFiNE Docker 이미지 빌드 가이드

## 개요

AFFiNE 웹 애플리케이션을 Docker 이미지로 빌드하는 방법을 설명합니다.

## 이미지 종류

| 이미지 | 용도 | Dockerfile |
|--------|------|------------|
| `affine` | 통합 이미지 (Backend + Frontend) | `.github/deployment/node/Dockerfile` |
| `affine-front` | 프론트엔드 전용 (Nginx) | `.github/deployment/front/Dockerfile` |

## 사전 요구사항

- Node.js 22.x
- Yarn 4.x (corepack)
- Rust 1.87.0 (서버 네이티브 모듈 빌드 시)
- Docker

## 통합 이미지 빌드 (권장)

Backend 서버와 Frontend 정적 파일이 함께 포함된 이미지입니다.

### 단계별 빌드

```bash
# 1. 의존성 설치
corepack enable && yarn install

# 2. 서버 네이티브 모듈 빌드 (Rust 필요)
yarn affine @affine/server-native build

# 3. 프론트엔드 빌드
yarn affine @affine/web build
yarn affine @affine/admin build
yarn affine @affine/mobile build

# 4. 서버 빌드
yarn workspace @affine/reader build
yarn workspace @affine/server build

# 5. Prisma 클라이언트 생성
yarn workspace @affine/server prisma generate

# 6. 프로덕션 의존성만 설치
yarn workspaces focus @affine/server --production

# 7. node_modules를 서버 디렉토리로 이동
mv ./node_modules ./packages/backend/server

# 8. Docker 이미지 빌드
docker build -f .github/deployment/node/Dockerfile -t affine:local .
```

### Dockerfile 구조

```dockerfile
# .github/deployment/node/Dockerfile
FROM node:22-bookworm-slim

COPY ./packages/backend/server /app
COPY ./packages/frontend/apps/web/dist /app/static
COPY ./packages/frontend/admin/dist /app/static/admin
COPY ./packages/frontend/apps/mobile/dist /app/static/mobile
WORKDIR /app

RUN apt-get update && \
  apt-get install -y --no-install-recommends openssl libjemalloc2 && \
  rm -rf /var/lib/apt/lists/*

# jemalloc으로 메모리 최적화
ENV LD_PRELOAD=libjemalloc.so.2

CMD ["node", "./dist/main.js"]
```

### 이미지 내부 구조

```
/app/
├── dist/                 # 서버 빌드 결과물
│   └── main.js          # 서버 엔트리포인트
├── static/              # 프론트엔드 정적 파일
│   ├── index.html       # 웹 앱
│   ├── admin/           # 관리자 페이지
│   └── mobile/          # 모바일 웹
├── node_modules/        # 프로덕션 의존성
├── prisma/              # Prisma 스키마
└── scripts/             # 배포 스크립트
```

## 프론트엔드 전용 이미지 빌드

Nginx(OpenResty)로 정적 파일만 서빙하는 이미지입니다.

### 빌드 단계

```bash
# 1. 의존성 설치
corepack enable && yarn install

# 2. 프론트엔드 빌드
yarn affine @affine/web build
yarn affine @affine/admin build
yarn affine @affine/mobile build

# 3. Docker 이미지 빌드
docker build -f .github/deployment/front/Dockerfile -t affine-front:local .
```

### Dockerfile 구조

```dockerfile
# .github/deployment/front/Dockerfile
FROM openresty/openresty:1.27.1.1-0-buster
WORKDIR /app

COPY ./packages/frontend/apps/web/dist ./dist
COPY ./packages/frontend/admin/dist ./admin
COPY ./packages/frontend/apps/mobile/dist ./mobile
COPY ./.github/deployment/front/nginx.conf /usr/local/openresty/nginx/conf/nginx.conf
COPY ./.github/deployment/front/affine.nginx.conf /etc/nginx/conf.d/affine.nginx.conf

RUN mkdir -p /var/log/nginx && \
  rm /etc/nginx/conf.d/default.conf

EXPOSE 8080
CMD ["/usr/local/openresty/bin/openresty", "-g", "daemon off;"]
```

### Nginx 설정 특징

- 포트: 8080
- 모바일 User-Agent 자동 감지하여 모바일 버전 서빙
- SPA 라우팅 지원 (`try_files`)
- 캐시 비활성화 (개발 환경용)

## 빌드 스크립트

### 전체 빌드 스크립트 (build-docker.sh)

```bash
#!/bin/bash
set -e

echo "=== AFFiNE Docker 이미지 빌드 ==="

# 색상 정의
GREEN='\033[0;32m'
NC='\033[0m'

step() {
  echo -e "${GREEN}[STEP]${NC} $1"
}

# 1. 서버 네이티브 모듈 빌드
step "서버 네이티브 모듈 빌드 중..."
yarn affine @affine/server-native build

# 2. 프론트엔드 빌드
step "웹 앱 빌드 중..."
yarn affine @affine/web build

step "관리자 페이지 빌드 중..."
yarn affine @affine/admin build

step "모바일 웹 빌드 중..."
yarn affine @affine/mobile build

# 3. 서버 빌드
step "Reader 빌드 중..."
yarn workspace @affine/reader build

step "서버 빌드 중..."
yarn workspace @affine/server build

# 4. Prisma 클라이언트 생성
step "Prisma 클라이언트 생성 중..."
yarn workspace @affine/server prisma generate

# 5. 프로덕션 의존성 준비
step "프로덕션 의존성 설치 중..."
yarn workspaces focus @affine/server --production

# 6. node_modules 이동
step "node_modules 이동 중..."
mv ./node_modules ./packages/backend/server

# 7. Docker 이미지 빌드
step "Docker 이미지 빌드 중..."
docker build -f .github/deployment/node/Dockerfile -t affine:local .

echo -e "${GREEN}=== 빌드 완료! ===${NC}"
echo "이미지: affine:local"
```

### 프론트엔드 전용 빌드 스크립트 (build-docker-front.sh)

```bash
#!/bin/bash
set -e

echo "=== AFFiNE Frontend Docker 이미지 빌드 ==="

# 프론트엔드 빌드
yarn affine @affine/web build
yarn affine @affine/admin build
yarn affine @affine/mobile build

# Docker 이미지 빌드
docker build -f .github/deployment/front/Dockerfile -t affine-front:local .

echo "=== 빌드 완료! ==="
echo "이미지: affine-front:local"
```

## 멀티 플랫폼 빌드

### ARM64 지원 (Apple Silicon, AWS Graviton 등)

```bash
# Docker Buildx 설정
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# 멀티 플랫폼 빌드 (로컬 저장)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f .github/deployment/node/Dockerfile \
  -t affine:local \
  --load .

# 멀티 플랫폼 빌드 (레지스트리 푸시)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f .github/deployment/node/Dockerfile \
  -t your-registry/affine:latest \
  --push .
```

### 지원 플랫폼

| 플랫폼 | 아키텍처 | 용도 |
|--------|----------|------|
| `linux/amd64` | x86_64 | 일반 서버, Intel Mac |
| `linux/arm64` | aarch64 | Apple Silicon, AWS Graviton |
| `linux/arm/v7` | armv7 | Raspberry Pi 등 |

## 환경 변수 설정

### 빌드 시 환경 변수

```bash
# 빌드 타입 (canary, beta, stable)
BUILD_TYPE=stable

# Sentry 설정 (선택)
SENTRY_ORG=your-org
SENTRY_PROJECT=affine-web
SENTRY_AUTH_TOKEN=your-token
SENTRY_DSN=your-dsn

# 빌드 예시
BUILD_TYPE=stable yarn affine @affine/web build
```

### 런타임 환경 변수

```bash
# 데이터베이스
DATABASE_URL=postgresql://user:pass@host:5432/affine

# Redis
REDIS_SERVER_HOST=redis

# 서버 설정
AFFINE_SERVER_HOST=0.0.0.0
AFFINE_SERVER_PORT=3010
```

## 빌드 결과물 확인

### 빌드된 디렉토리 구조

```
packages/
├── frontend/
│   ├── apps/
│   │   ├── web/dist/          # 웹 앱 빌드 결과
│   │   └── mobile/dist/       # 모바일 웹 빌드 결과
│   └── admin/dist/            # 관리자 페이지 빌드 결과
└── backend/
    └── server/
        ├── dist/              # 서버 빌드 결과
        ├── node_modules/      # 프로덕션 의존성
        └── prisma/            # DB 스키마
```

### 이미지 확인

```bash
# 이미지 목록
docker images | grep affine

# 이미지 크기 확인
docker image inspect affine:local --format='{{.Size}}' | numfmt --to=iec

# 이미지 레이어 확인
docker history affine:local
```

## 이미지 실행

### 단독 실행 (테스트용)

```bash
docker run -d \
  --name affine \
  -p 3010:3010 \
  -e DATABASE_URL=postgresql://user:pass@host:5432/affine \
  -e REDIS_SERVER_HOST=redis \
  affine:local
```

### Docker Compose와 함께 사용

```yaml
# docker-compose.yml
services:
  affine:
    image: affine:local
    ports:
      - "3010:3010"
    environment:
      - DATABASE_URL=postgresql://affine:affine@postgres:5432/affine
      - REDIS_SERVER_HOST=redis
    depends_on:
      - postgres
      - redis

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      - POSTGRES_USER=affine
      - POSTGRES_PASSWORD=affine
      - POSTGRES_DB=affine

  redis:
    image: redis
```

## 문제 해결

### 빌드 실패 시

1. **Rust 빌드 실패**
   ```bash
   # Rust 툴체인 확인
   rustc --version  # 1.87.0 이상

   # 재빌드
   yarn affine @affine/server-native build
   ```

2. **메모리 부족**
   ```bash
   # Node.js 메모리 증가
   export NODE_OPTIONS="--max-old-space-size=8192"
   ```

3. **Docker 빌드 컨텍스트 오류**
   ```bash
   # 프로젝트 루트에서 실행 확인
   pwd  # /path/to/AFFiNE

   # .dockerignore 확인
   cat .dockerignore
   ```

### 이미지 크기 최적화

```bash
# 멀티스테이지 빌드 사용 시
# 불필요한 파일 제외
# .dockerignore에 추가:
node_modules
*.md
tests
.git
```

## 공식 이미지 사용

직접 빌드 대신 공식 이미지를 사용할 수도 있습니다:

```bash
# 안정 버전
docker pull ghcr.io/toeverything/affine:stable

# 베타 버전
docker pull ghcr.io/toeverything/affine:beta

# 최신 개발 버전
docker pull ghcr.io/toeverything/affine:canary
```

## 관련 파일

| 파일 | 설명 |
|------|------|
| `.github/deployment/node/Dockerfile` | 통합 이미지 Dockerfile |
| `.github/deployment/front/Dockerfile` | 프론트엔드 Dockerfile |
| `.github/deployment/front/nginx.conf` | Nginx 기본 설정 |
| `.github/deployment/front/affine.nginx.conf` | Nginx 사이트 설정 |
| `.github/workflows/build-images.yml` | CI/CD 빌드 워크플로우 |
