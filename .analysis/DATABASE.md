# AFFiNE 데이터베이스 스키마 분석

이 문서는 AFFiNE 백엔드 서버의 PostgreSQL 데이터베이스 스키마를 분석한 내용입니다.

## 개요

- **데이터베이스**: PostgreSQL
- **ORM**: Prisma
- **확장**: pgvector (벡터 검색용)
- **스키마 위치**: `packages/backend/server/schema.prisma`

## 도메인별 테이블 분류

### 1. 사용자 및 인증 (User & Authentication)

#### `users` (User)
사용자 기본 정보를 저장하는 핵심 테이블.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| name | VARCHAR | 사용자 이름 |
| email | VARCHAR | 이메일 (유니크) |
| email_verified | TIMESTAMPTZ | 이메일 인증 일시 |
| avatar_url | VARCHAR | 아바타 URL |
| password | VARCHAR | 비밀번호 해시 (OAuth 사용자는 null) |
| registered | BOOLEAN | 가입 완료 여부 (초대받은 사용자는 false일 수 있음) |
| disabled | BOOLEAN | 계정 비활성화 여부 |
| created_at | TIMESTAMPTZ | 생성 일시 |

#### `user_connected_accounts` (ConnectedAccount)
OAuth 연결 계정 정보 (Google, GitHub 등).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| user_id | VARCHAR | 사용자 FK |
| provider | VARCHAR | OAuth 제공자 (google, github 등) |
| provider_account_id | VARCHAR | 제공자 측 계정 ID |
| access_token | TEXT | 액세스 토큰 |
| refresh_token | TEXT | 리프레시 토큰 |
| expires_at | TIMESTAMPTZ | 토큰 만료 일시 |

#### `multiple_users_sessions` (Session) & `user_sessions` (UserSession)
사용자 세션 관리. 멀티 사용자 세션을 지원.

| 테이블 | 설명 |
|--------|------|
| multiple_users_sessions | 세션 그룹 (여러 사용자가 공유 가능) |
| user_sessions | 개별 사용자-세션 연결 |

#### `verification_tokens` (VerificationToken)
이메일 인증, 비밀번호 재설정 등에 사용되는 일회용 토큰.

#### `access_tokens` (AccessToken)
API 액세스 토큰 (PAT - Personal Access Token).

---

### 2. 워크스페이스 (Workspace)

#### `workspaces` (Workspace)
워크스페이스 기본 정보.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| sid | INT | 시퀀셜 ID (자동 증가) |
| public | BOOLEAN | 공개 여부 |
| name | VARCHAR | 워크스페이스 이름 |
| avatar_key | VARCHAR | 아바타 파일 키 |
| enable_ai | BOOLEAN | AI 기능 활성화 (기본: true) |
| enable_url_preview | BOOLEAN | URL 미리보기 활성화 |
| enable_doc_embedding | BOOLEAN | 문서 임베딩 활성화 |
| indexed | BOOLEAN | 검색 인덱싱 여부 |
| last_check_embeddings | TIMESTAMPTZ | 마지막 임베딩 확인 시간 |

#### `workspace_pages` (WorkspaceDoc)
워크스페이스 내 문서(페이지) 메타데이터.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| workspace_id | VARCHAR | 워크스페이스 FK |
| page_id | VARCHAR | 문서 ID |
| public | BOOLEAN | 문서 공개 여부 |
| default_role | SMALLINT | 기본 권한 (30=Manager) |
| mode | SMALLINT | 모드 (0=Page, 1=Edgeless) |
| blocked | BOOLEAN | 차단 여부 |
| title | VARCHAR | 문서 제목 |
| summary | VARCHAR | 문서 요약 |

---

### 3. 권한 관리 (Permissions)

#### `workspace_user_permissions` (WorkspaceUserRole)
워크스페이스 멤버 권한.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| workspace_id | VARCHAR | 워크스페이스 FK |
| user_id | VARCHAR | 사용자 FK |
| type | SMALLINT | 역할 (Owner/Admin/Collaborator/External) |
| status | ENUM | 멤버 상태 (Pending/UnderReview/Accepted 등) |
| source | ENUM | 초대 방식 (Email/Link) |
| inviter_id | VARCHAR | 초대자 FK |

**WorkspaceMemberStatus 열거형:**
- `Pending`: 초대 수락 대기
- `UnderReview`: 관리자 검토 대기 (링크 초대)
- `AllocatingSeat`: 좌석 할당 중 (팀 워크스페이스)
- `NeedMoreSeat`: 좌석 부족
- `Accepted`: 활성 멤버

#### `workspace_page_user_permissions` (WorkspaceDocUserRole)
문서별 사용자 권한.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| workspace_id | VARCHAR | 워크스페이스 FK |
| page_id | VARCHAR | 문서 ID |
| user_id | VARCHAR | 사용자 FK |
| type | SMALLINT | 권한 (External/Reader/Editor/Manager/Owner) |

---

### 4. 기능 플래그 (Feature Flags)

#### `features` (Feature)
기능 정의 테이블.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT | 기본 키 (자동 증가) |
| feature | VARCHAR | 기능 이름 |
| configs | JSON | 기능 설정 |

#### `user_features` (UserFeature) & `workspace_features` (WorkspaceFeature)
사용자/워크스페이스별 기능 활성화 상태.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| user_id / workspace_id | VARCHAR | 대상 FK |
| feature_id | INT | 기능 FK |
| name | VARCHAR | 기능 이름 (빠른 조회용) |
| reason | VARCHAR | 활성화 사유 |
| activated | BOOLEAN | 활성화 여부 |
| expired_at | TIMESTAMPTZ | 만료 일시 |

---

### 5. 문서 동기화 (Document Sync)

AFFiNE는 CRDT(Yjs) 기반 실시간 동기화를 사용합니다.

#### `snapshots` (Snapshot)
문서의 최신 스냅샷.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| workspace_id | VARCHAR | 워크스페이스 FK |
| guid | VARCHAR | 문서 GUID |
| blob | BYTEA | Yjs 문서 바이너리 |
| state | BYTEA | Yjs 상태 벡터 |
| created_by | VARCHAR | 생성자 FK |
| updated_by | VARCHAR | 최종 수정자 FK |
| updated_at | TIMESTAMPTZ | 마지막 업데이트 병합 시간 |

#### `updates` (Update)
문서 업데이트 큐 (스냅샷에 병합되기 전).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| workspace_id | VARCHAR | 워크스페이스 FK |
| guid | VARCHAR | 문서 GUID |
| blob | BYTEA | Yjs 업데이트 바이너리 |
| created_at | TIMESTAMPTZ | 생성 시간 (복합 키의 일부) |
| created_by | VARCHAR | 생성자 FK |

#### `snapshot_histories` (SnapshotHistory)
문서 버전 히스토리 (시점 복원용).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| workspace_id | VARCHAR | 워크스페이스 FK |
| guid | VARCHAR | 문서 GUID |
| timestamp | TIMESTAMPTZ | 히스토리 시점 (복합 키의 일부) |
| blob | BYTEA | 해당 시점 스냅샷 |
| expired_at | TIMESTAMPTZ | 히스토리 만료 일시 |

#### `user_snapshots` (UserSnapshot)
사용자별 개인 설정 스냅샷 (앱 설정 등).

---

### 6. AI/Copilot

#### `ai_prompts_metadata` (AiPrompt)
AI 프롬프트 정의.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT | 기본 키 |
| name | VARCHAR(32) | 프롬프트 이름 (유니크) |
| action | VARCHAR | 프론트엔드 표시용 액션 |
| model | VARCHAR | 기본 모델 |
| optional_models | VARCHAR[] | 선택 가능한 대체 모델 |
| config | JSON | 프롬프트 설정 |

#### `ai_prompts_messages` (AiPromptMessage)
프롬프트 구성 메시지.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| prompt_id | INT | 프롬프트 FK |
| idx | INT | 메시지 순서 |
| role | ENUM | 역할 (system/assistant/user) |
| content | TEXT | 메시지 내용 |
| attachments | JSON | 첨부 파일 |
| params | JSON | 파라미터 |

#### `ai_sessions_metadata` (AiSession)
AI 대화 세션.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| user_id | VARCHAR | 사용자 FK |
| workspace_id | VARCHAR | 워크스페이스 ID |
| doc_id | VARCHAR | 연결된 문서 ID |
| prompt_name | VARCHAR(32) | 사용 프롬프트 |
| prompt_action | VARCHAR(32) | 액션 타입 |
| pinned | BOOLEAN | 고정 여부 |
| title | VARCHAR | 세션 제목 |
| parent_session_id | VARCHAR | 부모 세션 (포크된 경우) |
| message_cost | INT | 메시지 비용 |
| token_cost | INT | 토큰 비용 |
| deleted_at | TIMESTAMPTZ | 삭제 일시 (소프트 삭제) |

#### `ai_sessions_messages` (AiSessionMessage)
AI 세션 내 메시지.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| session_id | VARCHAR | 세션 FK |
| role | ENUM | 역할 (system/assistant/user) |
| content | TEXT | 메시지 내용 |
| stream_objects | JSON | 스트리밍 객체 |
| attachments | JSON | 첨부 파일 |

---

### 7. 벡터 임베딩 (AI Embeddings)

pgvector 확장을 사용한 벡터 검색 테이블들.

#### `ai_workspace_embeddings` (AiWorkspaceEmbedding)
워크스페이스 문서 임베딩.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| workspace_id | VARCHAR | 워크스페이스 FK |
| doc_id | VARCHAR | 문서 ID |
| chunk | INT | 청크 번호 |
| content | VARCHAR | 청크 텍스트 |
| embedding | vector(1024) | 임베딩 벡터 |

#### `ai_context_embeddings` (AiContextEmbedding)
AI 컨텍스트 파일 임베딩.

#### `ai_workspace_file_embeddings` (AiWorkspaceFileEmbedding)
워크스페이스 파일 임베딩.

#### `ai_workspace_blob_embeddings` (AiWorkspaceBlobEmbedding)
워크스페이스 Blob 임베딩.

#### `ai_workspace_files` (AiWorkspaceFiles)
AI에서 사용하는 워크스페이스 파일 메타데이터.

#### `ai_workspace_ignored_docs` (AiWorkspaceIgnoredDocs)
AI 임베딩에서 제외할 문서.

#### `ai_jobs` (AiJobs)
AI 비동기 작업 (음성 전사 등).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| workspace_id | VARCHAR | 워크스페이스 ID |
| blob_id | VARCHAR | Blob ID |
| type | ENUM | 작업 유형 (transcription) |
| status | ENUM | 상태 (pending/running/finished/claimed/failed) |
| payload | JSON | 작업 결과 |

---

### 8. 구독 및 결제 (Subscription & Billing)

#### `subscriptions` (Subscription)
구독 정보 (Stripe, RevenueCat 지원).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | INT | 기본 키 |
| target_id | VARCHAR | 대상 ID (사용자 또는 워크스페이스) |
| plan | VARCHAR(20) | 플랜 (Pro, Team 등) |
| recurring | VARCHAR(20) | 주기 (yearly/monthly/lifetime) |
| variant | VARCHAR(20) | 변형 (일회성 등) |
| quantity | INT | 수량 (좌석 수) |
| stripe_subscription_id | VARCHAR | Stripe 구독 ID |
| provider | ENUM | 결제 제공자 (stripe/revenuecat) |
| iap_store | ENUM | IAP 스토어 (app_store/play_store) |
| status | VARCHAR(20) | 상태 (active/past_due/canceled 등) |
| start | TIMESTAMPTZ | 시작일 |
| end | TIMESTAMPTZ | 종료일 |
| trial_start | TIMESTAMPTZ | 체험 시작일 |
| trial_end | TIMESTAMPTZ | 체험 종료일 |

#### `invoices` (Invoice)
청구서 정보.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| stripe_invoice_id | VARCHAR | Stripe 청구서 ID (기본 키) |
| target_id | VARCHAR | 대상 ID |
| currency | VARCHAR(3) | 통화 코드 |
| amount | INT | 금액 (센트 단위) |
| status | VARCHAR(20) | 상태 |
| link | TEXT | Stripe 호스팅 청구서 링크 |

#### `user_stripe_customers` (UserStripeCustomer)
사용자-Stripe 고객 ID 매핑.

#### `licenses` (License) & `installed_licenses` (InstalledLicense)
셀프 호스팅용 라이선스 관리.

---

### 9. 스토리지 (Storage)

#### `blobs` (Blob)
파일 Blob 메타데이터 (실제 데이터는 S3 등 외부 스토리지).

| 컬럼 | 타입 | 설명 |
|------|------|------|
| workspace_id | VARCHAR | 워크스페이스 FK |
| key | VARCHAR | Blob 키 |
| size | INT | 파일 크기 (바이트) |
| mime | VARCHAR | MIME 타입 |
| deleted_at | TIMESTAMPTZ | 삭제 일시 (소프트 삭제) |

---

### 10. 댓글 (Comments)

#### `comments` (Comment)
문서 댓글.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| sid | INT | 시퀀셜 ID |
| workspace_id | VARCHAR | 워크스페이스 FK |
| doc_id | VARCHAR | 문서 ID |
| user_id | VARCHAR | 작성자 FK |
| content | JSONB | 댓글 내용 (리치 텍스트) |
| resolved | BOOLEAN | 해결 여부 |
| deleted_at | TIMESTAMPTZ | 삭제 일시 |

#### `replies` (Reply)
댓글 답글.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| comment_id | VARCHAR | 댓글 FK |
| user_id | VARCHAR | 작성자 FK |
| content | JSONB | 답글 내용 |

#### `comment_attachments` (CommentAttachment)
댓글 첨부 파일.

---

### 11. 알림 (Notifications)

#### `notifications` (Notification)
사용자 알림.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| user_id | VARCHAR | 사용자 FK |
| type | ENUM | 알림 유형 |
| level | ENUM | 알림 레벨 (High/Default/Low/Min/None) |
| read | BOOLEAN | 읽음 여부 |
| body | JSONB | 알림 내용 |

**NotificationType 열거형:**
- `Mention`: 멘션
- `Invitation`: 초대
- `InvitationAccepted`: 초대 수락됨
- `InvitationBlocked`: 초대 차단됨
- `InvitationRejected`: 초대 거절됨
- `InvitationReviewRequest`: 초대 검토 요청
- `InvitationReviewApproved`: 초대 검토 승인
- `InvitationReviewDeclined`: 초대 검토 거절
- `Comment`: 댓글
- `CommentMention`: 댓글 멘션

---

### 12. 설정 (Settings)

#### `app_configs` (AppConfig)
애플리케이션 전역 설정.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR | 설정 키 (기본 키) |
| value | JSONB | 설정 값 |
| last_updated_by | VARCHAR | 마지막 수정자 FK |

#### `user_settings` (UserSettings)
사용자별 설정.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| user_id | VARCHAR | 사용자 FK (기본 키) |
| payload | JSONB | 설정 데이터 |

---

### 13. 시스템 (System)

#### `_data_migrations` (DataMigration)
데이터 마이그레이션 실행 이력.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | VARCHAR (UUID) | 기본 키 |
| name | VARCHAR | 마이그레이션 이름 (유니크) |
| started_at | TIMESTAMPTZ | 시작 시간 |
| finished_at | TIMESTAMPTZ | 완료 시간 |

---

## 주요 관계도

```
User
├── ConnectedAccount (1:N) - OAuth 연결
├── UserSession (1:N) - 세션
├── WorkspaceUserRole (1:N) - 워크스페이스 권한
├── WorkspaceDocUserRole (1:N) - 문서 권한
├── UserFeature (1:N) - 기능 플래그
├── AiSession (1:N) - AI 세션
├── Notification (1:N) - 알림
├── Comment (1:N) - 댓글
├── Reply (1:N) - 답글
└── UserSettings (1:1) - 설정

Workspace
├── WorkspaceDoc (1:N) - 문서 메타데이터
├── WorkspaceUserRole (1:N) - 멤버 권한
├── WorkspaceFeature (1:N) - 기능 플래그
├── Blob (1:N) - 파일
├── Comment (1:N) - 댓글
└── AiWorkspaceFiles (1:N) - AI 파일

Snapshot (문서 데이터)
├── Update (1:N) - 대기 중인 업데이트
├── SnapshotHistory (1:N) - 버전 히스토리
└── AiWorkspaceEmbedding (1:N) - 임베딩

AiSession
├── AiSessionMessage (1:N) - 메시지
└── AiContext (1:N) - 컨텍스트
    └── AiContextEmbedding (1:N) - 임베딩
```

---

## 참고 사항

1. **CRDT 기반 동기화**: `snapshots`와 `updates` 테이블은 Yjs CRDT 문서를 저장하며, 업데이트가 주기적으로 스냅샷에 병합됩니다.

2. **소프트 삭제**: 많은 테이블에서 `deleted_at` 컬럼을 사용한 소프트 삭제 패턴을 사용합니다.

3. **벡터 검색**: `pgvector` 확장을 사용하여 1024차원 벡터 임베딩을 저장하고 유사도 검색을 수행합니다.

4. **멀티 테넌시**: 대부분의 데이터가 `workspace_id`로 분리되어 있습니다.

5. **Deprecated 테이블**: `DeprecatedUserSubscription`, `DeprecatedUserInvoice`, `DeprecatedAppRuntimeSettings` 등은 이전 버전 호환성을 위해 유지되고 있습니다.
