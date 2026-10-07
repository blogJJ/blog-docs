# blog-docs

티스토리형 블로그 플랫폼의 기획·설계 문서 저장소입니다.

## 폴더 규칙

- 모든 문서는 `docs/` 아래에 둡니다. 목차는 `docs/README.md`입니다.
- 주제별 폴더는 `NN-주제` 형식(예: `01-requirements`, `02-database`)으로 번호를 붙여 순서를 유지합니다.
- 새 폴더나 파일을 추가하면 `docs/README.md` 목차도 함께 고칩니다.
- 파일을 옮길 때는 `git mv`를 쓰고, 그 파일을 가리키는 다른 문서의 링크를 같이 고칩니다.

## DB 설계 파일

- `docs/02-database/schema/erd_tables.sql`이 기준입니다. 테이블이나 컬럼을 바꾸면 `erdcloud/erdcloud_import.sql`과 `erdcloud/erdcloud_names.md`도 같이 맞춥니다.
