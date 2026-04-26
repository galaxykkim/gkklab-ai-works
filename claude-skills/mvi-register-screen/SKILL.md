---
name: mvi-register-screen
description: Flutter GoRouter 기반 프로젝트에서 AppRoute enum과 AppRouter의 _routes에 화면을 등록하는 스킬. 화면 path와 optional params를 입력받아 app_route.dart, app_router.dart 두 파일을 수정한다.
tools: AskUserQuestion, Read, Edit, Bash
---

# Register Screen

Flutter GoRouter 기반 프로젝트에서 새 화면을 라우터에 등록하는 스킬이다.
`{PROJECT_ROOT}/lib/core/route/app_route.dart`의 AppRoute enum과
`{PROJECT_ROOT}/lib/core/route/app_router.dart`의 `_routes` 리스트에 항목을 추가한다.

## 실행 절차

### Step 0: 프로젝트 정보 확인

Bash 도구로 `pubspec.yaml`을 찾아 프로젝트 루트 경로와 패키지명을 확인한다.

```bash
PROJECT_ROOT=$(dirname "$(find . -maxdepth 3 -name "pubspec.yaml" -not -path "*/.*" | head -1)")
PACKAGE_NAME=$(grep "^name:" "$PROJECT_ROOT/pubspec.yaml" | awk '{print $2}')
echo "PROJECT_ROOT=$PROJECT_ROOT"
echo "PACKAGE_NAME=$PACKAGE_NAME"
```

이후 모든 단계에서 `{PROJECT_ROOT}`와 `{PACKAGE_NAME}`을 이 값으로 대체한다.

### Step 1: 정보 입력받기

AskUserQuestion 도구로 다음 두 가지를 물어본다.

**질문 1 — 변수명과 path (동시 입력)**
```
변수명과 path를 쉼표로 구분하여 입력하세요 (예: rank, /rank  또는  my_profile, /my_profile):
```
- 첫 번째 값: AppRoute enum에 추가될 변수명 (snake_case)
- 두 번째 값: 라우트 path (슬래시로 시작)
- 두 번째 값 앞의 공백은 무시한다. 예) `"rank, /rank"` → 변수명 `rank`, path `/rank`
- 이미 `mvi-add-screen` 스킬로 생성한 화면 이름과 동일한 변수명을 사용한다.

**질문 2 — params (선택)**
```
path parameter 또는 extra 파라미터가 있으면 입력하세요.
형식: 이름:타입 (쉼표 구분, 예: place:Place, id:String)
없으면 빈칸으로 두세요.
```

### Step 2: 이름 변환

입력받은 `snake_case` 이름을 변환한다.

- `camelCase` (enum 키): `_`로 분리한 단어 중 첫 단어는 소문자 유지, 이후 단어는 첫 글자 대문자
  - `sign_up` → `signUp`, `my_profile` → `myProfile`, `rank` → `rank`
- `PascalCase` (클래스명): 모든 단어의 첫 글자 대문자
  - `sign_up` → `SignUp`, `my_profile` → `MyProfile`, `rank` → `Rank`

### Step 3: 파일 경로 확인

Bash 도구로 라우터 파일 존재 여부를 확인한다:
```bash
ls {PROJECT_ROOT}/lib/core/route/
```

파일이 없으면 사용자에게 경로가 다를 수 있다고 안내하고 중단한다.

### Step 4: app_route.dart 수정

Read 도구로 `{PROJECT_ROOT}/lib/core/route/app_route.dart`를 읽는다.

AppRoute enum의 마지막 항목 끝에 세미콜론(`;`)이 있으면 그 앞에 `,`로 구분하여 새 항목을 추가한다.

**파라미터가 없는 경우** (params가 빈 입력인 경우):
```dart
{camelName}(path: '{path}', params: []),
```

**파라미터가 있는 경우** (params 입력이 있는 경우):
params의 이름만 추출하여 리스트에 넣는다. 예) `place:Place, id:String` → `['place', 'id']`
```dart
{camelName}(path: '{path}', params: ['{param1}', '{param2}']),
```

Edit 도구로 마지막 enum 항목 뒤에 새 항목을 추가한다.

예시 — 기존 마지막 항목이 `signUp(path: '/sign_up', params: []);` 인 경우:
```
old: signUp(path: '/sign_up', params: []);
new: signUp(path: '/sign_up', params: []),
     rank(path: '/rank', params: []);
```

### Step 5: app_router.dart 수정

Read 도구로 `{PROJECT_ROOT}/lib/core/route/app_router.dart`를 읽는다.

#### 5-1. import 추가

기존 screen import 목록의 마지막 줄 다음에 새 import를 추가한다.
```dart
import 'package:{PACKAGE_NAME}/ui/{name}/{name}_screen.dart';
```

Edit 도구로 기존 마지막 screen import 뒤에 추가한다.

#### 5-2. GoRoute 추가

`_routes` getter의 마지막 `GoRoute(...)` 항목 뒤 `];` 직전에 새 항목을 추가한다.

**파라미터가 없는 경우:**
```dart
GoRoute(
  path: AppRoute.{camelName}.path,
  builder: (context, state) => const {Name}Screen(),
),
```

**파라미터가 있는 경우:**
파라미터가 1개이면 `state.extra`를 사용한다:
```dart
GoRoute(
  path: AppRoute.{camelName}.path,
  builder: (context, state) {
    final {param1Name} = state.extra as {Param1Type};
    return {Name}Screen({param1Name}: {param1Name});
  },
),
```
파라미터가 2개 이상이면 extra를 Map으로 사용한다:
```dart
GoRoute(
  path: AppRoute.{camelName}.path,
  builder: (context, state) {
    final args = state.extra as Map<String, dynamic>;
    final {param1Name} = args['{param1Name}'] as {Param1Type};
    final {param2Name} = args['{param2Name}'] as {Param2Type};
    return {Name}Screen({param1Name}: {param1Name}, {param2Name}: {param2Name});
  },
),
```

Edit 도구로 마지막 GoRoute 항목 뒤에 추가한다.

### Step 6: 완료 메시지 출력

```
✅ 라우트 등록이 완료되었습니다:

AppRoute.{camelName}  →  {path}

수정된 파일:
  {PROJECT_ROOT}/lib/core/route/app_route.dart  (enum 항목 추가)
  {PROJECT_ROOT}/lib/core/route/app_router.dart (import 및 GoRoute 추가)

다음 단계:
- {Name}Screen 생성자에 파라미터가 있다면 확인하세요.
- 다른 화면에서 context.go(AppRoute.{camelName}.path) 또는
  context.push(AppRoute.{camelName}.path, extra: ...) 로 이동할 수 있습니다.
```
