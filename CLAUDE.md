# blog-docs

티스토리형 블로그 플랫폼의 기획·설계 문서 저장소입니다.

## 폴더 규칙

- 원본 문서와 DB 설계 파일은 `docs/` 아래에 둡니다. 목차는 `docs/README.md`입니다.
- Spec Kit 문서는 Spec Kit 표준 위치에 둡니다: 원칙은 `.specify/memory/constitution.md`, 기능별 스펙은 `specs/NNN-이름/`(spec.md, plan.md, research.md, data-model.md). `.specify/templates`, `.specify/scripts`, `.claude/skills/speckit-*`는 `specify init`이 만든 파일이라 직접 고치지 않습니다.
- 주제별 폴더는 `NN-주제` 형식(예: `01-requirements`, `02-database`)으로 번호를 붙여 순서를 유지합니다.
- 새 폴더나 파일을 추가하면 `docs/README.md` 목차도 함께 고칩니다.
- 파일을 옮길 때는 `git mv`를 쓰고, 그 파일을 가리키는 다른 문서의 링크를 같이 고칩니다.
- 요구사항분석서는 PDF와 같은 이름의 md를 함께 둡니다. PDF가 바뀌면 md도 같은 내용으로 다시 맞춥니다.

## DB 설계 파일

- `docs/02-database/schema/erd_tables.sql`이 기준입니다. 테이블이나 컬럼을 바꾸면 `erdcloud/erdcloud_import.sql`과 `erdcloud/erdcloud_names.md`, `specs/001-main-blog/data-model.md`도 같이 맞춥니다.

## Spec Kit

- 새 기능은 `/speckit-specify`로 스펙을 만들고 `/speckit-clarify` → `/speckit-plan` → `/speckit-tasks` 순서로 진행합니다.
- 정책을 정하거나 바꾸면 `specs/001-main-blog/research.md` 결정 기록에 D-번호로 한 줄 추가하고, spec.md 본문도 같이 고칩니다.
