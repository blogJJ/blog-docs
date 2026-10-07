# Implementation Plan: 메인블로그 플랫폼

**Branch**: `001-main-blog` | **Date**: 2026-10-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-main-blog/spec.md`

## Summary

회원이 메인에서 가입하고, 블로그를 만들거나 참여하고, 공통 게시판 모듈로 글을 쓰는 티스토리형 블로그 플랫폼의 메인 부분이다. Spring Boot + Spring Security 서버가 API와 정적 화면을 내려주고, MySQL 8 한 곳에 모든 상태를 둔다. 보안(JWT HttpOnly 쿠키, 블로그별 역할 확인)과 이중화 대비(무상태 서버, ShedLock, FileStorage)를 처음부터 지킨다. 결정 근거는 [research.md](research.md), 데이터는 [data-model.md](data-model.md).

## Technical Context

**Language/Version**: Java + Spring Boot (버전은 코드 저장소에서 정함)

**Primary Dependencies**: Spring Security(인증·인가·CSRF·보안 헤더), Spring Mail(JavaMailSender, Gmail SMTP), jsoup(본문 걸러내기), 마크다운 → HTML 변환기, ShedLock, Cloudflare Turnstile(사람 확인), Toast UI Editor(화면), Flyway(DB 변경 관리, OPS-08), Spring Boot Actuator(상태 확인, OPS-04), springdoc-openapi(API 문서, local에서만, OPS-12)

**Storage**: MySQL 8 (InnoDB, utf8mb4, FULLTEXT + ngram). 이미지는 서버 디스크(`FileStorage` 뒤, 이중화 때 S3)

**Testing**: JUnit + Spring Boot Test. DB를 쓰는 테스트는 Testcontainers로 띄운 MySQL 8(OPS-09). 권한(SEC-07)·04:00 배치를 포함해 SC-002~SC-007, SC-009, SC-011은 자동 테스트로 확인. PR과 main 푸시마다 GitHub Actions로 빌드·테스트(OPS-05), Dependabot이 매주 업데이트 PR(OPS-10)

**Target Platform**: Linux 서버 1대(AWS), 이중화 때 ALB + 2대 이상

**Project Type**: 웹 서비스 (API 서버 + 정적 HTML/JS 화면)

**Performance Goals**: 메인·목록·검색 1초 목표, 늦어도 2초 (D-48)

**Constraints**: 5초 넘는 요청은 중단하고 다시 시도 안내. 서버 메모리에 상태를 두지 않음(IP 요청 제한만 예외). 시간대는 Asia/Seoul. 1차에 Redis 없음. 설정은 local·prod 프로필로 나누고 비밀값은 환경변수로만 받음(OPS-11)

**Scale/Scope**: 1차 서버 1대, 테이블 33개, 요구사항 USR 8 · BLG 13 · SOC 6 · BRD 11 · ADM 8 · SEC 13 · SCL 3 · OPS 12

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

[constitution.md](../../.specify/memory/constitution.md) v1.0.0 기준.

| 원칙 | 확인 | 결과 |
| --- | --- | --- |
| I. 보안이 먼저다 | bcrypt(SEC-01), JWT HttpOnly 쿠키 + DB Refresh Token(SEC-04), Spring Security(SEC-10~12), jsoup 서버 걸러내기(SEC-06), 가입 여부 비노출(USR-02, 06) | ✅ |
| II. 역할은 블로그마다 다르다 | `blog_members.role` + `blog_manager_permissions`, 요청마다 DB 확인, `@PreAuthorize`(SEC-07, 11), 정지는 `blog_members.suspended_until`로 블로그 단위 | ✅ |
| III. 개인정보는 필요한 만큼, 서버에서 가린다 | 수집 항목 4.6, 서버 마스킹 4.4, 관리자 전체 보기 기록(ADM-06), 30일 삭제 배치 4.5, 블랙리스트 해시 | ✅ |
| IV. 서버를 늘려도 코드를 고치지 않는다 | 상태는 DB(SCL-01), `FileStorage`(SCL-02), ShedLock(SCL-03), 프록시 헤더는 설정으로 | ✅ |
| V. 단순한 것을 고른다 | 폴링 알림, 볼 때마다 피드, DB 조회 기록, 서버 디스크 이미지, Gmail SMTP | ✅ |
| VI. 결정은 기록으로 남긴다 | research.md에 D-01~D-112. 요구사항분석서와 DB 설계 사이에 남은 차이 없음(Clarifications 2026-10-07, 팀원 검토까지 모두 해결) | ✅ |

위반으로 정당화할 복잡도는 없다(Complexity Tracking 비움).

## Project Structure

### Documentation (this feature)

```text
specs/001-main-blog/
├── spec.md              # 요구사항 (요구사항분석서 1·2를 Spec Kit 형식으로)
├── plan.md              # 이 파일
├── research.md          # 구현 방식 결정, 이중화 후보, 결정 기록 D-01~D-112
├── data-model.md        # 테이블 33개, 상태 전이, 검증 규칙
├── quickstart.md        # (아직 없음) 코드 저장소가 생기면 실행·검증 절차
├── contracts/           # (아직 없음) API 명세
└── tasks.md             # 구현 작업 147개 (/speckit-tasks)

docs/
├── 01-requirements/     # 요구사항분석서 1·2 PDF (근거 자료)
└── 02-database/         # 요구사항분석서 3(DB 설계 PDF), 기준 스키마 SQL, ERDCloud 파일
```

### Source Code

이 저장소는 기획·설계 문서만 둔다. 코드는 별도 저장소에서 만들며, 그 저장소의 구조가 정해지면 이 절을 채운다. 요구사항 모듈 기준으로 아래처럼 나누는 것을 제안한다.

```text
src/main/java/.../
├── auth/          # USR-01~08, SEC-01~05, 13 (가입, 로그인, JWT, 인증번호, 찾기)
├── blog/          # BLG-01~13 (블로그, 멤버, 참여, 위임, 폐쇄, 블랙리스트, 제재)
├── board/         # BRD-01~07, 11 (공통 게시판: 글, 댓글, 태그, 이미지, 조회수)
├── main/          # BRD-08~10 (통합 검색, 메인 피드, 공지)
├── social/        # SOC-01~06 (팔로우, 구독, 프로필, 알림, 차단, 신고)
├── admin/         # ADM-01~08
├── batch/         # 04:00 배치 (폐쇄, 알림, 30일 삭제) + ShedLock
└── common/        # Spring Security 설정, 마스킹, FileStorage, 요청 제한

src/main/resources/
├── static/        # 정적 HTML + JS (공통 api.js에서 X-XSRF-TOKEN 처리)
└── db/migration/  # Flyway SQL (docs/02-database/schema/erd_tables.sql에서 시작)
```

**Structure Decision**: 단일 Spring Boot 프로젝트. 화면은 같은 서버의 정적 파일로 내려주고 글 상세만 og 태그를 서버가 채운다(D-66).

## 다음 단계

1. 요구사항이 바뀌면 PDF와 함께 spec.md·research.md를 고친다. `/speckit-clarify`로 남은 모호한 곳을 더 찾을 수 있다. 남은 보류 항목은 이중화 기술(서버를 늘릴 때) 하나다.
2. 코드 저장소 구조가 정해지면 `/speckit-plan`으로 이 문서의 Source Code 절과 `contracts/`(API 명세), `quickstart.md`를 채운다.
3. [tasks.md](tasks.md)의 작업 순서대로 구현한다(MVP는 US1~US3). `contracts/`와 `quickstart.md`를 채운 뒤 `/speckit-analyze`로 스펙·계획·작업이 서로 맞는지 확인한다.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

해당 없음.
