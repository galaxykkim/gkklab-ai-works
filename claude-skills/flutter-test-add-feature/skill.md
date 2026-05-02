---
skill: flutter-test-add-feature
description: 테스트 화면에 UseCase를 연결합니다. UseCase의 call 메서드를 분석하여 파라미터 입력 필드, 실행 버튼, 결과 텍스트 영역을 자동으로 추가합니다.
---

## Instructions

아래 절차를 순서대로 수행합니다.

---

### Step 1. 의존성 확인

`pubspec.yaml`을 읽어 `flutter_riverpod`가 `dependencies`에 존재하는지 확인합니다.

- 없으면 작업을 중단하고 사용자에게 안내합니다.

---

### Step 2. 테스트 화면 클래스명 입력

사용자에게 UseCase를 연결할 테스트 화면의 클래스명을 직접 텍스트로 입력받습니다.

- `AskUserQuestion`을 사용하지 않고, 사용자가 프롬프트에 직접 타이핑하여 전달하도록 안내합니다.
- 안내 문구 예시: `"UseCase를 연결할 테스트 화면의 클래스명을 입력해 주세요. (예: PhrScreen)"`

입력받은 값을 `{ScreenClassName}`으로 저장합니다.

---

### Step 3. UseCase 클래스명 입력

사용자에게 연결할 UseCase의 클래스명을 직접 텍스트로 입력받습니다.

- `AskUserQuestion`을 사용하지 않고, 사용자가 프롬프트에 직접 타이핑하여 전달하도록 안내합니다.
- 안내 문구 예시: `"연결할 UseCase의 클래스명을 입력해 주세요. (예: FetchPhrDataUseCase)"`

입력받은 값을 `{UseCaseClassName}`으로 저장합니다.

---

### Step 4. 이름 규칙 도출

`{ScreenClassName}`에서 아래 이름들을 결정합니다.

| 항목 | 규칙 | 예시 (`PhrScreen`) |
|------|------|-------------------|
| `{FeatureName}` | 끝의 `Screen` 제거 후 PascalCase 유지. `Screen`이 없으면 그대로 사용 | `Phr` |
| `{feature_name}` | `{FeatureName}` → snake_case (대문자 앞에 `_` 삽입 후 소문자화) | `phr` |
| `{filePrefix}` | `{feature_name}`이 `test_`로 시작하면 `""`, 아니면 `"test_"` | `test_` |
| `{dirPath}` | `lib/test/{feature_name}/` | `lib/test/phr/` |
| `{IntentClass}` | `{FeatureName}Intent` | `PhrIntent` |
| `{StateClass}` | `{FeatureName}State` | `PhrState` |
| `{ViewModelClass}` | `{FeatureName}ViewModel` | `PhrViewModel` |
| `{providerName}` | `{feature_name}ViewModelProvider` (camelCase) | `phrViewModelProvider` |

> snake_case 변환 예시: `SamsungHealth` → `samsung_health`, `BloodGlucose` → `blood_glucose`

---

### Step 5. 파일 위치 탐색

아래 파일들을 프로젝트에서 탐색합니다.

| 역할 | 탐색 방법 |
|------|-----------|
| **feature intent 파일** | `{dirPath}` 내 `*_intent.dart` 파일 검색 |
| **feature state 파일** | `{dirPath}` 내 `*_state.dart` 파일 검색 |
| **feature viewmodel 파일** | `{dirPath}` 내 `*_viewmodel.dart` 파일 검색 |
| **feature screen 파일** | `{dirPath}` 내 `*_screen.dart` 파일 검색 |
| **UseCase 파일** | `lib/` 하위에서 `class {UseCaseClassName}` 패턴으로 `.dart` 파일 검색 |

파일을 하나라도 찾지 못하면 작업을 중단하고 사용자에게 안내합니다.

모든 파일을 읽어 현재 내용을 파악합니다.

---

### Step 6. UseCase 분석

UseCase 파일에서 아래 두 가지를 추출합니다.

#### 6-1. `call` 메서드 파라미터 추출

`call` 메서드 시그니처를 파싱하여 각 파라미터의 정보를 추출합니다.

```dart
// 파싱 대상 예시
Future<List<StepRecord>> call({
  required DateTime startTime,
  required DateTime endTime,
  HealthGroupUnit groupUnit = HealthGroupUnit.daily,
})
```

각 파라미터에 대해 추출하는 정보:

| 정보 | 설명 |
|------|------|
| `paramName` | 파라미터 이름 (예: `startTime`) |
| `paramType` | Dart 타입 문자열 (예: `DateTime`, `String`, `int`) |
| `isRequired` | `required` 키워드 존재 여부 |
| `defaultValue` | 기본값 문자열 (없으면 null) |

**파라미터가 없는 경우** (`call()` 또는 `call({})`) → 입력 필드 없이 버튼과 결과 영역만 추가합니다.

#### 6-2. Provider 변수명 추출

UseCase 파일 내에 선언된 Riverpod Provider 변수명을 찾습니다.

```dart
// 예시
final fetchPhrDataUseCaseProvider = Provider<FetchPhrDataUseCase>(...);
```

찾은 변수명을 `{useCaseProviderName}`으로 저장합니다.
Provider가 없으면 `{useCaseName}Provider` 패턴으로 추론합니다.
(`{useCaseName}` = `{UseCaseClassName}`의 첫 글자를 소문자로 변환, 예: `fetchPhrDataUseCase`)

#### 6-3. 파라미터 타입별 TextField 설정

| Dart 타입 | `keyboardType` | `hintText` | 파싱 코드 |
|-----------|---------------|------------|-----------|
| `String` | 기본값 생략 | `''` (없음) | `_{paramName}Controller.text` |
| `int` | `TextInputType.number` | `'정수 입력'` | `int.tryParse(_{paramName}Controller.text) ?? 0` |
| `double` | `TextInputType.numberWithOptions(decimal: true)` | `'소수 입력'` | `double.tryParse(_{paramName}Controller.text) ?? 0.0` |
| `bool` | 기본값 생략 | `'true 또는 false'` | `_{paramName}Controller.text.trim().toLowerCase() == 'true'` |
| `DateTime` | 기본값 생략 | `'yyyy-MM-ddTHH:mm:ss'` | `DateTime.tryParse(_{paramName}Controller.text) ?? DateTime.now()` |
| 그 외 (enum 등) | 기본값 생략 | 기본값이 있으면 기본값을, 없으면 타입명 표시 | 기본값이 있으면 기본값 코드를 직접 사용, 없으면 `_{paramName}Controller.text` 사용 후 주석 `// TODO: 타입 변환 필요` 추가 |

---

### Step 7. Intent 수정

`{dirPath}/{filePrefix}{feature_name}_intent.dart` 파일에 Execute intent를 추가합니다.

이미 `Execute{FeatureName}` 클래스가 존재하면 이 단계를 건너뜁니다.

**파라미터가 있는 경우:**

```dart
class Execute{FeatureName} extends {IntentClass} {
  final {Param1Type} {param1Name};
  final {Param2Type} {param2Name};
  // ... required 파라미터만 필드로 선언
  // default값 있는 파라미터도 필드로 선언하되 required 제외

  const Execute{FeatureName}({
    required this.{param1Name},
    required this.{param2Name},
    // ...
  });
}
```

**파라미터가 없는 경우:**

```dart
class Execute{FeatureName} extends {IntentClass} {
  const Execute{FeatureName}();
}
```

---

### Step 8. State 수정

`{dirPath}/{filePrefix}{feature_name}_state.dart` 파일에 `result` 필드를 추가합니다.

이미 `result` 필드가 존재하면 이 단계를 건너뜁니다.

```dart
class {StateClass} {
  final bool isLoading;
  final String result;   // ← 추가

  const {StateClass}({
    this.isLoading = false,
    this.result = '',    // ← 추가
  });

  {StateClass} copyWith({
    bool? isLoading,
    String? result,      // ← 추가
  }) {
    return {StateClass}(
      isLoading: isLoading ?? this.isLoading,
      result: result ?? this.result,  // ← 추가
    );
  }
}
```

---

### Step 9. ViewModel 수정

`{dirPath}/{filePrefix}{feature_name}_viewmodel.dart` 파일을 수정합니다.

**① UseCase import 추가** — viewmodel 파일 위치 기준 상대경로로 UseCase 파일을 import합니다.

**② switch case 추가** — `processIntent` switch 마지막에 추가합니다.

파라미터가 있는 경우:
```dart
case Execute{FeatureName}():
  await _execute(intent as Execute{FeatureName});
```

파라미터가 없는 경우:
```dart
case Execute{FeatureName}():
  await _execute();
```

> `processIntent` 반환 타입이 `void`이면 `Future<void>`로 변경합니다. 이미 `async`이면 유지합니다.

**③ execute 메서드 추가** — 파일 마지막 메서드 아래에 추가합니다.

파라미터가 있는 경우:
```dart
Future<void> _execute(Execute{FeatureName} intent) async {
  state = state.copyWith(isLoading: true, result: '');
  try {
    final useCase = ref.read({useCaseProviderName});
    final result = await useCase(
      {param1Name}: intent.{param1Name},
      {param2Name}: intent.{param2Name},
      // ... 각 파라미터
    );
    state = state.copyWith(isLoading: false, result: result.toString());
  } catch (e) {
    state = state.copyWith(isLoading: false, result: '오류: $e');
  }
}
```

파라미터가 없는 경우:
```dart
Future<void> _execute() async {
  state = state.copyWith(isLoading: true, result: '');
  try {
    final useCase = ref.read({useCaseProviderName});
    final result = await useCase();
    state = state.copyWith(isLoading: false, result: result.toString());
  } catch (e) {
    state = state.copyWith(isLoading: false, result: '오류: $e');
  }
}
```

---

### Step 10. Screen 수정

`{dirPath}/{filePrefix}{feature_name}_screen.dart` 파일을 수정합니다.

#### 10-1. ConsumerStatefulWidget 확인

이미 `ConsumerStatefulWidget`을 상속하면 유지합니다. `ConsumerWidget`이면 `ConsumerStatefulWidget`으로 변환합니다.

#### 10-2. TextEditingController 추가 (파라미터가 있는 경우)

State 클래스(`_{ScreenClassName}State`) 필드에 각 파라미터별 컨트롤러를 추가합니다.

```dart
// required 파라미터 + default값 있는 파라미터 각각 Controller 선언
final _{param1Name}Controller = TextEditingController();
final _{param2Name}Controller = TextEditingController();
```

`default값이 있는 파라미터`는 컨트롤러의 초기값을 `..text = '{defaultValue}'`로 설정합니다.

```dart
@override
void initState() {
  super.initState();
  // default값 있는 파라미터 초기화
  _{paramWithDefault}Controller.text = '{defaultValue}';
  Future.microtask(
    () => ref.read({providerName}.notifier).processIntent(const Init()),
  );
}

@override
void dispose() {
  _{param1Name}Controller.dispose();
  _{param2Name}Controller.dispose();
  // ...
  super.dispose();
}
```

#### 10-3. build() 메서드 수정

`result` 상태를 watch 합니다.

```dart
final isLoading = ref.watch({providerName}.select((s) => s.isLoading));
final result = ref.watch({providerName}.select((s) => s.result));
```

`body`를 아래 구조로 교체합니다.

**파라미터가 있는 경우:**

```dart
body: Padding(
  padding: const EdgeInsets.all(16),
  child: Column(
    crossAxisAlignment: CrossAxisAlignment.stretch,
    children: [
      TextField(
        controller: _{param1Name}Controller,
        decoration: const InputDecoration(
          labelText: '{param1Name}',
          hintText: '{hintText}',
        ),
        keyboardType: {keyboardType},
      ),
      const SizedBox(height: 8),
      // ... 나머지 파라미터 TextField

      const SizedBox(height: 16),
      ElevatedButton(
        onPressed: isLoading
            ? null
            : () => ref.read({providerName}.notifier).processIntent(
                  Execute{FeatureName}(
                    {param1Name}: {파싱 코드},
                    {param2Name}: {파싱 코드},
                    // ...
                  ),
                ),
        child: isLoading
            ? const SizedBox(
                width: 20,
                height: 20,
                child: CircularProgressIndicator(strokeWidth: 2),
              )
            : const Text('실행'),
      ),
      const SizedBox(height: 16),
      Expanded(
        child: SingleChildScrollView(
          child: SelectableText(
            result.isEmpty ? '결과가 여기에 표시됩니다.' : result,
            style: result.startsWith('오류:')
                ? const TextStyle(color: Colors.red)
                : null,
          ),
        ),
      ),
    ],
  ),
),
```

**파라미터가 없는 경우:**

```dart
body: Padding(
  padding: const EdgeInsets.all(16),
  child: Column(
    crossAxisAlignment: CrossAxisAlignment.stretch,
    children: [
      ElevatedButton(
        onPressed: isLoading
            ? null
            : () => ref.read({providerName}.notifier).processIntent(
                  const Execute{FeatureName}(),
                ),
        child: isLoading
            ? const SizedBox(
                width: 20,
                height: 20,
                child: CircularProgressIndicator(strokeWidth: 2),
              )
            : const Text('실행'),
      ),
      const SizedBox(height: 16),
      Expanded(
        child: SingleChildScrollView(
          child: SelectableText(
            result.isEmpty ? '결과가 여기에 표시됩니다.' : result,
            style: result.startsWith('오류:')
                ? const TextStyle(color: Colors.red)
                : null,
          ),
        ),
      ),
    ],
  ),
),
```

---

### Step 11. 완료 요약 출력

```
✅ flutter-test-add-feature 완료

수정된 파일:
  {dirPath}{filePrefix}{feature_name}_intent.dart    — Execute{FeatureName} 추가
  {dirPath}{filePrefix}{feature_name}_state.dart     — result 필드 추가
  {dirPath}{filePrefix}{feature_name}_viewmodel.dart — UseCase 연결·execute 메서드 추가
  {dirPath}{filePrefix}{feature_name}_screen.dart    — 입력 필드·실행 버튼·결과 영역 추가

연결된 UseCase: {UseCaseClassName}
파라미터: {param1Name} ({param1Type}), {param2Name} ({param2Type}), ...
```

---

## Notes

- import 경로는 각 파일 위치 기준 상대경로를 사용합니다 (패키지 절대경로 사용 금지).
- `Notifier`의 `ref`는 클래스 내부에서 직접 접근 가능합니다.
- enum 타입 파라미터는 기본값이 있으면 코드에 직접 기본값을 사용하고, TextField를 생략해도 됩니다.
- `SelectableText`를 사용하여 결과 텍스트를 사용자가 복사할 수 있도록 합니다.
- UseCase의 `call` 메서드가 `void` 또는 `Future<void>`를 반환하는 경우 결과 표시 대신 `'완료'` 문자열을 state에 저장합니다.
- 이미 Execute intent나 result 필드가 존재하면 해당 단계를 건너뛰고 사용자에게 알립니다.
- `processIntent`가 비동기 처리를 위해 `void` → `Future<void>`로 변경될 수 있으며, 이 경우 기존 case들도 영향받지 않으므로 안전합니다.
