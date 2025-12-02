# 빌드 및 빌드 결과물 정리 가이드

AFFiNE 프로젝트의 빌드 및 빌드 결과물 및 의존성을 정리하는 방법을 설명합니다.

---

## BUILD - 빌드 명령어

### 중요: `yarn build`는 에러가 발생합니다

package.json의 `yarn build`는 **패키지 지정이 필수**입니다. `yarn build`만 실행하면 에러가 발생합니다.

### 올바른 빌드 방법

**특정 패키지 빌드:**

```bash
yarn affine build -p @affine/web
yarn affine build -p web              # alias 사용 가능
```

**의존성 포함하여 빌드:**

```bash
yarn affine build -p @affine/web --deps
```

### 빌드 가능한 주요 패키지

| 패키지명                    | Alias      | 설명                 |
| --------------------------- | ---------- | -------------------- |
| `@affine/web`               | `web`      | 웹 애플리케이션      |
| `@affine/electron`          | `electron` | Electron 데스크톱 앱 |
| `@affine/electron-renderer` | -          | Electron 렌더러      |
| `@affine/mobile`            | `mobile`   | 모바일 웹            |
| `@affine/ios`               | `ios`      | iOS 앱               |
| `@affine/android`           | `android`  | Android 앱           |
| `@affine/server`            | `server`   | 백엔드 서버          |
| `@affine/admin`             | `admin`    | 관리자 페이지        |

### Bundle 명령어

Webpack을 사용한 프로덕션 번들링:

```bash
# 프로덕션 빌드
yarn affine bundle -p @affine/web

# 개발 서버 (webpack-dev-server)
yarn affine bundle -p @affine/web --dev
```

### 일반적인 빌드 시나리오

**웹 애플리케이션 빌드:**

```bash
yarn affine build -p web --deps
```

**Electron 데스크톱 앱 빌드:**

```bash
# 에셋 생성
yarn affine @affine/electron generate-assets

# Electron 빌드
yarn affine build -p electron
```

**백엔드 서버 빌드:**

```bash
# 서버 native 모듈 빌드 (Rust)
yarn affine @affine/server-native build

# 서버 빌드
yarn affine build -p server
```

### 메모리 부족 시 빌드

빌드 중 "JavaScript heap out of memory" 에러 발생 시:

```bash
# 방법 1: 환경 변수로 메모리 증가
NODE_OPTIONS="--max-old-space-size=8192" yarn affine build -p web

# 방법 2: 더 많은 메모리 할당 (16GB)
NODE_OPTIONS="--max-old-space-size=16384" yarn affine build -p web
```

### 빌드 전 정리 권장

빌드 전 이전 결과물을 정리하면 오류를 방지할 수 있습니다:

```bash
# 1. 이전 빌드 결과물 정리
yarn affine clean --dist

# 2. 빌드 실행
yarn affine build -p web --deps
```

---

## Clean 명령어

AFFiNE CLI 도구는 빌드 결과물을 정리하기 위한 `clean` 명령어를 제공합니다.

### 기본 사용법

```bash
yarn affine clean [옵션]
```

## 정리 옵션

### 1. dist 폴더만 정리 (빌드 결과물)

```bash
yarn affine clean --dist
```

- TypeScript 컴파일 결과물 삭제
- Webpack/Vite 번들링 결과물 삭제
- Cargo 빌드 결과물 삭제
- 각 패키지의 `dist` 디렉토리 정리

**사용 시나리오:**

- 빌드 에러 발생 후 재빌드 전
- 메모리 부족으로 빌드 실패 후
- 빌드 캐시 문제 해결

### 2. node_modules 정리

```bash
yarn affine clean --node-modules
```

- 모든 워크스페이스의 `node_modules` 삭제
- 루트 및 하위 패키지의 의존성 제거

**사용 시나리오:**

- 의존성 문제 발생 시
- package.json 수정 후 완전 재설치 필요 시
- 디스크 공간 확보 필요 시

### 3. 전체 정리

```bash
yarn affine clean --all
```

- `--dist`와 `--node-modules` 모두 실행
- 빌드 결과물 + 의존성 모두 삭제
- 프로젝트를 완전히 초기 상태로 복원

**사용 시나리오:**

- 심각한 빌드 문제 발생 시
- 프로젝트 완전 초기화 필요 시
- CI/CD 환경에서 클린 빌드 필요 시

## 일반적인 워크플로우

### 빌드 에러 해결

```bash
# 1. 빌드 결과물만 정리
yarn affine clean --dist

# 2. 다시 빌드
yarn build
```

### 의존성 문제 해결

```bash
# 1. node_modules 정리
yarn affine clean --node-modules

# 2. 의존성 재설치
yarn install

# 3. 빌드
yarn build
```

### 완전 초기화

```bash
# 1. 모든 것 정리
yarn affine clean --all

# 2. 의존성 설치
yarn install

# 3. 초기화
yarn affine init

# 4. 빌드
yarn build
```

## 메모리 부족 문제 해결

JavaScript heap out of memory 에러 발생 시:

```bash
# 1. 기존 빌드 결과물 정리
yarn affine clean --dist

# 2. 메모리 할당 증가하여 빌드
NODE_OPTIONS="--max-old-space-size=8192" yarn build
```

또는 package.json의 build 스크립트에 이미 메모리 설정이 포함되어 있다면:

```bash
yarn affine clean --dist
yarn build
```

## 디스크 공간 확보

프로젝트가 차지하는 공간을 최소화하려면:

```bash
# node_modules 제거 (가장 큰 용량 차지)
yarn affine clean --node-modules

# 또는 전체 정리
yarn affine clean --all
```

## 주의사항

- `--all` 옵션 사용 후에는 반드시 `yarn install`을 실행해야 합니다
- `--node-modules` 정리 후 재설치는 시간이 오래 걸릴 수 있습니다
- 빌드 중 clean을 실행하면 안 됩니다

---

## 관련 명령어

```bash
# CLI 도움말 확인
yarn affine --help

# 특정 명령어 도움말
yarn affine build --help
yarn affine bundle --help
yarn affine clean --help
```
