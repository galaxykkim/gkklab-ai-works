# PROJECT_DOCS_RULE

에이전트가 작성하는 문서 파일에 대한 지침 및 준수사항을 정의합니다.

---

## 1. 파일 생성 경로

```
{project_root}/docs/
```

---

## 2. 파일명 규칙

```
{date}_{project_name}_{task_name}_by_{agent_name}.md
```

| 필드 | 설명 | 예시 |
|---|---|---|
| `date` | 문서 생성일 (yyyymmdd) | `20260531` |
| `project_name` | Git 저장소 이름에서 첫 번째 `-` 이전 단어 | `mercury-app-flutter` → `mercury` / `explorer-web` → `explorer` |
| `task_name` | 작업의 핵심 기능 또는 요구사항 이름 | `user_auth`, `fanzone`, `push_notification` |
| `agent_name` | 문서를 작성한 에이전트 이름 | `dev-planner`, `dev-researcher`, `dev-lead` |

### 로그 파일 예외 규칙

로그 파일의 경우, 동일 날짜에 여러 파일이 생성될 수 있으므로 `time` 필드를 추가한다.

```
{date}_{time}_{project_name}_{task_name}_by_{agent_name}.md
```

| 필드 | 설명 | 예시 |
|---|---|---|
| `date` | 문서 생성일 (yyyymmdd) | `20260531` |
| `time` | 문서 생성 시각 (HHmmss) | `143022` |
| `project_name` | Git 저장소 이름에서 첫 번째 `-` 이전 단어 | `mercury` |
| `task_name` | 작업 또는 실행 내용 이름 | `deploy`, `migration` |
| `agent_name` | 문서를 작성한 에이전트 이름 | `dev-lead` |

예시: `20260531_143022_mercury_deploy_by_dev-lead.md`

---

## 3. 중복 파일명 처리

로그 파일은 `time` 필드로 파일명이 고유하게 생성되므로 이 절의 대상에서 제외된다.

그 외 파일명이 중복되는 경우, 기존 파일에 내용을 **추가**한다. 파일을 새로 생성하지 않는다.

### 변경이력 관리

문서 상단에 아래 형식으로 버전 테이블을 유지한다.

```markdown
| 버전 | 날짜 | 변경 내용 |
|---|---|---|
| v1.0 | 20260531 | 최초 작성 |
| v1.1 | 20260615 | 3절 요구사항 추가 |
```

- 내용을 파일에서 물리적으로 삭제하지 않는다. 변경·제거 대상 내용은 **취소선**(`~~텍스트~~`)으로 표시한다.

---

## 4. 작성 규칙

- 간결하고 명확하게 작성한다.
- 문서의 목적에 필요한 정보만 기술한다. 중복·불필요한 내용은 포함하지 않는다.
