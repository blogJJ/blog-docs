# API 명세 (contracts)

서버 API 목록입니다. 원본은 [openapi.json](openapi.json)이고, blog-server를 local 프로필로 띄워 `/v3/api-docs`에서 받은 OpenAPI 3.1 명세입니다(D-112, T134). 기준은 2026-10-14 main(v1.29.0)이고 경로 116개, 요청 133개입니다.

- 다시 받는 법: local 프로필로 띄운 뒤 `curl -s http://localhost:8080/v3/api-docs > openapi.json`. 화면으로는 http://localhost:8080/swagger-ui.html 에서 봅니다(운영에서는 닫혀 있음, OPS-12).
- 인증은 HttpOnly 쿠키(Access·Refresh Token)이고, 쓰기 요청에는 CSRF 헤더가 필요합니다. 오류 응답 모양과 code는 blog-server README의 "인증 / 오류"를 봅니다.
- 블로그별 권한(블로그장·부블로그장·멤버)은 요청마다 DB로 확인하므로 명세에는 드러나지 않습니다. spec.md의 BLG·BRD 요구사항과 data-model.md를 같이 봅니다.

## 회원·인증 (auth)

| 컨트롤러 | 요청 |
| --- | --- |
| `AuthController` | `GET /api/auth/config`<br>`POST /api/auth/extend`<br>`POST /api/auth/login`<br>`POST /api/auth/logout`<br>`POST /api/auth/refresh`<br>`GET /api/auth/session` |
| `FindEmailController` | `POST /api/auth/find-email` |
| `MeController` | `GET /api/me`<br>`PUT /api/me`<br>`PUT /api/me/password`<br>`PUT /api/me/profile-image`<br>`DELETE /api/me/profile-image`<br>`GET /api/me/withdraw`<br>`POST /api/me/withdraw` |
| `PasswordController` | `POST /api/auth/password/code`<br>`POST /api/auth/password/reset` |
| `SignupController` | `POST /api/auth/signup`<br>`POST /api/auth/signup/code`<br>`POST /api/auth/signup/verify` |
| `SocialController` | `GET /api/auth/social/pending`<br>`POST /api/auth/social/signup`<br>`GET /api/me/social`<br>`DELETE /api/me/social/{provider}`<br>`POST /api/me/social/{provider}/link` |

## 블로그 (blog)

| 컨트롤러 | 요청 |
| --- | --- |
| `BlacklistController` | `GET /api/blogs/{blogId}/blacklist`<br>`POST /api/blogs/{blogId}/blacklist-inquiries`<br>`POST /api/blogs/{blogId}/blacklist-inquiries/{inquiryId}/reject`<br>`POST /api/blogs/{blogId}/blacklist-inquiries/{inquiryId}/release` |
| `BlogCloseController` | `POST /api/blogs/{blogId}/close`<br>`POST /api/blogs/{blogId}/close/revoke` |
| `BlogController` | `GET /api/blogs`<br>`POST /api/blogs`<br>`GET /api/blogs/by-slug/{slug}`<br>`GET /api/blogs/search`<br>`GET /api/blogs/slug-check`<br>`PUT /api/blogs/{blogId}`<br>`PUT /api/blogs/{blogId}/cover`<br>`DELETE /api/blogs/{blogId}/cover`<br>`POST /api/blogs/{blogId}/leave`<br>`POST /api/blogs/{blogId}/share-link`<br>`GET /api/me/blogs` |
| `BlogReportController` | `GET /api/blogs/{blogId}/reports`<br>`POST /api/blogs/{blogId}/reports/{reportId}/resolve` |
| `JoinController` | `POST /api/blogs/{blogId}/join`<br>`GET /api/blogs/{blogId}/join-requests`<br>`DELETE /api/blogs/{blogId}/join-requests/{requestId}`<br>`POST /api/blogs/{blogId}/join-requests/{requestId}/approve`<br>`POST /api/blogs/{blogId}/join-requests/{requestId}/reject` |
| `MemberAdminController` | `PUT /api/blogs/{blogId}/managers/{userId}`<br>`GET /api/blogs/{blogId}/members`<br>`POST /api/blogs/{blogId}/members/{userId}/kick`<br>`POST /api/blogs/{blogId}/members/{userId}/release`<br>`GET /api/blogs/{blogId}/members/{userId}/sanctions`<br>`POST /api/blogs/{blogId}/members/{userId}/suspend`<br>`POST /api/blogs/{blogId}/members/{userId}/warn` |
| `TransferController` | `POST /api/blogs/{blogId}/transfers`<br>`GET /api/blogs/{blogId}/transfers/pending`<br>`POST /api/blogs/{blogId}/transfers/pending/cancel`<br>`GET /api/me/transfers`<br>`POST /api/transfers/{id}/accept`<br>`POST /api/transfers/{id}/reject` |

## 글·댓글·이미지 (board)

| 컨트롤러 | 요청 |
| --- | --- |
| `CategoryController` | `POST /api/blogs/{blogId}/categories`<br>`PUT /api/blogs/{blogId}/categories`<br>`PUT /api/blogs/{blogId}/categories/{categoryId}`<br>`DELETE /api/blogs/{blogId}/categories/{categoryId}` |
| `CommentController` | `PUT /api/comments/{commentId}`<br>`DELETE /api/comments/{commentId}`<br>`GET /api/posts/{postId}/comments`<br>`POST /api/posts/{postId}/comments` |
| `CommentReviewController` | `GET /api/blogs/{blogId}/comment-reviews`<br>`DELETE /api/blogs/{blogId}/comment-reviews/{commentId}`<br>`POST /api/blogs/{blogId}/comment-reviews/{commentId}/approve` |
| `ImageController` | `POST /api/images`<br>`GET /api/images/{storedName}` |
| `LikeController` | `POST /api/posts/{postId}/like` |
| `PostController` | `GET /api/blogs/{blogId}/categories`<br>`GET /api/blogs/{blogId}/posts`<br>`POST /api/blogs/{blogId}/posts`<br>`GET /api/posts/{postId}`<br>`PUT /api/posts/{postId}`<br>`DELETE /api/posts/{postId}` |
| `WritingAssistController` | `GET /api/posts/assist/config`<br>`POST /api/posts/assist/spelling`<br>`POST /api/posts/assist/tags` |

## 메인·검색 (main)

| 컨트롤러 | 요청 |
| --- | --- |
| `FeedController` | `GET /api/feed` |
| `NoticeController` | `GET /api/notices`<br>`GET /api/notices/{id}` |
| `ReleaseNoteController` | `GET /api/release-notes`<br>`GET /api/release-notes/{version}` |
| `SearchController` | `GET /api/me/recent-searches`<br>`DELETE /api/me/recent-searches`<br>`DELETE /api/me/recent-searches/{id}`<br>`GET /api/search` |
| `ShareImageController` | `GET /api/og/covers/{name}` |
| `SimilarPostController` | `GET /api/posts/{postId}/similar` |

## 소셜·알림 (social)

| 컨트롤러 | 요청 |
| --- | --- |
| `BlockController` | `GET /api/me/blocks`<br>`POST /api/users/{id}/block`<br>`DELETE /api/users/{id}/block` |
| `FollowController` | `POST /api/blogs/{blogId}/subscribe`<br>`POST /api/users/{id}/follow`<br>`GET /api/users/{id}/followers`<br>`GET /api/users/{id}/followings` |
| `NotificationController` | `GET /api/me/notification-settings`<br>`PUT /api/me/notification-settings`<br>`GET /api/notifications`<br>`DELETE /api/notifications`<br>`POST /api/notifications/read-all`<br>`GET /api/notifications/unread-count`<br>`POST /api/notifications/{id}/read` |
| `ProfileController` | `GET /api/users/{id}/profile` |
| `ReportController` | `POST /api/reports` |

## 메인 관리자 (admin)

| 컨트롤러 | 요청 |
| --- | --- |
| `AdminBlogController` | `GET /api/admin/blogs`<br>`POST /api/admin/blogs/{blogId}/close`<br>`POST /api/admin/blogs/{blogId}/hide`<br>`POST /api/admin/blogs/{blogId}/owner/demote`<br>`POST /api/admin/blogs/{blogId}/owner/warn`<br>`POST /api/admin/blogs/{blogId}/unhide` |
| `AdminContentController` | `POST /api/admin/comments/{commentId}/delete`<br>`POST /api/admin/comments/{commentId}/hide`<br>`GET /api/admin/posts`<br>`GET /api/admin/posts/{postId}/comments`<br>`POST /api/admin/posts/{postId}/delete`<br>`POST /api/admin/posts/{postId}/hide`<br>`POST /api/admin/posts/{postId}/restore` |
| `AdminLogController` | `GET /api/admin/logs` |
| `AdminNoticeController` | `POST /api/admin/notices` |
| `AdminReportController` | `GET /api/admin/reports`<br>`POST /api/admin/reports/{reportId}/resolve` |
| `AdminStatsController` | `GET /api/admin/stats` |
| `AdminUserController` | `GET /api/admin/users`<br>`GET /api/admin/users/{userId}`<br>`POST /api/admin/users/{userId}/release`<br>`POST /api/admin/users/{userId}/reveal` |
