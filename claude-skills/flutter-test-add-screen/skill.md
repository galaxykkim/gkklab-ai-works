---
skill: flutter-test-add-screen
description: TestScreen에 새 화면 이동 버튼을 추가합니다. flutter_riverpod + go_router 프로젝트인지 확인 후, 연결할 화면 클래스명을 입력받아 intent/viewmodel/screen/router를 자동으로 수정합니다.
---

## Instructions

아래 절차를 순서대로 수행합니다.

---

### Step 1. 의존성 확인

`pubspec.yaml`을 읽어 `flutter_riverpod`와 `go_router` 두 패키지가 `dependencies`에 모두 존재하는지 확인합니다.

- 둘 중 하나라도 없으면 작업을 중단하고 사용자에게 아래 메시지를 출력합니다.
  > "`pubspec.yaml`에 `{누락된 패키지}`가 없습니다. 패키지를 추가한 후 다시 실행해 주세요."

---

### Step 2. 사용자 입력 수집

사용자에게 아래 두 가지를 질문합니다. 한 번에 함께 질문합니다.

1. **연결할 화면의 클래스명** (예: `SamsungHealthScreen`)
2. **라우트 경로** (예: `/samsung-health`) — 입력하지 않으면 클래스명에서 자동 추론

> 자동 추론 규칙: 클래스명 끝의 `Screen`을 제거하고, PascalCase를 kebab-case로 변환합니다.  
> 예) `SamsungHealthScreen` → `/samsung-health`

---

### Step 3. 파일 위치 탐색

아래 파일들을 프로젝트에서 탐색합니다.

| 역할 | 탐색 방법 |
|------|-----------|
| **연결 대상 화면 파일** | `class {클래스명}` 패턴으로 `.dart` 파일 검색 |
| **test_intent.dart** | `lib/` 하위에서 `test_intent.dart` 파일 검색 |
| **test_viewmodel.dart** | `lib/` 하위에서 `test_viewmodel.dart` 파일 검색 |
| **test_screen.dart** | `lib/` 하위에서 `test_screen.dart` 파일 검색 |
| **app_router.dart** | `lib/` 하위에서 라우터 파일 검색 (`GoRouter` 포함 파일) |

연결 대상 화면 파일이 존재하지 않으면 작업을 중단하고 사용자에게 안내합니다.
> "프로젝트에서 `{클래스명}` 클래스를 찾을 수 없습니다. 파일 경로를 직접 입력해 주세요."

---

### Step 4. 이름 규칙 도출

입력받은 클래스명(`{ClassName}`)으로부터 아래 이름들을 결정합니다.

| 항목 | 규칙 | 예시 (`SamsungHealthScreen`) |
|------|------|------|
| **Intent 클래스명** | `GoTo{ClassName}` | `GoToSamsungHealthScreen` |
| **ViewModel 콜백명** | `onGoTo{ClassName}` | `onGoToSamsungHealthScreen` |
| **버튼 레이블** | `Screen` 제거 후 PascalCase → 띄어쓰기 | `Samsung Health` |
| **라우트 경로** | Step 2에서 확정된 값 | `/samsung-health` |

---

### Step 5. test_intent.dart 수정

`TestIntent`의 sealed class 마지막에 새 Intent 클래스를 추가합니다.

```dart
class GoTo{ClassName} extends TestIntent {
  const GoTo{ClassName}();
}
```

---

### Step 6. test_viewmodel.dart 수정

아래 세 곳을 수정합니다.

**① 콜백 필드 추가** — 기존 `onGoToMain` 필드 아래에 추가합니다.
```dart
VoidCallback? onGoTo{ClassName};
```

**② switch case 추가** — `processIntent`의 switch 마지막 case 아래에 추가합니다.
```dart
case GoTo{ClassName}():
  _goTo{ClassName}();
```

**③ 라우팅 메서드 추가** — `_goToMain()` 메서드 아래에 추가합니다.
```dart
void _goTo{ClassName}() {
  onGoTo{ClassName}?.call();
}
```

---

### Step 7. test_screen.dart 수정

아래 두 곳을 수정합니다.

**① 콜백 주입 추가** — `build()` 내 기존 `vm.onGoToMain = ...` 아래에 추가합니다.
```dart
vm.onGoTo{ClassName} = () => context.push('{라우트 경로}');
```

**② 버튼 추가** — `SingleChildScrollView > Column`의 `children` 목록에 추가합니다.
- 기존 `_TestButton`이 있으면 마지막 `_TestButton` 아래에 추가합니다.
- 기존 `_TestButton`이 없으면 (빈 Column) `children` 목록의 첫 번째 항목으로 추가하며, 앞에 `SizedBox`를 붙이지 않습니다.
```dart
// 기존 버튼이 있는 경우:
const SizedBox(height: 12),
_TestButton(
  label: '{버튼 레이블}',
  onPressed: () => vm.processIntent(const GoTo{ClassName}()),
),

// 빈 Column인 경우 (첫 번째 버튼):
_TestButton(
  label: '{버튼 레이블}',
  onPressed: () => vm.processIntent(const GoTo{ClassName}()),
),
```

---

### Step 8. app_router.dart 수정

아래 두 곳을 수정합니다.

**① import 추가** — 연결 대상 화면 파일의 경로를 `app_router.dart` 기준 상대경로로 계산하여 import 문을 추가합니다.

**② route 추가** — `routes` 목록에 새 `GoRoute`를 추가합니다.
```dart
GoRoute(
  path: '{라우트 경로}',
  builder: (context, state) => const {ClassName}(),
),
```

---

## Notes

- import 경로는 반드시 `app_router.dart`와 `test_screen.dart` 각각의 위치 기준으로 계산한 상대경로를 사용합니다.
- `context.push()`를 사용하여 뒤로가기가 가능하도록 합니다. `go()` 대신 `push()`를 사용합니다.
- 이미 동일한 Intent 클래스나 라우트 경로가 존재하면 해당 단계를 건너뛰고 사용자에게 알립니다.
- 버튼 레이블 변환 규칙: `SamsungHealthScreen` → `Screen` 제거 → `SamsungHealth` → 단어 분리 → `Samsung Health`
