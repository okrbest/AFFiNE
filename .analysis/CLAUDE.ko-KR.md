# CLAUDE.md

이 파일은 Claude Code (claude.ai/code)가 이 저장소의 코드를 다룰 때 참고할 수 있는 가이드입니다.

## 개요

AFFiNE는 문서, 화이트보드, 데이터베이스를 결합한 오픈소스 로컬 우선(local-first) 워크스페이스입니다. TypeScript/React 프론트엔드, NestJS 백엔드, Rust 네이티브 모듈, Electron 데스크톱 앱으로 구성되어 있습니다.

## 주요 명령어

### 개발

```bash
# 의존성 설치 (Node.js <23, Yarn 4.x, Rust 툴체인 필요)
corepack enable && yarn install

# 웹 앱 개발 서버 시작
yarn dev                    # 또는: yarn affine dev -p @affine/web

# 백엔드 서버와 함께 시작 (postgres/redis용 Docker 필요)
yarn affine server dev      # 백엔드 서버
yarn dev                    # 프론트엔드 (별도 터미널)

# 특정 앱 개발
yarn affine dev -p @affine/web              # 웹 앱
yarn affine dev -p @affine/electron         # 데스크톱 앱
yarn affine dev -p @affine/mobile           # 모바일 웹
yarn affine dev -p @affine/ios              # iOS 앱
yarn affine dev -p @affine/android          # Android 앱
```

### 빌드

```bash
yarn build                                      # 전체 빌드
yarn affine build -p @affine/web               # 웹 앱 빌드
yarn affine @affine/native build               # 프론트엔드 네이티브 모듈 빌드 (Rust)
yarn affine @affine/server-native build        # 서버 네이티브 모듈 빌드 (Rust)
yarn affine @affine/electron build             # Electron 앱 빌드
yarn affine @affine/electron generate-assets   # Electron 에셋 생성
```

### 테스트

```bash
yarn test                                             # 유닛 테스트 실행 (vitest)
yarn test:ui                                          # UI와 함께 테스트 실행
yarn workspace @affine-test/affine-local e2e          # E2E 테스트 (Playwright)
yarn workspace @affine-test/affine-cloud e2e          # 클라우드 E2E 테스트
yarn workspace @affine-test/affine-desktop e2e        # 데스크톱 E2E 테스트
yarn workspace @affine/server test                    # 서버 유닛 테스트 (ava)
```

### 린팅 & 타입 검사

```bash
yarn lint                   # ESLint + Prettier 검사
yarn lint:fix               # 린트 이슈 자동 수정
yarn typecheck              # TypeScript 타입 검사
yarn lint:ox                # oxlint로 빠른 린팅
```

### 서버 개발

```bash
# 최초 설정 (Docker 실행 필요)
cp .docker/dev/compose.yml.example .docker/dev/compose.yml
cp .docker/dev/.env.example .docker/dev/.env
docker compose -f .docker/dev/compose.yml up -d

cp packages/backend/server/.env.example packages/backend/server/.env
yarn affine server init     # 마이그레이션 실행

# 개발
yarn affine server dev      # 서버 시작 (포트 3010)
yarn affine server prisma studio  # 데이터베이스 GUI (포트 5555)

# 테스트 사용자: dev@affine.pro/dev, pro@affine.pro/pro, team@affine.pro/team
```

## 아키텍처

### 모노레포 구조

- **`packages/frontend/`** - 프론트엔드 애플리케이션 및 라이브러리
  - `apps/web/` - 웹 애플리케이션 (`@affine/web`)
  - `apps/electron/` - Electron 메인 프로세스 (`@affine/electron`)
  - `apps/electron-renderer/` - Electron 렌더러 (`@affine/electron-renderer`)
  - `apps/mobile/` - 모바일 웹 앱 (`@affine/mobile`)
  - `apps/ios/`, `apps/android/` - 네이티브 모바일 앱 (Capacitor)
  - `core/` - 메인 애플리케이션 로직 (`@affine/core`) - React 컴포넌트, 상태 관리
  - `component/` - UI 컴포넌트 라이브러리 (`@affine/component`)
  - `native/` - NAPI-RS를 통한 Rust 네이티브 바인딩 (`@affine/native`) - SQLite, 파일 시스템
  - `i18n/` - 국제화 (`@affine/i18n`)

- **`packages/backend/`** - 백엔드 서비스
  - `server/` - NestJS 백엔드 (`@affine/server`) - GraphQL API, 인증, 동기화, AI 코파일럿
  - `native/` - Rust 네이티브 바인딩 (`@affine/server-native`)

- **`packages/common/`** - 공유 동형(isomorphic) 코드
  - `infra/` - 핵심 인프라 (`@toeverything/infra`) - DI 프레임워크, 서비스, 상태
  - `nbstore/` - 문서 스토리지 추상화 (`@affine/nbstore`)
  - `graphql/` - GraphQL 클라이언트/타입 (`@affine/graphql`)
  - `env/` - 환경 설정 (`@affine/env`)

- **`blocksuite/`** - 에디터 프레임워크 (toeverything/blocksuite의 git subtree)
  - `framework/` - 핵심 에디터 프레임워크 (`@blocksuite/store`, `@blocksuite/std`)
  - `affine/` - AFFiNE 전용 블록 및 컴포넌트

- **`tools/`** - 개발 도구
  - `cli/` - 모노레포 CLI (`@affine-tools/cli`) - build, dev, bundle 명령어

- **`tests/`** - E2E 및 통합 테스트 (Playwright)

### 주요 기술

- **프론트엔드**: React 19, Jotai (상태 관리), Yjs (CRDT), vanilla-extract (CSS), Radix UI
- **백엔드**: NestJS, Prisma (PostgreSQL), Redis, Socket.io, GraphQL
- **에디터**: BlockSuite (Lit 기반 커스텀 블록 에디터)
- **네이티브**: Rust와 NAPI-RS를 통한 Node.js 바인딩
- **데스크톱**: Electron과 커스텀 네이티브 모듈 통합

### CLI 도구

`yarn affine` 명령어 (`yarn af`로 별칭 사용 가능)가 제공하는 기능:
- `dev -p <package>` - 개발 서버 시작
- `build -p <package>` - 패키지 빌드
- `bundle -p <package>` - 프로덕션용 번들링
- `<package> <script>` - 패키지 스크립트 실행 (예: `yarn affine server init`)
- `init` - 설치 후 워크스페이스 초기화
- `clean` - 빌드 결과물 정리

### 의존성 아키텍처

`@affine/core`는 비즈니스 로직을 포함하는 메인 애플리케이션 패키지입니다. 다음에 의존합니다:
- `@affine/component` - UI 컴포넌트
- `@toeverything/infra` - 서비스 인프라 및 의존성 주입
- `@blocksuite/*` 패키지들 - 에디터
- `@affine/nbstore` - 문서 스토리지

인프라 레이어 (`@toeverything/infra`)는 애플리케이션 상태와 사이드 이펙트를 관리하기 위한 서비스 패턴의 DI 프레임워크를 제공합니다.
