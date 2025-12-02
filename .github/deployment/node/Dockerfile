# ============================================
# Stage 1: Builder - 전체 빌드 환경
# ============================================
FROM node:22-bookworm AS builder

# Rust 설치 (네이티브 모듈 빌드용)
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

# 필수 시스템 패키지
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
    build-essential \
    python3 \
    openssl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Yarn 설정 파일 및 의존성 파일 먼저 복사 (캐싱 최적화)
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn ./.yarn

# Corepack 활성화
RUN corepack enable

# 전체 소스 코드 복사
COPY . .

# 의존성 설치
RUN yarn install --immutable

# 빌드 시 메모리 증가 (heap out of memory 방지)
ENV NODE_OPTIONS="--max-old-space-size=8192"

# 빌드 실행
RUN yarn affine @affine/server-native build && \
    yarn affine build -p @affine/reader && \
    yarn affine build -p @affine/server && \
    yarn affine build -p @affine/web && \
    yarn affine build -p @affine/admin && \
    yarn affine build -p @affine/mobile

# ============================================
# Stage 2: Dependencies - 프로덕션 의존성만 설치
# ============================================
FROM node:22-bookworm AS deps

WORKDIR /app

# 루트 package.json 복사
COPY --from=builder /build/package.json /build/yarn.lock /build/.yarnrc.yml ./
COPY --from=builder /build/.yarn ./.yarn

# 워크스페이스 구조 복사 (package.json들만)
COPY --from=builder /build/packages/backend/server/package.json ./packages/backend/server/
COPY --from=builder /build/packages/backend/server/schema.prisma ./packages/backend/server/

# Corepack 활성화 및 프로덕션 의존성 설치
RUN corepack enable && \
    yarn workspaces focus @affine/server --production

# Prisma Client 생성
RUN cd packages/backend/server && yarn prisma generate

# ============================================
# Stage 3: Production - 최종 런타임 이미지
# ============================================
FROM node:22-bookworm-slim

# 필수 런타임 패키지만 설치
RUN apt-get update && \
  apt-get install -y --no-install-recommends openssl libjemalloc2 && \
  rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 프로덕션 의존성 복사
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/packages/backend/server/node_modules ./packages/backend/server/node_modules

# 빌드된 서버 코드 복사
COPY --from=builder /build/packages/backend/server/dist ./dist
COPY --from=builder /build/packages/backend/server/package.json ./
COPY --from=builder /build/packages/backend/server/schema.prisma ./
COPY --from=builder /build/packages/backend/server/scripts ./scripts

# 정적 프론트엔드 파일 복사
COPY --from=builder /build/packages/frontend/apps/web/dist ./static
COPY --from=builder /build/packages/frontend/admin/dist ./static/admin
COPY --from=builder /build/packages/frontend/apps/mobile/dist ./static/mobile

# 빌드 시 설정한 NODE_OPTIONS 제거 (프로덕션에서는 불필요)
ENV NODE_OPTIONS=""

# jemalloc 메모리 할당자 활성화 (성능 향상)
ENV LD_PRELOAD=libjemalloc.so.2

# 포트 노출
EXPOSE 3010

# 헬스체크
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3010/api/healthz', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

CMD ["node", "./dist/main.js"]
