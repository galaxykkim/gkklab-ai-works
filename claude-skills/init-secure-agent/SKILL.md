---
name: init-secure-agent
description: 보안 설정 초기화 스킬. SKILL.md와 같은 위치의 secure_agent/ 폴더에서 .aiexclude 파일과 .claude 폴더 및 하위 파일들을 현재 프로젝트 root에 복사한다.
tools: Bash
---

# Init Secure Agent

이 스킬 파일(`SKILL.md`)과 같은 위치의 `secure_agent/` 폴더에 저장된 보안 설정 파일들을 현재 프로젝트 root 디렉토리에 복사하는 스킬이다.

## 복사 대상

- `.aiexclude` — AI 컨텍스트에서 제외할 파일 패턴 목록
- `.claude/` 폴더 및 하위 파일 전체 — Claude Code 프로젝트 설정 (permissions, ignore_patterns 등)

## 실행 절차

### Step 1: 스킬 디렉토리 경로 확인

Bash 도구로 이 스킬의 `SKILL.md` 파일 위치를 찾아 `secure_agent/` 경로를 `SKILL_SOURCE`로 저장한다.

```bash
SKILL_SOURCE="$(dirname "$(find ~ -name "SKILL.md" -path "*/init-secure-agent/SKILL.md" 2>/dev/null | head -1)")/secure_agent"
echo "$SKILL_SOURCE"
```

경로가 비어 있으면 스킬 파일을 찾을 수 없다는 오류 메시지를 출력하고 중단한다.

### Step 2: 소스 파일 존재 여부 확인

```bash
ls "$SKILL_SOURCE/.aiexclude" "$SKILL_SOURCE/.claude/"
```

파일이 없으면 오류 메시지를 출력하고 중단한다.

### Step 3: 현재 프로젝트 root 경로 확인

```bash
pwd
```

현재 작업 디렉토리를 `{PROJECT_ROOT}`로 사용한다.

### Step 4: .aiexclude 복사

```bash
cp "$SKILL_SOURCE/.aiexclude" "{PROJECT_ROOT}/.aiexclude"
```

### Step 5: .claude 폴더 복사

기존 `.claude` 폴더가 있을 경우 하위 파일을 덮어쓴다.

```bash
cp -r "$SKILL_SOURCE/.claude/." "{PROJECT_ROOT}/.claude/"
```

### Step 6: 복사 결과 검증

```bash
ls -la {PROJECT_ROOT}/.aiexclude {PROJECT_ROOT}/.claude/
```

### Step 7: 완료 메시지 출력

```
보안 설정 파일이 성공적으로 복사되었습니다:

  {PROJECT_ROOT}/
  ├── .aiexclude
  └── .claude/
      └── settings.json
```
