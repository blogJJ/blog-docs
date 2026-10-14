# Quickstart: 메인블로그 실행·검증

blog-server(코드 저장소)를 내 컴퓨터에서 띄우고, 성공 기준(spec.md SC-001~SC-011)을 어떻게 확인하는지 정리했습니다. 기준은 2026-10-14 main(v1.29.0)입니다. 자세한 설정과 배포는 [blog-server README](https://github.com/blogJJ/blog-server#readme)에 있습니다.

## 1. 준비

| 필요한 것 | 버전 |
| --- | --- |
| JDK | 21 |
| Docker | 로컬 MySQL과 통합 테스트(Testcontainers)에 필요 |
| Git | blogJJ/blog-server 받기 |

Gradle은 저장소의 wrapper(`./gradlew`, 9.8.1)를 씁니다.

## 2. 로컬에서 띄우기

```bash
git clone https://github.com/blogJJ/blog-server.git
cd blog-server
docker compose up -d          # MySQL 8.4 (blog DB, utf8mb4, Asia/Seoul, ngram-token-size=2)
cp .env.example .env          # JWT_SECRET을 채운다 (예: openssl rand -base64 48)
./gradlew bootRun             # local 프로필, http://localhost:8080
```

- 처음 뜰 때 Flyway가 테이블 34개를 만듭니다(V1~V6). 기준은 [erd_tables.sql](../../docs/02-database/schema/erd_tables.sql)입니다.
- 시험 데이터는 자동으로 넣지 않습니다. 필요하면 `docker exec -i blog-mysql mysql -ublog -pblog blog < dummy/dummy_data.sql`로 넣고, 시험 계정은 그 파일 맨 위 주석에 있습니다.
- local 프로필은 메일을 보내지 않고 콘솔에 인증번호를 찍습니다. Turnstile은 Cloudflare 시험용 키(항상 통과)를 씁니다.
- `ANTHROPIC_API_KEY`가 없어도 됩니다. 마스코트·글쓰기 도우미·댓글 검사는 대체 동작으로 돕니다.

## 3. 띄운 뒤 확인

| 확인 | 방법 | 기대 결과 |
| --- | --- | --- |
| 상태 | `curl -s http://localhost:8080/actuator/health` | `{"status":"UP"}` |
| API 명세 | http://localhost:8080/swagger-ui.html | 요청 133개. 저장해 둔 명세는 [contracts/](contracts/README.md) |
| 가입 | 화면에서 가입 → 콘솔의 인증번호 입력 | 가입 후 로그인 상태 |
| 블로그·글 | 블로그 만들기 → 글쓰기(이미지 포함) → 글 상세 | 글과 이미지가 보이고, 메인 피드 최신 탭에 나옴 |
| 메인 관리자 | dummy 데이터의 관리자 계정으로 `/admin` | 신고·회원·블로그 관리 화면 |

## 4. 자동 테스트

```bash
./gradlew build    # spotlessCheck + 컴파일 + 테스트(약 610개) + jar
```

통합 테스트는 Testcontainers로 MySQL 8.4를 띄우므로 Docker가 켜져 있어야 합니다. 커버리지 보고서는 `build/reports/jacoco/test/html/index.html`입니다. PR마다 GitHub Actions의 `build`·`migrations`가 같은 검사를 하고, 둘이 통과해야 main에 합칠 수 있습니다.

## 5. 성공 기준 확인 위치

| 기준 | 확인하는 곳 |
| --- | --- |
| SC-001 응답 1초, 5초 넘으면 다시 시도 안내 | `TimeoutConfigTest`(5초 제한). 응답 시간은 글 30만 개로 직접 잰 결과(D-144, T131) |
| SC-002 남의 블로그 관리 거부 | `BlogAuthzIntegrationTest`(실제 API 403), `BlogAuthzTest` |
| SC-003 가입 여부를 알 수 없음 | `AccountFlowTest`(없는 이메일도 같은 답), `AuthFlowTest` |
| SC-004 이메일·전화번호 가리기 | `MaskingResponseTest`, `MaskingTest`. 응답 DTO 점검 기록은 [tasks.md](tasks.md) T128 |
| SC-005 폐쇄 알림 3번과 04:00 폐쇄 | `DailyBatchIntegrationTest` |
| SC-006 공개 블로그 개수 제한(동시 요청) | `BlogFlowTest` |
| SC-007 조회수 하루 1번 | `PostFlowTest` |
| SC-008 앱 2대로 늘리기 | 운영에서 앱 두 대 롤링 배포로 확인(D-146, D-148). 이미지 폴더는 도커 볼륨을 같이 쓰고, 요청 제한은 IP 해시로 같은 앱에 보냄 |
| SC-009 오류 화면·로그에 내부 정보와 비밀값 없음 | `GlobalExceptionHandlerTest`, `ErrorPagesHttpTest`, `AuthFlowTest`(가입·로그인 실패·비밀번호 재설정 뒤 로그 검사) |
| SC-010 백업 하나로 복구 | blog-server `ops/backup/RESTORE.md` 순서로 직접 연습 |
| SC-011 prod에서 Swagger 404, 쿠키 Secure, 비밀키 없으면 안 뜸 | `ProdProfileTest`, `ProdSecretsCheckTest` |

## 6. 운영

운영 사이트는 https://jj-blog.java21.net 입니다. main에 합치면 CI → Deploy(앱 두 대를 한 대씩 교체) → Release(릴리즈 노트 버전 태그) 순서로 자동으로 올라갑니다. 매일 03:00에 DB를 암호화해 백업하고, 5분마다 사이트 상태를 확인해 응답이 없으면 blog-server 저장소에 `site-down` 이슈를 엽니다(D-150).
