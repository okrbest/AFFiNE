# AFFiNE 로컬 개발 가이드

## 개발환경 설정

### 사전 요구사항

| 도구 | 버전 | 비고 |
|------|------|------|
| Node.js | 22.16.0 | `.nvmrc` 파일 참조 |
| Yarn | 4.x | corepack 사용 |
| Rust | 1.87.0 | `rust-toolchain.toml` 참조 |
| Docker | 최신 | 백엔드 개발 시 필요 |

### 초기 설정 단계

```bash
# 1. Rust 설치 (rust-toolchain.toml에 의해 자동으로 1.87.0 사용)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 2. Node.js 설정 (nvm 또는 fnm 사용)
nvm use  # 자동으로 22.16.0 버전 사용

# 3. Yarn 활성화
corepack enable

# 4. 의존성 설치
yarn install

# 5. 네이티브 모듈 빌드 (선택사항 - Electron 앱 개발 시 필요)
yarn affine @affine/native build
yarn affine @affine/server-native build
```

## 사전 요구사항 (요약)

- Node.js < 23
- Yarn 4.x
- Rust 툴체인

## 개발 시작

```bash
# 1. 의존성 설치
corepack enable && yarn install

# 2. 웹 앱 개발 서버 시작
yarn dev
```

이렇게 하면 프론트엔드 개발 서버가 시작됩니다.

## 백엔드와 함께 개발 (선택사항)

백엔드 서버가 필요한 경우 (인증, 동기화 등):

```bash
# 1. Docker 실행 후 설정 파일 준비
cp .docker/dev/compose.yml.example .docker/dev/compose.yml
cp .docker/dev/.env.example .docker/dev/.env
docker compose -f .docker/dev/compose.yml up -d

# 2. 서버 환경 설정
cp packages/backend/server/.env.example packages/backend/server/.env
yarn affine server init

# 3. 백엔드 서버 시작 (터미널 1)
yarn affine server dev

# 4. 프론트엔드 시작 (터미널 2)
yarn dev
```

테스트 계정: `dev@affine.pro` / `dev`

## 유용한 명령어

```bash
yarn lint          # 린트 검사
yarn typecheck     # 타입 검사
yarn test          # 테스트 실행
```

## Rust 네이티브 모듈

AFFiNE에서 Rust는 **네이티브 모듈**을 통해 성능이 중요한 기능을 제공합니다.

### 모듈 위치

1. **`packages/frontend/native/`** (`@affine/native`) - 프론트엔드용
2. **`packages/backend/native/`** (`@affine/server-native`) - 서버용

### 주요 역할

Rust는 NAPI-RS를 통해 Node.js와 바인딩되어 다음 기능을 제공합니다:

| 기능 | 설명 |
|------|------|
| **SQLite** | 로컬 데이터 저장 및 쿼리 |
| **파일 시스템** | 빠른 파일 I/O 작업 |
| **암호화** | 데이터 암호화/복호화 |
| **이미지 처리** | 이미지 변환, 압축 |
| **동기화 엔진** | CRDT 기반 실시간 동기화 |

### 왜 Rust인가?

- **성능**: JavaScript보다 훨씬 빠른 연산
- **메모리 안전성**: 메모리 누수 방지
- **로컬 퍼스트**: AFFiNE의 핵심인 로컬 데이터 처리에 적합

### 빌드 명령어

```bash
yarn affine @affine/native build        # 프론트엔드 네이티브 모듈
yarn affine @affine/server-native build # 서버 네이티브 모듈
```

Electron 데스크톱 앱에서 특히 이 네이티브 모듈들이 중요한 역할을 합니다.

## 앱별 Rust 네이티브 모듈 필요 여부

`@affine/native` 모듈은 **모든 앱에서 필요한 것이 아닙니다**.

| 앱 | Rust 필요 | 이유 |
|----|-----------|------|
| **Web** (`apps/web`) | ❌ 불필요 | 브라우저 API (IndexedDB 등) 사용 |
| **Electron** (`apps/electron`) | ✅ 필요 | SQLite, 파일 시스템, 녹화 기능 |
| **iOS** (`apps/ios`) | ✅ 필요 | 네이티브 기능 접근 |
| **Android** (`apps/android`) | ✅ 필요 | 네이티브 기능 접근 |

### 웹 앱만 개발하는 경우

Rust 설치 및 빌드 없이 바로 시작할 수 있습니다:

```bash
corepack enable && yarn install
yarn dev
```

### Electron/Mobile 앱 개발하는 경우

Rust 네이티브 모듈 빌드가 필요합니다:

```bash
# Rust 설치 후
yarn affine @affine/native build
```

### 웹 vs Electron 저장소 차이

- **Web**: IndexedDB, Web Storage API 사용
- **Electron**: SQLite (네이티브), 로컬 파일 시스템 직접 접근

## Rust 개발환경 상세 설정

### Rust 툴체인 요구사항

프로젝트 루트의 `rust-toolchain.toml` 파일에서 Rust 버전이 관리됩니다:

```toml
[toolchain]
channel = "1.87.0"
profile = "default"
```

- **Edition**: 2021 (모든 패키지)
- **Resolver**: version 3

### Workspace 구조

`Cargo.toml` (루트)에서 13개의 crate를 workspace로 관리:

```
packages/
├── backend/native/          # @affine/server-native
├── common/native/           # affine_common (공유 라이브러리)
├── common/y-octo/           # CRDT 엔진
│   ├── core/
│   ├── node/
│   └── utils/
└── frontend/
    ├── native/              # @affine/native
    │   ├── nbstore/         # 문서 저장소
    │   ├── schema/          # 데이터 스키마
    │   └── sqlite_v1/       # SQLite 바인딩
    └── mobile-native/       # 모바일 네이티브
```

### NAPI-RS 설정

Rust와 Node.js를 연결하는 NAPI-RS 설정:

**의존성 버전**:
```toml
napi = "3.0.0-beta.3"
napi-build = "2"
napi-derive = "3.0.0-beta.3"
@napi-rs/cli = "3.0.0-alpha.89"
```

**지원 플랫폼** (6개):
- `x86_64-apple-darwin` (Intel macOS)
- `aarch64-apple-darwin` (Apple Silicon macOS)
- `x86_64-unknown-linux-gnu` (Intel Linux)
- `aarch64-unknown-linux-gnu` (ARM Linux)
- `x86_64-pc-windows-msvc` (Intel Windows)
- `aarch64-pc-windows-msvc` (ARM Windows)

### Frontend Native 상세 (`@affine/native`)

**경로**: `packages/frontend/native/`

**package.json 설정**:
```json
{
  "name": "@affine/native",
  "napi": {
    "binaryName": "affine",
    "targets": ["6개 플랫폼"],
    "constEnum": false
  }
}
```

**주요 모듈**:

| 모듈 | 경로 | 설명 |
|------|------|------|
| `affine_nbstore` | `native/nbstore/` | SQLite 기반 문서 저장소 |
| `affine_sqlite_v1` | `native/sqlite_v1/` | SQLite 바인딩 |
| `affine_schema` | `native/schema/` | 데이터 스키마 정의 |
| `affine_media_capture` | `native/media_capture/` | 스크린/마이크 캡처 |

**플랫폼별 기능 (media_capture)**:

| 기능 | macOS | Windows | Linux |
|------|-------|---------|-------|
| 스크린 캡처 | ScreenCaptureKit | Windows API | ❌ |
| 마이크 입력 | CoreAudio | WASAPI (cpal) | ❌ |
| 오디오 처리 | symphonia, rubato | symphonia, rubato | ❌ |

### Backend Native 상세 (`@affine/server-native`)

**경로**: `packages/backend/native/`

**주요 기능**:

| 기능 | 크레이트 | 설명 |
|------|----------|------|
| 문서 로딩 | `docx-parser`, `pdf-extract` | PDF, DOCX, HTML 파싱 |
| 파일 타입 감지 | `infer`, `file-format` | MIME 타입 감지 |
| AI 토큰 카운팅 | `tiktoken-rs` | OpenAI 토큰 계산 |
| CRDT 처리 | `y-octo` | YDOC 지원 |
| 메모리 최적화 | `mimalloc` | 고성능 메모리 할당 |

### Common Native (`affine_common`)

**경로**: `packages/common/native/`

**Feature Flags**:
```toml
[features]
default = ["hashcash"]
doc-loader = ["docx-parser", "infer", "pdf-extract", ...]
hashcash = ["sha3", "rand"]
tree-sitter = [...]  # 코드 분석
ydoc-loader = ["y-octo"]
```

**코드 분석 지원 언어** (tree-sitter):
C, C++, Go, Java, JavaScript, Python, Rust, TypeScript

### 빌드 최적화 설정

`Cargo.toml` 릴리스 프로필:
```toml
[profile.release]
codegen-units = 1    # LTO 최적화를 위해 단일 유닛
lto = true           # Link Time Optimization
opt-level = 3        # 최대 최적화
strip = "symbols"    # 디버그 심볼 제거

[profile.release.package.affine_mobile_native]
strip = "none"       # Android uniffi bindgen은 심볼 필요
```

### Rust 빌드 명령어

```bash
# Frontend 네이티브 모듈 (릴리스)
yarn affine @affine/native build

# Frontend 네이티브 모듈 (디버그)
yarn affine @affine/native build:debug

# Backend 네이티브 모듈
yarn affine @affine/server-native build
```

### 플랫폼별 주의사항

**macOS**:
- `strip` 명령은 시스템 내장 버전 사용 권장 (binutils보다)

**Windows**:
- 심볼릭 링크 지원을 위해 개발자 모드 활성화 필요

**Linux**:
- ARM64에서는 `mimalloc` 대신 기본 할당자 사용

## Corepack 이해하기

### Corepack이란?

**Corepack**은 Node.js에 내장된 패키지 매니저 관리 도구입니다. 프로젝트별로 **올바른 버전의 패키지 매니저**(Yarn, pnpm)를 자동으로 설치하고 사용하게 해줍니다.

### 왜 필요한가?

```
# 문제 상황
개발자 A: Yarn 3.x 사용
개발자 B: Yarn 1.x 사용
→ lock 파일 충돌, 의존성 문제 발생
```

Corepack은 `package.json`의 `packageManager` 필드를 읽어 **모든 개발자가 동일한 버전**을 사용하도록 강제합니다.

### 작동 방식

1. **프로젝트 설정** (`package.json`):
```json
{
  "packageManager": "yarn@4.5.1"
}
```

2. **Corepack 활성화**:
```bash
corepack enable
```

3. **자동 버전 관리**:
```bash
yarn install  # 자동으로 Yarn 4.5.1 사용
```

### 주요 명령어

```bash
# Corepack 활성화 (필수)
corepack enable

# 특정 버전 준비 (선택)
corepack prepare yarn@4.5.1 --activate

# Corepack 비활성화
corepack disable
```

### AFFiNE에서의 사용

AFFiNE은 Yarn 4.x를 사용하므로:

```bash
corepack enable      # Corepack 활성화
yarn install         # 자동으로 올바른 Yarn 버전 사용
```

### npm과의 관계

| 패키지 매니저 | Corepack 관리 |
|--------------|---------------|
| npm | ❌ (Node.js에 기본 포함) |
| Yarn | ✅ |
| pnpm | ✅ |

### 장점

- **버전 일관성**: 팀 전체가 동일한 패키지 매니저 버전 사용
- **자동 설치**: 필요한 버전이 없으면 자동 다운로드
- **zero-install 지원**: Yarn의 PnP(Plug'n'Play) 기능과 호환
- **별도 설치 불필요**: Node.js 16.10+ 에 기본 포함
