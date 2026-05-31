---
name: flutter-add-screen
description: Flutter 프로젝트에서 riverpod/go_router 의존성을 확인하고, 화면 이름과 패턴(MVVM/MVI/TCA)을 입력받아 필요한 Dart 파일을 생성한 뒤 go_router에 자동 등록하는 글로벌 스킬.
tools: AskUserQuestion, Read, Write, Edit, Bash
---

# Flutter Add Screen

Flutter 프로젝트에서 MVVM / MVI / TCA 패턴 중 하나를 선택하여 화면 관련 Dart 파일을 생성하고, go_router에 라우트를 자동 등록하는 스킬이다.

---

## 실행 절차

### Step 1: Flutter 프로젝트 확인 및 의존성 체크

Bash 도구로 `pubspec.yaml`을 찾는다.

```bash
find . -name "pubspec.yaml" -not -path "*/.*" -not -path "*/build/*" | head -5
```

파일을 찾지 못하면 아래 메시지를 출력하고 종료한다:
```
❌ pubspec.yaml을 찾을 수 없습니다. Flutter 프로젝트 루트에서 실행하세요.
```

찾은 첫 번째 `pubspec.yaml`을 Read 도구로 읽어 의존성을 확인한다.

**체크 항목:**
- `flutter_riverpod` 또는 `riverpod` 포함 여부
- `go_router` 포함 여부

누락된 의존성이 있으면 아래와 같이 안내하고 종료한다:
```
❌ 필수 의존성이 누락되었습니다:
  - flutter_riverpod: ❌ 없음
  - go_router: ❌ 없음

pubspec.yaml에 해당 패키지를 추가한 뒤 다시 실행하세요.
```

두 의존성이 모두 있으면 계속 진행한다.

---

### Step 2: 화면 이름 입력받기

AskUserQuestion 도구로 화면 이름을 입력받는다.

```
화면 이름을 입력하세요 (snake_case):
```

입력값을 `{name}`으로 사용한다. (예: `user_profile`, `sign_up`, `order_detail`)

---

### Step 3: 패턴 입력받기

AskUserQuestion 도구로 패턴을 입력받는다.

```
사용할 아키텍처 패턴을 입력하세요 (MVVM / MVI / TCA):
```

입력값을 정규화한다 (대소문자 무관):
- `mvvm` → `MVVM`
- `mvi` → `MVI`
- `tca` → `TCA`

MVVM / MVI / TCA 이외의 값이 입력되면 다시 물어본다.

---

### Step 4: 이름 변환

입력받은 `snake_case` 이름을 변환한다.

- **PascalCase** (`{Name}`): `_`로 분리한 각 단어의 첫 글자를 대문자로 변환하고 합친다.
  - `sign_up` → `SignUp`, `user_profile` → `UserProfile`, `home` → `Home`
- **camelCase** (`{camelName}`): PascalCase에서 첫 글자만 소문자로 변환한다.
  - `SignUp` → `signUp`, `UserProfile` → `userProfile`, `Home` → `home`

---

### Step 5: 출력 디렉토리 결정

Bash 도구로 프로젝트의 `lib/` 하위 디렉토리 구조를 확인한다:

```bash
find . -type d -name "ui" -not -path "*/.*" -not -path "*/build/*" | head -3
find . -type d -name "screens" -not -path "*/.*" -not -path "*/build/*" | head -3
find . -type d -name "features" -not -path "*/.*" -not -path "*/build/*" | head -3
find . -type d -name "presentation" -not -path "*/.*" -not -path "*/build/*" | head -3
```

결과를 바탕으로 아래 우선순위로 출력 디렉토리를 결정한다:
1. `lib/ui/` 존재 → `lib/ui/{name}/`
2. `lib/screens/` 존재 → `lib/screens/{name}/`
3. `lib/features/` 존재 → `lib/features/{name}/`
4. `lib/presentation/` 존재 → `lib/presentation/{name}/`
5. 위 모두 없을 경우 → `lib/ui/{name}/` (기본값)

결정된 경로를 `{OUTPUT_DIR}`로 사용한다.

Bash 도구로 디렉토리를 생성한다:
```bash
mkdir -p {OUTPUT_DIR}
```

---

### Step 6: 패키지명 확인

pubspec.yaml에서 `name:` 필드를 읽어 패키지명(`{package}`)을 추출한다.

예) `name: my_app` → `{package}` = `my_app`

---

### Step 7: 패턴별 파일 생성

선택한 패턴에 따라 아래 템플릿으로 파일을 생성한다.
`{name}` = snake_case, `{Name}` = PascalCase, `{camelName}` = camelCase, `{package}` = 패키지명

---

#### ■ MVVM 패턴 (3개 파일)

생성 파일: `{name}_state.dart`, `{name}_view_model.dart`, `{name}_screen.dart`

---

**`{OUTPUT_DIR}/{name}_state.dart`**

```dart
enum {Name}Status {
  initial,
  loading,
  success,
  error,
}

class {Name}State {
  final {Name}Status status;
  final String? errorMessage;

  const {Name}State({
    this.status = {Name}Status.initial,
    this.errorMessage,
  });

  {Name}State copyWith({
    {Name}Status? status,
    String? errorMessage,
  }) {
    return {Name}State(
      status: status ?? this.status,
      errorMessage: errorMessage ?? this.errorMessage,
    );
  }
}
```

---

**`{OUTPUT_DIR}/{name}_view_model.dart`**

```dart
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_state.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

final {camelName}ViewModelProvider =
    NotifierProvider<{Name}ViewModel, {Name}State>(() {
  return {Name}ViewModel();
});

class {Name}ViewModel extends Notifier<{Name}State> {
  @override
  {Name}State build() {
    return const {Name}State();
  }

  Future<void> load() async {
    state = state.copyWith(status: {Name}Status.loading);
    try {
      // TODO: 비즈니스 로직 구현
      state = state.copyWith(status: {Name}Status.success);
    } catch (e) {
      state = state.copyWith(
        status: {Name}Status.error,
        errorMessage: e.toString(),
      );
    }
  }
}
```

---

**`{OUTPUT_DIR}/{name}_screen.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_state.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_view_model.dart';

class {Name}Screen extends ConsumerStatefulWidget {
  const {Name}Screen({super.key});

  @override
  ConsumerState<{Name}Screen> createState() => _{Name}ScreenState();
}

class _{Name}ScreenState extends ConsumerState<{Name}Screen> {
  @override
  void initState() {
    super.initState();
    Future.microtask(
      () => ref.read({camelName}ViewModelProvider.notifier).load(),
    );
  }

  @override
  Widget build(BuildContext context) {
    ref.listen<{Name}State>({camelName}ViewModelProvider, (previous, next) {
      if (next.status == {Name}Status.error) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(next.errorMessage ?? 'Error')),
        );
      }
    });

    final state = ref.watch({camelName}ViewModelProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('{Name}')),
      body: switch (state.status) {
        {Name}Status.loading || {Name}Status.initial => const Center(
            child: CircularProgressIndicator(),
          ),
        {Name}Status.error => Center(
            child: Text(state.errorMessage ?? 'Error'),
          ),
        {Name}Status.success => const Center(
            child: Text('{Name} Screen'),
          ),
      },
    );
  }
}
```

---

#### ■ MVI 패턴 (4개 파일)

생성 파일: `{name}_state.dart`, `{name}_intent.dart`, `{name}_view_model.dart`, `{name}_screen.dart`

---

**`{OUTPUT_DIR}/{name}_state.dart`**

```dart
enum {Name}Status {
  initial,
  loading,
  success,
  error,
}

class {Name}State {
  final {Name}Status status;
  final String? errorMessage;

  const {Name}State({
    this.status = {Name}Status.initial,
    this.errorMessage,
  });

  {Name}State copyWith({
    {Name}Status? status,
    String? errorMessage,
  }) {
    return {Name}State(
      status: status ?? this.status,
      errorMessage: errorMessage ?? this.errorMessage,
    );
  }
}
```

---

**`{OUTPUT_DIR}/{name}_intent.dart`**

```dart
import 'package:flutter/foundation.dart';

@immutable
sealed class {Name}Intent {}

class {Name}Load extends {Name}Intent {}
```

---

**`{OUTPUT_DIR}/{name}_view_model.dart`**

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_state.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_intent.dart';

final {camelName}ViewModelProvider =
    NotifierProvider<{Name}ViewModel, {Name}State>(() {
  return {Name}ViewModel();
});

class {Name}ViewModel extends Notifier<{Name}State> {
  @override
  {Name}State build() {
    return const {Name}State();
  }

  void onIntent({Name}Intent intent) {
    switch (intent) {
      case {Name}Load():
        _load();
    }
  }

  Future<void> _load() async {
    state = state.copyWith(status: {Name}Status.loading);
    try {
      // TODO: 비즈니스 로직 구현
      state = state.copyWith(status: {Name}Status.success);
    } catch (e) {
      state = state.copyWith(
        status: {Name}Status.error,
        errorMessage: e.toString(),
      );
    }
  }
}
```

---

**`{OUTPUT_DIR}/{name}_screen.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_state.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_view_model.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_intent.dart';

class {Name}Screen extends ConsumerStatefulWidget {
  const {Name}Screen({super.key});

  @override
  ConsumerState<{Name}Screen> createState() => _{Name}ScreenState();
}

class _{Name}ScreenState extends ConsumerState<{Name}Screen> {
  @override
  void initState() {
    super.initState();
    Future.microtask(
      () => ref
          .read({camelName}ViewModelProvider.notifier)
          .onIntent({Name}Load()),
    );
  }

  @override
  Widget build(BuildContext context) {
    ref.listen<{Name}State>({camelName}ViewModelProvider, (previous, next) {
      if (next.status == {Name}Status.error) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(next.errorMessage ?? 'Error')),
        );
      }
    });

    final state = ref.watch({camelName}ViewModelProvider);

    return Scaffold(
      appBar: AppBar(title: const Text('{Name}')),
      body: switch (state.status) {
        {Name}Status.loading || {Name}Status.initial => const Center(
            child: CircularProgressIndicator(),
          ),
        {Name}Status.error => Center(
            child: Text(state.errorMessage ?? 'Error'),
          ),
        {Name}Status.success => const Center(
            child: Text('{Name} Screen'),
          ),
      },
    );
  }
}
```

---

#### ■ TCA 패턴 (4개 파일)

생성 파일: `{name}_state.dart`, `{name}_action.dart`, `{name}_reducer.dart`, `{name}_screen.dart`

---

**`{OUTPUT_DIR}/{name}_state.dart`**

```dart
class {Name}State {
  final bool isLoading;
  final String? errorMessage;

  const {Name}State({
    this.isLoading = false,
    this.errorMessage,
  });

  bool get hasError => errorMessage != null;

  {Name}State copyWith({
    bool? isLoading,
    String? errorMessage,
  }) {
    return {Name}State(
      isLoading: isLoading ?? this.isLoading,
      errorMessage: errorMessage,
    );
  }
}
```

---

**`{OUTPUT_DIR}/{name}_action.dart`**

```dart
sealed class {Name}Action {}

class {Name}OnAppear extends {Name}Action {}

class {Name}OnLoaded extends {Name}Action {}

class {Name}OnError extends {Name}Action {
  final String message;
  {Name}OnError(this.message);
}
```

---

**`{OUTPUT_DIR}/{name}_reducer.dart`**

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_state.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_action.dart';

final {camelName}Provider =
    NotifierProvider<{Name}Reducer, {Name}State>(() {
  return {Name}Reducer();
});

class {Name}Reducer extends Notifier<{Name}State> {
  @override
  {Name}State build() {
    return const {Name}State();
  }

  Future<void> send({Name}Action action) async {
    switch (action) {
      case {Name}OnAppear():
        state = state.copyWith(isLoading: true);
        try {
          // TODO: 비즈니스 로직 구현
          await send({Name}OnLoaded());
        } catch (e) {
          await send({Name}OnError(e.toString()));
        }

      case {Name}OnLoaded():
        state = state.copyWith(isLoading: false);

      case {Name}OnError(:final message):
        state = state.copyWith(isLoading: false, errorMessage: message);
    }
  }
}
```

---

**`{OUTPUT_DIR}/{name}_screen.dart`**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_state.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_action.dart';
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_reducer.dart';

class {Name}Screen extends ConsumerStatefulWidget {
  const {Name}Screen({super.key});

  @override
  ConsumerState<{Name}Screen> createState() => _{Name}ScreenState();
}

class _{Name}ScreenState extends ConsumerState<{Name}Screen> {
  @override
  void initState() {
    super.initState();
    Future.microtask(
      () => ref.read({camelName}Provider.notifier).send({Name}OnAppear()),
    );
  }

  @override
  Widget build(BuildContext context) {
    final state = ref.watch({camelName}Provider);

    if (state.hasError) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(state.errorMessage!)),
        );
      });
    }

    return Scaffold(
      appBar: AppBar(title: const Text('{Name}')),
      body: state.isLoading
          ? const Center(child: CircularProgressIndicator())
          : const Center(child: Text('{Name} Screen')),
    );
  }
}
```

---

### Step 8: import 경로 계산 규칙

Step 7에서 `{OUTPUT_DIR_PACKAGE}`는 `{OUTPUT_DIR}`에서 `lib/` 이후의 경로이다.

예시:
- `{OUTPUT_DIR}` = `lib/ui/sign_up` → `{OUTPUT_DIR_PACKAGE}` = `ui/sign_up`
- `{OUTPUT_DIR}` = `lib/features/home` → `{OUTPUT_DIR_PACKAGE}` = `features/home`

---

### Step 9: go_router 파일 찾기 및 라우트 등록

#### 9-1. 라우터 파일 탐색

Bash 도구로 GoRouter를 사용하는 파일을 찾는다:

```bash
grep -rl "GoRouter\|go_router" lib/ --include="*.dart" | grep -v ".g.dart" | head -10
```

결과에서 라우터 설정 파일(GoRouter 인스턴스를 정의하는 파일)을 식별한다.
`routes:` 또는 `_routes` 키워드가 있는 파일을 우선 선택한다.

```bash
grep -rl "routes:" lib/ --include="*.dart" | head -5
```

라우터 파일을 찾지 못하면:
```
⚠️ go_router 설정 파일을 찾을 수 없습니다.
생성된 파일에 다음 라우트를 수동으로 추가하세요:

GoRoute(
  path: '/{name}',
  builder: (context, state) => const {Name}Screen(),
),
```
위 메시지를 출력하고 Step 10으로 이동한다.

#### 9-2. AppRoute enum 확인 (있는 경우)

Bash 도구로 AppRoute enum 파일을 찾는다:

```bash
grep -rl "enum AppRoute\|class AppRoute" lib/ --include="*.dart" | head -3
```

AppRoute enum이 존재하는 경우:

Read 도구로 해당 파일을 읽는다.

enum 마지막 항목 뒤에 새 항목을 추가한다. 마지막 항목의 세미콜론(`;`)을 쉼표(`,`)로 바꾸고 새 항목을 추가한다.

```dart
{camelName}(path: '/{name}', params: []);
```

Edit 도구로 수정한다.

AppRoute enum이 없는 경우: 이 단계를 건너뛴다.

#### 9-3. 라우터 파일에 import 및 GoRoute 추가

Read 도구로 라우터 파일을 읽는다.

**import 추가:**

기존 dart import 블록의 끝부분을 찾아 새 import를 추가한다:

```dart
import 'package:{package}/{OUTPUT_DIR_PACKAGE}/{name}_screen.dart';
```

**GoRoute 추가:**

`routes:` 배열 또는 `_routes` getter의 마지막 `GoRoute(...)` 항목 뒤, `];` 직전에 추가한다.

AppRoute enum이 있는 경우:
```dart
GoRoute(
  path: AppRoute.{camelName}.path,
  builder: (context, state) => const {Name}Screen(),
),
```

AppRoute enum이 없는 경우:
```dart
GoRoute(
  path: '/{name}',
  builder: (context, state) => const {Name}Screen(),
),
```

Edit 도구로 수정한다.

---

### Step 10: 완료 메시지 출력

```
✅ {패턴} 화면 생성 완료!

생성된 파일:
  {OUTPUT_DIR}/
  ├── {name}_state.dart
  ├── {name}_intent.dart   (MVI만 해당)
  ├── {name}_action.dart   (TCA만 해당)
  ├── {name}_view_model.dart  (MVVM, MVI만 해당)
  ├── {name}_reducer.dart  (TCA만 해당)
  └── {name}_screen.dart

라우터 등록:
  - {라우터 파일 경로}: GoRoute 추가 완료
  - {AppRoute 파일 경로}: enum 항목 추가 완료  (AppRoute가 있는 경우)

네비게이션 사용법:
  context.go('/{name}');
  또는
  context.push('/{name}');
  또는 (AppRoute가 있는 경우)
  context.go(AppRoute.{camelName}.path);

다음 단계:
  - ViewModel/Reducer에 필요한 UseCase를 주입하세요.
  - State와 Intent/Action을 비즈니스 로직에 맞게 확장하세요.
```
