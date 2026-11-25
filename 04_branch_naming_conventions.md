# P Join · 브랜치 이름 규칙

> 팀 검토용 초안

작업 브랜치는 `<type>/<short-description>` 형식으로 작성합니다. 이슈를 추적하는 작업에는 `<type>/<issue-id>-<short-description>` 형식을 사용합니다. 설명에는 영문 소문자와 숫자를 사용하고 단어를 하이픈으로 구분합니다.

| 유형 | 용도 | 예시 |
| --- | --- | --- |
| `feat` | 기능 추가·변경 | `feat/42-project-search` |
| `fix` | 버그 수정 | `fix/github-link-state` |
| `docs` | 문서 변경 | `docs/split-project-specs` |
| `refactor` | 동작을 유지하는 코드 구조 개선 | `refactor/project-filters` |
| `test` | 테스트 추가·수정 | `test/application-cancel` |
| `chore` | 개발 도구·설정 정비 | `chore/update-lint-config` |

작업 유형은 [커밋 메시지 규칙](05_commit_naming_conventions.md)과 같은 의미로 사용합니다.
