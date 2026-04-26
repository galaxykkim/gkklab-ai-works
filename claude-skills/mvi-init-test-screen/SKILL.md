---
name: mvi-init-test-screen
description: Flutter 프로젝트에 테스트 허브 화면을 추가하는 스킬. test_screen, test_view_model, test_intent, test_state 4개의 Dart 파일을 {PROJECT_ROOT}/lib/test/ 폴더에 생성하고, 앱 초기 진입점을 TestScreen으로 변경한다. TestScreen 최상단에는 MainScreen으로 이동하는 버튼이 포함된다.
tools: Bash, Write, Read, Edit
---

# Add Test Screen

앱 개발 시 편의성을 위해 초기 실행 과정(스플래시, 튜토리얼, 인증 등)을 생략하고 원하는 화면에 바로 접근할 수 있는 테스트 허브 화면을 추가하는 스킬이다.

- `{PROJECT_ROOT}/lib/test/` 폴더에 MVI 패턴의 4개 파일 생성
- `AppRoute` enum에 `test` 항목 추가
- `app_router.dart`의 `initialLocation`을 `AppRoute.test.path`로 변경
- `TestScreen` 최상단에 MainScreen으로 이동하는 버튼 포함

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

### Step 1: 파일/디렉토리 확인

Bash 도구로 라우터 파일과 프로젝트 구조를 확인한다:
```bash
ls {PROJECT_ROOT}/lib/core/route/
```

파일이 없으면 사용자에게 경로가 다를 수 있다고 안내하고 중단한다.

### Step 2: test 디렉토리 생성

```bash
mkdir -p {PROJECT_ROOT}/lib/test
```

### Step 3: 4개의 파일 생성

Write 도구로 아래 4개 파일을 생성한다.

---

#### 3-1. `{PROJECT_ROOT}/lib/test/test_state.dart`

```dart
enum TestStatus {
  initial,
}

class TestState {
  final TestStatus status;

  const TestState({
    this.status = TestStatus.initial,
  });

  TestState copyWith({
    TestStatus? status,
  }) {
    return TestState(
      status: status ?? this.status,
    );
  }
}
```

---

#### 3-2. `{PROJECT_ROOT}/lib/test/test_intent.dart`

```dart
import 'package:flutter/foundation.dart';

@immutable
sealed class TestIntent {}
```

---

#### 3-3. `{PROJECT_ROOT}/lib/test/test_view_model.dart`

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:{PACKAGE_NAME}/test/test_state.dart';
import 'package:{PACKAGE_NAME}/test/test_intent.dart';

final testViewModelProvider = NotifierProvider<TestViewModel, TestState>(() {
  return TestViewModel();
});

class TestViewModel extends Notifier<TestState> {
  @override
  TestState build() {
    return const TestState();
  }

  void onIntent(TestIntent intent) {
    // TODO: 인텐트 처리 로직 추가
  }
}
```

---

#### 3-4. `{PROJECT_ROOT}/lib/test/test_screen.dart`

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:{PACKAGE_NAME}/core/route/app_route.dart';
import 'package:{PACKAGE_NAME}/test/test_view_model.dart';

class TestScreen extends ConsumerWidget {
  const TestScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Test Hub'),
        backgroundColor: Colors.amber,
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // 정상 앱 실행 흐름
            ElevatedButton.icon(
              onPressed: () => context.go(AppRoute.main.path),
              icon: const Icon(Icons.play_arrow),
              label: const Text('앱 정상 실행 (Main Screen)'),
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.green,
                foregroundColor: Colors.white,
                padding: const EdgeInsets.symmetric(vertical: 14),
              ),
            ),
            const SizedBox(height: 8),
            const Divider(height: 32),
            const Text(
              '테스트 화면 바로가기',
              style: TextStyle(fontSize: 14, color: Colors.grey),
            ),
            const SizedBox(height: 8),
            // TODO: 테스트할 화면/기능 버튼을 아래에 추가하세요.
            // 예시:
            // ElevatedButton(
            //   onPressed: () => context.push(AppRoute.rank.path),
            //   child: const Text('Rank Screen'),
            // ),
          ],
        ),
      ),
    );
  }
}
```

---

### Step 4: app_route.dart 수정

Read 도구로 `{PROJECT_ROOT}/lib/core/route/app_route.dart`를 읽는다.

AppRoute enum의 **첫 번째 항목 앞**에 `test` 항목을 추가한다.
기존 첫 번째 항목 앞에 새 항목을 넣고, 기존 마지막 항목의 세미콜론(`;`)이 남아 있어야 한다.

예시 — 기존:
```dart
enum AppRoute {
  splash(path: '/', params: []),
  ...
  signUp(path: '/sign_up', params: []);
```

변경 후:
```dart
enum AppRoute {
  test(path: '/test', params: []),
  splash(path: '/', params: []),
  ...
  signUp(path: '/sign_up', params: []);
```

Edit 도구로 enum의 첫 번째 항목 앞에 `test` 항목을 추가한다.

### Step 5: app_router.dart 수정

Read 도구로 `{PROJECT_ROOT}/lib/core/route/app_router.dart`를 읽는다.

#### 5-1. import 추가

기존 screen import 목록의 마지막 줄 다음에 추가한다:
```dart
import 'package:{PACKAGE_NAME}/test/test_screen.dart';
```

#### 5-2. initialLocation 변경

현재 파일에서 `initialLocation:` 값을 읽어 `AppRoute.test.path`로 교체한다.

```
old: initialLocation: AppRoute.{현재값}.path,
new: initialLocation: AppRoute.test.path,
```

#### 5-3. GoRoute 추가

`_routes` getter의 **첫 번째** GoRoute 앞에 test 라우트를 추가한다:
```dart
GoRoute(
  path: AppRoute.test.path,
  builder: (context, state) => const TestScreen(),
),
```

Edit 도구로 첫 번째 GoRoute 앞에 추가한다. 예시:

```
old:
    GoRoute(
      path: AppRoute.splash.path,
      ...

new:
    GoRoute(
      path: AppRoute.test.path,
      builder: (context, state) => const TestScreen(),
    ),
    GoRoute(
      path: AppRoute.splash.path,
      ...
```

### Step 6: 완료 메시지 출력

```
✅ Test Hub 화면이 추가되었습니다.

생성된 파일:
  {PROJECT_ROOT}/lib/test/
  ├── test_state.dart
  ├── test_intent.dart
  ├── test_view_model.dart
  └── test_screen.dart

수정된 파일:
  {PROJECT_ROOT}/lib/core/route/app_route.dart  (AppRoute.test 항목 추가)
  {PROJECT_ROOT}/lib/core/route/app_router.dart (import, initialLocation, GoRoute 추가)

앱을 실행하면 TestScreen이 가장 먼저 표시됩니다.
TestScreen 최상단의 '앱 정상 실행' 버튼으로 MainScreen으로 이동할 수 있습니다.

테스트할 화면을 추가하려면 /mvi-add-test-screen 스킬을 실행하세요.
```
