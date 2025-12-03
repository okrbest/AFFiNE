# ============================================
# AFFiNE Production Dockerfile (로컬 빌드 방식)
# ============================================
#
# 이 Dockerfile은 로컬에서 빌드된 결과물을 Docker 이미지로 패키징합니다.
#
# 빌드 전 필수 작업 (로컬 또는 CI에서 실행):
#   01. yarn install
#   02. yarn affine @affine/server-native build
#   03. yarn affine build -p @affine/reader
#   04. yarn affine build -p @affine/server
#   05. NODE_OPTIONS="--max-old-space-size=8192" yarn affine build -p @affine/web
#   06. NODE_OPTIONS="--max-old-space-size=8192" yarn affine build -p @affine/admin (선택)
#   07. NODE_OPTIONS="--max-old-space-size=8192" yarn affine build -p @affine/mobile (선택)
#
# Docker 이미지 빌드:
#   08. docker build -t affine-app:latest -f .github/deployment/node/Dockerfile .
#
# 컨테이너 재시작:
#   09. docker compose -f .docker/selfhost/compose.yml down
#   10. docker compose -f .docker/selfhost/compose.yml up -d

FROM node:22-bookworm-slim

# 필수 런타임 패키지 설치
RUN apt-get update && \
  apt-get install -y --no-install-recommends openssl libjemalloc2 && \
  rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 빌드된 서버 파일 복사
COPY ./packages/backend/server/dist ./dist
COPY ./packages/backend/server/package.json ./
COPY ./packages/backend/server/schema.prisma ./
COPY ./packages/backend/server/scripts ./scripts

# Node modules 복사 (프로덕션 의존성 포함)
COPY ./node_modules ./node_modules

# 네이티브 모듈 복사 (선택적)
COPY ./packages/backend/native ./native

# 정적 프론트엔드 파일 복사
COPY ./packages/frontend/apps/web/dist ./static
COPY ./packages/frontend/admin/dist ./static/admin
COPY ./packages/frontend/apps/mobile/dist ./static/mobile

# jemalloc 메모리 할당자 활성화
ENV LD_PRELOAD=libjemalloc.so.2

# 포트 노출
EXPOSE 3010

# 헬스체크
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3010/api/healthz', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

CMD ["node", "./dist/main.js"]
