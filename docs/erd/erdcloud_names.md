# ERDCloud 이름 바꾸기 목록

ERDCloud에서 테이블과 컬럼마다 **물리 이름 = 영어(DB에 실제로 만들어지는 이름)**, **논리 이름 = 한글(화면 표시용)** 으로 넣으세요. 물리 이름은 erd_tables.sql과 똑같아야 합니다.

## users → 회원

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| email | 이메일 |
| password_hash | 비밀번호 해시 |
| name | 이름 |
| nickname | 닉네임 |
| phone | 전화번호 |
| profile_image | 프로필 사진 |
| bio | 소개 |
| role | 역할 |
| status | 상태 |
| login_fail_count | 로그인 실패 횟수 |
| locked_until | 잠금 해제 시각 |
| notification_keep_days | 알림 보관 일수 |
| terms_agreed_at | 약관 동의 시각 |
| privacy_agreed_at | 개인정보 동의 시각 |
| withdrawn_at | 탈퇴 시각 |
| created_at | 생성 시각 |
| updated_at | 수정 시각 |

## verification_codes → 이메일 인증번호

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| email | 이메일 |
| purpose | 용도 |
| code_hash | 인증번호 해시 |
| fail_count | 틀린 횟수 |
| expires_at | 만료 시각 |
| verified_at | 인증 완료 시각 |
| used_at | 사용 시각 |
| created_at | 생성 시각 |

## account_find_tokens → 이메일 찾기 임시 토큰

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| user_id | 찾은 회원 번호 |
| token_hash | 토큰 해시 |
| expires_at | 만료 시각 |
| created_at | 생성 시각 |

## refresh_tokens → Refresh Token

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| user_id | 회원 번호 |
| token_hash | 토큰 해시 |
| remember_me | 로그인 유지 |
| user_agent | 기기 정보 |
| expires_at | 만료 시각 |
| revoked_at | 폐기 시각 |
| created_at | 생성 시각 |

## blogs → 블로그

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| owner_id | 블로그장 번호 |
| slug | 블로그 주소 |
| name | 블로그 이름 |
| description | 소개 |
| cover_image | 대표 이미지 |
| visibility | 공개 범위 |
| share_key | 공유 링크 값 |
| join_policy | 참여 방식 |
| status | 상태 |
| is_hidden | 숨김 여부 |
| close_scheduled_at | 폐쇄 예정 시각 |
| close_reason | 폐쇄 사유 |
| closed_at | 폐쇄 시각 |
| member_count | 멤버 수 |
| post_count | 글 수 |
| subscriber_count | 구독자 수 |
| created_at | 생성 시각 |
| updated_at | 수정 시각 |

## blog_members → 블로그 멤버

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| user_id | 회원 번호 |
| role | 멤버 역할 |
| manager_since | 부블로그장 지정 시각 |
| suspended_until | 정지 종료 시각 |
| suspension_count | 정지 횟수 |
| joined_at | 참여 시각 |

## blog_manager_permissions → 부블로그장 권한

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_member_id | 멤버 번호 |
| permission | 권한 |
| granted_at | 권한 부여 시각 |

## blog_join_requests → 참여 신청

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| user_id | 신청한 회원 번호 |
| status | 상태 |
| handled_by | 처리자 번호 |
| handled_at | 처리 시각 |
| created_at | 생성 시각 |

## blog_owner_transfers → 블로그장 위임 요청

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| from_user_id | 요청자 번호 |
| to_user_id | 받는 멤버 번호 |
| status | 상태 |
| expires_at | 만료 시각 |
| responded_at | 응답 시각 |
| created_at | 생성 시각 |

## blog_subscriptions → 블로그 구독

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| user_id | 구독한 회원 번호 |
| created_at | 생성 시각 |

## blog_close_notices → 폐쇄 알림 발송 기록

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| close_scheduled_at | 폐쇄 예정 시각 |
| stage | 알림 단계 |
| sent_at | 발송 시각 |

## categories → 카테고리

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| name | 카테고리 이름 |
| sort_order | 표시 순서 |
| created_at | 생성 시각 |

## blog_tags → 블로그 태그

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| tag_id | 태그 번호 |

## member_sanctions → 멤버 경고·정지·강제 퇴장

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| user_id | 대상 멤버 번호 |
| type | 제재 종류 |
| suspend_days | 정지 일수 |
| ends_at | 정지 종료 시각 |
| reason | 사유 |
| report_id | 신고 번호 |
| issued_by | 조치자 번호 |
| released_at | 정지 해제 시각 |
| released_by | 해제자 번호 |
| created_at | 생성 시각 |

## owner_sanctions → 블로그장 경고·권한 박탈

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| user_id | 대상 블로그장 번호 |
| type | 제재 종류 |
| reason | 사유 |
| report_id | 신고 번호 |
| admin_id | 관리자 번호 |
| created_at | 생성 시각 |

## blog_blacklist → 블로그 블랙리스트

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| user_id | 강퇴된 회원 번호 |
| name_hash | 이름 해시 |
| email_hash | 이메일 해시 |
| phone_hash | 전화번호 해시 |
| registered_by | 등록자 번호 |
| released_at | 해제 시각 |
| released_by | 해제자 번호 |
| created_at | 생성 시각 |

## blacklist_inquiries → 블랙리스트 해제 문의

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blacklist_id | 블랙리스트 번호 |
| user_id | 문의한 회원 번호 |
| name_match | 이름 일치 |
| phone_match | 전화번호 일치 |
| message | 문의 내용 |
| status | 상태 |
| handled_by | 처리자 번호 |
| handled_at | 처리 시각 |
| created_at | 생성 시각 |

## reports → 신고

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| reporter_id | 신고자 번호 |
| target_type | 대상 종류 |
| target_id | 대상 번호 |
| blog_id | 블로그 번호 |
| handler_scope | 처리 담당 |
| reason | 신고 사유 |
| detail | 상세 내용 |
| target_snapshot | 신고 당시 내용 |
| status | 처리 결과 |
| handled_by | 처리자 번호 |
| handled_at | 처리 시각 |
| created_at | 생성 시각 |

## admin_action_logs → 관리자 활동 기록

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| admin_id | 관리자 번호 |
| action | 조치 |
| target_type | 대상 종류 |
| target_id | 대상 번호 |
| reason | 사유 |
| created_at | 생성 시각 |

## posts → 게시글

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blog_id | 블로그 번호 |
| author_id | 작성자 번호 |
| category_id | 카테고리 번호 |
| title | 제목 |
| content | 본문 |
| is_notice | 공지 여부 |
| status | 상태 |
| author_hidden | 작성자 숨김 |
| view_count | 조회수 |
| like_count | 좋아요 수 |
| comment_count | 댓글 수 |
| deleted_by | 삭제자 번호 |
| deleted_at | 삭제 시각 |
| created_at | 생성 시각 |
| updated_at | 수정 시각 |

## tags → 태그

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| name | 태그 이름 |
| created_at | 생성 시각 |

## post_tags → 글-태그 연결

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| post_id | 글 번호 |
| tag_id | 태그 번호 |

## post_images → 글 이미지

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| post_id | 글 번호 |
| uploader_id | 업로더 번호 |
| stored_name | 저장 파일명 |
| original_name | 원래 파일명 |
| content_type | 파일 형식 |
| size_bytes | 파일 크기 |
| sort_order | 이미지 순서 |
| deleted_at | 삭제 시각 |
| created_at | 생성 시각 |

## comments → 댓글

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| post_id | 글 번호 |
| author_id | 작성자 번호 |
| parent_id | 부모 댓글 번호 |
| reply_to_user_id | 답글 대상 회원 번호 |
| content | 본문 |
| status | 상태 |
| is_edited | 수정 여부 |
| deleted_at | 삭제 시각 |
| created_at | 생성 시각 |
| updated_at | 수정 시각 |

## post_likes → 좋아요

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| post_id | 글 번호 |
| user_id | 누른 회원 번호 |
| created_at | 생성 시각 |

## post_views → 조회 기록

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| post_id | 글 번호 |
| viewer_key | 조회자 키 |
| view_date | 조회 날짜 |
| created_at | 생성 시각 |

## user_follows → 회원 팔로우

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| follower_id | 팔로우하는 회원 번호 |
| followee_id | 팔로우받는 회원 번호 |
| created_at | 생성 시각 |

## user_blocks → 회원 차단

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| blocker_id | 차단한 회원 번호 |
| blocked_id | 차단된 회원 번호 |
| created_at | 생성 시각 |

## notifications → 알림

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| receiver_id | 받는 회원 번호 |
| actor_id | 보낸 회원 번호 |
| tab | 탭 |
| type | 알림 종류 |
| target_type | 대상 종류 |
| target_id | 대상 번호 |
| message | 알림 문구 |
| is_read | 읽음 여부 |
| created_at | 생성 시각 |

## notification_settings → 알림 설정

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| user_id | 회원 번호 |
| type | 알림 종류 |
| enabled | 켜짐 여부 |

## recent_searches → 최근 검색어

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| id | 번호 |
| user_id | 회원 번호 |
| keyword | 검색어 |
| searched_at | 검색 시각 |

## shedlock → 배치 잠금 (ShedLock)

| 물리 이름 (영어) | 논리 이름 (한글) |
| --- | --- |
| name | 작업 이름 |
| lock_until | 잠금 종료 시각 |
| locked_at | 잠금 시각 |
| locked_by | 잠근 서버 |
