---

description: "메인블로그 플랫폼 구현 작업 목록"
---

# Tasks: 메인블로그 플랫폼

**Input**: `/specs/001-main-blog/`의 설계 문서 ([spec.md](spec.md), [plan.md](plan.md), [research.md](research.md), [data-model.md](data-model.md))

**Prerequisites**: plan.md, spec.md, research.md, data-model.md. `contracts/`(API 명세)와 `quickstart.md`는 아직 없어서 API 주소는 각 작업에 직접 적었습니다.

**Tests**: OPS-09(D-109)가 성공 기준 SC-002~SC-007, SC-009, SC-011을 자동 테스트로 확인하라고 정해, 그 테스트(T085, T127, T130, T141, T147)와 실제 MySQL로 도는 테스트 기반(T146)을 작업으로 넣었습니다. 기능별 단위 테스트는 따로 작업으로 두지 않았습니다.

**진행 상황 (2026-10-14 코드 대조)**: 171개 중 166개 완료([x]). 바뀐 작업(T069, T148, T152, T156)은 바뀐 모습으로 끝나 완료로 표시했습니다. T127·T141·T147은 blog-server PR #71에서 테스트를 채웠습니다(T127의 부블로그장·정지 경우는 `BlogAuthzTest` 단위 테스트와 `BlogAuthzIntegrationTest`, T147의 나머지 기준은 `AccountFlowTest`·`PostFlowTest`·`ProdProfileTest`·`ProdSecretsCheckTest`). 남은 5개: T060(조회 기록을 엔티티 없이 SQL로 처리, 동작은 됨), T128(응답 DTO 점검 기록), T132(요청 제한 공유 저장소 전환 설정, D-146으로 보류), T134(contracts/·quickstart.md), T140(브라우저 확인 기록).

**Organization**: 사용자 스토리(US1~US9)별로 나눠 스토리마다 따로 만들고 확인할 수 있게 했습니다.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 다른 파일을 고치고 앞의 끝나지 않은 작업에 기대지 않아 동시에 할 수 있음
- **[Story]**: 이 작업이 속한 사용자 스토리 (US1~US9)
- 설명에 고칠 파일 경로를 적음

## Path Conventions

- 코드는 이 문서 저장소가 아니라 별도 코드 저장소에 만든다. 아래 경로는 그 저장소의 루트 기준이다.
- plan.md의 Source Code 제안을 따라 단일 Spring Boot 프로젝트로 둔다. 기본 패키지는 `com.blog`로 적었다. 팀이 다른 이름을 정하면 경로의 `com/blog`만 바꾼다.
- DB 접근은 Spring Data JPA로 적었다(research.md에서 `ddl-auto=update`는 운영 금지, 스키마는 Flyway가 만든다).
- 모듈마다 `domain/`(엔티티), `repository/`, `service/`, `api/`(컨트롤러) 하위 패키지를 둔다.
- 화면은 `src/main/resources/static/`의 정적 HTML + JS, 공통 요청은 `static/js/api.js`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 코드 저장소와 빌드 준비

- [x] T001 plan.md의 Source Code 구조대로 Spring Boot 프로젝트를 만들고 패키지 `auth`, `blog`, `board`, `main`, `social`, `admin`, `batch`, `common`을 `src/main/java/com/blog/` 아래에 만든다
- [x] T002 `build.gradle`에 의존성 추가: spring-boot-starter-web, -security, -data-jpa, -validation, -mail, -test, MySQL 드라이버, Flyway(flyway-core, flyway-mysql), jjwt, jsoup, commonmark(마크다운 → HTML), shedlock-spring, shedlock-provider-jdbc-template, metadata-extractor 또는 같은 역할의 EXIF 제거 라이브러리
- [x] T003 [P] `src/main/resources/application.yml`에 MySQL 연결(환경변수), `spring.jpa.hibernate.ddl-auto=validate`, `spring.jackson.time-zone=Asia/Seoul`, JVM 기본 시간대 Asia/Seoul, `blog.limit.public=${BLOG_LIMIT_PUBLIC:3}`, `blog.limit.private=5`, 메일(Gmail SMTP) 설정, Turnstile 키, JWT 비밀키를 환경변수로 받게 적는다
- [x] T004 [P] 포맷·린트 설정(예: Spotless + google-java-format)을 `build.gradle`에 추가한다
- [x] T005 [P] 로컬 MySQL 8 실행용 `docker-compose.yml`(utf8mb4, ngram 기본 설정, 시간대 Asia/Seoul)을 만든다
- [x] T135 [P] PR과 main 푸시마다 `./gradlew build`(코드 모양 검사, 테스트)를 돌리는 GitHub Actions를 `.github/workflows/ci.yml`에 만든다(JDK 21, Gradle 캐시). 저장소 관리자가 GitHub 설정의 main 브랜치 보호 규칙에서 이 검사를 merge 필수로 켠다 (OPS-05, D-105)
- [x] T142 [P] 라이브러리 업데이트 확인을 `.github/dependabot.yml`에 만든다: `gradle`과 `github-actions`를 매주 월요일(Asia/Seoul) 확인하고, 열린 업데이트 PR은 종류별 5개까지. 저장소 관리자가 GitHub 설정에서 Dependabot alerts와 security updates를 켠다. 자동 merge는 켜지 않는다 (OPS-10, D-110)
- [x] T143 [P] `.github/workflows/ci.yml`에 DB 변경 규칙 검사를 넣는다: PR이 main에 이미 있는 `src/main/resources/db/migration/` 파일을 고치거나 지우면 실패하고, 새 파일 이름이 `V번호__설명.sql` 형식이며 번호가 겹치지 않는지 확인한다 (OPS-08, D-108)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 모든 스토리가 기대는 공통 기반

**⚠️ CRITICAL**: 이 단계가 끝나야 사용자 스토리 작업을 시작할 수 있다

- [x] T006 이 문서 저장소의 `docs/02-database/schema/erd_tables.sql`을 그대로 옮겨 `src/main/resources/db/migration/V1__init.sql`을 만든다(테이블 33개, 관계 65개)
- [x] T007 `docs/02-database/schema/add_indexes.sql`을 옮겨 `src/main/resources/db/migration/V2__indexes.sql`을 만들고, 블로그 이름·소개와 글 제목의 FULLTEXT ngram 인덱스가 있는지 확인한다(BRD-08)
- [x] T008 [P] 공통 응답·오류 형식과 `@RestControllerAdvice` 예외 처리를 `src/main/java/com/blog/common/error/GlobalExceptionHandler.java`에 만든다(검증 실패 400, 권한 없음 403, 없음 404, 제한 초과 429). 오류 본문은 모두 `{code, message}` 형식이고 스택 트레이스·SQL·클래스 이름·서버 경로를 담지 않으며(`server.error.include-stacktrace=never`), 500에는 로그에서 찾을 오류 번호(요청 ID)를 담는다 (OPS-02, D-102)
- [x] T009 [P] 요청 처리 5초 제한(서버 쿼리 타임아웃 + `spring.mvc.async.request-timeout`)과 시간 초과 시 "다시 시도" 응답을 `src/main/java/com/blog/common/config/TimeoutConfig.java`에 둔다(D-48)
- [x] T010 [P] `users` 엔티티를 `src/main/java/com/blog/auth/domain/User.java`에 만든다: `email`(소문자, UNIQUE, 탈퇴 즉시 NULL), `name`(변경 불가), `nickname`(2~12자, UNIQUE), `phone`(숫자만, 1차 중복 허용), `role` USER/ADMIN, `status` ACTIVE/WITHDRAWN, `login_fail_count`·`locked_until`, `notification_keep_days` 30/7, 약관·개인정보 동의 시각, `suspended_until`(영구는 9999-12-31)
- [x] T011 [P] `blogs` 엔티티를 `src/main/java/com/blog/blog/domain/Blog.java`에 만든다: `slug`(3~30자, UNIQUE, 폐쇄 30일 뒤 NULL), `visibility` PUBLIC/LINK_ONLY/PRIVATE, `share_key`, `join_policy` OPEN/APPROVAL, `status` ACTIVE/CLOSING/CLOSED, `is_hidden`, `close_scheduled_at`, `close_reason` OWNER/OWNER_DEMOTED/ADMIN, `closed_at`
- [x] T012 [P] `blog_members` 엔티티를 `src/main/java/com/blog/blog/domain/BlogMember.java`에 만든다: `role` OWNER/MANAGER/MEMBER, `manager_since`, `suspended_until`(영구는 9999-12-31), `suspension_count`
- [x] T013 [P] `blog_manager_permissions` 엔티티를 `src/main/java/com/blog/blog/domain/BlogManagerPermission.java`에 만든다: `permission` EDIT_INFO/MANAGE_MEMBERS/MANAGE_POSTS
- [x] T014 T010~T013의 리포지토리를 `src/main/java/com/blog/auth/repository/UserRepository.java`, `src/main/java/com/blog/blog/repository/BlogRepository.java`, `BlogMemberRepository.java`, `BlogManagerPermissionRepository.java`에 만든다
- [x] T015 JWT 발급·검증(토큰에는 회원 번호만)을 `src/main/java/com/blog/common/security/JwtProvider.java`에, 쿠키에서 Access Token을 읽어 인증하는 필터를 `src/main/java/com/blog/common/security/JwtCookieAuthFilter.java`에 만든다(HttpOnly·Secure 쿠키, localStorage 금지, SEC-04)
- [x] T016 Spring Security 설정을 `src/main/java/com/blog/common/config/SecurityConfig.java`에 만든다: 주소별 비회원·회원·관리자 구분, `CookieCsrfTokenRepository`로 CSRF(POST·PUT·DELETE), 보안 헤더(X-Frame-Options, X-Content-Type-Options: nosniff, Content-Security-Policy, Strict-Transport-Security, Referrer-Policy), `@EnableMethodSecurity` (SEC-10~12)
- [x] T017 블로그별 역할을 요청마다 DB로 확인하는 빈을 `src/main/java/com/blog/common/security/BlogAuthz.java`에 만든다: `isOwner(blogId)`, `isMember(blogId)`, `hasPermission(blogId, EDIT_INFO|MANAGE_MEMBERS|MANAGE_POSTS)`, 블로그 단위 정지(`blog_members.suspended_until`) 확인. `@PreAuthorize("@blogAuthz....")`로 쓴다 (SEC-07, constitution II)
- [x] T018 계정 상태 확인을 `src/main/java/com/blog/common/security/AccountGuard.java`에 만든다: `role = ADMIN`도 일반 회원과 같이 블로그 만들기·참여 신청·글쓰기·댓글·좋아요·팔로우·구독을 허용하고(D-122), `users.suspended_until`이 지금보다 뒤면 글·댓글·좋아요·팔로우·구독·블로그 만들기·참여 신청 거부(ADM-08)
- [x] T019 [P] 개인정보 가리기를 `src/main/java/com/blog/common/privacy/Masking.java`에 만든다: 이메일은 @ 앞 처음 2글자만 보이고 나머지 `***`(@ 앞이 2글자 이하면 첫 글자만, `abcdef@gmail.com → ab***@gmail.com`), 전화번호는 가운데 4자리 가림(`010-1234-5678 → 010-****-5678`). 응답을 만들 때 서버에서 적용 (D-22)
- [x] T020 [P] 해시 도구(SHA-256 + 서버 비밀값)를 `src/main/java/com/blog/common/crypto/HashUtil.java`에 만든다. 인증번호·토큰·블랙리스트 개인정보에 쓴다
- [x] T021 [P] `FileStorage` 인터페이스와 디스크 구현을 `src/main/java/com/blog/common/storage/FileStorage.java`, `LocalDiskFileStorage.java`에 만든다. 저장 이름은 UUID (SCL-02)
- [x] T022 [P] IP별 요청 제한(1차는 메모리, 이중화 때 Redis로 바꿀 수 있게 인터페이스 뒤)을 `src/main/java/com/blog/common/ratelimit/RateLimiter.java`, `InMemoryRateLimiter.java`에 만든다. 프록시 헤더(X-Forwarded-For) 신뢰 여부는 설정으로 (SCL-01)
- [x] T023 [P] 메일 발송을 `src/main/java/com/blog/common/mail/MailService.java`에 만든다(JavaMailSender, Gmail SMTP, D-69). 발송이 실패하면(Gmail 오류, 5초 시간 초과, 하루 한도 초과) 자동으로 다시 보내지 않고 오류 로그를 남긴 뒤 "메일을 보내지 못했어요. 잠시 뒤 다시 시도해 주세요" 오류를 돌려준다. 이 문구는 가입된 이메일인지와 상관없이 같고, 보내지 못한 인증번호는 바로 무효로 하며 1분 재발송 제한에 세지 않는다 (OPS-07, D-107)
- [x] T024 [P] ShedLock 설정(`@EnableSchedulerLock`, JdbcTemplateLockProvider, `shedlock` 테이블)을 `src/main/java/com/blog/common/config/SchedulerConfig.java`에 만든다 (SCL-03)
- [x] T025 [P] `notifications`, `notification_settings` 엔티티를 `src/main/java/com/blog/social/domain/Notification.java`, `NotificationSetting.java`에 만든다: `tab` COMMENT/LIKE/FOLLOW/BLOG/OPERATION, `type`, 대상, `message`, `is_read`. 설정은 끌 수 있는 알림 종류만 행을 둔다
- [x] T026 알림 만들기 공통 서비스를 `src/main/java/com/blog/social/service/NotificationService.java`에 만든다: spec.md 알림 종류 표의 종류·탭·끌 수 있음 여부를 enum으로 두고, 끈 종류는 만들지 않으며 끌 수 없는 종류는 항상 만든다
- [x] T027 [P] 공통 화면 틀을 만든다: `src/main/resources/static/js/api.js`(모든 요청에 `X-XSRF-TOKEN` 헤더, 5초 넘으면 중단하고 다시 시도 안내, 401이면 토큰 재발급 시도), `static/js/layout.js`(메인으로 가는 상단 메뉴, 로그인 상태, 종 아이콘, 모든 화면 아래 개인정보 처리방침 링크), `static/css/common.css`(PC·모바일 반응형)
- [x] T136 [P] 오류 안내 화면을 `src/main/resources/static/error/404.html`, `403.html`, `500.html`에 만든다: 내부 정보 없이 안내 문구와 메인으로 가기 버튼, 500은 오류 번호 표시 (OPS-02, D-102)
- [x] T137 로그 설정을 `src/main/resources/logback-spring.xml`에 만든다: 서버 파일에 날짜별로, 30일 지나면 자동 삭제, 모든 줄에 요청 ID(`src/main/java/com/blog/common/logging/RequestIdFilter.java`가 MDC에 넣음). 로그인 실패(가린 이메일, IP)·권한 거부(403)·요청 제한 초과(429)는 `src/main/java/com/blog/common/logging/SecurityEventLogger.java`로 남기고 이메일·전화번호는 Masking(T019)으로 가린다. 비밀번호·JWT·Refresh Token·인증번호·쿠키·CSRF 토큰은 로그에 넣지 않는다 (OPS-03, D-103)
- [x] T138 상태 확인 주소를 연다: `build.gradle`에 spring-boot-starter-actuator를 추가하고, `src/main/resources/application.yml`에 `management.endpoints.web.exposure.include=health`, `management.endpoint.health.show-details=never`를 적고, `src/main/java/com/blog/common/config/SecurityConfig.java`에서 `/actuator/health`만 비회원에게 연다(DB 연결 포함 UP/DOWN만) (OPS-04, D-104)
- [x] T144 실행 환경을 나눈다: `src/main/resources/application.yml`은 운영 기준(쿠키 Secure, Swagger 꺼짐, SQL 로그 꺼짐)으로 두고 `application-local.yml`에서만 푼다(Secure 없는 쿠키, Swagger 열기, SQL 로그). `application-prod.yml`은 프록시 헤더(SCL-01)를 켜고, `src/main/java/com/blog/common/config/SecurityConfig.java`에서 prod일 때 http 요청을 https로 돌려보낸다. `src/main/java/com/blog/common/config/ProdSecretsCheck.java`는 prod에서 JWT 비밀키·DB 비밀번호·Gmail 앱 비밀번호·Turnstile 비밀키가 비었거나 `.env.example`의 예시 값이면 서버를 멈춘다. `./gradlew bootRun`은 local로 뜨게 한다 (OPS-11, D-111)
- [x] T145 API 문서를 연다: `build.gradle`에 springdoc-openapi-starter-webmvc-ui를 추가하고, `application.yml`에 `springdoc.api-docs.enabled=false`, `springdoc.swagger-ui.enabled=false`를 적고 `application-local.yml`에서만 true로 둔다 (OPS-12, D-112)
- [x] T146 [P] 테스트 기반을 `src/test/java/com/blog/support/`에 만든다: Testcontainers로 MySQL 8 컨테이너를 한 번 띄워 테스트끼리 같이 쓰고(`MySqlTestcontainersConfig.java`), 스키마는 Flyway V1부터 그대로 적용하며, 통합 테스트에 붙일 `@IntegrationTest`를 둔다. H2는 쓰지 않는다. README에 테스트에는 Docker가 필요하다고 적는다 (OPS-09, D-109)

**Checkpoint**: 기반 완료. 이제 사용자 스토리를 시작할 수 있다

---

## Phase 3: User Story 1 - 가입하고 로그인한다 (Priority: P1) 🎯 MVP

**Goal**: 방문자가 이메일 인증을 거쳐 가입하고, 로그인 상태로 모든 개별 블로그를 오간다

**Independent Test**: 새 이메일로 가입 → 로그아웃 → 로그인 → 다른 블로그 주소로 이동했을 때 로그인 상태가 유지되는지 확인한다

### Implementation for User Story 1

- [x] T028 [P] [US1] `verification_codes` 엔티티와 리포지토리를 `src/main/java/com/blog/auth/domain/VerificationCode.java`, `src/main/java/com/blog/auth/repository/VerificationCodeRepository.java`에 만든다: 회원 번호 대신 `email` 기준, `purpose` SIGNUP(10분)/PASSWORD_RESET(30분), `code_hash`, `fail_count`(5번 무효), `verified_at`, `used_at`(1회용)
- [x] T029 [P] [US1] `refresh_tokens` 엔티티와 리포지토리를 `src/main/java/com/blog/auth/domain/RefreshToken.java`, `src/main/java/com/blog/auth/repository/RefreshTokenRepository.java`에 만든다: `token_hash`, `remember_me`(14일/30분), `user_agent`, `revoked_at`
- [x] T030 [US1] 인증번호 서비스를 `src/main/java/com/blog/auth/service/VerificationCodeService.java`에 만든다: SecureRandom 6자리, 해시 저장, 가입 10분 만료, 5번 틀리면 무효, 재발송은 1분에 한 번이고 이전 번호 무효 (USR-02, D-10, D-28, D-42)
- [x] T031 [US1] 가입 1단계 API `POST /api/auth/signup/code`, `POST /api/auth/signup/verify`를 `src/main/java/com/blog/auth/api/SignupController.java`에 만든다. 이미 가입된 이메일이어도 화면 응답은 "인증번호를 보냈습니다"로 같고, 메일로만 "이미 가입된 이메일입니다"를 보낸다 (USR-02, SC-003)
- [x] T032 [US1] 가입 완료 API `POST /api/auth/signup`을 `src/main/java/com/blog/auth/service/SignupService.java`와 `SignupController.java`에 만든다: 서버가 인증 완료 여부를 다시 확인, 비밀번호 8~15자 영문·숫자·특수문자 모두 포함(SEC-02), bcrypt 저장(SEC-01), 이메일 소문자 저장·중복 불가, 닉네임 2~12자 한글·영문·숫자·중복 불가·예약어(관리자, admin, 운영자, 탈퇴한 회원) 불가, 전화번호 숫자만, [필수] 이용약관·[필수] 개인정보 수집·이용 동의 시각 저장, 만 14세 확인 없음 (USR-01, D-91)
- [x] T033 [US1] Turnstile 서버 검증을 `src/main/java/com/blog/auth/service/TurnstileVerifier.java`에 만든다 (SEC-13, D-79)
- [x] T034 [US1] 로그인 API `POST /api/auth/login`을 `src/main/java/com/blog/auth/service/LoginService.java`, `src/main/java/com/blog/auth/api/AuthController.java`에 만든다: 계정별 5회 연속 실패 시 5분 잠금(`login_fail_count`, `locked_until`), 3번째 실패부터 Turnstile 요구, IP별 요청 제한(T022), 탈퇴 계정 거부, 성공 시 Access Token + Refresh Token을 HttpOnly·Secure 쿠키로 (USR-03, SEC-03, SEC-04, D-80)
- [x] T035 [US1] "로그인 유지"를 `LoginService.java`에 반영한다: 미체크면 세션 쿠키 + 30분 무활동 만료, 체크면 14일. 토큰 재발급 API `POST /api/auth/refresh`는 Refresh Token을 DB에서 확인해 바꿔 준다 (SEC-04, D-61, D-62)
- [x] T036 [US1] 30분 연장 규칙을 `src/main/java/com/blog/auth/service/ActivityPolicy.java`에 만든다: 사용자가 직접 한 행동만 연장하고, 알림 폴링 같은 자동 요청(`X-Auto-Request: true` 헤더)은 세지 않는다 (D-72)
- [x] T037 [US1] 로그아웃 API `POST /api/auth/logout`을 `AuthController.java`에 만든다: Refresh Token 폐기(`revoked_at`), 토큰 쿠키 삭제 (USR-04)
- [x] T038 [P] [US1] 가입 화면 3단계(이메일 인증 → 비밀번호 → 이름·닉네임·전화번호·약관 동의)를 `src/main/resources/static/signup.html`, `static/js/signup.js`에 만든다
- [x] T039 [P] [US1] 로그인 화면(로그인 유지 체크박스, 3번째 실패부터 Turnstile 위젯)을 `src/main/resources/static/login.html`, `static/js/login.js`에 만든다
- [x] T040 [US1] 글쓰기 중 입력이 있으면 연장 요청을 보내고 만료 5분 전에 "로그인을 연장할까요?" 창을 띄우는 처리를 `src/main/resources/static/js/session.js`에 만든다

**Checkpoint**: 가입·로그인·로그아웃이 따로 동작한다

---

## Phase 4: User Story 2 - 블로그를 만들고, 찾고, 참여한다 (Priority: P1)

**Goal**: 회원이 블로그를 만들어 블로그장이 되거나, 메인에서 블로그를 찾아 참여한다

**Independent Test**: 회원 A가 승인제 블로그를 만들고, 회원 B가 검색으로 찾아 참여 신청 → A가 승인 → B의 "내 블로그 목록"에 나타나는지 확인한다

### Implementation for User Story 2

- [x] T041 [P] [US2] `blog_tags`, `tags` 엔티티와 리포지토리를 `src/main/java/com/blog/blog/domain/BlogTag.java`, `src/main/java/com/blog/board/domain/Tag.java`와 각 리포지토리에 만든다: `tags.name` 영문 소문자, 1~20자, UNIQUE. 블로그·태그 UNIQUE
- [x] T042 [P] [US2] `blog_join_requests` 엔티티와 리포지토리를 `src/main/java/com/blog/blog/domain/BlogJoinRequest.java`, `src/main/java/com/blog/blog/repository/BlogJoinRequestRepository.java`에 만든다: `status` PENDING/APPROVED/REJECTED/CANCELED, `handled_at`
- [x] T043 [US2] 태그 정리 규칙을 `src/main/java/com/blog/board/service/TagNormalizer.java`에 만든다: 앞의 #과 앞뒤 공백 제거, 가운데 공백은 `_`, 영문은 소문자, 1~20자 한글·영문·숫자·`_`, 한 글·블로그 안 중복은 합침, 10개까지, 금칙어 없음 (BRD-04, D-88, D-98). 블로그와 글이 같이 쓴다
- [x] T044 [US2] 블로그 만들기 API `POST /api/blogs`를 `src/main/java/com/blog/blog/service/BlogService.java`, `src/main/java/com/blog/blog/api/BlogController.java`에 만든다: 이름(중복 허용)·주소·소개·대표 이미지(3MB 1장)·태그·공개 범위·참여 방식, 주소는 영문 소문자·숫자·`-` 3~30자, 중복 불가, 예약어(main, admin, api, login, signup, search 등) 불가, 만든 회원을 OWNER로 `blog_members`에 추가, AccountGuard 적용 (BLG-01, D-70)
- [x] T045 [US2] 생성 개수 제한을 `BlogService.java`에 넣는다: 공개(일부 공개 포함) 3개(`blog.limit.public`, 최대 5), 비공개 5개. 회원 행을 `SELECT ... FOR UPDATE`로 잠근 뒤 세고 만든다. `role = ADMIN`은 세지 않는다 (BLG-10, D-68, D-122, SC-006)
- [x] T046 [US2] 일부 공개 공유 링크를 `src/main/java/com/blog/blog/service/ShareLinkService.java`에 만든다: 주소와 별개인 무작위 `share_key`, 새로 만들면 이전 링크 무효, 링크 없이 주소로만 오면 막음, 링크로 온 비회원도 열람 (BLG-01, D-37, D-49, D-50)
- [x] T047 [US2] 블로그 열람 권한 확인을 `src/main/java/com/blog/blog/service/BlogAccessService.java`에 만든다: 공개는 누구나, 일부 공개는 공유 링크, 비공개는 멤버만, 숨김·폐쇄 블로그 제외, 그 블로그에서 정지된 멤버는 기간·사유 안내와 함께 막음
- [x] T048 [US2] 블로그 목록 API `GET /api/blogs?sort=latest|popular&page=`를 `BlogController.java`에 만든다: 공개 블로그만, 인기순은 `member_count` 순, 번호 페이지 (BLG-02, D-89)
- [x] T049 [US2] 블로그 검색 API `GET /api/blogs/search?q=`를 `BlogController.java`에 만든다: 이름·소개(FULLTEXT ngram), 블로그 태그 일치 (BLG-03)
- [x] T050 [US2] 참여 신청 API `POST /api/blogs/{blogId}/join`을 `src/main/java/com/blog/blog/service/JoinService.java`, `src/main/java/com/blog/blog/api/JoinController.java`에 만든다: 자유 참여는 바로 MEMBER, 승인제는 PENDING, 대기 중이면 다시 신청 불가, 거절 후 `handled_at` + 7일 전에는 불가, 블로그장이 계정 정지 중이고 부블로그장이 없으면 "블로그장 정지로 참여 신청이 일시 중지되었어요 (끝나는 날짜까지)"로 거부(D-115), AccountGuard 적용, 블로그장·승인 권한 부블로그장에게 "참여 신청" 알림 (BLG-04, D-01)
- [x] T051 [US2] 참여 승인·거절·취소 API `POST /api/blogs/{blogId}/join-requests/{id}/approve|reject`, `DELETE /api/blogs/{blogId}/join-requests/{id}`를 `JoinController.java`에 만든다. 승인·거절은 `@PreAuthorize` 블로그장 또는 MANAGE_MEMBERS 권한, 결과는 신청자에게 알림 (BLG-05)
- [x] T052 [US2] 내 블로그 목록 API `GET /api/me/blogs`를 `BlogController.java`에 만든다: 만든 블로그와 참여한 블로그를 나눠서 (BLG-06)
- [x] T053 [US2] 블로그 정보 수정 API `PUT /api/blogs/{blogId}`(이름·소개·대표 이미지·태그·공개 범위·참여 방식)를 `BlogController.java`에 만든다. 블로그장 또는 EDIT_INFO 권한
- [x] T054 [P] [US2] 메인 블로그 목록·검색·블로그 만들기 화면을 `src/main/resources/static/index.html`, `static/blog-new.html`, `static/js/blog-list.js`, `static/js/blog-new.js`에 만든다
- [x] T055 [P] [US2] 블로그 첫 화면(`/blog/{주소}`, 참여·구독 버튼)과 내 블로그 목록 화면을 `src/main/resources/static/blog.html`, `static/my-blogs.html`, `static/js/blog.js`에 만든다
- [x] T056 [P] [US2] 블로그 관리 화면의 참여 신청 탭과 정보 수정 탭을 `src/main/resources/static/blog-admin.html`, `static/js/blog-admin.js`에 만든다

**Checkpoint**: US1·US2가 각각 동작한다

---

## Phase 5: User Story 3 - 글을 쓰고 읽는다 (Priority: P1)

**Goal**: 멤버가 참여한 블로그에 글을 쓰고, 누구나 공개 글을 읽고 댓글·좋아요를 남긴다

**Independent Test**: 멤버가 이미지와 태그가 있는 글을 쓰고, 다른 회원이 댓글·대댓글·좋아요를 남기고, 비회원이 목록·상세를 읽는지 확인한다

### Implementation for User Story 3

- [x] T057 [P] [US3] `posts` 엔티티와 리포지토리를 `src/main/java/com/blog/board/domain/Post.java`, `src/main/java/com/blog/board/repository/PostRepository.java`에 만든다: `title` 1~30자, `content` 마크다운 TEXT(최대 5,000자는 서버에서 검사), `is_notice`, `status` PUBLISHED/HIDDEN/DELETED, `author_hidden`, 개수 캐시, `deleted_by`·`deleted_at`
- [x] T058 [P] [US3] `post_tags`, `post_images` 엔티티와 리포지토리를 `src/main/java/com/blog/board/domain/PostTag.java`, `PostImage.java`와 각 리포지토리에 만든다: `post_images`는 `stored_name`(UUID), `original_name`(DB에만), `content_type`, `size_bytes`(3MB 이하), `sort_order`, 글 저장 전 업로드면 `post_id` NULL
- [x] T059 [P] [US3] `comments` 엔티티와 리포지토리를 `src/main/java/com/blog/board/domain/Comment.java`, `src/main/java/com/blog/board/repository/CommentRepository.java`에 만든다: `parent_id`(1단계), `reply_to_user_id`(@닉네임, D-87), `content` 1~500자, `status` ACTIVE/HIDDEN/DELETED, `is_edited`
- [ ] T060 [P] [US3] `post_likes`, `post_views`, `categories` 엔티티와 리포지토리를 `src/main/java/com/blog/board/domain/PostLike.java`, `PostView.java`, `Category.java`와 각 리포지토리에 만든다: 좋아요는 회원·글 UNIQUE, 조회 기록은 `viewer_key`(회원 번호 또는 IP+브라우저 해시)·`view_date`·같은 사람·같은 날·같은 글 UNIQUE, 카테고리는 블로그별 `sort_order`
- [x] T061 [US3] 본문 변환을 `src/main/java/com/blog/board/service/ContentRenderer.java`에 만든다: 마크다운 → HTML 변환 후 jsoup Safelist로 위험한 태그 제거, 최대 5,000자(공백·마크다운 기호 포함) 검사 (SEC-06, D-64, D-92)
- [x] T062 [US3] 이미지 업로드 API `POST /api/images`를 `src/main/java/com/blog/board/service/ImageService.java`, `src/main/java/com/blog/board/api/ImageController.java`에 만든다: jpg·jpeg·png·gif·webp만, 확장자·MIME·파일 시그니처 모두 확인, 장당 3MB, 글 하나에 10장(10MB), EXIF 제거, UUID 이름으로 FileStorage에 저장, 원래 이름은 DB에만 (BRD-05, SEC-08, D-75)
- [x] T063 [US3] 글 쓰기·수정·삭제 API `POST /api/blogs/{blogId}/posts`, `PUT /api/posts/{id}`, `DELETE /api/posts/{id}`를 `src/main/java/com/blog/board/service/PostService.java`, `src/main/java/com/blog/board/api/PostController.java`에 만든다: 쓰기는 그 블로그 멤버만(AccountGuard 적용), 수정은 작성자만, 삭제는 작성자·블로그장(자기 블로그 글)·MANAGE_POSTS 부블로그장·관리자, 남이 지우면 작성자에게 "내 글 삭제됨" 알림, 태그는 TagNormalizer, 이미지 연결 (BRD-01, SEC-07)
- [x] T064 [US3] 글 목록 API `GET /api/blogs/{blogId}/posts?category=&size=10|20|30&page=`를 `PostController.java`에 만든다: 최신순, 번호 페이지, 기본 10개, 열람 권한은 BlogAccessService (BRD-02, BRD-03, D-76)
- [x] T065 [US3] 글 상세 API `GET /api/posts/{id}`와 조회수 처리를 `PostService.java`, `src/main/java/com/blog/board/service/ViewCountService.java`에 만든다: `post_views` UNIQUE로 같은 사람·같은 날·같은 글은 1번만, 비회원은 IP+브라우저 해시 (BRD-11, D-77, SC-007)
- [x] T066 [US3] 카테고리 관리 API `POST|PUT|DELETE /api/blogs/{blogId}/categories`를 `src/main/java/com/blog/board/api/CategoryController.java`에 만든다. 블로그장 또는 EDIT_INFO 권한 (BRD-03)
- [x] T067 [US3] 댓글 API `POST /api/posts/{id}/comments`, `PUT|DELETE /api/comments/{id}`를 `src/main/java/com/blog/board/service/CommentService.java`, `src/main/java/com/blog/board/api/CommentController.java`에 만든다: 1~500자, 대댓글은 1단계(답글에 답하면 같은 부모 + `reply_to_user_id`), 수정하면 "수정됨", 답글 있는 댓글을 지우면 "삭제된 댓글입니다", 글 작성자·댓글 작성자에게 알림, AccountGuard 적용 (BRD-06, D-78, D-87)
- [x] T068 [US3] 좋아요 API `POST /api/posts/{id}/like`(다시 누르면 취소)를 `src/main/java/com/blog/board/api/LikeController.java`에 만든다. 글 작성자에게 알림, AccountGuard 적용 (BRD-06)
- [x] T069 [US3] 글 상세 주소의 og 태그(og:title, og:description, og:image)를 서버가 채우는 컨트롤러를 `src/main/java/com/blog/board/api/PostPageController.java`에 만든다: 이미지는 글의 첫 이미지, 없으면 블로그 대표 이미지 (BRD-07, D-66) (바뀜: T161에서 글·블로그 공유 미리보기를 `main/api/SharePageController.java` 한 곳으로 모음)
- [x] T070 [P] [US3] 글쓰기 화면(Toast UI Editor, 임시저장은 브라우저에만, 삭제 전 확인 창)을 `src/main/resources/static/post-edit.html`, `static/js/post-edit.js`에 만든다 (D-74)
- [x] T071 [P] [US3] 글 목록·상세 화면(본문만 innerHTML, 나머지는 textContent, 링크 복사 공유 버튼, 제목에 마우스를 올리면 사진 표시)을 `src/main/resources/static/js/post-list.js`, `static/js/post-view.js`에 만든다 (SEC-06, BRD-07)

**Checkpoint**: MVP(US1~US3) 완료. 가입 → 블로그 만들기·참여 → 글쓰기·읽기가 된다

---

## Phase 6: User Story 4 - 블로그장이 멤버를 관리한다 (Priority: P2)

**Goal**: 블로그장(멤버 관리 권한을 받은 부블로그장 포함)이 자기 블로그의 멤버를 경고·정지·강제 퇴장하고, 블로그 안 신고를 처리한다

**Independent Test**: 블로그장이 멤버를 14일 정지 → 그 멤버가 그 블로그에만 못 들어가고 다른 블로그는 쓸 수 있는지 → 강제 퇴장 후 같은 전화번호로 재가입해도 참여 신청이 막히는지 확인한다

### Implementation for User Story 4

- [x] T072 [P] [US4] `member_sanctions` 엔티티와 리포지토리를 `src/main/java/com/blog/blog/domain/MemberSanction.java`, `src/main/java/com/blog/blog/repository/MemberSanctionRepository.java`에 만든다: `type` WARNING/SUSPENSION/KICK, `suspend_days` 3/14/30(영구 NULL), `ends_at`, `report_id`, `released_at`
- [x] T073 [P] [US4] `blog_blacklist`, `blacklist_inquiries` 엔티티와 리포지토리를 `src/main/java/com/blog/blog/domain/BlogBlacklist.java`, `BlacklistInquiry.java`와 각 리포지토리에 만든다: 블랙리스트는 `name_hash`, `email_hash`, `phone_hash`, 고치거나 지우지 않고 `released_at`으로만 해제. 문의는 `name_match`, `phone_match`, `status` PENDING/RELEASED/REJECTED
- [x] T074 [P] [US4] `reports` 엔티티와 리포지토리를 `src/main/java/com/blog/social/domain/Report.java`, `src/main/java/com/blog/social/repository/ReportRepository.java`에 만든다: `target_type` USER/BLOG/POST/COMMENT, `handler_scope` BLOG_OWNER/ADMIN, `reason`, `target_id`(FK 없음), `target_snapshot`, `status` PENDING/NO_ISSUE/WARNED/SUSPENDED/KICKED/OWNER_DEMOTED/BLOG_CLOSED/PROFILE_RESET/ACCOUNT_SUSPENDED
- [x] T075 [US4] 신고 접수 API `POST /api/reports`를 `src/main/java/com/blog/social/service/ReportService.java`, `src/main/java/com/blog/social/api/ReportController.java`에 만든다: 같은 사람·같은 대상 2주에 한 번, 신고 당시 대상 내용(글 제목·본문 앞부분, 댓글, 닉네임, 블로그 이름)을 `target_snapshot`에 저장, `handler_scope`는 블로그 안의 회원·글·댓글이면 BLOG_OWNER, 블로그 자체·블로그장 본인·블로그장이 쓴 글·댓글·메인 프로필(`blog_id` 없음)·블로그장이 계정 정지 중이고 부블로그장이 없는 블로그면 ADMIN (SOC-06, D-45, D-85, D-94, D-95, D-100, D-114)
- [x] T076 [US4] 멤버 제재 API `POST /api/blogs/{blogId}/members/{userId}/warn|suspend|kick`, `POST .../release`를 `src/main/java/com/blog/blog/service/MemberSanctionService.java`, `src/main/java/com/blog/blog/api/MemberAdminController.java`에 만든다: 블로그장 또는 MANAGE_MEMBERS 권한, 블로그장이 계정 정지 중이면 블로그장 거부, 정지는 3·14·30일·영구(9999-12-31), `suspension_count` 증가, 3번이면 관리 화면에 표시, 대상에게 알림 (BLG-13, D-29, D-43, D-46, D-57)
- [x] T077 [US4] 강제 퇴장 처리를 `MemberSanctionService.java`에 넣는다: 멤버에서 빼고 그 블로그의 그 사람 글을 `author_hidden = TRUE`로, 이름·이메일·전화번호 해시를 `blog_blacklist`에 등록 (BLG-11, D-34, D-55)
- [x] T078 [US4] 참여 신청 때 블랙리스트 해시 비교를 `JoinService.java`에 넣는다: 이메일 또는 전화번호 해시가 일치하면 거절(재가입해도) (BLG-11)
- [x] T079 [US4] 블랙리스트 해제 문의 API `POST /api/blogs/{blogId}/blacklist-inquiries`와 처리 API `POST /api/blogs/{blogId}/blacklist-inquiries/{id}/release|reject`를 `src/main/java/com/blog/blog/service/BlacklistService.java`, `src/main/java/com/blog/blog/api/BlacklistController.java`에 만든다: 서버가 이름·전화번호 일치 여부를 계산해 보여 주고, 결과는 문의자에게 알림. 블랙리스트는 블로그장이 바뀌어도 새 블로그장·부블로그장은 열람만 (BLG-12)
- [x] T080 [US4] 블로그 안 신고 처리 API `POST /api/blogs/{blogId}/reports/{id}/resolve`(문제 없음·경고·정지·강제 퇴장)를 `src/main/java/com/blog/blog/api/BlogReportController.java`에 만든다: `handler_scope = BLOG_OWNER`만, 블로그장 또는 MANAGE_MEMBERS 권한, 처리 결과는 신고자에게 알림 (BLG-13)
- [x] T081 [US4] 부블로그장 지정·해제와 권한 체크박스 API `PUT /api/blogs/{blogId}/managers/{userId}`를 `src/main/java/com/blog/blog/api/ManagerController.java`에 만든다: 블로그장만, `manager_since` 기록, 권한 EDIT_INFO/MANAGE_MEMBERS/MANAGE_POSTS (D-03, D-71)
- [x] T082 [US4] 멤버 목록 API `GET /api/blogs/{blogId}/members`를 `MemberAdminController.java`에 만든다: 블로그장·MANAGE_MEMBERS 부블로그장만, 이메일·전화번호는 Masking으로 가림, 정지 3번 멤버 표시 (4.4)
- [x] T083 [P] [US4] 블로그 관리 화면의 멤버·부블로그장·신고·블랙리스트 탭을 `src/main/resources/static/blog-admin.html`, `static/js/blog-admin-members.js`에 만든다
- [x] T084 [P] [US4] 신고하기 창(사유 고르기)을 `src/main/resources/static/js/report.js`에 만든다

**Checkpoint**: 블로그 단위 운영이 따로 동작한다

---

## Phase 7: User Story 5 - 블로그를 넘기거나 닫고, 탈퇴한다 (Priority: P2)

**Goal**: 블로그장이 블로그를 다른 멤버에게 위임하거나 폐쇄하고, 회원이 블로그나 서비스를 떠난다

**Independent Test**: 10월 2일 오후 3시에 폐쇄 버튼 → 10월 9일 04:00 폐쇄 예정으로 저장, 즉시·10월 6일·10월 8일에 멤버와 구독자에게 알림이 가는지, 철회하면 멈추는지 확인한다

### Tests for User Story 5 (plan.md가 요구한 통합 테스트) ⚠️

- [x] T085 [P] [US5] 04:00 배치 통합 테스트를 `src/test/java/com/blog/batch/DailyBatchIntegrationTest.java`에 만든다: 고정 시계로 10월 2일 15:00 폐쇄 → `close_scheduled_at` 10월 9일 04:00, 10월 6일·8일 배치에서 D3·D1 알림이 각각 한 번, 같은 날 배치를 두 번 돌려도 중복 없음, 서버가 꺼졌던 날 다음 배치가 밀린 폐쇄를 처리, 철회하면 알림이 멈춤 (SC-005)

### Implementation for User Story 5

- [x] T086 [P] [US5] `blog_owner_transfers`, `blog_close_notices` 엔티티와 리포지토리를 `src/main/java/com/blog/blog/domain/BlogOwnerTransfer.java`, `BlogCloseNotice.java`와 각 리포지토리에 만든다: 위임은 `status` PENDING/ACCEPTED/REJECTED/CANCELED/EXPIRED, `expires_at`(+7일). 폐쇄 알림은 `close_scheduled_at` + `stage` IMMEDIATE/D3/D1/REVOKED
- [x] T087 [US5] 위임 API `POST /api/blogs/{blogId}/transfers`, `POST /api/transfers/{id}/accept|reject`를 `src/main/java/com/blog/blog/service/TransferService.java`, `src/main/java/com/blog/blog/api/TransferController.java`에 만든다: 받는 멤버가 수락해야 OWNER ↔ 받는 멤버 역할 교체와 `blogs.owner_id` 갱신, 폐쇄 예정 중에는 위임 불가, 받는 멤버에게 "위임 요청" 알림, 결과는 블로그장에게 알림 (BLG-08, D-16, D-20, D-31)
- [x] T088 [US5] 폐쇄·철회 API `POST /api/blogs/{blogId}/close`, `POST /api/blogs/{blogId}/close/revoke`를 `src/main/java/com/blog/blog/service/BlogCloseService.java`, `src/main/java/com/blog/blog/api/BlogCloseController.java`에 만든다: 누른 날짜 + 7일 04:00(Asia/Seoul)을 `close_scheduled_at`에, `status` CLOSING, `close_reason` OWNER, 대기 중인 위임은 CANCELED, 블로그장을 뺀 멤버·구독자에게 즉시 알림(`blog_close_notices` IMMEDIATE), 철회는 그 블로그의 블로그장만 7일 안에 하고 같은 대상에게 철회 알림. 관리자 강제 폐쇄(US9)와 블로그장 박탈 폐쇄도 이 서비스를 쓴다 (BLG-09, D-12, D-13, D-67)
- [x] T089 [US5] 04:00 배치를 `src/main/java/com/blog/batch/DailyBatchJob.java`에 만든다: `@Scheduled(cron = "0 0 4 * * *", zone = "Asia/Seoul")` + `@SchedulerLock`. ① 예정 시각 지난 CLOSING → CLOSED ② 폐쇄까지 3일·1일 남은 블로그에 `blog_close_notices`에 없으면 알림 ③ 7일 지난 PENDING 위임 → EXPIRED, 블로그장에게 알림 (data-model.md 04:00 배치 1~3)
- [x] T090 [US5] 같은 배치에 정리 작업을 `src/main/java/com/blog/batch/RetentionCleanup.java`로 넣는다: ④ 30일 지난 삭제 글·댓글·사진 완전 삭제, 폐쇄 30일 지난 블로그의 글·사진·카테고리·블랙리스트 삭제와 `slug` NULL(행은 남김) ⑤ 탈퇴 30일 지난 회원의 name·nickname·phone·profile_image NULL ⑥ 보관 기간 지난 알림, 만료 인증번호·임시 토큰, 만료 Refresh Token 삭제 ⑦ 1년 지난 운영 기록 삭제 (4.5, D-40, D-60, D-83, D-86)
- [x] T091 [US5] 블로그 탈퇴 API `POST /api/blogs/{blogId}/leave`를 `src/main/java/com/blog/blog/api/BlogController.java`에 만든다: 블로그장은 거부, 떠난 멤버의 그 블로그 글은 `author_hidden = TRUE` (BLG-07, D-33)
- [x] T092 [US5] 회원 탈퇴 API `POST /api/me/withdraw`를 `src/main/java/com/blog/auth/service/WithdrawService.java`, `src/main/java/com/blog/auth/api/MeController.java`에 만든다: 비밀번호 재확인, 블로그장인 ACTIVE·CLOSING 블로그가 있으면 거부하고 그 목록과 위임·폐쇄 방법을 응답, 통과하면 `status` WITHDRAWN, `email` 즉시 NULL, Refresh Token 모두 폐기, 글은 "탈퇴한 회원" (USR-05, D-05, D-15, D-18)
- [x] T093 [P] [US5] 블로그 관리 화면의 위임·폐쇄 탭(위임할 멤버가 없으면 폐쇄 버튼만, 폐쇄 예정이면 철회 버튼만)과 위임 수락 화면을 `src/main/resources/static/js/blog-admin-close.js`, `static/transfer.html`에 만든다
- [x] T094 [P] [US5] 회원 탈퇴 화면을 `src/main/resources/static/withdraw.html`, `static/js/withdraw.js`에 만든다

**Checkpoint**: 위임·폐쇄·탈퇴와 04:00 배치가 동작한다

---

## Phase 8: User Story 6 - 계정을 찾고 정보를 고친다 (Priority: P2)

**Goal**: 이메일이나 비밀번호를 잊은 회원이 스스로 계정을 되찾고, 회원 정보를 고친다

**Independent Test**: 이름+전화번호로 이메일 찾기 → 가려진 이메일 옆 [비밀번호 재설정] → 인증번호 입력 → 새 비밀번호로 로그인, 다른 기기는 로그아웃되는지 확인한다

### Implementation for User Story 6

- [x] T095 [P] [US6] `account_find_tokens` 엔티티와 리포지토리를 `src/main/java/com/blog/auth/domain/AccountFindToken.java`, `src/main/java/com/blog/auth/repository/AccountFindTokenRepository.java`에 만든다: `token_hash`, 10분 만료
- [x] T096 [US6] 비밀번호 찾기 API `POST /api/auth/password/code`, `POST /api/auth/password/reset`을 `src/main/java/com/blog/auth/service/PasswordResetService.java`, `src/main/java/com/blog/auth/api/PasswordController.java`에 만든다: 가입 여부와 관계없이 같은 문구, 6자리 번호(30분, 1회용, 5번 틀리면 무효), 새 비밀번호 두 번 입력, 바뀌면 그 회원의 Refresh Token 모두 폐기와 로그인 잠금 해제 (USR-06, SEC-05, D-25, SC-003)
- [x] T097 [US6] 이메일 찾기 API `POST /api/auth/find-email`을 `src/main/java/com/blog/auth/service/FindEmailService.java`, `src/main/java/com/blog/auth/api/FindEmailController.java`에 만든다: 이름(앞뒤 공백 제거) + 전화번호(숫자만), 일치하는 계정의 가린 이메일과 가입일을 모두, 탈퇴 계정 제외, 계정마다 10분짜리 임시 토큰을 주고 [비밀번호 재설정]은 그 토큰으로 요청, 같은 IP 10분에 5번까지(T022) (USR-08, D-23, D-26, D-27)
- [x] T098 [US6] 내 정보 API `GET /api/me`, `PUT /api/me`를 `src/main/java/com/blog/auth/api/MeController.java`에 만든다: 본인에게는 이메일·이름·닉네임·전화번호를 그대로, 바꿀 수 있는 것은 닉네임·프로필 사진(3MB 1장)·소개·전화번호·비밀번호(현재 비밀번호 확인)뿐, 이메일과 이름은 불가, 비밀번호를 바꾸면 Refresh Token 모두 폐기 (USR-07, D-21, D-32)
- [x] T099 [P] [US6] 이메일 찾기·비밀번호 찾기·내 정보 화면을 `src/main/resources/static/find-email.html`, `static/find-password.html`, `static/me.html`과 각 js 파일에 만든다

**Checkpoint**: 계정 찾기와 정보 수정이 따로 동작한다

---

## Phase 9: User Story 7 - 사람과 블로그를 따라가고, 찾는다 (Priority: P2)

**Goal**: 회원이 다른 회원을 팔로우하고 블로그를 구독해 메인 피드에서 새 글을 보고, 통합 검색으로 블로그와 글을 찾는다

**Independent Test**: A가 B를 팔로우하고 C 블로그를 구독 → B와 C에 새 글 → A의 메인 피드 팔로우 탭·구독 탭에 각각 나오는지 확인한다

### Implementation for User Story 7

- [x] T100 [P] [US7] `user_follows`, `user_blocks`, `blog_subscriptions`, `recent_searches` 엔티티와 리포지토리를 `src/main/java/com/blog/social/domain/UserFollow.java`, `UserBlock.java`, `src/main/java/com/blog/blog/domain/BlogSubscription.java`, `src/main/java/com/blog/main/domain/RecentSearch.java`와 각 리포지토리에 만든다: follower·followee UNIQUE, blocker·blocked UNIQUE, 회원·블로그 UNIQUE, 최근 검색어는 본인만·최대 10개·같은 검색어면 시각만 갱신
- [x] T101 [US7] 팔로우·구독 API `POST /api/users/{id}/follow`, `POST /api/blogs/{blogId}/subscribe`(다시 누르면 취소)를 `src/main/java/com/blog/social/service/FollowService.java`, `src/main/java/com/blog/social/api/FollowController.java`에 만든다: AccountGuard 적용, 새 팔로워·블로그 구독 알림, 일부 공개 블로그는 링크로 들어온 회원만 구독 (SOC-01, D-02)
- [x] T102 [US7] 팔로워·팔로잉 목록 API `GET /api/users/{id}/followers|followings`를 `FollowController.java`에 만든다 (SOC-02)
- [x] T103 [US7] 프로필 API `GET /api/users/{id}/profile`을 `src/main/java/com/blog/social/api/ProfileController.java`에 만든다: 닉네임, 사진, 소개, 운영하는 블로그, 참여한 블로그, 팔로워 수만(이메일·이름·전화번호 없음) (SOC-03, 4.4)
- [x] T104 [US7] 차단 API `POST|DELETE /api/users/{id}/block`을 `src/main/java/com/blog/social/service/BlockService.java`, `src/main/java/com/blog/social/api/BlockController.java`에 만든다: 서로의 팔로우를 풀고, 차단한 회원의 글·댓글을 목록·피드·검색에서 숨김(같은 블로그 멤버끼리는 그 블로그 안 글은 보임) (SOC-05, D-36)
- [x] T105 [US7] 블로그장이 차단한 회원의 참여 신청 자동 거절을 `JoinService.java`에 넣는다 (SOC-05)
- [x] T106 [US7] 메인 피드 API `GET /api/feed?tab=latest|popular|following|subscribed`를 `src/main/java/com/blog/main/service/FeedService.java`, `src/main/java/com/blog/main/api/FeedController.java`에 만든다: 공개 블로그의 공개 글, 인기는 좋아요 수, 볼 때마다 모음, 차단한 회원 제외 (BRD-09, D-65, D-73, D-89)
- [x] T107 [US7] 통합 검색 API `GET /api/search?q=&type=blog|post|tag|nickname&sort=relevance|latest|popular`를 `src/main/java/com/blog/main/service/SearchService.java`, `src/main/java/com/blog/main/api/SearchController.java`에 만든다: 검색어 2~20자, 공백·특수문자만이면 거부, 블로그(이름·소개)·글 제목은 FULLTEXT + ngram, 태그·닉네임은 일치 검색, 비공개 블로그·숨김/삭제 글·내가 차단한 회원의 글 제외, 결과가 없으면 "검색 결과가 없습니다", 자동완성 없음 (BRD-08)
- [x] T108 [US7] 최근 검색어 API `GET|DELETE /api/me/recent-searches`(개별·전체 삭제)를 `SearchController.java`에 만든다 (BRD-08)
- [x] T109 [US7] 공개 → 비공개 전환 시 구독자에게 "비공개 전환" 알림, 멤버 아닌 구독자에게 글을 숨기는 처리를 `src/main/java/com/blog/blog/service/BlogService.java`의 정보 수정에 넣는다 (BLG-01, D-50)
- [x] T110 [P] [US7] 메인 피드 탭·통합 검색·프로필·팔로워 목록 화면을 `src/main/resources/static/index.html`, `static/search.html`, `static/profile.html`과 `static/js/feed.js`, `static/js/search.js`, `static/js/profile.js`에 만든다

**Checkpoint**: 팔로우·구독·피드·검색이 따로 동작한다

---

## Phase 10: User Story 8 - 알림을 받는다 (Priority: P2)

**Goal**: 회원이 종 아이콘을 눌러 탭별로 알림을 보고, 받을 알림 종류를 고른다

**Independent Test**: 댓글 알림을 끈 회원에게 댓글이 달려도 알림이 생기지 않고, 폐쇄 예정 알림은 끌 수 없는지 확인한다

### Implementation for User Story 8

- [x] T111 [US8] 알림 목록·읽음·전체 삭제 API `GET /api/notifications?tab=`, `POST /api/notifications/{id}/read`, `DELETE /api/notifications`를 `src/main/java/com/blog/social/api/NotificationController.java`에 만든다: 탭은 전체·댓글·좋아요·팔로우·구독·블로그·운영 (SOC-04)
- [x] T112 [US8] 안 읽은 알림 수 API `GET /api/notifications/unread-count`를 `NotificationController.java`에 만든다. 폴링용이라 30분 연장 활동으로 세지 않는다(ActivityPolicy) (D-72)
- [x] T113 [US8] 알림 설정 API `GET|PUT /api/me/notification-settings`와 보관 기간 `notification_keep_days` 30/7 변경을 `src/main/java/com/blog/social/api/NotificationSettingController.java`에 만든다: 끌 수 없는 알림(위임 요청, 폐쇄 예정, 내 글 삭제됨 등)은 설정에 나오지 않고 바꿀 수도 없다. 끄면 새 알림만 막고 받은 알림은 그대로 (3.6)
- [x] T114 [P] [US8] 종 아이콘 사이드바(탭, 읽은 알림 강조 해제, 30초 폴링에 `X-Auto-Request: true`)를 `src/main/resources/static/js/notifications.js`에, 알림 설정 화면을 `static/me-notifications.html`에 만든다

**Checkpoint**: 알림이 따로 동작한다

---

## Phase 11: User Story 9 - 신고하고, 메인 관리자가 처리한다 (Priority: P3)

**Goal**: 회원이 문제를 신고하고, 신고 대상에 따라 블로그장이나 메인 관리자가 처리한다. 메인 관리자는 블로그와 블로그장을 관리한다

**Independent Test**: 블로그장에 대한 신고 → 메인 관리자가 경고 3번 → 권한 박탈 → 가장 먼저 부블로그장이 된 사람에게 넘어가는지(없으면 7일 뒤 폐쇄 예정) 확인한다

### Implementation for User Story 9

- [x] T115 [P] [US9] `owner_sanctions`, `user_sanctions`, `admin_action_logs` 엔티티와 리포지토리를 `src/main/java/com/blog/admin/domain/OwnerSanction.java`, `UserSanction.java`, `AdminActionLog.java`와 각 리포지토리에 만든다: 블로그장 제재는 `type` WARNING/DEMOTION, 계정 제재는 `type` WARNING/PROFILE_RESET/SUSPENSION·`suspend_days` 3/14/30(영구 NULL)·`ends_at`·`report_id`·`admin_id`·`released_at`, 활동 기록은 `action`(예: OWNER_WARN, CLOSE_BLOG, VIEW_PRIVATE_INFO)·대상·사유
- [x] T116 [US9] 관리자 활동 기록 공통 처리를 `src/main/java/com/blog/admin/service/AdminActionLogger.java`에 만든다. 모든 관리자 조치 서비스가 부른다 (ADM-06, SEC-09)
- [x] T117 [US9] 회원 관리 API `GET /api/admin/users?q=`, `GET /api/admin/users/{id}`, `POST /api/admin/users/{id}/reveal`을 `src/main/java/com/blog/admin/api/AdminUserController.java`에 만든다: 이메일·전화번호는 기본 가림, "전체 보기"는 원문을 주고 VIEW_PRIVATE_INFO 기록 (ADM-01, ADM-06)
- [x] T118 [US9] 블로그 관리 API `GET /api/admin/blogs?q=`, `POST /api/admin/blogs/{id}/hide|unhide`, `POST /api/admin/blogs/{id}/close`를 `src/main/java/com/blog/admin/api/AdminBlogController.java`에 만든다: 블로그장·멤버 수·글 수 표시, 강제 폐쇄는 BlogCloseService로 `close_reason` ADMIN·누른 날짜 + 7일 04:00·같은 알림·그 블로그의 블로그장이 철회 가능 (ADM-02, D-96)
- [x] T119 [US9] 글·댓글 숨김·삭제 API `POST /api/admin/posts/{id}/hide|delete`, `POST /api/admin/comments/{id}/hide|delete`와 30일 안 복구 `POST /api/admin/posts/{id}/restore`를 `src/main/java/com/blog/admin/api/AdminContentController.java`에 만든다. 삭제하면 작성자에게 알림 (ADM-03, 4.5)
- [x] T120 [US9] 블로그장 경고·강퇴를 `src/main/java/com/blog/admin/service/OwnerSanctionService.java`에 만든다: 그 블로그에서 최근 1년 안 경고가 3번이거나 바로 강퇴하면 권한 박탈, 박탈된 사람은 MEMBER로, 가장 이른 `manager_since`의 부블로그장을 OWNER로, 없으면 BlogCloseService로 `close_reason` OWNER_DEMOTED와 "블로그장 권한 박탈로 인한 폐쇄 조치" 공지, 블로그장에게 알림 (ADM-07, D-30, D-54, D-97)
- [x] T121 [US9] 계정 제재를 `src/main/java/com/blog/admin/service/UserSanctionService.java`에 만든다: 경고, 프로필 초기화(닉네임을 "회원"+회원 번호로, 사진·소개 삭제), 계정 정지(3·14·30일·영구, `users.suspended_until`, 영구는 9999-12-31), 해제(`released_at`, `suspended_until` NULL), 대상에게 알림, 영구 정지면 그 사람이 블로그장인 블로그마다 OwnerSanctionService의 박탈과 같이 처리 (ADM-08, D-99, D-100)
- [x] T122 [US9] 블로그장 정지 중 처리를 넣는다: `src/main/java/com/blog/common/security/BlogAuthz.java`에서 계정 정지 중인 블로그장의 관리 권한을 막고(그동안 모든 부블로그장이 멤버 관리 권한과 신고 처리를 임시로 가짐, D-114), 블로그 응답에 "블로그장 정지 중 (끝나는 날짜)" 공지를 `src/main/java/com/blog/blog/service/BlogAccessService.java`에서 붙인다 (ADM-08, D-100, D-114)
- [x] T123 [US9] 관리자 신고 처리 API `GET /api/admin/reports`, `POST /api/admin/reports/{id}/resolve`를 `src/main/java/com/blog/admin/service/AdminReportService.java`, `src/main/java/com/blog/admin/api/AdminReportController.java`에 만든다: `handler_scope = ADMIN`만. 블로그·블로그장 신고는 NO_ISSUE/WARNED/OWNER_DEMOTED/BLOG_CLOSED, 메인 프로필 신고는 NO_ISSUE/WARNED/PROFILE_RESET/ACCOUNT_SUSPENDED, 블로그장 정지 중인 블로그의 신고는 NO_ISSUE/WARNED/SUSPENDED/KICKED(MemberSanctionService로 대신 처리). 신고자에게 결과 알림 (ADM-04, ADM-08)
- [x] T124 [US9] 통계 API `GET /api/admin/stats`(회원 수, 블로그 수, 일별 가입·글 수)를 `src/main/java/com/blog/admin/api/AdminStatsController.java`에 만든다 (ADM-05)
- [x] T125 [US9] 메인 공지 API `POST /api/admin/notices`를 `src/main/java/com/blog/admin/api/AdminNoticeController.java`에 만든다: `main` 블로그의 글(`is_notice = TRUE`)로 저장, 전체 회원에게 "공지사항" 알림 (BRD-10)
- [x] T126 [P] [US9] 관리자 화면(회원·블로그·글·신고·통계·공지·활동 기록)을 `src/main/resources/static/admin/index.html`, `static/admin/js/admin.js`에 만든다. `/admin/**` 주소는 SecurityConfig에서 ADMIN만

**Checkpoint**: 모든 사용자 스토리가 따로 동작한다

---

## Phase 12: Polish & Cross-Cutting Concerns

**Purpose**: 여러 스토리에 걸친 확인과 마무리

- [x] T127 블로그별 권한 통합 테스트를 `src/test/java/com/blog/security/BlogAuthzIntegrationTest.java`에 만든다: 블로그 A의 블로그장으로 블로그 B의 멤버 제재, 글 삭제, 정보 수정, 폐쇄를 호출하면 모두 403. 권한 없는 부블로그장, 그 블로그에서 정지된 멤버, 계정 정지된 블로그장도 거부(ADMIN의 관리 권한과 회원 활동은 T153에서 확인) (SEC-07, SC-002, plan.md)
- [ ] T128 [P] 응답 개인정보 점검: 모든 API 응답 DTO를 `src/main/java/com/blog/**/api/` 기준으로 훑어 권한 없는 사람에게 가리지 않은 이메일·전화번호가 없는지 확인하고, 비밀번호 칸이 어느 응답에도 없는지 확인한다 (SC-004, 4.4)
- [x] T129 [P] 보안 헤더와 CSP를 실제 화면(Toast UI Editor, Turnstile 스크립트 포함)에서 확인하고 `src/main/java/com/blog/common/config/SecurityConfig.java`의 CSP를 맞춘다 (SEC-12)
- [x] T130 [P] 동시 요청으로 공개 블로그 생성 제한이 지켜지는지 `src/test/java/com/blog/blog/BlogLimitConcurrencyTest.java`로 확인한다 (SC-006)
- [x] T131 메인·블로그 목록·검색 응답 시간을 측정해 1초 목표(늦어도 2초)를 넘는 쿼리에 `add_indexes.sql`의 인덱스를 다음 번호 Flyway 파일 `src/main/resources/db/migration/V6__query_indexes.sql`로 추가하고, 큰 행을 정렬하는 쿼리는 글 번호만 먼저 고르게 바꾼다. 측정 결과는 D-144 (SC-001, D-48, D-144)
- [ ] T132 [P] 서버 2대 대비 점검: 이미지 저장소·IP 요청 제한 저장소·프록시 헤더가 설정만으로 바뀌는지 `src/main/resources/application.yml`에서 확인한다 (SC-008, SCL-01~03)
- [x] T133 [P] 개인정보 처리방침 화면을 `src/main/resources/static/privacy.html`에 만든다(4.6 표 기준, 판매·제공 없음)
- [ ] T134 이 문서 저장소에 `specs/001-main-blog/quickstart.md`(실행·검증 절차)와 `specs/001-main-blog/contracts/`(local에서 받은 `/v3/api-docs` OpenAPI 명세, D-112)를 만들고 `/speckit-analyze`로 스펙·계획·작업이 맞는지 확인한다
- [x] T139 DB 백업을 만든다: 서버의 `ops/backup/db-backup.sh`(mysqldump `--single-transaction` 전체 백업, 서버에 7일치 남기고 지난 파일 삭제, 그 계정만 읽기)와, 매일 03:00 Asia/Seoul(cron 18:00 UTC)에 서버에 ssh로 접속해 이 스크립트를 돌리고 받은 파일을 저장소 비밀값 `BACKUP_PASSPHRASE`로 AES-256 암호화해 Actions 아티팩트로 7일 보관하는 Backup 워크플로 `.github/workflows/backup.yml`, 복구 절차 `ops/backup/RESTORE.md`. 전날 백업으로 빈 DB를 복구해 `/actuator/health`가 UP인지 확인한다 (OPS-01, SC-010, D-101, D-121)
- [ ] T140 [P] 지원 브라우저 확인: 가입, 로그인, 블로그 만들기, 글쓰기(Toast UI Editor), 글 상세, 알림 화면을 Windows·Mac Chrome, Mac Safari, Android Chrome, iPhone Safari 최신 버전에서 확인하고 깨지는 곳을 고친다 (OPS-06, D-106)
- [x] T141 [P] 오류·로그 점검: 없는 주소, 잘못된 입력, 일부러 낸 서버 오류의 응답과 화면에 스택 트레이스·SQL이 없는지 `src/test/java/com/blog/common/ErrorResponseTest.java`로 확인하고, 가입·로그인 실패·비밀번호 재설정을 한 번씩 한 뒤 로그 파일에 비밀번호·토큰·인증번호가 없는지 검색한다 (OPS-02, OPS-03, SC-009)
- [x] T147 [P] 남은 성공 기준 자동 테스트를 만든다: 가입·비밀번호 찾기 응답이 가입된 이메일과 없는 이메일에서 같은지(`src/test/java/com/blog/auth/AccountEnumerationTest.java`, SC-003), 회원·블로그·글 API 응답에 권한 없는 사람에게 가리지 않은 이메일·전화번호가 없는지(`src/test/java/com/blog/common/MaskingResponseTest.java`, SC-004), 같은 사람이 같은 날 같은 글을 여러 번 열면 조회수가 1만 오르는지(`src/test/java/com/blog/board/ViewCountTest.java`, SC-007), prod 프로필에서 Swagger 주소가 404이고 로그인 쿠키에 Secure가 붙으며 JWT 비밀키가 없으면 서버가 뜨지 않는지(`src/test/java/com/blog/common/ProdProfileTest.java`, SC-011) (OPS-09, D-109)

## Phase 13: 블로그 마스코트와 화면 디자인 (D-116~D-119)

**Purpose**: 2026-10-12에 정한 블로그 마스코트(BLG-14)와 정돈된 목록형 화면·밝은/어두운 모드를 넣는다

**Independent Test**: 태그가 "고양이"인 블로그를 만들면 저장은 바로 끝나고 잠시 뒤 블로그 첫 화면 제목 옆과 블로그 카드 이름 앞에 고양이 그림이 나타나고, 태그가 "개발"이면 컴퓨터 그림이 나타난다. API 키 없이 띄워도 태그 단어 사전으로 골라진다. 상단 메뉴의 버튼으로 어두운 모드로 바꾸고 새로고침해도 유지된다

- [x] T148 블로그 마스코트 고르기를 만든다: Flyway `src/main/resources/db/migration/V3__blog_mascot.sql`로 `blogs.mascot VARCHAR(40) NULL`을 `cover_image` 뒤에 추가하고(`erd_tables.sql` 기준, 이미 V3가 있으면 다음 번호), `Blog` 엔티티에 필드를 넣는다. `src/main/java/com/blog/blog/service/MascotService.java`가 블로그 만들기(T044)·정보 수정(T053) 저장이 커밋된 뒤 비동기로 블로그 이름과 태그만 Claude API에 보내 18종, 동물 10종(bear, cat, dog, rabbit, fox, penguin, bird, frog, hamster, owl)과 물건 8종(computer, camera, book, plane, headphones, gamepad, coffee, cake) 중 하나를 고른다(소개·글 내용은 보내지 않음). 주제가 물건과 뚜렷하게 맞으면 물건(개발 → computer, 사진 → camera, 독서 → book), 아니면 동물을 고르게 프롬프트에 적는다. API 키(환경변수 `ANTHROPIC_API_KEY`)가 없거나 호출이 실패하거나 목록에 없는 답이 오면 태그 단어 사전(`MascotDictionary.java`)으로, 맞는 단어도 없으면 이름의 해시로 정한 기본값을 쓴다. `mascot`이 NULL인 기존 블로그는 `@Scheduled` + `@SchedulerLock` 예약 작업이 한 번에 조금씩 채운다. Claude는 바탕과 함께 색·모자·장식도 고르고, 폐쇄되지 않은 다른 블로그나 이 블로그의 지금 값과 같은 조합이면 같은 바탕 안의 비어 있는 조합으로 바꿔 `bear.mint.beret.heart`처럼 저장한다(단어 사전으로 고를 때는 색·모자·장식을 이름·태그로 정함). 블로그 응답 DTO에 `mascot`을 넣는다 (BLG-14, D-116, D-118). → T169(D-143)로 바뀜: 바탕 30종, 후보 바탕 중 열린 블로그가 가장 적게 쓰는 바탕을 먼저 쓰고 한 바탕에 몰린 기존 블로그를 한 번 옮긴다
- [x] T149 [P] 마스코트 그림 18종(동물 10종, 물건 8종)의 픽셀 그림과 움직임을 `src/main/resources/static/img/mascot/`와 `static/js/mascot.js`에 만들고, 색 6가지로 다시 칠하기와 모자 8가지·장식 6가지 그림을 더한다. 블로그 첫 화면(`static/blog.html`, `static/js/blog.js`)에는 제목 옆에 크게, 메인·내 블로그 목록의 블로그 카드(`static/js/blog-list.js`, `static/my-blogs.html`)에는 이름 앞에 작게 보여 주고, `mascot`이 NULL이면 기본 그림을 보여 준다 (BLG-14, D-116). → T169(D-143)에서 30종(동물 20종, 물건 10종)으로 늘림
- [x] T150 [P] 화면 디자인을 정돈된 목록형(흰 바탕, 회색 구분선, 포인트 빨강 `#f04452`, 목록 중심)으로 `src/main/resources/static/css/common.css`에서 바꾸고, 밝은/어두운 모드를 CSS 변수로 둔다. 처음에는 `prefers-color-scheme`을 따르고, `static/js/layout.js`의 상단 메뉴 버튼으로 바꾸면 그 브라우저의 localStorage에 기억한다(로그인과 무관, 서버 변경 없음). 바꾼 뒤 T140의 브라우저에서 두 모드를 확인한다 (D-117)
- [x] T151 기본 대표 픽셀 장면을 만든다: Flyway로 `blogs.cover_scene VARCHAR(60) NULL`을 `mascot` 뒤에 추가하고(T148과 같은 V3), `MascotService`가 마스코트와 함께 Claude에게 장면·시간·날씨·색을 고르게 해 `study.night.rain.mint.4821`처럼 저장한다. 배치 번호는 저장할 때마다 새로 뽑고, Claude를 못 쓰면 마스코트 바탕에 맞는 장면을 쓴다. `mascot`이나 `cover_scene`이 NULL인 블로그는 예약 작업이 채운다. `static/js/pixel-cover.js`가 192x80 픽셀 장면 14가지를 그리고 움직이며(화면 밖이나 움직임 줄이기 설정이면 멈춤), 대표 이미지가 없는 블로그의 첫 화면(`static/blog.html`, `static/js/blog.js`)과 블로그 카드(`static/js/blog-card.js`)에 보여 준다. 블로그 응답 DTO에 `coverScene`을 넣는다 (BLG-15, D-119)
- [x] T155 Claude가 그린 대표 그림: `src/main/java/com/blog/blog/service/CoverArtService.java`가 블로그 만들기(T044) 저장이 커밋된 뒤 비동기로, 블로그장이 대표 이미지를 올리지 않았으면 블로그 이름과 태그만 Claude API(`ANTHROPIC_API_KEY`)에 보내 글자 없는 가로 그림 SVG(viewBox 1200×500)를 그리게 한다. `SvgSanitizer.java`는 허용한 요소·속성만 남기고(script·foreignObject·style·text·image·a, on* 속성, href·xlink:href, 외부 `url()` 제거), DOCTYPE·엔티티가 있거나 200KB를 넘거나 XML로 읽히지 않으면 버린다. 통과한 SVG는 `FileStorage`(SCL-02)에 UUID 이름으로 저장하고 `blogs.cover_image`에 경로를 넣는다. 그사이 블로그장이 이미지를 올렸으면 저장하지 않는다. 이 그림은 `image/svg+xml`과 `Content-Security-Policy: default-src 'none'; style-src 'unsafe-inline'; sandbox`, `X-Content-Type-Options: nosniff`로 보낸다. 이름·태그 수정(T053)에서는 다시 그리지 않고, 기존 블로그는 채우지 않는다. 블로그장이 대표 이미지를 지우면 `cover_image`를 NULL로 두고 픽셀 장면(T151)이 보인다. 회원 업로드(T062, T044의 대표 이미지)는 그대로 SVG를 거부한다. 테스트: 키가 없거나 호출이 실패하면 `cover_image`가 NULL이고 픽셀 장면이 보이는지, script·onload·외부 url()·DOCTYPE이 든 SVG가 걸러지거나 버려지는지, 블로그장이 올린 이미지를 덮어쓰지 않는지 (BLG-15, BRD-05, D-126)
- [x] T158 픽셀 대표 장면 넓히기: `CoverScene.java`에 장면 26개(library, rooftop, campsite, mountain, train, aquarium, atelier, workshop, flower-shop, observatory, bedroom, street-market, riverside, greenhouse, laundromat, classroom, gym, record-shop, convenience, office, subway, bus-stop, hanok, farm, space-station, pojangmacha)와 색 3개(ocean, forest, pastel), 소품 주제 15개(`Prop`, 여섯째 칸), 섞기 모드의 배경 20개(`Backdrop`)와 계절·행사 9개(`Event`)를 더하고 일곱째 칸 `배경~계절`을 읽고 쓴다(주제가 없으면 여섯째 칸 `-`, 가장 긴 값도 60자라 `blogs.cover_scene` VARCHAR(60) 그대로, Flyway 없음). `ClaudeMascotClient`의 구조화된 답에 `prop`·`backdrop`·`event`를 더하고 NONE이면 `MascotService.withLayers`가 태그 단어 사전(`KeywordScenePicker.java`) → 무작위(배경 10번 중 7번, 계절 20번 중 9번쯤)로 정하며, 크리스마스·핼러윈·생일은 태그에 있을 때만 붙인다. Claude를 못 쓰면 `KeywordScenePicker`로 장면·주제를 고르고, 없으면 `MascotService.scenesFor`의 바탕 그림별 장면 몇 개 중 하나를 이름·태그로 고른다. `static/js/pixel-cover.js`는 장면 40개·색 8개를 그리고, 섞기 모드면 배경·계절을 겹치고 벽·바닥 재질, 가구 색, 마스코트 자세, 좌우 뒤집기를 배치 번호로 고른다. 테스트: 일곱째 칸이 없는 예전 값(처음 14개 장면, 여섯 칸 값)은 그림이 픽셀까지 같은지, 모든 조합의 저장값이 60자 이하인지, 모든 장면에 사전 단어가 있는지, 크리스마스·핼러윈·생일이 태그 없이 붙지 않는지 (BLG-15, D-119, D-129). D-132로 넓힘: 섞기 모드의 구도 4가지(세로 띠를 잘라 순서 바꾸기, 창 모양·먼 풍경 높이), 배경 13개 추가(33개), 모니터·TV 화면 그림 12가지, 새 블로그는 배경을 늘 붙이고, `MascotService.refreshLegacyCovers`가 예약 작업(10분마다 100개, Claude 호출 없음)으로 일곱째 칸이 없는 열린 블로그의 장면을 한 번 새로 고른다 (D-132)
- [x] T163 운동 장면: `CoverScene.java`에 장면 SOCCER·TRACK·POOL·COURT·BASEBALL(저장값 soccer, track, pool, court, baseball)을 더하고 `static/js/pixel-cover.js`에 축구장(공 몰기), 달리기 트랙(달리기), 수영장(헤엄치기), 농구 코트(드리블과 슛), 야구장(타격)을 그린다. 섞기 모드의 배경·계절·배치 번호를 모두 쓰고, 골대·다이아몬드가 있는 장면은 큰 배치를 장면이 직접 고르며, 전광판 글자는 뒤집히지 않게 그린다. `KeywordScenePicker.java`: 축구·수영·러닝/마라톤/조깅·농구·야구는 종목 장면, 헬스·요가는 헬스장, 스키·스노보드·겨울·스케이트만 눈밭, 종목 없는 운동 단어는 `SPORT_POOL`(9개) 중 무작위, 운동 단어가 섞인 다른 말(머신러닝, 위스키, 독립운동 등)은 따로 처리한다. `ClaudeMascotClient`의 장면 설명을 고친다. `MascotService.refreshSportCovers`를 마스코트 채우기 예약 작업(ShedLock)에 붙여 운동 태그가 있는 블로그와 겨울 단어 없는 눈밭 블로그의 장면 칸만 한 번 다시 고른다(`BlogRepository.findIdsForSportRefresh`, 지금 장면이 그 태그로 고를 수 있으면 그대로, 폐쇄 블로그 건너뜀, AI 호출 없음). 릴리즈 노트 v1.25.0. 테스트: 종목 단어별 장면, 섞인 말 오탐 없음, 두 번째 실행에서 바뀌지 않는지, 예전 값 그림이 그대로인지 (BLG-15, D-137)
- [x] T152 무중단 배포: `deploy/deploy.sh`를 블루그린으로 바꾼다. 받은 포트에 전용 프록시(nginx 컨테이너)를 두고 서버 컨테이너 blue/green을 번갈아 띄우며, 새 버전이 UP이면 프록시를 reload해 넘기고 옛 버전을 graceful하게 내린다. 새 버전이 안 뜨면 새 버전만 지운다. Flyway 연결에는 `spring.flyway.init-sqls: SET SESSION lock_wait_timeout = 30`을 걸어, 누가 트랜잭션을 열어 둔 테이블을 고치는 마이그레이션이 30초 뒤 "Lock wait timeout exceeded"로 실패하고 새 버전이 뜨지 않게 한다(운영 중인 버전은 그대로). 새 버전이 UP이 되기를 기다리는 한도는 `HEALTH_TIMEOUT`(기본 240초, 4분)이다 (OPS-13, D-120, D-131) → T171(D-146)로 바뀜: 프록시 뒤 앱 두 대, 롤링
- [x] T156 릴리즈 노트: 별도 워크플로(`.github/workflows/release.yml`)가 운영 서버 배포(Deploy, T152)가 성공한 뒤 배포한 커밋에 GitHub Release를 만든다. 버전은 가장 최근 `v*` 태그의 마지막 자리를 1 올리고(태그가 없으면 v1.0.0), 그 커밋에 이미 릴리즈 태그가 있으면 새로 만들지 않는다. 내용은 GitHub 자동 릴리즈 노트(이전 릴리즈 뒤로 merge된 PR 제목)로 만들고, `.github/release.yml`에서 `dependencies` 라벨(Dependabot PR, T142)을 따로 묶는다. 배포가 실패하면 릴리즈를 만들지 않는다 (OPS-14, D-127). → T159(D-130)로 바뀜: 버전을 스스로 올리지 않고 `release-notes.json` 맨 위 버전으로 태그를 붙이며, 그 태그가 이미 있으면 건너뛴다. 본문은 파일의 바뀐 점 뒤에 자동 노트를 붙인다
- [x] T159 사이트 릴리즈 노트: `src/main/resources/release-notes.json`(새 버전이 맨 위, 버전·날짜·제목·PR 번호·바뀐 점)을 원본으로 두고, `src/main/java/com/blog/main/repository/ReleaseNoteRepository.java`가 서버가 뜰 때 읽는다(버전 형식 `v1.0.0`, 빈 제목, 같은 버전 두 번, 새 버전이 맨 위가 아니면 서버가 뜨지 않음). `GET /api/release-notes`(목록), `GET /api/release-notes/{version}`(바뀐 점, 없으면 404)을 `ReleaseNoteController.java`에 만들고 비회원에게 연다. 화면 `static/release-notes.html`(목록)·`static/release-note.html?v=버전`(상세)과 `static/js/release-notes.js`를 만들고, `static/js/layout.js` 아래쪽에 "릴리즈 노트" 링크를 둔다. 버전 규칙: 첫 PR을 v1.0.0으로 시작해 MAJOR는 아주 큰 업데이트만, MINOR는 기능 추가, PATCH는 버그 수정·작은 변경. `.github/workflows/release.yml`은 배포한 커밋의 파일 맨 위 버전·제목으로 태그와 GitHub Release를 만들고(본문은 파일의 바뀐 점 + `--generate-notes`), 그 태그가 이미 있으면 건너뛰며, 마지막 태그보다 낮으면 실패한다. 스스로 번호를 올리지 않는다. 테스트: API·화면이 비회원에게 열리는지, 없는 버전 404, 잘못된 파일을 거부하는지 (OPS-14, D-130)
- [x] T153 메인 관리자 권한: `src/main/java/com/blog/common/security/BlogAuthz.java`에서 `role = ADMIN`이면 폐쇄되지 않은 모든 블로그의 관리 권한(정보·대표 이미지·카테고리·공유 링크 수정, 참여 승인·거절, 멤버 경고·정지·강제 퇴장·해제, 블랙리스트, 글·댓글 관리, 블로그 안 신고 처리, 부블로그장 지정·권한 변경, 폐쇄·철회)을 블로그장과 같이 주고, 위임은 그 블로그의 진짜 블로그장(`blog_members.role = OWNER`)만 하게 한다. 관리만 하는 블로그에는 `blog_members`에 넣지 않는다. 관리자도 블로그 관리 화면에 들어가고 비공개 블로그를 읽게 한다. `AccountGuard`(T018)는 관리자에게 일반 회원 활동(블로그 만들기, 참여 신청, 참여한 블로그의 글쓰기, 댓글, 좋아요, 팔로우, 구독, 차단)을 모두 허용하고, 블로그 개수 제한(T045)에서 관리자를 뺀다. 글쓰기는 관리자도 그 블로그의 멤버여야 한다. 블로그 화면에서 관리자가 남의 블로그를 폐쇄하면 `close_reason = ADMIN`으로, 자기 블로그면 `OWNER`로 저장한다. 관리자 조치는 AdminActionLogger(T116)로 기록한다. 테스트: 관리자가 남의 블로그에서 멤버 정지·신고 처리·비공개 글 읽기가 되고 위임은 막히며, 블로그 만들기(4개째 공개 블로그 포함)·참여 신청·참여한 블로그의 글쓰기·댓글·좋아요·팔로우·구독이 되고, 참여하지 않은 블로그의 글쓰기는 막히며, 직접 만든 블로그는 위임·폐쇄(OWNER)가 되고 남의 블로그 폐쇄는 ADMIN으로 기록되는지 (SEC-09, BLG-10, ADM-02, D-122)
- [x] T154 블로그장 위임 요청 취소: `POST /api/blogs/{blogId}/transfers/pending/cancel`을 TransferService(T087)에 만들고 PENDING → CANCELED, 받는 멤버에게 "위임 요청 취소" 알림을 보낸다. 대기 중인 요청이 있으면 새 요청을 막고 "이미 ○○님에게 보낸 요청이 있어요 (언제까지)"를 돌려주며(D-123), 받는 멤버가 블로그를 떠나면 요청을 CANCELED로 바꾼다. 관리 화면(`static/js/blog-admin-close.js`)에 "요청 취소" 버튼을 둔다. 그 블로그에서 정지 중인 멤버의 블로그 탈퇴를 막는다(D-125). 테스트: 취소와 알림, 두 번째 요청 거부, 정지 중 탈퇴 거부 (BLG-07, BLG-08, D-123, D-124, D-125)
- [x] T157 구글·깃허브 소셜 로그인: Flyway `V4__user_social_accounts.sql`로 `user_social_accounts`를 만들고 `users.password_hash`를 NULL 가능으로 바꾼다(`erd_tables.sql` 기준, 테이블 추가와 NOT NULL 풀기만). `spring-boot-starter-security-oauth2-client`를 넣고, `SecurityConfig`(T016)는 키(`GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`, `GH_CLIENT_ID`/`GH_CLIENT_SECRET`)가 있는 제공자가 하나라도 있을 때만 `oauth2Login`을 켠다(`/oauth2/authorization/{google|github}` → `/login/oauth2/code/{google|github}`, 제공자 토큰은 보관하지 않음). `SocialLoginService.java`: 연결된 계정이면 그 회원으로, 아니면 제공자가 인증한 이메일과 같은 활성 회원에게 연결, 그것도 없으면 세션에 10분 담아 가입 마무리(`GET /api/auth/social/pending`, `POST /api/auth/social/signup`, `static/social-signup.html`: 이름 미리 채움·닉네임·필수 약관 동의, 비밀번호·전화번호 없음)로 보낸다. 인증 안 된 이메일은 거절하고, 로그인 잠금은 막고 계정 정지는 읽기만 허용한다. 돌아온 뒤에는 일반 로그인과 같은 JWT 쿠키(T035)를 만든다. `GET /api/auth/config`에 켜진 제공자 목록을 넣어 `login.html`·`signup.html`이 키가 있는 버튼만 보인다. 내 정보(T098, T099): `GET /api/me/social`, `POST /api/me/social/{provider}/link`, `DELETE /api/me/social/{provider}`와 `me.html`의 "소셜 로그인" 칸, `hasPassword`가 false면 현재 비밀번호 없이 "비밀번호 만들기", 비밀번호가 없으면 마지막 연결 해제 거부, 전화번호는 비워 둔 채 저장 가능. 비밀번호 없는 회원의 이메일·비밀번호 로그인(T034)은 비밀번호 틀림과 같게 실패(실패 횟수 셈). 탈퇴(T092)는 비밀번호가 없으면 확인을 건너뛰고 연결을 모두 지운다. 설정 방법은 서버 저장소 `docs/social-login.md`. 테스트: 키가 없으면 버튼·주소가 꺼지는지, 새 가입·이메일로 연결·이미 연결된 계정 로그인, 인증 안 된 이메일 거절, 다른 회원에게 연결된 계정 거절, 마지막 연결 해제 거부 (USR-01, USR-03, USR-05, USR-07, USR-09, SEC-01, D-128)
- [x] T160 https 고정: `src/main/java/com/blog/common/web/SiteUrl.java`에 사이트 주소(`app.site-url`, 환경변수 `SITE_URL`, prod 기본값은 운영 https 주소)를 두고, `require-https`인데 https가 아니면 서버가 뜨지 않게 한다. `SecurityConfig`(T016)는 prod(require-https)에서만 https 응답에 HSTS `max-age=31536000`을 붙이고 `includeSubDomains`·`preload`는 빼며, 프록시 뒤 http 요청은 이 주소의 https로 돌려보낸다. 소셜 로그인 돌아오는 주소(`SocialClientRegistrations`, T157)와 공유 미리보기 절대 주소(T161)를 이 값으로 만든다. 릴리즈 노트 v1.22.1. 테스트: `X-Forwarded-Proto` https면 HSTS, http면 https로 리다이렉트하고 Location이 http가 아닌지, local·test에는 HSTS가 없는지 (SEC-10, SEC-12, OPS-11, D-134)
- [x] T161 공유 미리보기 카드: `src/main/java/com/blog/main/api/SharePageController.java`가 메인(`/`), 블로그(`/blog/{주소}`), 글(`/blog/{주소}/posts/{번호}`) HTML의 `<title>`과 `<!--og-->` 자리에 Open Graph·트위터 카드 태그를 넣는다(값은 HTML 이스케이프, 예전 글 상세 og(T069)는 이리로 옮김). `ShareMetaService.java`는 비회원이 볼 수 있는 블로그·글만 제목·설명(본문 앞 100자 → 블로그 소개 → 사이트 소개)·그림을 채우고 나머지는 사이트 기본값. 그림은 글의 첫 비트맵 이미지 → 블로그 대표 이미지 → AI가 그린 SVG 대표 그림(T155)을 `SvgRasterizer.java`(JSVG)로 1200×630 PNG로 바꾼 것(`ShareImageController`의 `/api/og/covers/{UUID}.png`, 바깥 자원 읽지 않음, 동시에 2장, 메모리 64장, 30일 캐시) → `static/img/og-default.png`. 릴리즈 노트 v1.23.0. 테스트: 공개 글·블로그 카드, 비공개·일부 공개·삭제 글은 기본 카드, 그림 순서, SVG → PNG (BRD-07, SEC-06, D-135)
- [x] T162 이미지 최적화: `src/main/java/com/blog/board/service/ImageOptimizer.java`를 만들고 `ImageService`(T062)·블로그 대표 이미지·프로필 사진 업로드가 부르게 한다. 쓰임별 상한 POST·COVER 가로 1600px, PROFILE 512×512. JPEG·PNG는 EXIF 방향대로 세운 뒤 WebP(JPEG 손실 품질 80, PNG 무손실, 사진 같은 PNG 손실)로 쓰고 메타데이터를 모두 지운다. 줄이지도 돌리지도 않았는데 10% 넘게 작아지지 않거나 풀 수 없으면 메타데이터만 지운 원래 파일, GIF·WebP·APNG는 그대로(메타데이터만 지움). 5천만 화소를 넘으면 파일 머리만 보고 413. 동시에 푸는 요청 2개, 큰 그림은 솎아 읽기. 인코더는 webp-imageio, 못 쓰면 JPEG·PNG로 줄인다. 저장 그림 응답은 1년 immutable 캐시. 릴리즈 노트 v1.24.0. 테스트: 크기 상한, EXIF 회전·GPS 제거, 투명도 유지, 움직이는 그림 그대로, 화소 초과 413 (BRD-05, SEC-08, D-136)
- [x] T164 화면 빨리 띄우기: `application.yml`에 `server.compression`(HTML·JS·CSS·JSON·SVG, 1KB 이상)을 켠다. `src/main/java/com/blog/common/web/AssetVersions.java`가 JS·CSS 내용 해시를 구하고 `StaticResourceConfig.java`가 화면 HTML의 JS·CSS 주소에 `?v=`를 붙이며, 버전이 맞으면 1년 immutable, 버전이 없으면 ETag 확인, 화면 HTML은 no-cache, `/img/**`는 7일 캐시. `SharePageController`(T161)는 메인·블로그·글 HTML에 곧 부를 API를 `<link rel="preload" as="fetch">`로 넣고, 화면 HTML에 `Speculation-Rules` 헤더와 `/speculation-rules.json`(메인·블로그·글 링크 prefetch, 블로그 관리 제외, moderate)을 준다. `JsonSecurityErrorHandler`는 로그인 쿠키가 없는 401에 `X-Auth-Session: none`을 붙이고 `static/js/api.js`는 이때 재발급을 건너뛴다. `static/js/blog-card.js`·`blog.js`는 픽셀 장면을 화면에 가까워질 때·첫 화면을 그린 뒤 그린다. 테스트: gzip 응답, 버전 주소 캐시 헤더, preload 주소가 화면 JS 주소와 같은지, 탭 없는 메인 주소에서 500이 나지 않는지 (4.2, D-138)
- [x] T165 README와 Git Flow: 서버 저장소 README를 다시 쓴다(소개, 주요 기능, 기술 스택, 아키텍처·Git Flow 그림, 패키지 구조, 개발환경, 파일 업로드, 소셜 로그인, CI/CD 무중단 배포). 브랜치는 `main`·`develop`·`feature/*`·`release/vX.Y.Z`·`hotfix/*`. `.github/workflows/ci.yml`은 모든 PR과 main·develop 푸시에서 돌고 Deploy·Release(T152, T159)는 main만 이어진다. `.github/pull_request_template.md`(무엇을/왜, 관련 T-번호, 근거, 확인한 것)를 더하고, 배포 파일 주석에서 내부 주소를 뺀다. 저장소 관리자가 main·develop 브랜치 보호(PR 필수, build·migrations 통과)를 켠다 (OPS-05, OPS-14, D-139)
- [x] T166 새 댓글 자동 검사: Flyway `V5__comment_moderation.sql`로 `comments.review_status`·`review_reason`·`held_at`과 `idx_comments_review`를 더한다(`erd_tables.sql` 기준, NULL 가능 컬럼 추가만). `src/main/java/com/blog/board/service/ClaudeCommentFilterClient.java`(마스코트와 같은 키·모델, 구조화된 답 OK/SUSPECT + 이유, 댓글은 태그 안 데이터, 20초·다시 시도 1번, 실패면 빈 값)와 `CommentModerationService.java`(댓글 저장 커밋 뒤 작업 스레드에서 검사, 회원·블로그·전체 시간당 횟수 제한, 수상하면 HIDDEN + PENDING·댓글 수 빼기·블로그장과 글 관리 부블로그장에게 `COMMENT_REVIEW` 알림)를 만든다. `CommentService`(T067)는 블로그장·부블로그장·메인 관리자가 아니면 쓰거나 고친 뒤 검사를 부르고, 확인 대기 댓글은 쓴 사람에게만 보이며 쓴 사람은 고치기·지우기·답글을 그대로 한다. `GET /api/blogs/{id}/comment-reviews`, `POST .../{commentId}/approve`, `DELETE .../{commentId}`(글 관리 권한)를 `CommentReviewController.java`에, 화면은 `static/blog-admin.html`의 "댓글 확인" 탭과 `static/js/blog-admin-comments.js`에 둔다. 테스트: 가짜 서버로 답 읽기·실패·시간 초과·거절, 통합 테스트로 숨김·쓴 사람에게 보임·승인·삭제·고친 뒤 다시 숨김·fail-open·블로그장 댓글 건너뜀·횟수 제한 (BRD-06, SOC-04, D-133)
- [x] T167 글쓰기 도우미: `src/main/java/com/blog/board/api/WritingAssistController.java`에 `POST /api/posts/assist/tags`(본문 앞 3,000자로 태그 후보 3~5개, 그 블로그에서 쓰던 태그 먼저, `TagNormalizer`(T043)로 정리, 이미 단 태그 제외), `POST /api/posts/assist/spelling`(본문 앞 5,000자, 후보 `{start,end,original,suggestion,reason}`을 글 안에서 찾은 것만 30개까지), `GET /api/posts/assist/config`(버튼 표시 여부)를 둔다. `WritingAssistService.java`는 그 블로그에 글을 쓸 수 있는 회원만 허용하고 회원별 횟수 제한(`app.writing-assist`: 태그 1분 5번·하루 60번, 맞춤법 1분 3번·하루 20번, 넘으면 429)을 건다. `ClaudeWritingClient.java`는 마스코트와 같은 키·모델로 구조화된 답을 받고(글은 구분 블록 안 데이터, 태그 20초·맞춤법 50초, 트랜잭션 밖), 키가 없거나 실패하면 태그는 `KeywordTagSuggester.java`(단어 세기)로, 맞춤법은 503(`ASSIST_UNAVAILABLE`). 서버는 아무것도 저장하지 않는다. 화면은 `static/post-edit.html`, `static/js/writing-assist.js`, `static/css/writing-assist.css`: 태그 칩을 눌러 추가, 맞춤법 후보를 하나씩 적용/무시, 키가 없으면 맞춤법 버튼을 숨김. 테스트: 가짜 서버로 답 읽기·실패·시간 초과, 권한, 횟수 제한, 글 밖 후보 버림, 키 없을 때 단어 세기 (BRD-12, BRD-04, D-140, D-141)
- [x] T168 비슷한 글 추천: `GET /api/posts/{id}/similar?limit=4`(1~8)를 `src/main/java/com/blog/main/api/SimilarPostController.java`에 만든다. `SimilarPostService.java`는 원래 글을 글 상세와 같은 규칙(BlogAccessService)으로 확인하고, 비회원이 볼 수 있는 글만 후보로(공개 블로그, 게시 중, 블로그 공지 아님, 폐쇄·숨김 아님), 보는 사람이 차단한 작성자는 꺼낼 때 뺀다. `SimilarPostRepository.java`는 태그마다 `idx_post_tags_tag`로 최근 100개, 제목은 `ft_posts_title`로 30개를 읽는다. `SimilarPostRanker.java`: 겹치는 태그(1 + ln(표본/태그 글 수)/2) + 제목 bigram Dice × 2, 주제 점수 0.3 미만 제외, 최신·인기 가산 각 최대 0.3, 다른 블로그 먼저 블로그당 2개, 같은 블로그는 채우기용, 동점이면 최근 글. 결과는 글마다 메모리에 5분(`app.similar-posts.cache-ttl`, 최대 5,000개). 화면은 `static/js/similar-posts.js`가 글 끝이 가까워지면(IntersectionObserver) 불러와 카드(그림·제목·발췌·블로그 이름)를 그리고, 없으면 숨긴다. 스키마 변경 없음. 테스트: 순위 규칙, 비공개·숨김·공지 글 제외, 차단 작성자 제외, 블로그당 2개 (BRD-13, SOC-05, D-142)
- [x] T169 마스코트 바탕 늘리기와 고르게 나누기: `src/main/java/com/blog/blog/domain/Mascot.java`에 동물 10종(PANDA, KOALA, PIG, SHEEP, DUCK, TURTLE, HEDGEHOG, WHALE, LION, MOUSE)과 물건 2종(PENCIL, PLANT)을 더해 30종으로 한다(가장 긴 값도 34자라 `blogs.mascot` VARCHAR(40) 그대로, Flyway 없음). `static/js/mascot.js`에 새 바탕 12종의 픽셀 그림·이름을 그리고 순서를 동물 20종 먼저로, `static/css/mascot.css`의 깜빡임 엇갈림을 맞추고, `static/js/pixel-cover.js`가 장면 안에 새 동물 10종을 그린다. `ClaudeMascotClient.java`의 구조화된 답에 어울리는 다른 바탕(최대 2개)을 더하고 안내문에 새 바탕을 적는다. `KeywordMascotPicker.java`는 태그·이름에 맞는 바탕을 모두 돌려주고(`matches`), 맞는 단어가 없으면 이름으로 정한 기본값부터 모든 바탕을 후보로 내며, 새 바탕의 단어와 `MascotService.scenesFor`의 어울리는 장면을 더한다. `MascotService.leastUsed`는 후보 중 폐쇄되지 않은 블로그가 가장 적게 쓰는 바탕을 고르고(같으면 앞 후보, 이 블로그의 지금 바탕은 세지 않음) 그 바탕 안에서 겹치지 않는 조합(T148)을 쓴다. 바탕별 수는 `BlogRepository.countMascotBases`(쿼리 하나, 바탕만 있는 예전 값도 셈)로 센다. `MascotService.rebalanceMascots`를 마스코트 채우기 예약 작업(ShedLock, 10분마다)에 붙여, 몫(열린 블로그 수 ÷ 30, 올림)을 넘는 바탕의 블로그(`BlogRepository.findOpenIdsByMascotBase`)를 단어 사전에 맞는 바탕, 없으면 동물 중 몫보다 적게 쓰는 바탕에서 가장 적게 쓰는 것으로 한 번에 50개까지 옮긴다. 옮길 곳이 없으면 그대로 두고, `blogs.mascot`만 바꾸며(색·모자·장식은 그대로 두되 겹치면 다른 조합) 장면·대표 이미지는 그대로, AI 호출 없음, 폐쇄 블로그 건너뜀. 테스트: `MascotBalanceTest`(덜 쓰인 바탕 고르기, 몫을 넘는 바탕 옮기기, 몇 번 돈 뒤 더 바뀌지 않는지, 장면·대표 이미지 그대로), `MascotDesignTest`(30종 저장값 읽기·길이), `ClaudeMascotClientTest`(다른 바탕 답 읽기), `KeywordMascotPickerTest`(맞는 바탕 모두), `MascotServiceTest` (BLG-14, D-143)
- [x] T170 테스트 커버리지를 소나 기준(줄+분기) 90% 넘게 올린다: `build.gradle`에 JaCoCo(HTML·XML 보고서)를 붙이고, 빈 곳이 큰 이미지 검사·변환, 신고·관리자, 위임·인증·블랙리스트·알림에 정책 규칙과 실패 경우를 확인하는 테스트를 더한다(값만 채우는 테스트는 쓰지 않음). 테스트로 찾은 버그를 같이 고친다: PNG 조각 길이 넘침(500), 대신 쓰는 길의 뒤집힌 사진 방향, 신고 당시 내용의 이모지 깨짐, 관리자 박탈 때 위임 요청 취소 (D-145, BLG-08, ADM-07, OPS-09)
- [x] T171 서버 이중화와 롤링 배포: `deploy/deploy.sh`가 받은 포트의 전용 프록시(nginx 컨테이너) 뒤에 앱 컨테이너 app-1·app-2(`REPLICAS`, 기본 2)를 늘 같이 띄우고 한 대씩 바꾼다(프록시가 넘겨주지 않는 자리부터, 바꿀 앱을 프록시에서 빼고 reload → 30초 안에 마치게 한 뒤 내림 → 새 이미지 → health UP 4분 → 프록시에 다시 넣음 → 다음 대). UP이 안 되면 그 컨테이너만 지우고 멈춘다. 프록시는 같은 IP를 같은 앱으로(X-Real-IP 해시), 실패한 앱은 다른 앱으로 넘기고(proxy_next_upstream), 컨테이너 주소가 바뀌어도 따라간다(resolve). 블루그린의 blue/green은 첫 배포 때 내린다. 서버 nginx 설정과 secret은 그대로다. 로컬 도커로 블루그린에서 바꾸기, 롤링을 부하를 걸며 확인 (OPS-13, SCL-01, D-146)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 바로 시작
- **Foundational (Phase 2)**: Setup이 끝나야 함. 모든 사용자 스토리를 막음
- **User Stories (Phase 3~11)**: Foundational이 끝나면 시작
- **Polish (Phase 12)**: 원하는 스토리가 끝난 뒤
- **마스코트·화면 디자인 (Phase 13)**: T148은 US2(블로그 만들기 T044, 정보 수정 T053)와 ShedLock(T024) 뒤, T149는 T148(응답의 `mascot`)과 US2 화면(T054, T055) 뒤, T150은 공통 화면 틀(T027) 뒤면 언제든, T155는 T044·T151 뒤, T156은 T152 뒤, T157은 US1(T034, T035, T038, T039)·US6(T098, T099)·탈퇴(T092) 뒤, T158은 T151 뒤, T159는 T156 뒤, T160은 T016·T157 뒤, T161은 T069·T155·T160 뒤, T162는 T062 뒤, T163은 T158 뒤, T164는 T161 뒤, T165는 T135·T152·T159 뒤, T166은 T067 뒤, T167은 T043·T063 뒤, T168은 T065·T100 뒤, T169는 T148·T158 뒤, T171은 T152·T165 뒤

### User Story Dependencies

스토리끼리 실제로 기대는 곳만 적었다. 화살표 왼쪽이 먼저다.

- **US1 (P1)**: Foundational 뒤 바로. 다른 스토리에 기대지 않음
- **US2 (P1)**: US1(로그인한 회원) → US2
- **US3 (P1)**: US2(블로그·멤버, TagNormalizer T043, BlogAccessService T047) → US3
- **US4 (P2)**: US2(참여 신청 JoinService), US3(신고할 글·댓글) → US4
- **US5 (P2)**: US2 → US5. 구독자 알림은 US7의 구독이 있으면 함께 가고, 없어도 멤버 알림으로 확인 가능
- **US6 (P2)**: US1 → US6. US2~US5와 따로
- **US7 (P2)**: US3(글) → US7
- **US8 (P2)**: Foundational의 NotificationService(T026)만 있으면 됨. 알림을 만드는 스토리가 하나 이상 있어야 확인 가능
- **US9 (P3)**: US4(신고 접수 T075, MemberSanctionService), US5(BlogCloseService T088) → US9

```text
Setup → Foundational → US1 → US2 → US3 ─┬─ US4 ─┐
                         │       │       ├─ US7   ├─ US9 → Polish
                         │       └─ US5 ─┼───────┘
                         └─ US6          └─ US8
```

### Within Each User Story

- 엔티티 → 서비스 → API → 화면
- `[P]` 엔티티와 화면은 동시에, 같은 서비스 파일을 고치는 작업은 순서대로
- 스토리 Checkpoint에서 Independent Test를 해 보고 다음으로

### Parallel Opportunities

- Setup: T003, T004, T005, T135, T142, T143
- Foundational: T008~T013, T019~T025, T027, T136, T146 (T014는 T010~T013 뒤, T015~T018은 T014 뒤, T026은 T025 뒤, T137은 T019 뒤, T138·T144는 T016 뒤, T145는 T144 뒤)
- Foundational이 끝나면 US1과 함께 US6의 T095를 시작할 수 있고, US3이 끝나면 US4·US5·US7을 사람별로 나눠 동시에 할 수 있다
- 각 스토리의 `[P]` 엔티티 작업과 `[P]` 화면 작업
- Phase 13: T150은 T148·T149와 동시에 할 수 있다(서버를 건드리지 않음)

---

## Parallel Example: User Story 3

```bash
# 엔티티 네 개를 동시에:
Task: "posts 엔티티와 리포지토리 in src/main/java/com/blog/board/domain/Post.java"
Task: "post_tags, post_images 엔티티 in src/main/java/com/blog/board/domain/PostTag.java, PostImage.java"
Task: "comments 엔티티 in src/main/java/com/blog/board/domain/Comment.java"
Task: "post_likes, post_views, categories 엔티티 in src/main/java/com/blog/board/domain/"

# API가 끝난 뒤 화면 두 개를 동시에:
Task: "글쓰기 화면 in src/main/resources/static/post-edit.html"
Task: "글 목록·상세 화면 in src/main/resources/static/js/post-list.js, post-view.js"
```

---

## Implementation Strategy

### MVP First (US1~US3)

US1만으로는 가입·로그인뿐이라 보여 줄 것이 없어서, P1 세 개를 MVP로 묶는다.

1. Phase 1 Setup
2. Phase 2 Foundational (모든 스토리를 막음)
3. Phase 3~5: US1 → US2 → US3
4. **멈추고 확인**: 가입 → 블로그 만들기 → 다른 회원 참여 → 글쓰기·댓글·좋아요 → 비회원 읽기
5. 시연

### Incremental Delivery

1. Setup + Foundational → 기반
2. US1 → US2 → US3 → MVP 시연
3. US4·US5 → 블로그 운영과 폐쇄 (04:00 배치 테스트 T085 포함)
4. US6·US7·US8 → 계정 찾기, 소셜, 알림
5. US9 → 메인 관리자
6. Polish → 권한 통합 테스트 T127 등
7. 마스코트·화면 디자인 → T148~T151 (D-116~D-119), Claude가 그린 대표 그림 → T155 (D-126), 픽셀 장면 넓히기 → T158 (D-129), 무중단 배포 → T152 (D-120, D-131), 릴리즈 노트 → T156 (D-127), T159 (D-130), 메인 관리자 권한·위임 요청 취소 → T153~T154 (D-122~D-125), 소셜 로그인 → T157 (D-128), 댓글 자동 검사 → T166 (D-133), https·공유 카드·이미지·속도 → T160~T162, T164 (D-134~D-136, D-138), 운동 장면 → T163 (D-137), Git Flow → T165 (D-139), 글쓰기 도우미 → T167 (D-140, D-141), 비슷한 글 → T168 (D-142), 마스코트 바탕 늘리기·고르게 나누기 → T169 (D-143), 테스트 커버리지·박탈 때 위임 취소 → T170 (D-145), 서버 이중화·롤링 배포 → T171 (D-146)

### Parallel Team Strategy

1. Setup + Foundational을 다 같이
2. US1 → US2 → US3을 함께 끝낸 뒤 나눈다:
   - 개발자 A: US4 → US9
   - 개발자 B: US5 → US8
   - 개발자 C: US6 → US7

---

## Notes

- 작업 수 171개 (Setup 8, Foundational 28, US1 13, US2 16, US3 15, US4 13, US5 10, US6 5, US7 11, US8 4, US9 12, Polish 12, 마스코트·화면 디자인·대표 그림·픽셀 장면 8, 무중단 배포·릴리즈 노트 3, 메인 관리자 권한·위임 요청 취소 2, 소셜 로그인 1, https·공유 카드·이미지·속도 4, Git Flow 1, 댓글 검사·글쓰기 도우미·비슷한 글 3, 테스트 커버리지 1, 서버 이중화 1)
- T135~T147은 운영 요구사항(OPS, D-101~D-112)을 넣으며 뒤에 붙인 번호라 단계 안의 번호 순서와 실행 순서가 다를 수 있다. 단계는 각 작업이 놓인 Phase를 따른다
- T148~T151은 블로그 마스코트(BLG-14, D-116, D-118), 화면 디자인(D-117), 기본 대표 픽셀 장면(BLG-15, D-119)을 정하며 Phase 13으로 붙였다
- T153~T154는 메인 관리자 권한(D-122)과 위임 요청 취소·정지 중 탈퇴 불가(D-123~D-125)를 정하며 Phase 13 끝에 붙였다. T153은 T017·T018·T045·T116 뒤, T154는 T087·T093 뒤
- T155(대표 그림, D-126)는 T151 바로 다음에, T156(릴리즈 노트, D-127)은 T152 바로 다음에 두어 번호와 놓인 순서가 다르다. T155는 T044·T151 뒤, T156은 T152 뒤
- T157~T159는 소셜 로그인(D-128), 픽셀 장면 넓히기(D-129), 사이트 릴리즈 노트(D-130)를 서버에 넣은 뒤 기록했다. T158은 T155 다음에, T159는 T156 다음에, T157은 Phase 13 끝에 두었다. T156의 자동 번호 올리기는 T159가 바꿨다. 배포 중 DB 잠금 대기 제한(D-131)은 새 작업 없이 T152에 넣었다
- T166(새 댓글 자동 검사, D-133)은 서버에 넣으며 기록했다. T067 뒤면 되고 Phase 13 끝 T157 다음에 두었다
- T160~T165, T167~T169는 2026-10-13에 서버에 넣은 뒤 기록했다(https 고정 D-134, 공유 카드 D-135, 이미지 최적화 D-136, 운동 장면 D-137, 화면 빨리 띄우기 D-138, Git Flow D-139, 글쓰기 도우미 D-140·D-141, 비슷한 글 D-142, 마스코트 바탕 고르게 나누기 D-143). T170(테스트 커버리지, 박탈 때 위임 취소 D-145)도 2026-10-13. T163은 T158 다음에, 나머지는 Phase 13 끝에 번호 순서로 두어 T166이 T165와 T167 사이에 있다
- T171(서버 이중화와 롤링 배포, D-146)은 2026-10-14에 기록했다. T152의 블루그린을 T171이 바꿨다. Phase 13 끝 T170 다음에 두었다
- 요구사항 ID(USR, BLG, SOC, BRD, ADM, SEC, SCL, OPS)와 결정 기록(D-번호)을 작업 끝에 적어 근거를 찾을 수 있게 했다
- 테이블·컬럼을 바꿔야 하면 코드보다 먼저 이 저장소의 `erd_tables.sql`과 관련 파일을 고친다(CLAUDE.md)
- 작업 하나나 묶음마다 커밋하고, Checkpoint에서 스토리를 따로 확인한다
