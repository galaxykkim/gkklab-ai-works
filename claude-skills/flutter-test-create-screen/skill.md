---
skill: flutter-test-create-screen
description: Flutter 프로젝트에 사용자로부터 입력받은 이름의 Screen/ViewModel/State/Intent 파일을 생성하고, TestScreen 버튼 및 GoRouter에 자동 등록합니다.
---

## Instructions

아래 절차를 순서대로 수행합니다.

---

### Step 1. 의존성 확인

`pubspec.yaml`을 읽어 `flutter_riverpod`와 `go_router` 두 패키지가 `dependencies`에 모두 존재하는지 확인합니다.

- 둘 중 하나라도 없으면 작업을 중단하고 사용자에게 알립니다.

---

### Step 2. 클래스명 입력

`AskUserQuestion`으로 생성할 화면의 클래스명을 직접 입력받습니다.

- 예시 형태 없이 사용자에게 직접 클래스명을 입력받는 옵션만 제공합니다.

입력받은 클래스명을 `{ClassName}`으로 저장합니다. (예: `SamsungHealthScreen`)

---

### Step 3. 이름 규칙 도출

`{ClassName}`에서 아래 이름들을 결정합니다.

| 항목 | 규칙 | 예시 (`TestSamsungHealthScreen`) |
|------|------|------|
| `{FeatureName}` | 앞의 `Test` 및 뒤의 `Screen` 제거 후 PascalCase 유지 | `SamsungHealth` |
| `{feature_name}` | `{FeatureName}`을 snake_case로 변환 | `samsung_health` |
| `{IntentClass}` | `{FeatureName}Intent` | `SamsungHealthIntent` |
| `{StateClass}` | `{FeatureName}State` | `SamsungHealthState` |
| `{ViewModelClass}` | `{FeatureName}ViewModel` | `SamsungHealthViewModel` |
| `{providerName}` | `{feature_name}ViewModelProvider` (camelCase) | `samsungHealthViewModelProvider` |
| `{routePath}` | `/{feature_name}` (하이픈 구분) | `/samsung-health` |
| `{buttonLabel}` | `{FeatureName}`의 단어를 공백으로 구분 | `Samsung Health` |
| `{dirPath}` | `lib/test/{feature_name}/` | `lib/test/samsung_health/` |

> **`{FeatureName}` 결정 규칙**: `{ClassName}` 앞의 `Test` prefix와 뒤의 `Screen` suffix를 제거합니다.  
> 예) 입력 `TestSamsungHealthScreen` → `{FeatureName}` = `SamsungHealth`  
> 예) 입력 `SamsungHealthScreen` → `{FeatureName}` = `SamsungHealth`  
> 예) 입력 `SamsungHealth` (Screen 없음) → `{FeatureName}` = `SamsungHealth`

> 파일명 prefix 규칙: 모든 생성 파일에 `test_`를 prefix로 고정 사용합니다. `{feature_name}`은 항상 `test_`를 포함하지 않으므로 중복이 발생하지 않습니다.

> snake_case 변환 규칙: PascalCase의 각 대문자 앞에 `_`를 삽입하고 소문자화합니다.  
> 예) `SamsungHealth` → `samsung_health`

> kebab-case 변환 규칙 (라우트 경로): snake_case의 `_`를 `-`로 교체합니다.  
> 예) `samsung_health` → `samsung-health` → `/samsung-health`

---

### Step 4. 파일 위치 탐색

아래 파일들을 프로젝트에서 탐색합니다.

| 역할 | 탐색 방법 |
|------|-----------|
| **test_intent.dart** | `lib/` 하위에서 `test_intent.dart` 파일 검색 |
| **test_viewmodel.dart** | `lib/` 하위에서 `test_viewmodel.dart` 파일 검색 |
| **test_screen.dart** | `lib/` 하위에서 `test_screen.dart` 파일 검색 (TestScreen 클래스 포함 파일) |
| **app_router.dart** | `lib/` 하위에서 `GoRouter` 포함 파일 검색 |

`{dirPath}` 디렉터리가 이미 존재하거나 같은 클래스명 파일이 있으면 덮어쓰기 전 사용자에게 확인합니다.

---

### Step 5. `test_{feature_name}_intent.dart` 생성

경로: `{dirPath}/test_{feature_name}_intent.dart`

```dart
// {dirPath}test_{feature_name}_intent.dart

sealed class {IntentClass} {
  const {IntentClass}();
}

class Init extends {IntentClass} {
  const Init();
}
```

---

### Step 6. `test_{feature_name}_state.dart` 생성

경로: `{dirPath}/test_{feature_name}_state.dart`

```dart
// {dirPath}test_{feature_name}_state.dart

class {StateClass} {
  final bool isLoading;

  const {StateClass}({
    this.isLoading = false,
  });

  {StateClass} copyWith({
    bool? isLoading,
  }) {
    return {StateClass}(
      isLoading: isLoading ?? this.isLoading,
    );
  }
}
```

---

### Step 7. `test_{feature_name}_viewmodel.dart` 생성

경로: `{dirPath}/test_{feature_name}_viewmodel.dart`

```dart
// {dirPath}test_{feature_name}_viewmodel.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'test_{feature_name}_intent.dart';
import 'test_{feature_name}_state.dart';

class {ViewModelClass} extends Notifier<{StateClass}> {
  @override
  {StateClass} build() => const {StateClass}();

  void processIntent({IntentClass} intent) {
    switch (intent) {
      case Init():
        _init();
    }
  }

  void _init() {
    // TODO: 초기화 로직
  }
}

/// Provider 선언
final {providerName} =
    NotifierProvider<{ViewModelClass}, {StateClass}>({ViewModelClass}.new);
```

---

### Step 8. `test_{feature_name}_screen.dart` 생성

경로: `{dirPath}/test_{feature_name}_screen.dart`

`{appBarTitle}`은 `{buttonLabel}`과 동일하게 사용합니다.

```dart
// {dirPath}test_{feature_name}_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

import 'test_{feature_name}_intent.dart';
import 'test_{feature_name}_viewmodel.dart';

class {ClassName} extends ConsumerStatefulWidget {
  const {ClassName}({super.key});

  @override
  ConsumerState<{ClassName}> createState() => _{ClassName}State();
}

class _{ClassName}State extends ConsumerState<{ClassName}> {
  @override
  void initState() {
    super.initState();
    Future.microtask(
      () => ref.read({providerName}.notifier).processIntent(const Init()),
    );
  }

  @override
  Widget build(BuildContext context) {
    final isLoading = ref.watch(
      {providerName}.select((s) => s.isLoading),
    );

    return Scaffold(
      appBar: AppBar(title: const Text('{appBarTitle}')),
      body: Center(
        child: isLoading
            ? const CircularProgressIndicator()
            : Container(),
      ),
    );
  }
}
```

---

### Step 9. test_intent.dart 수정

`TestIntent` sealed class 마지막에 추가합니다.

```dart
class GoTo{ClassName} extends TestIntent {
  const GoTo{ClassName}();
}
```

---

### Step 10. test_viewmodel.dart 수정

아래 세 곳을 수정합니다.

**① 콜백 필드 추가** — 기존 마지막 `VoidCallback?` 필드 아래에 추가합니다.
```dart
VoidCallback? onGoTo{ClassName};
```

**② switch case 추가** — `processIntent`의 switch 마지막 case 아래에 추가합니다.
```dart
case GoTo{ClassName}():
  _goTo{ClassName}();
```

**③ 라우팅 메서드 추가** — 마지막 `_goToXxx()` 메서드 아래에 추가합니다.
```dart
void _goTo{ClassName}() {
  onGoTo{ClassName}?.call();
}
```

---

### Step 11. test_screen.dart 수정

아래 두 곳을 수정합니다.

**① import 추가** — `test_intent.dart` import 아래에 새 화면 파일 import를 추가합니다.  
경로는 `test_screen.dart` 기준 상대경로로 계산하며, 파일명은 `test_{feature_name}_screen.dart`입니다.

**② 콜백 주입 추가** — `build()` 내 마지막 `vm.onGoToXxx = ...` 아래에 추가합니다.
```dart
vm.onGoTo{ClassName} = () => context.push('{routePath}');
```

**③ 버튼 추가** — `SingleChildScrollView > Column > children` 마지막 `_TestButton` 아래에 추가합니다.
```dart
const SizedBox(height: 12),
_TestButton(
  label: '{buttonLabel}',
  onPressed: () => vm.processIntent(const GoTo{ClassName}()),
),
```

---

### Step 12. app_router.dart 수정

아래 두 곳을 수정합니다.

**① import 추가** — 기존 import 블록 마지막에 추가합니다.  
경로는 `app_router.dart` 기준 상대경로로 계산합니다.

```dart
import '{상대경로}/test_{feature_name}_screen.dart';
```

**② route 추가** — `routes` 목록 마지막 `GoRoute` 아래에 추가합니다.

```dart
GoRoute(
  path: '{routePath}',
  builder: (context, state) => const {ClassName}(),
),
```

---

### Step 13. 완료 요약 출력

```
✅ flutter-test-create-screen 완료

생성된 파일:
  {dirPath}test_{feature_name}_intent.dart
  {dirPath}test_{feature_name}_state.dart
  {dirPath}test_{feature_name}_viewmodel.dart
  {dirPath}test_{feature_name}_screen.dart

수정된 파일:
  lib/test/test_intent.dart      — GoTo{ClassName} 추가
  lib/test/test_viewmodel.dart   — 콜백·case·메서드 추가
  lib/test/test_screen.dart      — 콜백 주입·버튼 추가
  lib/core/router/app_router.dart — import·route 추가

라우트 경로: {routePath}
```

---

## Notes

- import 경로는 각 파일 위치 기준 상대경로를 사용합니다 (패키지 절대경로 사용 금지).
- `context.push()`를 사용하여 뒤로가기가 가능하도록 합니다.
- 이미 동일한 라우트 경로나 Intent 클래스가 존재하면 해당 단계를 건너뛰고 사용자에게 알립니다.
- `{ClassName}` 끝에 `Screen`이 없는 경우 (예: `HealthData`), `{FeatureName}`은 `Test` 제거 후 그대로 사용합니다.
- `{FeatureName}` 도출 예시 (`Test`/`Screen` 제거 후):
  - `TestSamsungHealthScreen` → `SamsungHealth`
  - `SamsungHealthScreen` → `SamsungHealth`
  - `TestBloodGlucose` → `BloodGlucose`
- PascalCase → snake_case 변환 예시:
  - `SamsungHealth` → `samsung_health`
  - `BloodGlucose` → `blood_glucose`
  - `Webview` → `webview`
