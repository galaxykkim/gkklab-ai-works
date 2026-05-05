---
name: android-add-screen
description: Android Jetpack Compose 프로젝트에서 navigation-compose NavHost 의존성을 확인하고, 화면 이름과 패턴(MVVM/MVI/TCA)을 입력받아 필요한 Kotlin 파일을 생성한 뒤 NavHost에 자동 등록하는 글로벌 스킬.
tools: AskUserQuestion, Read, Write, Edit, Bash
---

# Android Add Screen

Android Jetpack Compose 프로젝트에서 MVVM / MVI / TCA 패턴 중 하나를 선택하여 화면 관련 Kotlin 파일을 생성하고, NavHost에 라우트를 자동 등록하는 스킬이다.

---

## 실행 절차

### Step 1: Android 프로젝트 확인

Bash 도구로 AndroidManifest.xml을 찾아 Android 프로젝트 여부를 검증한다:

```bash
find . -name "AndroidManifest.xml" -maxdepth 6 2>/dev/null | head -1
```

찾지 못하면 아래 메시지를 출력하고 종료한다:
```
❌ AndroidManifest.xml을 찾을 수 없습니다. Android 프로젝트 루트에서 실행하세요.
```

---

### Step 2: NavHost(navigation-compose) 의존성 체크

Bash 도구로 build.gradle 또는 build.gradle.kts 파일에서 navigation-compose 의존성을 찾는다:

```bash
grep -r "navigation-compose\|navigation\.compose\|androidx\.navigation" \
  --include="*.gradle" --include="*.kts" --include="*.toml" \
  -l . 2>/dev/null | head -5
```

아무 파일도 없으면 추가로 소스 파일에서 NavHost 사용 여부를 확인한다:

```bash
grep -r "NavHost\b" --include="*.kt" -l . 2>/dev/null | head -3
```

두 검사 모두 결과가 없으면 아래 메시지를 출력하고 종료한다:

```
❌ navigation-compose 의존성을 찾을 수 없습니다.

build.gradle.kts에 아래 의존성을 추가한 뒤 다시 실행하세요:
  implementation("androidx.navigation:navigation-compose:<version>")
```

의존성이 확인되면 계속 진행한다.

---

### Step 3: 패키지명 및 소스 루트 추출

#### 3-1. applicationId 추출

```bash
grep -r "applicationId" --include="*.kts" --include="*.gradle" -h . 2>/dev/null \
  | grep -v "//" | grep -v "test" | head -1
```

따옴표 안의 값을 파싱하여 `{packageName}`으로 사용한다.
예) `applicationId = "com.example.myapp"` → `com.example.myapp`

찾지 못하면 AndroidManifest.xml에서 추출한다:

```bash
find . -name "AndroidManifest.xml" -maxdepth 6 \
  | xargs grep -o 'package="[^"]*"' 2>/dev/null | head -1
```

#### 3-2. 소스 루트 탐색

```bash
find . -type d \( -path "*/src/main/kotlin" -o -path "*/src/main/java" \) \
  2>/dev/null | head -1
```

찾은 경로를 `{srcRoot}`로 사용한다. (예: `./app/src/main/kotlin`)

소스 루트를 찾지 못하면 아래 메시지를 출력하고 종료한다:
```
❌ 소스 루트 디렉토리(src/main/kotlin 또는 src/main/java)를 찾을 수 없습니다.
```

#### 3-3. 패키지 경로 변환

`{packageName}`의 `.`을 `/`로 변환하여 `{packagePath}`를 만든다.
예) `com.example.myapp` → `com/example/myapp`

---

### Step 4: 화면 이름 입력받기

AskUserQuestion 도구로 화면 이름을 입력받는다. 선택 옵션을 절대 제시하지 않고 직접 입력받는다.

```
생성할 화면 이름을 입력하세요:
```

입력값을 아래 규칙으로 변환한다:

| 입력 형식 | 예시 | {Name} (PascalCase) | {name} (소문자, 언더스코어 제거) | {routeName} (snake_case) |
|-----------|------|---------------------|----------------------------------|--------------------------|
| PascalCase | `UserProfile` | `UserProfile` | `userprofile` | `user_profile` |
| camelCase | `userProfile` | `UserProfile` | `userprofile` | `user_profile` |
| snake_case | `user_profile` | `UserProfile` | `userprofile` | `user_profile` |
| 단어 | `Home` | `Home` | `home` | `home` |

- `{Name}`: 클래스명에 사용 (PascalCase)
- `{name}`: 패키지명에 사용 (소문자, 구분자 없음)
- `{routeName}`: NavHost route 문자열에 사용 (snake_case)

---

### Step 5: 패턴 입력받기

AskUserQuestion 도구로 패턴을 입력받는다. 선택 옵션을 절대 제시하지 않고 직접 입력받는다.

```
사용할 아키텍처 패턴을 입력하세요 (MVVM / MVI / TCA):
```

입력값을 정규화한다 (대소문자 무관):
- `mvvm` → `MVVM`
- `mvi` → `MVI`
- `tca` → `TCA`

MVVM / MVI / TCA 이외의 값이 입력되면 다시 물어본다.

---

### Step 6: 출력 디렉토리 생성

출력 경로: `{srcRoot}/{packagePath}/ui/{name}/`

```bash
mkdir -p {srcRoot}/{packagePath}/ui/{name}
```

---

### Step 7: 패턴별 Kotlin 파일 생성

선택한 패턴에 따라 아래 템플릿으로 파일을 생성한다.

---

#### ■ MVVM 패턴 (3개 파일)

생성 파일: `{Name}State.kt`, `{Name}ViewModel.kt`, `{Name}Screen.kt`

---

**`{Name}State.kt`**

```kotlin
package {packageName}.ui.{name}

data class {Name}State(
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
)
```

---

**`{Name}ViewModel.kt`**

```kotlin
package {packageName}.ui.{name}

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

class {Name}ViewModel : ViewModel() {

    private val _state = MutableStateFlow({Name}State())
    val state: StateFlow<{Name}State> = _state.asStateFlow()

    fun load() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                // TODO: 비즈니스 로직 구현
                _state.update { it.copy(isLoading = false) }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, errorMessage = e.message) }
            }
        }
    }
}
```

---

**`{Name}Screen.kt`**

```kotlin
package {packageName}.ui.{name}

import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.window.Dialog
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun {Name}Screen(
    viewModel: {Name}ViewModel = viewModel(),
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.load()
    }

    if (state.isLoading) {
        Dialog(onDismissRequest = {}) {
            CircularProgressIndicator()
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(title = { Text("{Name}") })
        },
    ) { paddingValues ->
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues),
            contentAlignment = Alignment.Center,
        ) {
            Text("{Name} Screen")
        }
    }
}
```

---

#### ■ MVI 패턴 (4개 파일)

생성 파일: `{Name}State.kt`, `{Name}Intent.kt`, `{Name}ViewModel.kt`, `{Name}Screen.kt`

---

**`{Name}State.kt`**

```kotlin
package {packageName}.ui.{name}

data class {Name}State(
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
)
```

---

**`{Name}Intent.kt`**

```kotlin
package {packageName}.ui.{name}

sealed class {Name}Intent {
    data object Load : {Name}Intent()
}
```

---

**`{Name}ViewModel.kt`**

```kotlin
package {packageName}.ui.{name}

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

class {Name}ViewModel : ViewModel() {

    private val _state = MutableStateFlow({Name}State())
    val state: StateFlow<{Name}State> = _state.asStateFlow()

    fun onIntent(intent: {Name}Intent) {
        when (intent) {
            is {Name}Intent.Load -> load()
        }
    }

    private fun load() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                // TODO: 비즈니스 로직 구현
                _state.update { it.copy(isLoading = false) }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, errorMessage = e.message) }
            }
        }
    }
}
```

---

**`{Name}Screen.kt`**

```kotlin
package {packageName}.ui.{name}

import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.window.Dialog
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun {Name}Screen(
    viewModel: {Name}ViewModel = viewModel(),
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.onIntent({Name}Intent.Load)
    }

    if (state.isLoading) {
        Dialog(onDismissRequest = {}) {
            CircularProgressIndicator()
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(title = { Text("{Name}") })
        },
    ) { paddingValues ->
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues),
            contentAlignment = Alignment.Center,
        ) {
            Text("{Name} Screen")
        }
    }
}
```

---

#### ■ TCA 패턴 (4개 파일)

생성 파일: `{Name}State.kt`, `{Name}Action.kt`, `{Name}Reducer.kt`, `{Name}Screen.kt`

---

**`{Name}State.kt`**

```kotlin
package {packageName}.ui.{name}

data class {Name}State(
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
) {
    val hasError: Boolean get() = errorMessage != null
}
```

---

**`{Name}Action.kt`**

```kotlin
package {packageName}.ui.{name}

sealed class {Name}Action {
    data object OnAppear : {Name}Action()
    data object OnLoaded : {Name}Action()
    data class OnError(val message: String) : {Name}Action()
}
```

---

**`{Name}Reducer.kt`**

```kotlin
package {packageName}.ui.{name}

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch

class {Name}Reducer : ViewModel() {

    private val _state = MutableStateFlow({Name}State())
    val state: StateFlow<{Name}State> = _state.asStateFlow()

    fun send(action: {Name}Action) {
        when (action) {
            is {Name}Action.OnAppear -> {
                _state.update { it.copy(isLoading = true) }
                viewModelScope.launch {
                    try {
                        // TODO: 비즈니스 로직 구현
                        send({Name}Action.OnLoaded)
                    } catch (e: Exception) {
                        send({Name}Action.OnError(e.message ?: "Unknown error"))
                    }
                }
            }
            is {Name}Action.OnLoaded -> {
                _state.update { it.copy(isLoading = false) }
            }
            is {Name}Action.OnError -> {
                _state.update { it.copy(isLoading = false, errorMessage = action.message) }
            }
        }
    }
}
```

---

**`{Name}Screen.kt`**

```kotlin
package {packageName}.ui.{name}

import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.Scaffold
import androidx.compose.material3.SnackbarHost
import androidx.compose.material3.SnackbarHostState
import androidx.compose.material3.Text
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.remember
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.window.Dialog
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun {Name}Screen(
    reducer: {Name}Reducer = viewModel(),
) {
    val state by reducer.state.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        reducer.send({Name}Action.OnAppear)
    }

    LaunchedEffect(state.errorMessage) {
        state.errorMessage?.let { snackbarHostState.showSnackbar(it) }
    }

    if (state.isLoading) {
        Dialog(onDismissRequest = {}) {
            CircularProgressIndicator()
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(title = { Text("{Name}") })
        },
        snackbarHost = { SnackbarHost(snackbarHostState) },
    ) { paddingValues ->
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues),
            contentAlignment = Alignment.Center,
        ) {
            Text("{Name} Screen")
        }
    }
}
```

---

### Step 8: NavHost 파일 탐색

Bash 도구로 `NavHost(`를 실제로 호출하는 파일을 찾는다:

```bash
grep -r "NavHost(" --include="*.kt" -rl . 2>/dev/null | head -5
```

결과가 없으면 더 넓게 탐색한다:

```bash
grep -r "NavHost\b" --include="*.kt" -rl . 2>/dev/null | head -5
```

파일이 여러 개이면 `composable(` 호출도 함께 있는 파일을 우선 선택한다:

```bash
grep -rl "NavHost\b" --include="*.kt" . 2>/dev/null \
  | xargs grep -l "composable(" 2>/dev/null | head -1
```

NavHost 파일을 찾지 못하면 아래 메시지를 출력하고 Step 9로 이동한다:

```
⚠️ NavHost 파일을 자동으로 찾을 수 없습니다.
아래 코드를 NavHost 블록 안에 수동으로 추가하세요:

        composable("{routeName}") {
            {Name}Screen()
        }

그리고 파일 상단에 import를 추가하세요:
import {packageName}.ui.{name}.{Name}Screen
```

---

### Step 9: NavHost에 라우트 등록

#### 9-1. NavHost 파일 읽기

Read 도구로 NavHost 파일을 읽는다.

#### 9-2. import 추가

패턴별 필요한 import가 없으면 파일 상단 import 블록 마지막 줄 뒤에 추가한다:

**MVVM / MVI:**
```kotlin
import {packageName}.ui.{name}.{Name}Screen
```

**TCA:**
```kotlin
import {packageName}.ui.{name}.{Name}Screen
```

Edit 도구로 추가한다.

#### 9-3. composable 라우트 추가

`NavHost(...)` 블록 내부의 마지막 `composable(...)` 항목 바로 뒤에 Edit 도구로 추가한다:

```kotlin
        composable("{routeName}") {
            {Name}Screen()
        }
```

NavHost 블록에서 마지막 `composable(...)` 항목을 찾을 때, `}` 와 `}` 사이의 닫는 위치에 주의한다. NavHost 블록 전체의 닫는 `}` 바로 앞, 마지막 composable 항목 뒤에 삽입한다.

---

### Step 10: 완료 메시지 출력

```
✅ {패턴} 화면 생성 완료!

생성된 파일:
  {srcRoot}/{packagePath}/ui/{name}/
  ├── {Name}State.kt
  ├── {Name}Intent.kt      (MVI만 해당)
  ├── {Name}Action.kt      (TCA만 해당)
  ├── {Name}ViewModel.kt   (MVVM, MVI만 해당)
  ├── {Name}Reducer.kt     (TCA만 해당)
  └── {Name}Screen.kt

패키지: {packageName}.ui.{name}

NavHost 등록:
  - {NavHost 파일 경로}: composable("{routeName}") 추가 완료

네비게이션 사용법:
  navController.navigate("{routeName}")

다음 단계:
  - ViewModel/Reducer에 필요한 UseCase를 주입하세요.
  - State와 Intent/Action을 비즈니스 로직에 맞게 확장하세요.
```
