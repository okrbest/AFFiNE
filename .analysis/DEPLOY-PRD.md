# AFFiNE 프로덕션 배포 가이드

## 목차

1. [배포 방식 개요](#배포-방식-개요)
2. [사전 준비](#사전-준비)
3. [프로덕션 빌드](#프로덕션-빌드)
4. [Docker 설정 파일 작성](#docker-설정-파일-작성)
5. [Docker 이미지 빌드](#docker-이미지-빌드)
6. [서비스 배포 및 운영](#서비스-배포-및-운영)
7. [고급 설정](#고급-설정)
8. [문제 해결](#문제-해결)

---

## 배포 방식 개요

AFFiNE는 두 가지 배포 방식을 지원합니다:

### 1. 통합 배포 (권장)

- **파일**: `node/Dockerfile`
- **특징**: Backend 서버가 정적 파일도 함께 서빙
- **장점**: 설정 간단, 관리 용이
- **적합**: 중소규모 배포, 단일 서버 환경

### 2. 분리 배포

- **파일**: `front/Dockerfile`
- **특징**: Nginx로 정적 파일만 별도 서빙
- **장점**: 대규모 트래픽 처리에 유리
- **적합**: 대규모 서비스, CDN 활용 필요 시

**이 가이드는 통합 배포 방식을 기준으로 작성되었습니다.**

---

## 사전 준비

### SECTION 1: 개발 환경 설치 (로컬 머신에서 실행)

#### 1-1. Node.js 설치

```bash
# nvm으로 Node.js 22.16.0 설치
nvm install 22.16.0
nvm use 22.16.0

# 버전 확인
node --version  # v22.16.0 출력 확인
```

#### 1-2. Rust 설치

```bash
# Rust 1.87.0 설치
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 환경 변수 적용
source $HOME/.cargo/env

# 버전 확인
rustc --version  # rustc 1.87.0 확인
```

#### 1-3. 프로젝트 의존성 설치

```bash
# 프로젝트 루트 디렉토리로 이동
cd /path/to/AFFiNE

# Corepack 활성화
corepack enable

# 의존성 설치 (시간이 오래 걸릴 수 있음)
yarn install
```

---

## 프로덕션 빌드

### SECTION 2: 애플리케이션 빌드 (명령어 실행)

#### 2-1. 네이티브 모듈 빌드

```bash
# Rust 네이티브 모듈 빌드 (서버용)
yarn affine @affine/server-native build
```

#### 2-2. reader 모듈 빌드

```bash
# Reader 모듈 빌드
yarn affine build -p @affine/reader
```

#### 2-3. 백엔드 빌드

```bash
# NestJS 백엔드 빌드
yarn affine build -p @affine/server
```

#### 2-4. 프론트엔드 빌드

```bash
# React 웹 애플리케이션 빌드
# yarn affine build -p @affine/web
NODE_OPTIONS="--max-old-space-size=8192" yarn affine build -p @affine/web # memory issue 가 있을 때 사용
```

#### 2-5. 선택적 빌드 (필요시)

```bash
# Admin 패널 빌드
yarn affine build -p @affine/admin

# Mobile 웹 빌드
yarn affine build -p @affine/mobile
```

**빌드 완료 확인**:

- `packages/backend/server/dist/` 디렉토리 확인
- `packages/frontend/apps/web/dist/` 디렉토리 확인

---

## Docker 설정 파일 작성

### SECTION 3: 필수 설정 파일 생성

#### 3-1. Dockerfile 생성 (파일 작성)

**파일 위치**: 프로젝트 루트에 `Dockerfile` 생성 또는 `.github/deployment/node/Dockerfile` 파일 사용

**중요**: Migration job에서 `prisma` CLI를 사용하려면 node_modules가 포함되어야 합니다!

```dockerfile
# 프로덕션 배포용 Dockerfile (Prisma 지원)
FROM node:22-bookworm-slim

# 필수 시스템 패키지 설치
RUN apt-get update && \
  apt-get install -y --no-install-recommends openssl libjemalloc2 && \
  rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 의존성 파일 먼저 복사 (Docker 레이어 캐싱 최적화)
COPY ./packages/backend/server/package.json ./packages/backend/server/yarn.lock* ./
COPY ./packages/backend/server/prisma ./prisma

# 의존성 설치 (prisma CLI 포함)
RUN corepack enable && \
  yarn install --production --frozen-lockfile && \
  yarn cache clean

# Prisma Client 생성
RUN yarn prisma generate

# 나머지 애플리케이션 파일 복사
COPY ./packages/backend/server /app
COPY ./packages/frontend/apps/web/dist /app/static
COPY ./packages/frontend/admin/dist /app/static/admin
COPY ./packages/frontend/apps/mobile/dist /app/static/mobile

# jemalloc 메모리 할당자 활성화 (성능 향상)
ENV LD_PRELOAD=libjemalloc.so.2

EXPOSE 3010

CMD ["node", "./dist/main.js"]
```

**변경 사항 설명**:

- `yarn install`로 node_modules 설치 (prisma CLI 포함)
- `prisma generate`로 Prisma Client 생성
- Docker 레이어 캐싱을 활용한 빌드 최적화
- Migration job에서 `yarn prisma migrate deploy` 실행 가능

#### 3-2. Docker Compose 파일 생성 (파일 작성)

**파일 위치**: 프로젝트 루트에 `docker-compose.yml` 생성 또는 `.docker/selfhost/compose.yml` 파일 사용

```yaml
name: affine
services:
  affine:
    #    image: ghcr.io/toeverything/affine:${AFFINE_REVISION:-stable}
    image: affine-app:latest
    container_name: affine_server
    ports:
      - '${PORT:-3010}:3010'
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
      affine_migration:
        condition: service_completed_successfully
    volumes:
      # custom configurations
      - ${UPLOAD_LOCATION}:/root/.affine/storage
      - ${CONFIG_LOCATION}:/root/.affine/config
    env_file:
      - .env
    environment:
      - REDIS_SERVER_HOST=redis
      - DATABASE_URL=postgresql://${DB_USERNAME}:${DB_PASSWORD}@postgres:5432/${DB_DATABASE:-affine}
      - AFFINE_INDEXER_ENABLED=false
    restart: unless-stopped

  affine_migration:
    #    image: ghcr.io/toeverything/affine:${AFFINE_REVISION:-stable}
    image: affine-app:latest
    container_name: affine_migration_job
    volumes:
      # custom configurations
      - ${UPLOAD_LOCATION}:/root/.affine/storage
      - ${CONFIG_LOCATION}:/root/.affine/config
    command: ['sh', '-c', 'node ./scripts/self-host-predeploy.js']
    env_file:
      - .env
    environment:
      - REDIS_SERVER_HOST=redis
      - DATABASE_URL=postgresql://${DB_USERNAME}:${DB_PASSWORD}@postgres:5432/${DB_DATABASE:-affine}
      - AFFINE_INDEXER_ENABLED=false
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  redis:
    image: redis
    container_name: affine_redis
    healthcheck:
      test: ['CMD', 'redis-cli', '--raw', 'incr', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  postgres:
    image: pgvector/pgvector:pg16
    container_name: affine_postgres
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_DATABASE:-affine}
      POSTGRES_INITDB_ARGS: '--data-checksums'
      # you better set a password for you database
      # or you may add 'POSTGRES_HOST_AUTH_METHOD=trust' to ignore postgres security policy
      POSTGRES_HOST_AUTH_METHOD: trust
    healthcheck:
      test: ['CMD', 'pg_isready', '-U', '${DB_USERNAME}', '-d', '${DB_DATABASE:-affine}']
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
```

#### 3-3. 환경 변수 파일 생성 (선택적, 파일 작성)

**파일 위치**: 프로젝트 루트에 `.env` 생성

```env
# 데이터베이스 인증 정보 (반드시 변경!)
POSTGRES_USER=affine
POSTGRES_PASSWORD=your_very_secure_password_here
POSTGRES_DB=affine

# 애플리케이션 포트
AFFINE_PORT=3010

# 외부 URL (실제 도메인 사용 시)
# AFFINE_SERVER_HTTPS=true
# AFFINE_SERVER_EXTERNAL_URL=https://affine.yourdomain.com
```

**주의**: `.env` 파일 사용 시 `docker-compose.yml`의 비밀번호도 `.env` 변수로 교체해야 합니다.

---

## Docker 이미지 빌드

### SECTION 4: 이미지 생성 (명령어 실행)

#### 4-1. Docker 이미지 빌드

```bash
# 프로젝트 루트에서 실행
docker build -t affine-app:latest -f Dockerfile .
```

**빌드 시간**: 약 5-10분 소요

#### 4-2. 이미지 확인

```bash
# 생성된 이미지 목록 확인
docker images affine-app

# 예상 출력:
# REPOSITORY    TAG       IMAGE ID       CREATED          SIZE
# affine-app    latest    abc123def456   10 seconds ago   1.2GB
```

---

## 서비스 배포 및 운영

### SECTION 5: 서비스 시작 (명령어 실행)

#### 5-1. 서비스 시작

```bash
# Docker Compose로 모든 서비스 시작
docker compose up -d
```

**실행 순서**:

1. postgres 컨테이너 시작 → 헬스체크 대기
2. redis 컨테이너 시작 → 헬스체크 대기
3. affine-migration 실행 (DB 마이그레이션)
4. affine 애플리케이션 시작

#### 5-2. 서비스 상태 확인

```bash
# 컨테이너 상태 확인
docker compose ps

# 예상 출력:
# NAME                STATUS              PORTS
# affine-app          Up 2 minutes        0.0.0.0:3010->3010/tcp
# affine-postgres     Up 3 minutes        5432/tcp
# affine-redis        Up 3 minutes        6379/tcp
```

#### 5-3. 로그 확인

```bash
# 전체 서비스 로그 (실시간)
docker compose logs -f

# 특정 서비스만 확인
docker compose logs -f affine
docker compose logs -f postgres
```

#### 5-4. 접속 확인

**브라우저에서 접속**:

- 로컬: http://localhost:3010
- 원격: http://서버IP:3010

**첫 로그인**:

- 회원가입 페이지에서 계정 생성
- 관리자 계정은 첫 번째 가입자에게 자동 부여

---

### SECTION 6: 서비스 관리 명령어

#### 6-1. 서비스 제어

```bash
# 서비스 중지
docker compose down

# 서비스 재시작
docker compose restart affine

# 전체 재시작 (마이그레이션 포함)
docker compose down && docker compose up -d

# 특정 서비스만 재시작
docker compose restart affine
```

#### 6-2. 컨테이너 접속

```bash
# 애플리케이션 컨테이너 쉘 접속
docker compose exec affine sh

# PostgreSQL 접속
docker compose exec postgres psql -U affine -d affine

# Redis CLI 접속
docker compose exec redis redis-cli
```

#### 6-3. 데이터 백업

```bash
# PostgreSQL 데이터베이스 백업
docker compose exec postgres pg_dump -U affine affine > backup_$(date +%Y%m%d).sql

# 볼륨 전체 백업
docker run --rm \
  -v affine-production_postgres_data:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/postgres_backup.tar.gz /data
```

#### 6-4. 애플리케이션 업데이트

```bash
# 1. 새 버전 코드 pull
git pull origin main

# 2. 의존성 업데이트
yarn install

# 3. 빌드
yarn affine @affine/server-native build
yarn affine build -p @affine/server
yarn affine build -p @affine/web

# 4. 새 Docker 이미지 생성
docker build -t affine-app:latest .

# 5. 서비스 재시작
docker compose down
docker compose up -d
```

---

## 고급 설정

### SECTION 7-1: 멀티 스테이지 빌드 (자동화)

**목적**: 빌드 환경까지 Docker로 자동화 (로컬 빌드 불필요)

**파일 위치**: 프로젝트 루트에 `Dockerfile.multistage` 생성

```dockerfile
# Stage 1: 빌드 환경
FROM node:22-bookworm AS builder

# Rust 설치
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

WORKDIR /build

# 의존성 파일 먼저 복사 (캐시 활용)
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn ./.yarn

# Corepack 활성화 및 의존성 설치
RUN corepack enable && yarn install --immutable

# 소스 코드 복사
COPY . .

# 빌드 실행
RUN yarn affine @affine/server-native build && \
    yarn affine build -p @affine/server && \
    yarn affine build -p @affine/web && \
    yarn affine build -p @affine/admin && \
    yarn affine build -p @affine/mobile

# Stage 2: 프로덕션 이미지
FROM node:22-bookworm-slim

# 빌드된 파일만 복사
COPY --from=builder /build/packages/backend/server /app
COPY --from=builder /build/packages/frontend/apps/web/dist /app/static
COPY --from=builder /build/packages/frontend/admin/dist /app/static/admin
COPY --from=builder /build/packages/frontend/apps/mobile/dist /app/static/mobile

WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends openssl libjemalloc2 && \
    rm -rf /var/lib/apt/lists/*

ENV LD_PRELOAD=libjemalloc.so.2

EXPOSE 3010
CMD ["node", "./dist/main.js"]
```

**사용법**:

```bash
# 멀티 스테이지로 빌드 (로컬 빌드 없이 전부 Docker 안에서)
docker build -t affine-app:latest -f Dockerfile.multistage .
```

---

### SECTION 7-2: Nginx 리버스 프록시 (HTTPS)

**목적**: SSL/TLS 인증서로 HTTPS 제공

**파일 위치**: 프로젝트 루트에 `nginx.conf` 생성

```nginx
# HTTP -> HTTPS 리다이렉트
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS 서버
server {
    listen 443 ssl http2;
    server_name your-domain.com;

    # SSL 인증서 경로
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    # SSL 설정
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # 프록시 설정
    location / {
        proxy_pass http://affine:3010;
        proxy_http_version 1.1;

        # WebSocket 지원
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';

        # 헤더 전달
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache_bypass $http_upgrade;
    }
}
```

**Docker Compose에 Nginx 추가**:

`docker-compose.yml`에 서비스 추가:

```yaml
nginx:
  image: nginx:alpine
  container_name: affine-nginx
  ports:
    - '80:80'
    - '443:443'
  volumes:
    - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    - ./ssl:/etc/nginx/ssl:ro
  depends_on:
    - affine
  restart: unless-stopped
  networks:
    - affine-network
```

---

### SECTION 7-3: 리소스 제한 설정

**목적**: 컨테이너 메모리/CPU 사용량 제한

`docker-compose.yml`의 `affine` 서비스에 추가:

```yaml
affine:
  # ... 기존 설정 ...
  deploy:
    resources:
      limits:
        cpus: '2' # 최대 2 CPU 코어
        memory: 2G # 최대 2GB RAM
      reservations:
        cpus: '1' # 최소 1 CPU 코어
        memory: 512M # 최소 512MB RAM
```

---

## 문제 해결

### SECTION 8: 일반적인 문제와 해결 방법

#### 8-1. 마이그레이션 실패

**증상**: `affine-migration` 컨테이너가 실패

```bash
# 마이그레이션 로그 확인
docker compose logs affine-migration

# 수동 마이그레이션 실행
docker compose run --rm affine-migration node ./scripts/self-host-predeploy.js
```

#### 8-2. 포트 충돌

**증상**: "address already in use" 오류

**해결**: `docker-compose.yml`에서 호스트 포트 변경

```yaml
ports:
  - '8080:3010' # 3010 대신 8080 포트 사용
```

#### 8-3. 데이터베이스 연결 실패

**증상**: "Cannot connect to database" 오류

```bash
# PostgreSQL 헬스 체크
docker compose exec postgres pg_isready -U affine

# 연결 테스트
docker compose exec postgres psql -U affine -d affine -c "SELECT 1;"
```

#### 8-4. 데이터 완전 초기화

**주의**: 모든 데이터가 삭제됩니다!

```bash
# 컨테이너 + 볼륨 모두 삭제
docker compose down -v

# 재시작
docker compose up -d
```

#### 8-5. 빌드 실패

**증상**: Docker 이미지 빌드 중 오류

```bash
# 빌드 캐시 무시하고 재빌드
docker build --no-cache -t affine-app:latest .

# 이전 빌드 정리
docker builder prune -a
```

#### 8-6. 메모리 부족

**증상**: 컨테이너가 갑자기 종료됨

```bash
# Docker 시스템 리소스 확인
docker stats

# 불필요한 이미지/컨테이너 정리
docker system prune -a
```

---

## 체크리스트

### 배포 전 확인사항

- [ ] Node.js 22.16.0 설치 완료
- [ ] Rust 1.87.0 설치 완료
- [ ] Docker 및 Docker Compose 설치 완료
- [ ] 프로젝트 빌드 완료
- [ ] `Dockerfile` 생성 완료
- [ ] `docker-compose.yml` 생성 완료
- [ ] 비밀번호 변경 완료 (`POSTGRES_PASSWORD`)
- [ ] 포트 충돌 확인 (3010 포트)

### 배포 후 확인사항

- [ ] `docker compose ps`로 모든 컨테이너 실행 확인
- [ ] 브라우저에서 접속 확인
- [ ] 회원가입 및 로그인 테스트
- [ ] 문서 생성/저장 테스트
- [ ] 로그에 에러 없는지 확인 (`docker compose logs`)

---

## 추가 참고사항

### 권장 서버 사양

- **최소**: CPU 2코어, RAM 4GB, 디스크 20GB
- **권장**: CPU 4코어, RAM 8GB, 디스크 50GB
- **OS**: Ubuntu 22.04 LTS 이상, Docker 24.0 이상

### 보안 권장사항

1. **방화벽 설정**: 3010 포트만 개방
2. **비밀번호 강화**: 최소 16자, 영문+숫자+특수문자
3. **SSL 인증서**: Let's Encrypt 무료 인증서 사용
4. **정기 백업**: 매일 자동 백업 스크립트 설정
5. **업데이트**: 월 1회 보안 패치 확인

### 유용한 명령어 모음

```bash
# 실시간 리소스 모니터링
docker stats

# 특정 컨테이너 로그 (최근 100줄)
docker compose logs --tail=100 affine

# 컨테이너 재시작 없이 설정 리로드
docker compose up -d --force-recreate affine

# 전체 시스템 정보
docker system info

# 디스크 사용량 확인
docker system df
```

---

## 기타

### 왜 `affine-migration`과 `affine` 서비스가 분리되어 있나요?

Docker Compose 설정에서 `affine-migration`과 `affine`이 별도 서비스로 구성된 이유를 설명합니다.

#### 1. 역할의 차이

- **`affine-migration`**: 일회성 초기화 작업

  - 데이터베이스 스키마 생성/업데이트 (Prisma 마이그레이션)
  - 초기 데이터 세팅
  - 실행 후 자동 종료 (상시 실행 X)
  - 명령어: `node ./scripts/self-host-predeploy.js`

- **`affine`**: 상시 실행 애플리케이션 서버
  - 웹 서버 실행
  - API 요청 처리
  - 사용자 요청 대기 (계속 실행)
  - 명령어: `node ./dist/main.js`

#### 2. 실행 순서 보장

```yaml
affine:
  depends_on:
    affine-migration:
      condition: service_completed_successfully # 마이그레이션 성공 후에만 시작
```

이렇게 설정하면 안전한 실행 순서가 보장됩니다:

1. **postgres, redis 시작** → healthy 상태 확인
2. **affine-migration 실행** → DB 스키마 생성
3. **마이그레이션 완료 확인** → Exited (0) 상태
4. **affine 서버 시작** → 정상적으로 DB 사용 가능

#### 3. 실제 동작 예시

```bash
# 첫 배포 시
docker compose up -d

# 실행 흐름:
# [1] postgres 시작 → healthcheck 통과
# [2] redis 시작 → healthcheck 통과
# [3] affine-migration 실행
#     └─ Prisma migrate deploy
#     └─ 초기 데이터 세팅
#     └─ 완료 후 종료 (Exited 0)
# [4] affine 서버 시작 (계속 실행 중)
```

#### 4. 만약 하나로 합치면 발생하는 문제점

만약 `affine` 컨테이너에서 마이그레이션도 함께 실행한다면:

```dockerfile
# 안티 패턴 (권장하지 않음)
CMD ["sh", "-c", "node ./scripts/self-host-predeploy.js && node ./dist/main.js"]
```

**발생하는 문제**:

- 서버 재시작할 때마다 마이그레이션 재실행 (불필요한 작업)
- 마이그레이션 실패 시 서버도 시작되지 않음
- 로그 추적 어려움 (초기화 로그 vs 서버 로그 혼재)
- 디버깅 복잡도 증가

#### 5. 실제 상태 확인 방법

```bash
# 서비스 상태 확인
docker compose ps

# 정상 상태 예시:
# NAME                STATUS                       PORTS
# affine-app          Up 2 minutes                 0.0.0.0:3010->3010/tcp
# affine-migration    Exited (0) 3 minutes ago     -
# affine-postgres     Up 3 minutes                 5432/tcp
# affine-redis        Up 3 minutes                 6379/tcp
```

**중요**: `affine-migration`은 `Exited (0)` 상태가 정상입니다!

#### 6. 비교 표

| 구분          | affine-migration                        | affine                |
| ------------- | --------------------------------------- | --------------------- |
| **역할**      | DB 마이그레이션                         | 웹 서버               |
| **실행 타입** | 한 번 실행 후 종료                      | 계속 실행 (데몬)      |
| **명령어**    | `node ./scripts/self-host-predeploy.js` | `node ./dist/main.js` |
| **정상 상태** | `Exited (0)`                            | `Up`                  |
| **재시작**    | 필요 없음                               | 필요 시 재시작 가능   |
| **로그 유형** | 초기화 로그                             | 서버 요청 로그        |

#### 7. 유사 패턴

이러한 분리 패턴은 컨테이너 오케스트레이션에서 일반적으로 사용됩니다:

- **Kubernetes**: Init Container 패턴
- **Docker Compose**: depends_on + service_completed_successfully
- **Cloud Run**: Cloud SQL Proxy 사이드카 패턴

#### 8. 마이그레이션 재실행이 필요한 경우

업데이트로 인해 새로운 마이그레이션이 필요할 때:

```bash
# 마이그레이션만 다시 실행
docker compose run --rm affine-migration

# 또는 전체 재시작
docker compose down
docker compose up -d  # 마이그레이션 자동 실행됨
```

---

**문서 버전**: 1.0
**최종 업데이트**: 2025-12-02
**작성자**: AI Assistant
