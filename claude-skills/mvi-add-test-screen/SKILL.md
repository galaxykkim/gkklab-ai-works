---
name: mvi-add-test-screen
description: TestScreen에 특정 화면으로 이동하는 버튼을 추가하는 스킬. 사용자로부터 연결할 화면 정보를 입력받아 AppRoute enum과 대조하여 유효성을 검사한 뒤, test_screen.dart에 이동 버튼을 추가한다.
tools: AskUserQuestion, Read, Edit, Bash
---

# Add Test Screen Button

`test_screen.dart`에 특정 화면으로 이동하는 버튼을 추가하는 스킬이다.
AppRoute enum에 등록된 경로만 허용하며, 유효하지 않은 경로는 등록하지 않는다.

## 실행 절차

### Step 0: 프로젝트 정보 확인

Bash 도구로 `pubspec.yaml`을 찾아 프로젝트 루트 경로를 확인한다.

```bash
PROJECT_ROOT=$(dirname "$(find . -maxdepth 3 -name "pubspec.yaml" -not -path "*/.*" | head -1)")
echo "PROJECT_ROOT=$PROJECT_ROOT"
```

이후 모든 단계에서 `{PROJECT_ROOT}`를 이 값으로 대체한다.

### Step 1: 사전 조건 확인

Bash 도구로 `test_screen.dart` 파일이 존재하는지 확인한다.

```bash
find {PROJECT_ROOT} -name "test_screen.dart" -not -path "*/.*" | head -1
```

파일이 존재하지 않으면 아래 메시지를 출력하고 중단한다.
```
❌ test_screen.dart 파일을 찾을 수 없습니다.

먼저 /mvi-init-test-screen 스킬을 실행하여 TestScreen을 초기화하세요.
```

### Step 2: AppRoute 목록 파악

Read 도구로 `{PROJECT_ROOT}/lib/core/route/app_route.dart`를 읽는다.
enum 항목에서 camelCase 이름, path, params를 모두 추출한다.

예시 — 파일 내용:
```dart
enum AppRoute {
  test(path: '/test', params: []),
  splash(path: '/', params: []),
  main(path: '/main', params: []),
  rank(path: '/rank', params: []);
}
```
→ 유효한 항목: `test /test []`, `splash / []`, `main /main []`, `rank /rank []`

### Step 3: 사용자 입력 받기

AskUserQuestion 도구로 **질문 1개만** 물어본다.

**질문 — 연결할 화면 선택**
```
TestScreen에 추가할 화면을 선택하세요:
```

options 구성 규칙:
- Step 2에서 추출한 항목 중 `test`와 `main`과 `splash`를 **제외**한 항목들을 최대 3개까지 option으로 제공한다.
- 각 option의 label은 camelCase 이름, description은 path로 표시한다. 예) label: `rank`, description: `/rank`
- 나머지 항목이 있거나 직접 입력이 필요한 경우를 위해 마지막 option으로 `기타 (직접 입력)` / `Other 선택 후 AppRoute 이름 입력` 을 추가한다.

### Step 4: 유효성 검사 (직접 입력한 경우에만)

Step 3에서 predefined option을 선택한 경우 → 유효성 검사 생략, Step 5로 바로 이동.

`기타 (직접 입력)` 또는 Other를 통해 직접 입력한 경우:
- 입력값이 Step 2의 유효한 이름 목록에 포함되는지 확인한다.
- **유효하지 않은 경우**: 아래 메시지를 출력하고 중단한다.
  ```
  ❌ '{입력값}'은 AppRoute에 등록되지 않은 경로입니다.

  유효한 AppRoute 목록:
  {목록 출력}

  먼저 /mvi-register-screen 스킬로 라우트를 등록하세요.
  ```
- `test`, `main`, `splash`가 입력된 경우:
  ```
  ⚠️ '{입력값}'은 TestScreen에서 제외된 경로입니다. 다른 화면을 선택하세요.
  ```
  메시지를 출력하고 중단한다.

### Step 5: 버튼 레이블 자동 생성

camelCase 이름에서 버튼 레이블을 자동 생성한다.
단어 경계(대문자 앞)에 공백을 삽입하고 각 단어의 첫 글자를 대문자로 만든다.

변환 예시:
- `rank` → `Rank`
- `signUp` → `Sign Up`
- `myProfile` → `My Profile`
- `suggestPlace` → `Suggest Place`

### Step 6: test_screen.dart 수정

Read 도구로 `{PROJECT_ROOT}/lib/test/test_screen.dart`를 읽는다.

**중복 확인**: 이미 `AppRoute.{camelName}.path`가 파일 내에 존재하면:
```
⚠️ '{camelName}' 버튼은 이미 TestScreen에 추가되어 있습니다.
```
메시지를 출력하고 중단한다.

**버튼 추가**: TODO 주석 바로 앞에 버튼을 삽입한다.

파라미터가 없는 경우 (params가 `[]`인 경우):
```dart
            ElevatedButton(
              onPressed: () => context.push(AppRoute.{camelName}.path),
              child: const Text('{레이블}'),
            ),
            const SizedBox(height: 8),
            // TODO: 테스트할 화면/기능 버튼을 아래에 추가하세요.
```

파라미터가 있는 경우 (params가 비어 있지 않은 경우):
```dart
            ElevatedButton(
              onPressed: () {
                // TODO: {camelName} 화면에 필요한 파라미터를 설정하세요.
                // context.push(AppRoute.{camelName}.path, extra: ...);
              },
              child: const Text('{레이블}'),
            ),
            const SizedBox(height: 8),
            // TODO: 테스트할 화면/기능 버튼을 아래에 추가하세요.
```

Edit 도구로 `// TODO: 테스트할 화면/기능 버튼을 아래에 추가하세요.` 앞에 삽입한다.

### Step 7: 완료 메시지 출력

```
✅ TestScreen에 버튼이 추가되었습니다:

  '{레이블}'  →  AppRoute.{camelName} ({path})

수정된 파일:
  {PROJECT_ROOT}/lib/test/test_screen.dart

추가 버튼이 필요하면 /mvi-add-test-screen 을 다시 실행하세요.
```
