# 블로그 플랫폼 문서

요구사항과 설계는 [GitHub Spec Kit](https://github.com/github/spec-kit) 형식으로 정리되어 있습니다. 구현할 때는 아래 Spec Kit 문서를 먼저 보고, 이 폴더의 원본은 근거 자료로 봅니다.

| 문서 | 내용 |
| --- | --- |
| [constitution.md](../.specify/memory/constitution.md) | 모든 스펙·구현이 지킬 원칙 (보안, 블로그별 역할, 개인정보, 이중화 대비, 단순함, 결정 기록) |
| [spec.md](../specs/001-main-blog/spec.md) | 메인블로그 요구사항: 사용자 스토리, 기능·비기능 요구사항, 성공 기준, 정리한 질문(Clarifications) |
| [plan.md](../specs/001-main-blog/plan.md) | 기술 맥락, 원칙 점검, 구조, 다음 단계 |
| [research.md](../specs/001-main-blog/research.md) | 구현 방식을 고른 이유, 이중화 기술 후보, 결정 기록 D-01~D-92 |
| [data-model.md](../specs/001-main-blog/data-model.md) | 테이블 32개를 요구사항과 연결, 상태 전이, 검증 규칙 |

## 원본 문서

| 폴더 | 내용 |
| --- | --- |
| [01-requirements](01-requirements/) | 요구사항분석서 1·2 (기본 요구사항, 세부 정책 결정 기록) |
| [02-database](02-database/) | DB 설계: 요구사항분석서 3(ERD), MySQL 스키마, ERDCloud용 파일 |

## 01-requirements

요구사항분석서는 같은 내용을 PDF와 Markdown 두 가지로 둡니다. GitHub에서 바로 읽거나 검색할 때는 md를 보면 됩니다.

- 요구사항분석서 1 · 기본 요구사항: [md](01-requirements/요구사항분석서_1_기본요구사항.md) · [PDF](01-requirements/요구사항분석서_1_기본요구사항.pdf)
- 요구사항분석서 2 · 세부 정책과 결정 기록: [md](01-requirements/요구사항분석서_2_세부정책_결정기록.md) · [PDF](01-requirements/요구사항분석서_2_세부정책_결정기록.pdf)
- `images/`: 요구사항분석서 1의 md에 들어가는 그림 (서비스 구조, 서버 2대 구성)

## 02-database

- 요구사항분석서 3 · DB 설계(ERD): [md](02-database/요구사항분석서_3_DB설계_ERD.md) · [PDF](02-database/요구사항분석서_3_DB설계_ERD.pdf). 테이블 32개의 컬럼 표, 관계도, 인덱스 메모 (아래 SQL과 같은 내용). md의 관계도는 Mermaid로 다시 그렸습니다.
- `schema/`: 실제 DB에 적용하는 SQL (MySQL 8)
  - [erd_tables.sql](02-database/schema/erd_tables.sql): 테이블 정의 (PK, UNIQUE, FK)
  - [add_indexes.sql](02-database/schema/add_indexes.sql): 나중에 추가할 조회용 인덱스
- `erdcloud/`: ERDCloud에 그리기 위한 파일
  - [erdcloud_import.sql](02-database/erdcloud/erdcloud_import.sql): ERDCloud 가져오기용 SQL
  - [erdcloud_names.md](02-database/erdcloud/erdcloud_names.md): 물리 이름(영어) ↔ 논리 이름(한글) 목록
