# 블로그 플랫폼 문서

| 폴더 | 내용 |
| --- | --- |
| [01-requirements](01-requirements/) | 요구사항분석서 (기본 요구사항, 세부 정책 결정 기록) |
| [02-database](02-database/) | DB 설계: MySQL 스키마와 ERDCloud용 파일 |

## 01-requirements

- [요구사항분석서_1_기본요구사항.pdf](01-requirements/요구사항분석서_1_기본요구사항.pdf)
- [요구사항분석서_2_세부정책_결정기록.pdf](01-requirements/요구사항분석서_2_세부정책_결정기록.pdf)

## 02-database

- `schema/`: 실제 DB에 적용하는 SQL (MySQL 8)
  - [erd_tables.sql](02-database/schema/erd_tables.sql): 테이블 정의 (PK, UNIQUE, FK)
  - [add_indexes.sql](02-database/schema/add_indexes.sql): 나중에 추가할 조회용 인덱스
- `erdcloud/`: ERDCloud에 그리기 위한 파일
  - [erdcloud_import.sql](02-database/erdcloud/erdcloud_import.sql): ERDCloud 가져오기용 SQL
  - [erdcloud_names.md](02-database/erdcloud/erdcloud_names.md): 물리 이름(영어) ↔ 논리 이름(한글) 목록
