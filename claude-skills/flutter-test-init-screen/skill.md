---
skill: flutter-test-init-screen
description: 앱 개발 초기에 여러 기능을 테스트하기 위한 테스트 화면을 구현하여 앱 실행 시 가장 먼저 표시되도록 합니다. 화면 최상단에는 정상적인 동작 흐름을 위한 Main 이동 버튼을 배치합니다.
---

> **전제 패키지** : `flutter_riverpod`, `go_router`

---

## Instructions

Flutter 프로젝트에 테스트 화면을 구성하는 파일들을 아래 구조와 규칙에 따라 생성합니다.
만약 생성할 경로에 동일한 이름의 폴더나 파일이 존재할 경우, 사용자에게 직접 경로를 입력받도록 합니다.

### 생성 파일 목록

```
lib/
└── test/
    ├── test_screen.dart
    ├── test_viewmodel.dart
    ├── test_intent.dart
    └── test_state.dart
```

---

### 1. `test_intent.dart`

MVI 패턴의 Intent를 sealed class로 정의합니다.

```dart
// lib/test/test_intent.dart

sealed class TestIntent {
  const TestIntent();
}

class GoToMain extends TestIntent {
  const GoToMain();
}
```

---

### 2. `test_state.dart`

MVI 패턴의 State를 불변 클래스로 정의합니다. `copyWith`를 지원합니다.

```dart
// lib/test/test_state.dart

class TestState {
  final bool isLoading;

  const TestState({
    this.isLoading = false,
  });

  TestState copyWith({
    bool? isLoading,
  }) {
    return TestState(
      isLoading: isLoading ?? this.isLoading,
    );
  }
}
```

---

### 3. `test_viewmodel.dart`

`Notifier<TestState>` 기반의 ViewModel을 작성합니다.
- `processIntent(TestIntent)` 로 모든 Intent를 처리합니다.
- `GoToMain` Intent는 라우팅 콜백(`onGoToMain`)을 통해 처리합니다. ViewModel이 `GoRouter`에 직접 의존하지 않도록 Screen에서 콜백을 주입합니다.

```dart
// lib/test/test_viewmodel.dart

import 'dart:ui';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'test_intent.dart';
import 'test_state.dart';

class TestViewModel extends Notifier<TestState> {
  /// GoToMain 처리 시 호출할 라우팅 콜백 (Screen에서 주입)
  VoidCallback? onGoToMain;

  @override
  TestState build() => const TestState();

  void processIntent(TestIntent intent) {
    switch (intent) {
      case GoToMain():
        _goToMain();
    }
  }

  void _goToMain() {
    onGoToMain?.call();
  }
}

/// Provider 선언
final testViewModelProvider =
    NotifierProvider<TestViewModel, TestState>(TestViewModel.new);
```

---

### 4. `test_screen.dart`

`ConsumerWidget` 기반 화면을 작성합니다.
- **"Main으로 이동"** 버튼을 배치합니다 (전체 너비).
- `GoRouter`의 `context.go()`로 라우팅하며, 콜백을 ViewModel에 주입합니다.
- `/flutter-test-add-screen` 스킬로 새 화면 이동 버튼을 추가할 수 있습니다.

```dart
// lib/test/test_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'test_intent.dart';
import 'test_viewmodel.dart';

class TestScreen extends ConsumerWidget {
  const TestScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final vm = ref.read(testViewModelProvider.notifier);

    // GoRouter 라우팅 콜백 주입 — ViewModel이 context에 의존하지 않도록 분리
    vm.onGoToMain = () => context.go('/main');

    return Scaffold(
      appBar: AppBar(title: const Text('Test Screen')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // ── Main 이동 버튼 ──────────────────────────────────────
            ElevatedButton(
              onPressed: () => vm.processIntent(const GoToMain()),
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.symmetric(vertical: 16),
              ),
              child: const Text('Main으로 이동'),
            ),

            const SizedBox(height: 32),

            // ── 세로 스크롤 버튼 영역 ────────────────────────────────
            const Text(
              'Test Buttons',
              style: TextStyle(fontSize: 14, color: Colors.grey),
            ),
            const SizedBox(height: 12),
            const SingleChildScrollView(
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.stretch,
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _TestButton extends StatelessWidget {
  final String label;
  final VoidCallback onPressed;

  const _TestButton({required this.label, required this.onPressed});

  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: onPressed,
      style: ElevatedButton.styleFrom(
        padding: const EdgeInsets.symmetric(vertical: 16),
      ),
      child: Text(label),
    );
  }
}
```

---

### 5. GoRouter 등록 안내

기존 라우터 설정 파일에 `/test` 경로를 **초기 경로(initialLocation)** 로 등록해 주세요.

```dart
// lib/core/router/app_router.dart (또는 동등한 경로)

import 'package:go_router/go_router.dart';
import 'package:your_app/presentation/test/test_screen.dart';
import 'package:your_app/presentation/main/main_screen.dart';

final goRouter = GoRouter(
  initialLocation: '/test',       // 앱 시작 시 테스트 화면 표시
  routes: [
    GoRoute(
      path: '/test',
      builder: (context, state) => const TestScreen(),
    ),
    GoRoute(
      path: '/main',
      builder: (context, state) => const MainScreen(),
    ),
  ],
);
```

`main.dart`에서 `MaterialApp.router`에 연결합니다.

```dart
// lib/main.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'core/router/app_router.dart';

void main() {
  runApp(
    const ProviderScope(
      child: MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: goRouter,
    );
  }
}
```

---

## Notes

- `pubspec.yaml`에 아래 패키지가 등록되어 있어야 합니다. 없으면 추가해 주세요.
  ```yaml
  dependencies:
    flutter_riverpod: ^2.x.x
    go_router: ^14.x.x
  ```
- 새 화면 이동 버튼을 추가하려면 `/flutter-test-add-screen` 스킬을 사용하세요.
- 화면 이동에 `go()` 대신 `push()`가 필요한 경우 `onGoToMain` 콜백을 `() => context.push('/main')`으로 변경해 주세요.
- 개발 완료 후 `initialLocation`을 `/main`으로 변경하거나 `/test` 라우트를 제거하여 배포 빌드에서 테스트 화면이 노출되지 않도록 합니다.
