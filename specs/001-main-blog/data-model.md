# Data Model: 메인블로그 플랫폼

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

기준은 [erd_tables.sql](../../docs/02-database/schema/erd_tables.sql)(MySQL 8, InnoDB, utf8mb4)이고, 같은 내용을 표와 관계도로 정리한 원문은 [요구사항분석서 3 · DB 설계(ERD)](../../docs/02-database/요구사항분석서_3_DB설계_ERD.pdf)입니다(테이블 33개, 관계 65개, D-84, D-99). 이 문서는 테이블을 요구사항과 연결하고 상태 전이와 검증 규칙을 정리합니다. 컬럼 하나하나의 정의와 한글 이름은 SQL 파일과 [erdcloud_names.md](../../docs/02-database/erdcloud/erdcloud_names.md)를 보세요. 조회용 인덱스는 [add_indexes.sql](../../docs/02-database/schema/add_indexes.sql)에 있습니다(나중에 추가).

테이블이나 컬럼을 바꾸면 `erd_tables.sql` → `erdcloud_import.sql` → `erdcloud_names.md` → 이 문서 순서로 같이 고칩니다.

## 공통 규칙

- 모든 테이블의 PK는 `id BIGINT AUTO_INCREMENT`(shedlock만 `name`).
- 시각 컬럼은 `created_at`, 고칠 수 있는 테이블은 `updated_at`(ON UPDATE).
- 지우는 대신 상태나 `deleted_at`을 두고, 04:00 배치가 30일 뒤 완전 삭제한다(4.5).
- 인증번호·토큰·블랙리스트 개인정보는 원문 대신 해시로 저장한다.
- 개수 컬럼(`member_count`, `post_count`, `like_count` 등)은 정렬·표시용 캐시다.

## 엔티티 목록 (33개)

### 회원·인증

| 테이블 | 한글 | 주요 요구사항 | 핵심 컬럼·규칙 |
| --- | --- | --- | --- |
| `users` | 회원 | USR-01~08, SEC-01~03, ADM-08 | `email`(소문자, UNIQUE, 탈퇴 즉시 NULL), `name`(변경 불가), `nickname`(2~12자, UNIQUE), `phone`(숫자만, 1차 중복 허용), `role` USER/ADMIN(ADMIN은 블로그 활동 불가, 메인 공지·관리만, D-90), `status` ACTIVE/WITHDRAWN, `login_fail_count`·`locked_until`(5회 5분 잠금), `notification_keep_days` 30/7, 약관·개인정보 동의 시각, `suspended_until`(메인 관리자 계정 정지, 영구는 9999-12-31, D-99) |
| `verification_codes` | 이메일 인증번호 | USR-02, USR-06 | 회원 번호 대신 `email` 기준. `purpose` SIGNUP(10분)/PASSWORD_RESET(30분), `code_hash`, `fail_count`(5번 무효), `verified_at`, `used_at`(1회용) |
| `account_find_tokens` | 이메일 찾기 임시 토큰 | USR-08 | `token_hash`, 10분 만료 |
| `refresh_tokens` | Refresh Token | SEC-04, USR-04 | `token_hash`, `remember_me`(14일/30분), `user_agent`, `revoked_at`(로그아웃·비밀번호 변경) |

### 블로그

| 테이블 | 한글 | 주요 요구사항 | 핵심 컬럼·규칙 |
| --- | --- | --- | --- |
| `blogs` | 블로그 | BLG-01~03, 09, 10, ADM-02 | `slug`(3~30자, UNIQUE, 폐쇄 30일 뒤 NULL, D-86), `visibility` PUBLIC/LINK_ONLY/PRIVATE, `share_key`(일부 공개 링크 무작위 값), `join_policy` OPEN/APPROVAL, `status` ACTIVE/CLOSING/CLOSED, `is_hidden`, `close_scheduled_at`, `close_reason` OWNER/OWNER_DEMOTED/ADMIN, `closed_at` |
| `blog_members` | 블로그 멤버 | BLG-04~08, 13, SEC-07 | `role` OWNER/MANAGER/MEMBER, `manager_since`(자동 위임 순서), `suspended_until`(영구는 9999-12-31), `suspension_count`(3번이면 표시) |
| `blog_manager_permissions` | 부블로그장 권한 | 2장, D-71 | `permission` EDIT_INFO/MANAGE_MEMBERS/MANAGE_POSTS |
| `blog_join_requests` | 참여 신청 | BLG-04, 05 | `status` PENDING/APPROVED/REJECTED/CANCELED, `handled_at`(거절 7일 뒤 재신청 기준) |
| `blog_owner_transfers` | 블로그장 위임 요청 | BLG-08 | `status` PENDING/ACCEPTED/REJECTED/CANCELED/EXPIRED, `expires_at`(+7일) |
| `blog_subscriptions` | 블로그 구독 | SOC-01 | 회원·블로그 UNIQUE |
| `blog_close_notices` | 폐쇄 알림 발송 기록 | BLG-09 | `close_scheduled_at` + `stage` IMMEDIATE/D3/D1/REVOKED로 중복 발송 방지 |
| `categories` | 카테고리 | BRD-03 | 블로그별, `sort_order` |
| `blog_tags` | 블로그 태그 | BLG-01, BLG-03 | 블로그 생성 때 다는 태그(최대 10개, D-88). 블로그·태그 UNIQUE |

### 운영·제재

| 테이블 | 한글 | 주요 요구사항 | 핵심 컬럼·규칙 |
| --- | --- | --- | --- |
| `member_sanctions` | 멤버 경고·정지·강제 퇴장 | BLG-13, 3.7 | `type` WARNING/SUSPENSION/KICK, `suspend_days` 3/14/30(영구 NULL), `ends_at`, `report_id`, `released_at`. 1년 보관 |
| `owner_sanctions` | 블로그장 경고·권한 박탈 | ADM-07 | `type` WARNING/DEMOTION. 그 블로그에서 최근 1년 안의 경고가 3번이면 박탈(블로그별, D-97) |
| `user_sanctions` | 계정 경고·프로필 초기화·정지 | ADM-08 | 메인 프로필 신고 처리 기록. `type` WARNING/PROFILE_RESET/SUSPENSION, `suspend_days` 3/14/30(영구 NULL), `ends_at`, `report_id`, `admin_id`, `released_at`. 정지 중인지는 `users.suspended_until`로 확인. 1년 보관 (D-99) |
| `blog_blacklist` | 블로그 블랙리스트 | BLG-11 | `name_hash`, `email_hash`, `phone_hash`. 고치거나 지우지 않고 `released_at`으로만 해제 |
| `blacklist_inquiries` | 블랙리스트 해제 문의 | BLG-12 | `name_match`, `phone_match`(서버 비교), `status` PENDING/RELEASED/REJECTED |
| `reports` | 신고 | SOC-06, ADM-04 | `target_type` USER/BLOG/POST/COMMENT, `handler_scope` BLOG_OWNER/ADMIN(블로그장 본인·블로그장 글·댓글 신고, `blog_id`가 없는 메인 프로필 신고, 블로그장이 계정 정지 중이고 부블로그장이 없는 블로그의 신고는 ADMIN, D-94, D-95, D-100, D-114), `reason`, `target_id`(종류마다 테이블이 달라 FK 없음), `target_snapshot`(원본이 지워져도 1년 확인, D-85), `status` PENDING/NO_ISSUE/WARNED/SUSPENDED/KICKED/OWNER_DEMOTED/BLOG_CLOSED/PROFILE_RESET/ACCOUNT_SUSPENDED |
| `admin_action_logs` | 관리자 활동 기록 | ADM-06, SEC-09 | `action`(예: OWNER_WARN, CLOSE_BLOG, VIEW_PRIVATE_INFO), 대상, 사유. 1년 보관 |

### 게시판

| 테이블 | 한글 | 주요 요구사항 | 핵심 컬럼·규칙 |
| --- | --- | --- | --- |
| `posts` | 게시글 | BRD-01, 02, 10 | `title` 1~30자, `content` 마크다운 TEXT(최대 5,000자는 서버에서 검사, D-92), `is_notice`, `status` PUBLISHED/HIDDEN/DELETED, `author_hidden`("탈퇴한 계정"), 개수 캐시, `deleted_by`·`deleted_at`. 메인 공지는 `main` 블로그의 글 |
| `tags` | 태그 | BRD-04 | `name` 영문 소문자, 1~20자, UNIQUE |
| `post_tags` | 글-태그 연결 | BRD-04 | `uk_post_tags`로 한 글 안 중복 방지 |
| `post_images` | 글 이미지 | BRD-05, SEC-08 | `stored_name`(UUID), `original_name`(DB에만), `content_type`, `size_bytes`(3MB 이하), `sort_order`(첫 이미지가 공유 미리보기), 글 저장 전 업로드면 `post_id` NULL |
| `comments` | 댓글 | BRD-06 | `parent_id`(1단계), `reply_to_user_id`(@닉네임, D-87), `content` 1~500자, `status` ACTIVE/HIDDEN/DELETED, `is_edited` |
| `post_likes` | 좋아요 | BRD-06, 6.5 | 회원·글 UNIQUE |
| `post_views` | 조회 기록 | BRD-11, D-77 | `viewer_key`(회원 번호 또는 IP+브라우저 해시), `view_date`. 같은 사람·같은 날·같은 글 UNIQUE |

### 소셜·알림·검색

| 테이블 | 한글 | 주요 요구사항 | 핵심 컬럼·규칙 |
| --- | --- | --- | --- |
| `user_follows` | 회원 팔로우 | SOC-01, 02 | follower·followee UNIQUE |
| `user_blocks` | 회원 차단 | SOC-05 | blocker·blocked UNIQUE |
| `notifications` | 알림 | SOC-04, 3.6 | `tab` COMMENT/LIKE/FOLLOW/BLOG/OPERATION, `type`(예: BLOG_CLOSING), 대상, `message`, `is_read`. 30일(또는 7일) 뒤 삭제 |
| `notification_settings` | 알림 설정 | 3.6 | 끌 수 있는 알림 종류만 행을 둠 |
| `recent_searches` | 최근 검색어 | BRD-08 | 본인만, 최대 10개, 같은 검색어면 시각만 갱신 |
| `shedlock` | 배치 잠금 | SCL-03 | ShedLock 표준 테이블 |

## 상태 전이

### 블로그 (`blogs.status`)

```text
ACTIVE ──폐쇄 버튼(BLG-09) / 관리자 강제 폐쇄(ADM-02, D-96) / 블로그장 박탈 + 부블로그장 없음(ADM-07)──▶ CLOSING
  ▲                                                                                                  │
  └──────────────── 폐쇄 철회 (7일 안, 그 블로그의 블로그장만) ◀───────────────────────────────────────┤
                                                                                                     │
                                     CLOSED ◀── close_scheduled_at 지난 뒤 04:00 배치 ◀───────────────┘
                                       │
                                       └─ 30일 뒤: 글·사진·카테고리·블랙리스트 삭제, slug NULL (행은 남김, D-86)
```

- CLOSING 중에도 글쓰기·댓글은 평소처럼 된다(D-67). 위임은 막힌다(D-20).
- CLOSING으로 바뀌면 대기 중인 위임 요청은 CANCELED.
- `close_reason`: OWNER(블로그장 폐쇄), OWNER_DEMOTED(권한 박탈), ADMIN(강제 폐쇄).

### 참여 신청 (`blog_join_requests.status`)

```text
PENDING ──승인──▶ APPROVED (blog_members 행 생성)
PENDING ──거절──▶ REJECTED (handled_at + 7일 뒤 재신청 가능)
PENDING ──신청자 취소──▶ CANCELED
(자유 참여 블로그는 신청 없이 바로 blog_members 행 생성)
(블로그장이 차단한 회원, 블랙리스트 해시 일치는 신청 단계에서 거절)
```

### 블로그장 위임 (`blog_owner_transfers.status`)

```text
PENDING ──받는 멤버 수락──▶ ACCEPTED (OWNER ↔ 받는 멤버 역할 교체, blogs.owner_id 갱신)
PENDING ──거절──▶ REJECTED
PENDING ──폐쇄 버튼──▶ CANCELED
PENDING ──7일 무응답──▶ EXPIRED
```

### 멤버 역할 (`blog_members.role`)

```text
MEMBER ──블로그장이 지정──▶ MANAGER (manager_since 기록, 권한은 blog_manager_permissions)
MANAGER ──해제──▶ MEMBER
MEMBER/MANAGER ──위임 수락 / 블로그장 박탈 시 가장 이른 manager_since──▶ OWNER
OWNER ──위임 / 관리자 박탈(ADM-07)──▶ MEMBER
MEMBER/MANAGER ──블로그 탈퇴 / 강제 퇴장──▶ (멤버에서 빠짐, 글은 author_hidden = TRUE, 강제 퇴장이면 blog_blacklist 등록)
```

### 회원 (`users.status`)

```text
ACTIVE ──탈퇴(운영 중·폐쇄 예정 블로그 없음)──▶ WITHDRAWN
  email 즉시 NULL (같은 이메일로 바로 재가입 가능)
  30일 뒤 name·nickname·phone·profile_image NULL, 글은 "탈퇴한 회원"
```

### 신고 (`reports.status`)

```text
PENDING ──블로그장 처리(handler_scope = BLOG_OWNER)──▶ NO_ISSUE | WARNED | SUSPENDED | KICKED
PENDING ──관리자 처리(handler_scope = ADMIN)──▶ NO_ISSUE | WARNED | OWNER_DEMOTED | BLOG_CLOSED
                                                메인 프로필 신고: NO_ISSUE | WARNED | PROFILE_RESET | ACCOUNT_SUSPENDED (D-99)
                                                블로그장 정지 중이고 부블로그장 없는 블로그의 신고: NO_ISSUE | WARNED | SUSPENDED | KICKED (D-100, D-114)
```

## 검증 규칙 (서버에서 검사)

| 대상 | 규칙 | 근거 |
| --- | --- | --- |
| 이메일 | 소문자로 바꿔 저장, 중복 불가 | 6.5 |
| 비밀번호 | 8~15자, 영문·숫자·특수문자 모두, 대소문자 구분 | SEC-02 |
| 닉네임 | 2~12자, 한글·영문·숫자, 중복 불가, 예약어(관리자, admin, 운영자, 탈퇴한 회원) 불가 | 6.5 |
| 전화번호 | 숫자만 저장 | USR-01 |
| 블로그 주소 | 영문 소문자·숫자·`-` 3~30자, 중복 불가, 예약어(main, admin, api, login, signup, search 등) 불가 | 6.5 |
| 블로그 개수 | 공개(일부 공개 포함) 3개(설정, 최대 5), 비공개 5개. 회원 행 잠금 후 셈 | BLG-10, D-68 |
| 글 제목 | 1~30자 | 6.6 |
| 글 본문 | 최대 5,000자 (공백·마크다운 기호 포함) | 6.6, D-92 |
| 댓글 | 1~500자, 대댓글은 1단계까지 | 6.6 |
| 태그 | 1~20자, 한글·영문·숫자·`_`, 글·블로그 하나에 10개, 영문 소문자 | 6.4, D-88 |
| 이미지 | jpg·jpeg·png·gif·webp, 장당 3MB, 글 하나에 10장(10MB), 확장자·MIME·시그니처 확인, EXIF 제거 | 6.3 |
| 검색어 | 2~20자, 공백·특수문자만이면 거부 | 6.1 |
| 신고 | 같은 사람·같은 대상 2주에 한 번 | 6.5 |
| 참여 신청 | 대기 중이면 불가, 거절 후 7일 | 6.5 |
| 인증번호 | 가입 10분 / 재설정 30분, 5번 틀리면 무효, 재발송 1분에 한 번 | USR-02, SEC-05 |
| 이메일 찾기 | 같은 IP 10분에 5번 | USR-08 |

## 04:00 배치가 하는 일

`@Scheduled(cron = "0 0 4 * * *", zone = "Asia/Seoul")` + ShedLock 하나에서 처리합니다.

1. `close_scheduled_at`이 지난 CLOSING 블로그 → CLOSED
2. 폐쇄까지 3일·1일 남은 블로그 → `blog_close_notices`에 없으면 알림
3. 7일 지난 PENDING 위임 요청 → EXPIRED, 블로그장에게 알림
4. 30일 지난 삭제 글·댓글·사진 완전 삭제. 폐쇄 30일 지난 블로그는 글·사진·카테고리·블랙리스트를 지우고 `slug`를 NULL로 비움(블로그 행은 남김, D-86)
5. 탈퇴 30일 지난 회원의 개인정보 NULL
6. 보관 기간 지난 알림, 만료 인증번호·임시 토큰 삭제, 만료 Refresh Token 삭제(D-83)
7. 1년 지난 운영 기록 삭제
