# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

AFFiNE is an open-source, local-first workspace combining docs, whiteboards, and databases. It consists of a TypeScript/React frontend, NestJS backend, Rust native modules, and an Electron desktop app.

## Common Commands

### Development

```bash
# Install dependencies (requires Node.js <23, Yarn 4.x, Rust toolchain)
corepack enable && yarn install

# Start web app development server
yarn dev                    # or: yarn affine dev -p @affine/web

# Start with backend server (requires Docker for postgres/redis)
yarn affine server dev      # Backend server
yarn dev                    # Frontend (separate terminal)

# Develop specific apps
yarn affine dev -p @affine/web              # Web app
yarn affine dev -p @affine/electron         # Desktop app
yarn affine dev -p @affine/mobile           # Mobile web
yarn affine dev -p @affine/ios              # iOS app
yarn affine dev -p @affine/android          # Android app
```

### Building

```bash
yarn build                                      # Build all
yarn affine build -p @affine/web               # Build web app
yarn affine @affine/native build               # Build frontend native module (Rust)
yarn affine @affine/server-native build        # Build server native module (Rust)
yarn affine @affine/electron build             # Build electron app
yarn affine @affine/electron generate-assets   # Generate electron assets
```

### Testing

```bash
yarn test                                             # Run unit tests (vitest)
yarn test:ui                                          # Run tests with UI
yarn workspace @affine-test/affine-local e2e          # E2E tests (Playwright)
yarn workspace @affine-test/affine-cloud e2e          # Cloud E2E tests
yarn workspace @affine-test/affine-desktop e2e        # Desktop E2E tests
yarn workspace @affine/server test                    # Server unit tests (ava)
```

### Linting & Type Checking

```bash
yarn lint                   # ESLint + Prettier check
yarn lint:fix               # Auto-fix lint issues
yarn typecheck              # TypeScript type checking
yarn lint:ox                # Fast linting with oxlint
```

### Server Development

```bash
# First time setup (requires Docker running)
cp .docker/dev/compose.yml.example .docker/dev/compose.yml
cp .docker/dev/.env.example .docker/dev/.env
docker compose -f .docker/dev/compose.yml up -d

cp packages/backend/server/.env.example packages/backend/server/.env
yarn affine server init     # Run migrations

# Development
yarn affine server dev      # Start server (port 3010)
yarn affine server prisma studio  # Database GUI (port 5555)

# Test users: dev@affine.pro/dev, pro@affine.pro/pro, team@affine.pro/team
```

## Architecture

### Monorepo Structure

- **`packages/frontend/`** - Frontend applications and libraries
  - `apps/web/` - Web application (`@affine/web`)
  - `apps/electron/` - Electron main process (`@affine/electron`)
  - `apps/electron-renderer/` - Electron renderer (`@affine/electron-renderer`)
  - `apps/mobile/` - Mobile web app (`@affine/mobile`)
  - `apps/ios/`, `apps/android/` - Native mobile apps (Capacitor)
  - `core/` - Main application logic (`@affine/core`) - React components, state management
  - `component/` - UI component library (`@affine/component`)
  - `native/` - Rust native bindings via NAPI-RS (`@affine/native`) - SQLite, file system
  - `i18n/` - Internationalization (`@affine/i18n`)

- **`packages/backend/`** - Backend services
  - `server/` - NestJS backend (`@affine/server`) - GraphQL API, auth, sync, AI copilot
  - `native/` - Rust native bindings (`@affine/server-native`)

- **`packages/common/`** - Shared isomorphic code
  - `infra/` - Core infrastructure (`@toeverything/infra`) - DI framework, services, state
  - `nbstore/` - Document storage abstraction (`@affine/nbstore`)
  - `graphql/` - GraphQL client/types (`@affine/graphql`)
  - `env/` - Environment config (`@affine/env`)

- **`blocksuite/`** - Editor framework (git subtree from toeverything/blocksuite)
  - `framework/` - Core editor framework (`@blocksuite/store`, `@blocksuite/std`)
  - `affine/` - AFFiNE-specific blocks and components

- **`tools/`** - Development tooling
  - `cli/` - Monorepo CLI (`@affine-tools/cli`) - build, dev, bundle commands

- **`tests/`** - E2E and integration tests (Playwright)

### Key Technologies

- **Frontend**: React 19, Jotai (state), Yjs (CRDT), vanilla-extract (CSS), Radix UI
- **Backend**: NestJS, Prisma (PostgreSQL), Redis, Socket.io, GraphQL
- **Editor**: BlockSuite (custom block-based editor built on Lit)
- **Native**: Rust with NAPI-RS for Node.js bindings
- **Desktop**: Electron with custom native module integration

### CLI Tool

The `yarn affine` command (aliased as `yarn af`) provides:
- `dev -p <package>` - Start development server
- `build -p <package>` - Build a package
- `bundle -p <package>` - Bundle for production
- `<package> <script>` - Run package script (e.g., `yarn affine server init`)
- `init` - Initialize workspace after install
- `clean` - Clean build artifacts

### Dependency Architecture

`@affine/core` is the main application package containing business logic. It depends on:
- `@affine/component` for UI components
- `@toeverything/infra` for service infrastructure and dependency injection
- `@blocksuite/*` packages for the editor
- `@affine/nbstore` for document storage

The infrastructure layer (`@toeverything/infra`) provides a DI framework with services pattern for managing application state and side effects.
