FROM node:22-bookworm-slim

# Install system dependencies
RUN apt-get update && \
  apt-get install -y --no-install-recommends openssl libjemalloc2 && \
  rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy package files for dependency installation
COPY ./packages/backend/server/package.json ./packages/backend/server/yarn.lock* ./
COPY ./packages/backend/server/prisma ./prisma

# Install dependencies including prisma CLI
RUN corepack enable && \
  yarn install --production --frozen-lockfile && \
  yarn cache clean

# Generate Prisma Client
RUN yarn prisma generate

# Copy the rest of the application
COPY ./packages/backend/server /app
COPY ./packages/frontend/apps/web/dist /app/static
COPY ./packages/frontend/admin/dist /app/static/admin
COPY ./packages/frontend/apps/mobile/dist /app/static/mobile

# Enable jemalloc by preloading the library
ENV LD_PRELOAD=libjemalloc.so.2

CMD ["node", "./dist/main.js"]
