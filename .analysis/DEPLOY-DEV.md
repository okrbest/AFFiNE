# AFFiNE 개발 서버 배포 가이드

## 개요

AFFiNE 웹 애플리케이션을 개발 서버에 Docker Compose를 사용하여 배포하는 방법을 설명합니다.

## 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Compose 환경                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   AFFiNE    │  │  PostgreSQL │  │       Redis         │  │
│  │   Server    │──│  (pgvector) │  │  (캐시/세션)        │  │
│  │  :3010      │  │   :5432     │  │     :6379           │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│         │                                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Migration  │  │   Mailpit   │  │   Manticoresearch   │  │
│  │    Job      │  │ (이메일테스트)│  │   (검색엔진)        │  │
│  │             │  │ :1025/:8025 │  │      :9308          │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 배포 방식 비교

| 방식 | 용도 | 구성 파일 |
|------|------|----------|
| **Self-Hosted** | 운영/개발 서버 배포 | `.docker/selfhost/compose.yml` |
| **Dev** | 로컬 개발 환경 | `.docker/dev/compose.yml` |

## Self-Hosted 배포 (권장)

### 1. 환경 설정 파일 준비

```bash
# 환경 변수 파일 복사
cp .docker/selfhost/.env.example .docker/selfhost/.env
```

### 2. 환경 변수 설정

`.docker/selfhost/.env` 파일을 편집합니다:

```bash
# AFFiNE 버전 (stable, beta, canary)
AFFINE_REVISION=stable

# 서버 포트
PORT=3010

# 외부 URL 설정 (선택사항)
# AFFINE_SERVER_HTTPS=true
# AFFINE_SERVER_HOST=affine.yourdomain.com
# AFFINE_SERVER_EXTERNAL_URL=https://affine.yourdomain.com

# 데이터 저장 경로
DB_DATA_LOCATION=~/.affine/self-host/postgres/pgdata
UPLOAD_LOCATION=~/.affine/self-host/storage
CONFIG_LOCATION=~/.affine/self-host/config

# 데이터베이스 인증 정보 (반드시 변경!)
DB_USERNAME=affine
DB_PASSWORD=your_secure_password_here
DB_DATABASE=affine
```

### 3. Docker Compose 실행

```bash
# 서비스 시작
docker compose -f .docker/selfhost/compose.yml up -d

# 로그 확인
docker compose -f .docker/selfhost/compose.yml logs -f

# 서비스 상태 확인
docker compose -f .docker/selfhost/compose.yml ps
```

### 4. 접속 확인

브라우저에서 `http://서버IP:3010` 접속

## Docker Compose 구성 상세

### Self-Hosted compose.yml 구조

```yaml
name: affine

services:
  # 메인 애플리케이션 서버
  affine:
    image: ghcr.io/toeverything/affine:${AFFINE_REVISION:-stable}
    ports:
      - "${PORT:-3010}:3010"
    depends_on:
      redis:
        condition: service_healthy
      postgres:
        condition: service_healthy
      affine_migration:
        condition: service_completed_successfully
    volumes:
      - ${UPLOAD_LOCATION}:/root/.affine/storage
      - ${CONFIG_LOCATION}:/root/.affine/config
    environment:
      - REDIS_SERVER_HOST=redis
      - DATABASE_URL=postgresql://${DB_USERNAME}:${DB_PASSWORD}@postgres:5432/${DB_DATABASE:-affine}
      - AFFINE_INDEXER_ENABLED=false
    restart: unless-stopped

  # 데이터베이스 마이그레이션 작업
  affine_migration:
    image: ghcr.io/toeverything/affine:${AFFINE_REVISION:-stable}
    volumes:
      - ${CONFIG_LOCATION}:/root/.affine/config
    environment:
      - REDIS_SERVER_HOST=redis
      - DATABASE_URL=postgresql://${DB_USERNAME}:${DB_PASSWORD}@postgres:5432/${DB_DATABASE:-affine}
    command: ["node", "./scripts/self-host-predeploy.js"]
    depends_on:
      postgres:
        condition: service_healthy

  # PostgreSQL (pgvector 지원)
  postgres:
    image: pgvector/pgvector:pg16
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=${DB_USERNAME}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_DATABASE:-affine}
      - POSTGRES_INITDB_ARGS=--data-checksums
    healthcheck:
      test: pg_isready -U ${DB_USERNAME} -d ${DB_DATABASE:-affine}
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Redis 캐시
  redis:
    image: redis
    healthcheck:
      test: redis-cli --raw incr ping
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
```

## 포트 구성

### 필수 서비스 포트

| 서비스 | 내부 포트 | 외부 노출 | 설명 |
|--------|----------|----------|------|
| AFFiNE Server | 3010 | ✅ | 웹 애플리케이션 |
| PostgreSQL | 5432 | ❌ | 데이터베이스 |
| Redis | 6379 | ❌ | 캐시/세션 |

### 개발용 추가 서비스 포트

| 서비스 | 포트 | 설명 |
|--------|------|------|
| Mailpit SMTP | 1025 | 이메일 테스트 송신 |
| Mailpit Web | 8025 | 이메일 확인 UI |
| Manticoresearch | 9308 | 전문 검색 엔진 |
| Elasticsearch | 9200 | 대체 검색 엔진 (선택) |

## 볼륨 및 데이터 저장

### 데이터 영속성

```
~/.affine/self-host/
├── postgres/
│   └── pgdata/          # PostgreSQL 데이터
├── storage/             # 사용자 업로드 파일 (이미지, 첨부파일)
└── config/              # 설정 및 개인키
    └── private.key      # 암호화 키 (자동 생성)
```

### 주요 볼륨 마운트

| 호스트 경로 | 컨테이너 경로 | 용도 |
|------------|--------------|------|
| `${DB_DATA_LOCATION}` | `/var/lib/postgresql/data` | DB 데이터 |
| `${UPLOAD_LOCATION}` | `/root/.affine/storage` | 사용자 파일 |
| `${CONFIG_LOCATION}` | `/root/.affine/config` | 앱 설정 |

## 헬스 체크 설정

서비스 안정성을 위해 헬스 체크가 구성되어 있습니다:

```yaml
# PostgreSQL 헬스 체크
healthcheck:
  test: pg_isready -U ${DB_USERNAME} -d ${DB_DATABASE:-affine}
  interval: 10s
  timeout: 5s
  retries: 5

# Redis 헬스 체크
healthcheck:
  test: redis-cli --raw incr ping
  interval: 10s
  timeout: 5s
  retries: 5
```

## 서비스 의존성

```
affine
├── redis (service_healthy)
├── postgres (service_healthy)
└── affine_migration (service_completed_successfully)
    └── postgres (service_healthy)
```

마이그레이션 작업이 성공적으로 완료된 후에만 메인 서버가 시작됩니다.

## Docker 이미지

### 공식 이미지

| 이미지 | 태그 옵션 | 설명 |
|--------|----------|------|
| `ghcr.io/toeverything/affine` | `stable` | 안정 버전 (권장) |
| `ghcr.io/toeverything/affine` | `beta` | 베타 버전 |
| `ghcr.io/toeverything/affine` | `canary` | 최신 개발 버전 |
| `pgvector/pgvector` | `pg16` | PostgreSQL 16 + pgvector |
| `redis` | `latest` | Redis 캐시 |

## 운영 명령어

### 서비스 관리

```bash
# 서비스 시작
docker compose -f .docker/selfhost/compose.yml up -d

# 서비스 중지
docker compose -f .docker/selfhost/compose.yml down

# 서비스 재시작
docker compose -f .docker/selfhost/compose.yml restart

# 특정 서비스만 재시작
docker compose -f .docker/selfhost/compose.yml restart affine

# 로그 확인
docker compose -f .docker/selfhost/compose.yml logs -f affine

# 서비스 상태 확인
docker compose -f .docker/selfhost/compose.yml ps
```

### 데이터 백업

```bash
# PostgreSQL 데이터 백업
docker compose -f .docker/selfhost/compose.yml exec postgres \
  pg_dump -U affine affine > backup_$(date +%Y%m%d).sql

# 전체 데이터 디렉토리 백업
tar -czvf affine_backup_$(date +%Y%m%d).tar.gz ~/.affine/self-host/
```

### 업데이트

```bash
# 최신 이미지 다운로드
docker compose -f .docker/selfhost/compose.yml pull

# 서비스 재시작 (마이그레이션 자동 실행)
docker compose -f .docker/selfhost/compose.yml up -d
```

## 고급 설정

### HTTPS 설정 (리버스 프록시 사용)

Nginx나 Traefik 같은 리버스 프록시를 사용하여 HTTPS를 설정할 수 있습니다.

```bash
# .env에 외부 URL 설정
AFFINE_SERVER_HTTPS=true
AFFINE_SERVER_HOST=affine.yourdomain.com
AFFINE_SERVER_EXTERNAL_URL=https://affine.yourdomain.com
```

### 검색 엔진 활성화

Manticoresearch를 사용한 전문 검색 기능:

```bash
# .env에 추가
AFFINE_INDEXER_ENABLED=true
```

### 이메일 설정

실제 이메일 발송을 위한 SMTP 설정:

```bash
# .env 또는 config.json에 설정
MAILER_HOST=smtp.example.com
MAILER_PORT=587
MAILER_USER=noreply@example.com
MAILER_PASSWORD=your_password
MAILER_SECURE=true
```

### 스토리지 설정

AWS S3 또는 Cloudflare R2 사용:

```json
// config.json
{
  "storage": {
    "provider": "aws",
    "aws": {
      "accessKeyId": "YOUR_ACCESS_KEY",
      "secretAccessKey": "YOUR_SECRET_KEY",
      "bucket": "your-bucket-name",
      "region": "ap-northeast-2"
    }
  }
}
```

## 문제 해결

### 일반적인 문제

1. **마이그레이션 실패**
   ```bash
   # 마이그레이션 로그 확인
   docker compose -f .docker/selfhost/compose.yml logs affine_migration
   ```

2. **데이터베이스 연결 실패**
   ```bash
   # PostgreSQL 상태 확인
   docker compose -f .docker/selfhost/compose.yml exec postgres pg_isready
   ```

3. **서비스 시작 안됨**
   ```bash
   # 전체 로그 확인
   docker compose -f .docker/selfhost/compose.yml logs

   # 특정 서비스 로그
   docker compose -f .docker/selfhost/compose.yml logs affine
   ```

### 데이터 초기화 (주의!)

```bash
# 서비스 중지
docker compose -f .docker/selfhost/compose.yml down

# 데이터 삭제 (복구 불가!)
rm -rf ~/.affine/self-host/

# 다시 시작
docker compose -f .docker/selfhost/compose.yml up -d
```

## 설정 파일 위치 요약

| 파일 | 경로 | 설명 |
|------|------|------|
| Compose 파일 | `.docker/selfhost/compose.yml` | 서비스 정의 |
| 환경 변수 예제 | `.docker/selfhost/.env.example` | 환경 설정 템플릿 |
| 설정 스키마 | `.docker/selfhost/schema.json` | 전체 설정 옵션 |
| 설정 예제 | `.docker/selfhost/config.example.json` | 앱 설정 템플릿 |

## 개발 환경과의 차이점

| 항목 | Self-Hosted (개발서버) | Dev (로컬 개발) |
|------|----------------------|----------------|
| 목적 | 서버 배포 | 로컬 개발 |
| 이미지 | 빌드된 이미지 사용 | 소스 빌드 필요 |
| Mailpit | 미포함 | 포함 (이메일 테스트) |
| 포트 노출 | 최소화 (3010만) | 전부 노출 |
| 데이터 경로 | `~/.affine/self-host/` | Docker 볼륨 |

## 참고 자료

- [AFFiNE 공식 문서](https://affine.pro/docs)
- [Docker Compose 문서](https://docs.docker.com/compose/)
- [Self-Hosting 가이드](https://affine.pro/docs/self-host)
