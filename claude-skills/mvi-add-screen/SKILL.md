---
name: mvi-add-screen
description: Flutter MVI 패턴 화면 생성 스킬. 이름을 입력받아 {name}_screen, {name}_view_model, {name}_intent, {name}_state 4개의 Dart 파일을 생성한다.
tools: AskUserQuestion, Write, Bash
---

# Add MVI Screen

Flutter MVI(Model-View-Intent) 패턴에 따라 화면을 구성하는 4개의 Dart 파일을 자동 생성하는 스킬이다.

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

### Step 1: 이름 입력받기

AskUserQuestion 도구로 사용자에게 화면 이름을 물어본다.
이때, 예시는 제공하지 않고 사용자로부터 직접 입력받는다.

```
화면 이름을 입력하세요 (snake_case, 예: my_profile, home_tab, sign_up):
```

입력받은 이름을 `{name}`으로 사용한다. (예: `sign_up`)

### Step 2: 이름 변환

입력받은 `snake_case` 이름을 `PascalCase`로 변환한다.

- `sign_up` → `SignUp`
- `my_profile` → `MyProfile`
- `home_tab` → `HomeTab`

변환 규칙: `_`로 분리한 각 단어의 첫 글자를 대문자로 변환하고 합친다.

### Step 3: 출력 디렉토리 결정

파일을 생성할 디렉토리는 `{PROJECT_ROOT}/lib/ui/{name}/`이다.

Bash 도구로 디렉토리를 생성한다:
```bash
mkdir -p {PROJECT_ROOT}/lib/ui/{name}
```

### Step 4: 4개의 파일 생성

아래 템플릿을 사용하여 파일을 생성한다. `{name}`은 snake_case, `{Name}`은 PascalCase로 치환한다.

---

#### 4-1. `{name}_state.dart`

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

#### 4-2. `{name}_intent.dart`

```dart
import 'package:flutter/foundation.dart';

@immutable
sealed class {Name}Intent {}

class {Name}Load extends {Name}Intent {}
```

---

#### 4-3. `{name}_view_model.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:{PACKAGE_NAME}/ui/{name}/{name}_state.dart';
import 'package:{PACKAGE_NAME}/ui/{name}/{name}_intent.dart';

final {camelName}ViewModelProvider = NotifierProvider<{Name}ViewModel, {Name}State>(() {
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

`{camelName}`은 PascalCase의 첫 글자만 소문자로 바꾼 camelCase이다. (예: `SignUp` → `signUp`, `HomeTab` → `homeTab`)

---

#### 4-4. `{name}_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:{PACKAGE_NAME}/ui/{name}/{name}_state.dart';
import 'package:{PACKAGE_NAME}/ui/{name}/{name}_view_model.dart';
import 'package:{PACKAGE_NAME}/ui/{name}/{name}_intent.dart';

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
      () => ref.read({camelName}ViewModelProvider.notifier).onIntent({Name}Load()),
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

### Step 5: 완료 메시지 출력

생성된 파일 목록을 사용자에게 안내한다:

```
✅ MVI 화면 파일이 생성되었습니다:

  {PROJECT_ROOT}/lib/ui/{name}/
  ├── {name}_state.dart
  ├── {name}_intent.dart
  ├── {name}_view_model.dart
  └── {name}_screen.dart

다음 단계:
- 라우터에 {Name}Screen을 등록하세요.
- ViewModel에 필요한 UseCase를 주입하세요.
- Intent와 State를 비즈니스 로직에 맞게 확장하세요.
```
