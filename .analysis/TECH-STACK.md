# AFFiNE 기술 스택 분석

## 개요

AFFiNE은 문서, 화이트보드, 데이터베이스를 결합한 로컬 퍼스트 워크스페이스입니다. 이 문서는 프로젝트를 분석하고 수정 개발하기 위한 기술 스택을 정리합니다.

## 핵심 기술 버전 요약

| 기술 | 버전 | 용도 |
|------|------|------|
| React | 19.1.0 | UI 프레임워크 |
| TypeScript | 5.7.2 | 프로그래밍 언어 |
| NestJS | 11.0.12 | 백엔드 프레임워크 |
| Vite | 7.0.0 | 빌드 도구 |
| Electron | 36.0.0 | 데스크톱 앱 |
| Rust | 1.87.0 | 네이티브 모듈 |
| Yjs | 13.6.21 | CRDT/실시간 동기화 |
| Prisma | 6.6.0 | ORM |
| PostgreSQL | 16 | 데이터베이스 |
| Redis | latest | 캐시/세션 |

---

## 1. 프론트엔드 기술

### 프레임워크 & 코어

| 기술 | 버전 | 설명 |
|------|------|------|
| React | 19.1.0 | 최신 React 버전 |
| React DOM | 19.1.0 | DOM 렌더링 |
| React Router DOM | 6.28.0 | 클라이언트 라우팅 |

### 상태 관리

| 기술 | 버전 | 설명 |
|------|------|------|
| Jotai | 2.10.3 | 원자 기반 상태 관리 |
| Jotai-Effect | 2.0.0 | 사이드 이펙트 관리 |
| Jotai-Scope | 0.7.2 | 스코프 관리 |
| Preact/Signals-Core | 1.8.0 | 세밀한 반응성 시그널 |
| RxJS | 7.8.1 | 반응형 프로그래밍 |

### 스타일링

| 기술 | 버전 | 설명 |
|------|------|------|
| Vanilla Extract | 1.17.0 | CSS-in-JS (제로 런타임) |
| @vanilla-extract/vite-plugin | 5.0.0 | Vite 통합 |
| @vanilla-extract/dynamic | 2.1.2 | 동적 스타일링 |
| Emotion | 11.14.0 | CSS-in-JS |
| next-themes | 0.4.4 | 테마 전환 |

### UI 컴포넌트

| 기술 | 버전 | 설명 |
|------|------|------|
| Radix UI | - | 헤드리스 UI 컴포넌트 |
| Lit | 3.2.1 | 웹 컴포넌트 |
| Storybook | 9.0.0 | UI 컴포넌트 문서화 |

**Radix UI 컴포넌트 목록**:
- react-collapsible, react-context-menu, react-dialog
- react-popover, react-scroll-area, react-toolbar
- react-dropdown-menu, react-progress, react-slider
- react-tabs, react-toast, react-tooltip

### 빌드 도구

| 기술 | 버전 | 설명 |
|------|------|------|
| Vite | 7.0.0 | 빌드 도구 & 개발 서버 |
| esbuild | 0.25.0 | 번들러 |
| SWC | - | Rust 기반 JS 컴파일러 |
| @vitejs/plugin-react-swc | 3.7.2 | SWC React 플러그인 |

### 데이터 & 문서 처리

| 기술 | 버전 | 설명 |
|------|------|------|
| Yjs | 13.6.21 | CRDT 구현 (실시간 협업) |
| y-protocols | 1.0.6 | Yjs 프로토콜 |
| lib0 | 0.2.99 | Yjs 유틸리티 |
| GraphQL | 16.9.0 | GraphQL 클라이언트 |
| SWR | 2.3.3 | 데이터 페칭 & 캐싱 |

### 추가 프론트엔드 라이브러리

| 기술 | 버전 | 용도 |
|------|------|------|
| Shiki | 3.7.0 | 문법 하이라이팅 |
| KaTeX | 0.16.11 | LaTeX 렌더링 |
| Mermaid | 11.1.0 | 다이어그램 생성 |
| Fuse.js | 7.0.0 | 퍼지 검색 |
| Lodash-es | 4.17.21 | 유틸리티 |
| Nanoid | 5.0.9 | 고유 ID 생성 |
| Zod | 3.24.1 | 스키마 검증 |
| Lottie | 2.4.0 | 애니메이션 |
| Socket.io-client | 4.8.1 | 실시간 통신 |
| idb | 8.0.0 | IndexedDB 래퍼 |

---

## 2. 백엔드 기술

### 서버 프레임워크

| 기술 | 버전 | 설명 |
|------|------|------|
| NestJS | 11.0.12 | TypeScript 우선 Node.js 프레임워크 |
| @nestjs/common | 11.0.12 | 코어 데코레이터 & 파이프 |
| @nestjs/core | 11.0.12 | 코어 프레임워크 |
| @nestjs/platform-express | 11.0.12 | Express 통합 |
| @nestjs/platform-socket.io | 11.0.12 | WebSocket 지원 |
| @nestjs/schedule | 6.0.0 | 태스크 스케줄링 |
| @nestjs/throttler | 6.4.0 | 레이트 리미팅 |
| @nestjs/swagger | 11.2.0 | API 문서화 |
| Express | 5.0.1 | HTTP 서버 |

### GraphQL

| 기술 | 버전 | 설명 |
|------|------|------|
| Apollo Server | 4.11.3 | GraphQL 서버 |
| @nestjs/apollo | 13.0.4 | NestJS Apollo 통합 |
| @nestjs/graphql | 13.0.4 | GraphQL 지원 |
| GraphQL | 16.9.0 | GraphQL 코어 |
| GraphQL Scalars | 1.24.0 | 커스텀 스칼라 타입 |
| graphql-upload | 17.0.0 | 파일 업로드 |

### 데이터베이스

| 기술 | 버전 | 설명 |
|------|------|------|
| PostgreSQL | 16 | 메인 데이터베이스 |
| pgvector | - | 벡터 검색 확장 |
| Prisma | 6.6.0 | ORM |
| @prisma/client | 6.6.0 | Prisma 클라이언트 |

### 캐시 & 메시지 큐

| 기술 | 버전 | 설명 |
|------|------|------|
| Redis | latest | 캐시 & 메시지 브로커 |
| ioredis | 5.4.1 | Redis 클라이언트 |
| @socket.io/redis-adapter | 8.3.0 | Socket.io Redis 어댑터 |
| BullMQ | 5.40.2 | 작업 큐 |
| @nestjs/bullmq | 11.0.2 | NestJS BullMQ 통합 |

### 인증 & 보안

| 기술 | 버전 | 설명 |
|------|------|------|
| jsonwebtoken | 9.0.2 | JWT 처리 |
| @node-rs/argon2 | 2.0.2 | 비밀번호 해싱 |
| @marsidev/react-turnstile | 1.1.0 | Turnstile CAPTCHA |

### AI 통합

| 기술 | 버전 | 설명 |
|------|------|------|
| AI SDK (Vercel) | 5.0.81 | AI SDK 코어 |
| @ai-sdk/anthropic | 2.0.38 | Claude 모델 |
| @ai-sdk/openai | 2.0.56 | OpenAI 모델 |
| @ai-sdk/google | 2.0.24 | Google AI |
| @ai-sdk/google-vertex | 3.0.54 | Vertex AI |
| @ai-sdk/perplexity | 2.0.14 | Perplexity |
| @modelcontextprotocol/sdk | 1.16.0 | MCP SDK |

### 외부 서비스

| 기술 | 버전 | 설명 |
|------|------|------|
| @aws-sdk/client-s3 | 3.779.0 | AWS S3 |
| Stripe | 17.4.0 | 결제 처리 |
| @fal-ai/serverless-client | 0.15.0 | ML 추론 |
| Nodemailer | 7.0.0 | 이메일 발송 |
| React Email | 4.0.11 | 이메일 템플릿 |

### 모니터링 & 관측성

| 기술 | 버전 | 설명 |
|------|------|------|
| @opentelemetry/api | 1.9.0 | OpenTelemetry API |
| @opentelemetry/sdk-trace-node | 1.30.1 | 트레이싱 |
| @opentelemetry/sdk-metrics | 1.30.1 | 메트릭 |
| @opentelemetry/exporter-prometheus | 0.57.2 | Prometheus 내보내기 |
| @sentry/react | - | 에러 트래킹 |
| Mixpanel | 0.18.0 | 분석 |

---

## 3. 에디터/BlockSuite 기술

### 에디터 프레임워크

| 기술 | 버전 | 설명 |
|------|------|------|
| @blocksuite/store | 0.25.5 | CRDT 기반 상태 관리 |
| @blocksuite/std | 0.25.5 | 표준 라이브러리 |
| @blocksuite/global | 0.25.5 | 전역 유틸리티 |
| @blocksuite/sync | 0.25.5 | 동기화 |

BlockSuite는 toeverything/blocksuite에서 git subtree로 관리됩니다.

### CRDT 구현

```
Yjs (JavaScript)
    ↓
y-protocols (프로토콜 정의)
    ↓
y-octo (Rust CRDT 엔진)
```

### 블록 타입 (25개 이상)

| 카테고리 | 블록 타입 |
|----------|----------|
| 텍스트 | Paragraph, List, Code, Callout, Divider |
| 미디어 | Image, Embed, Bookmark, Attachment |
| 데이터 | Table, Database, Data View |
| 캔버스 | Frame, Surface (Edgeless), Note |
| 고급 | LaTeX, Reference, Footnote, Comment |

### 그래픽 & 렌더링

- **Turbo Renderer**: 커스텀 성능 최적화 렌더러
- **Shape System**: Brush, connector, group, link, mindmap
- **Canvas**: 확대/축소, 변환, 뷰포트 관리

---

## 4. 데스크톱/모바일 기술

### Electron (데스크톱)

| 기술 | 버전 | 설명 |
|------|------|------|
| Electron | 36.0.0 | 최신 버전 |
| Electron Forge | 7.10.2 | 빌드 시스템 |
| Electron Updater | 6.6.2 | 자동 업데이트 |
| Electron Window State | 5.0.3 | 창 상태 유지 |
| Electron Log | 5.4.3 | 로깅 |

**빌드 타겟**:
- macOS: DMG
- Windows: Squirrel
- Linux: DEB, Flatpak, AppImage

### 네이티브 바인딩 (NAPI-RS)

| 모듈 | 경로 | 용도 |
|------|------|------|
| @affine/native | packages/frontend/native/ | 파일 시스템, SQLite |
| @affine/server-native | packages/backend/native/ | 서버 네이티브 기능 |

**지원 플랫폼**:
- x86_64-apple-darwin (Intel Mac)
- aarch64-apple-darwin (Apple Silicon)
- x86_64-unknown-linux-gnu
- aarch64-unknown-linux-gnu
- x86_64-pc-windows-msvc
- aarch64-pc-windows-msvc

### Capacitor (모바일)

| 기술 | 버전 | 설명 |
|------|------|------|
| Capacitor | 7.0.0 | 크로스 플랫폼 모바일 |
| Capacitor CLI | 7.0.0 | CLI 도구 |

---

## 5. 개발 도구

### 패키지 매니저 & 모노레포

| 도구 | 버전 | 설명 |
|------|------|------|
| Yarn | 4.9.1 | 워크스페이스 패키지 매니저 |
| @affine-tools/cli | - | 커스텀 CLI (build, dev, bundle) |

**모노레포 구조**: 80개 이상의 패키지

### TypeScript

| 설정 | 값 |
|------|-----|
| 버전 | 5.7.2 |
| Target | ES2024 |
| Module Resolution | Bundler |
| Strict Mode | Enabled |

### 린팅 & 코드 품질

| 도구 | 버전 | 설명 |
|------|------|------|
| ESLint | 9.16.0 | 정적 분석 |
| typescript-eslint | 8.18.0 | TS 지원 |
| Oxlint | 1.18.0 | Rust 기반 빠른 린터 |
| Prettier | 3.4.2 | 코드 포맷터 |
| lint-staged | 16.0.0 | pre-commit 훅 |
| Husky | 9.1.7 | Git 훅 관리 |

**ESLint 플러그인**:
- eslint-plugin-react
- eslint-plugin-react-hooks
- eslint-plugin-simple-import-sort
- eslint-plugin-sonarjs
- eslint-plugin-unicorn
- eslint-plugin-import-x

### 테스팅

| 도구 | 버전 | 용도 |
|------|------|------|
| Vitest | 3.1.3 | 유닛 테스트 |
| @vitest/ui | 3.1.3 | 테스트 UI |
| @vitest/coverage-istanbul | 3.1.3 | 커버리지 |
| Playwright | 1.52.0 | E2E 테스트 |
| @testing-library/react | 16.1.0 | React 컴포넌트 테스트 |
| MSW | 2.6.8 | API 모킹 |
| AVA | 6.2.0 | 서버 테스트 |
| Supertest | 7.0.0 | HTTP 테스트 |
| Sinon | 21.0.0 | 테스트 스텁/목 |

**E2E 테스트 스위트**:
- affine-local
- affine-cloud
- affine-desktop
- affine-cloud-copilot
- affine-mobile
- blocksuite

---

## 6. 인프라

### CI/CD (GitHub Actions)

| 워크플로우 | 용도 |
|-----------|------|
| build-test.yml | PR 빌드 & 테스트 |
| build-images.yml | Docker 이미지 빌드 |
| release.yml | 릴리스 자동화 |
| release-desktop.yml | 데스크톱 릴리스 |
| release-mobile.yml | 모바일 릴리스 |
| sync-i18n.yml | 다국어 동기화 |
| windows-signer.yml | Windows 코드 서명 |

### Docker

| 서비스 | 이미지 | 용도 |
|--------|--------|------|
| AFFiNE | ghcr.io/toeverything/affine | 메인 애플리케이션 |
| PostgreSQL | pgvector/pgvector:pg16 | 데이터베이스 |
| Redis | redis:latest | 캐시 |
| Mailpit | axllent/mailpit | 이메일 테스트 |
| Manticoresearch | manticoresearch/manticore | 검색 엔진 |

### 국제화 (i18n)

| 기술 | 설명 |
|------|------|
| @affine/i18n | 커스텀 i18n 패키지 |
| @magic-works/i18n-codegen | 번역 코드 생성 |

---

## 7. 아키텍처 패턴

### 의존성 주입

```
@toeverything/infra
    ↓
Service Pattern (상태, 사이드 이펙트 관리)
    ↓
Jotai Atoms (반응형 상태)
```

### 실시간 협업

```
Yjs (CRDT)
    ↓
y-protocols (동기화 프로토콜)
    ↓
Socket.io (양방향 통신)
```

### 데이터 동기화 아키텍처

```
@affine/nbstore (스토리지 추상화)
    ├── IndexedDB (브라우저)
    ├── SQLite (Electron)
    └── Cloud Sync (서버)
```

### 블록 시스템

```
BlockSuite
    ├── Extensible Block Model
    ├── Block Hierarchy (부모-자식)
    └── Block Events (변경 전파)
```

---

## 8. 개발 환경 설정 요약

### 필수 도구

```bash
# Node.js (22.x)
nvm use

# Yarn (4.x)
corepack enable

# Rust (1.87.0) - 네이티브 모듈 개발 시
rustup default 1.87.0

# Docker - 백엔드 개발 시
docker --version
```

### 개발 시작

```bash
# 의존성 설치
yarn install

# 웹 개발 서버
yarn dev

# 린트 검사
yarn lint

# 타입 검사
yarn typecheck

# 테스트
yarn test
```

### 유용한 CLI 명령어

```bash
# 특정 패키지 개발
yarn affine dev -p @affine/web
yarn affine dev -p @affine/electron

# 빌드
yarn affine build -p @affine/web
yarn affine @affine/native build

# 서버 개발
yarn affine server dev
yarn affine server init
```

---

## 9. 패키지 구조

```
packages/
├── frontend/
│   ├── apps/
│   │   ├── web/              # 웹 앱 (@affine/web)
│   │   ├── electron/         # Electron 메인 (@affine/electron)
│   │   ├── mobile/           # 모바일 웹 (@affine/mobile)
│   │   ├── ios/              # iOS 앱
│   │   └── android/          # Android 앱
│   ├── core/                 # 코어 로직 (@affine/core)
│   ├── component/            # UI 컴포넌트 (@affine/component)
│   ├── native/               # Rust 네이티브 (@affine/native)
│   └── i18n/                 # 다국어 (@affine/i18n)
├── backend/
│   ├── server/               # NestJS 서버 (@affine/server)
│   └── native/               # 서버 네이티브 (@affine/server-native)
├── common/
│   ├── infra/                # DI 프레임워크 (@toeverything/infra)
│   ├── nbstore/              # 문서 저장소 (@affine/nbstore)
│   ├── graphql/              # GraphQL 클라이언트 (@affine/graphql)
│   └── env/                  # 환경 설정 (@affine/env)
└── blocksuite/               # 에디터 프레임워크 (git subtree)
    ├── framework/            # 코어 에디터
    └── affine/               # AFFiNE 블록
```

---

## 10. 학습 리소스

### 프레임워크 문서
- [React 19](https://react.dev/)
- [NestJS](https://docs.nestjs.com/)
- [Vite](https://vite.dev/)
- [Electron](https://www.electronjs.org/docs)

### 상태 관리
- [Jotai](https://jotai.org/)
- [RxJS](https://rxjs.dev/)

### CRDT & 협업
- [Yjs](https://docs.yjs.dev/)
- [y-protocols](https://github.com/yjs/y-protocols)

### 데이터베이스
- [Prisma](https://www.prisma.io/docs)
- [PostgreSQL](https://www.postgresql.org/docs/)

### 테스팅
- [Vitest](https://vitest.dev/)
- [Playwright](https://playwright.dev/)
