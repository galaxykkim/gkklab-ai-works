---
name: check-sources
description: 작성된 코드가 클린아키텍처 기반의 MVVM / MVI / TCA 패턴을 준수했는지 검사하고 보완 여부를 사용자에게 확인받는 스킬.
tools: AskUserQuestion, Read, Edit, Bash
---

# Check Sources

작성된 코드가 클린아키텍처 원칙과 선택한 UI 패턴(MVVM / MVI / TCA)을 준수하는지 검사한다.

---

## 실행 절차

### Step 1: 검사 대상 입력받기

AskUserQuestion 도구로 검사할 파일 또는 폴더를 직접 입력받는다.
선택지는 제시하지 않는다.

```
검사할 파일 또는 폴더를 입력하세요.
- 파일: 경로를 그대로 입력 (예: lib/ui/home/home_screen.dart)
- 폴더: 이름 앞에 /를 붙여 입력 (예: /home, /ui)
- 복수 입력 시 쉼표(,)로 구분 (예: /home, lib/ui/rank/rank_screen.dart, /core)
```

입력값을 `RAW_INPUT`으로 저장한다.

---

### Step 2: 파일 목록 수집

`RAW_INPUT`을 쉼표(`,`)로 분리하고 각 항목을 trim한다.

각 항목을 아래 규칙으로 처리하여 `TARGET_FILES` 목록을 구성한다.

#### 항목이 `/`로 시작하는 경우 (폴더)

`/` 를 제거한 이름을 폴더명으로 사용하고 Bash 도구로 하위 소스 파일을 수집한다.

```bash
find . -type d -name "{폴더명}" ! -path "*/.*" ! -path "*/build/*" ! -path "*/node_modules/*" | head -1
```

찾은 디렉토리 경로를 기준으로 소스 파일을 수집한다:

```bash
find {디렉토리경로} \
  -type f \( -name "*.dart" -o -name "*.swift" -o -name "*.kt" -o -name "*.tsx" -o -name "*.ts" \) \
  ! -name "*.freezed.dart" \
  ! -name "*.g.dart" \
  2>/dev/null | sort
```

폴더를 찾지 못하면 사용자에게 아래 메시지를 출력하고 해당 항목을 건너뛴다:
```
⚠️ '{폴더명}' 폴더를 찾을 수 없습니다. 건너뜁니다.
```

#### 항목이 `/`로 시작하지 않는 경우 (파일)

해당 경로의 파일이 존재하는지 확인한다:

```bash
ls {파일경로} 2>/dev/null
```

파일이 존재하면 `TARGET_FILES`에 추가한다.
파일이 없으면 사용자에게 아래 메시지를 출력하고 해당 항목을 건너뛴다:
```
⚠️ '{파일경로}' 파일을 찾을 수 없습니다. 건너뜁니다.
```

#### 수집 완료 후

`TARGET_FILES`가 비어 있으면 아래 메시지를 출력하고 종료한다:
```
❌ 검사할 파일을 찾지 못했습니다. 입력값을 확인하세요.
```

---

### Step 3: 패턴 선택 입력받기

AskUserQuestion 도구로 검사 기준 패턴을 물어본다.

```
어떤 아키텍처 패턴 기준으로 검사할까요?

1) MVVM
2) MVI
3) TCA

번호 또는 이름을 입력하세요:
```

입력값을 정규화한다.
- `1` 또는 `mvvm` (대소문자 무관) → `MVVM`
- `2` 또는 `mvi` → `MVI`
- `3` 또는 `tca` → `TCA`
- 그 외 → 다시 물어본다.

---

### Step 4: 파일 내용 읽기

Read 도구로 TARGET_FILES의 각 파일을 읽는다.
파일이 20개를 초과하면 먼저 파일 경로 목록을 사용자에게 보여주고, 계속 진행할지 확인한다.

---

### Step 5: 클린아키텍처 + 패턴 준수 여부 검사

읽어들인 파일들을 아래 기준으로 분석한다.

#### 공통 클린아키텍처 원칙

| 항목 | 설명 |
|------|------|
| CA-1 | **레이어 분리**: Presentation / Domain / Data 레이어가 구분되어 있는가 |
| CA-2 | **의존성 방향**: Presentation → Domain ← Data 방향이어야 하며, Domain이 Data를 import하면 위반 |
| CA-3 | **UseCase 존재**: 비즈니스 로직이 ViewModel/Reducer가 아닌 UseCase에 있는가 |
| CA-4 | **Repository 추상화**: Repository는 인터페이스(추상 클래스)가 Domain에, 구현체가 Data에 있는가 |
| CA-5 | **Entity 순수성**: 도메인 모델(Entity)이 외부 프레임워크나 데이터 모델에 의존하지 않는가 |

#### MVVM 전용 체크리스트

| 항목 | 설명 |
|------|------|
| MVVM-1 | View(Screen)는 UI 렌더링만 담당하고, 비즈니스 로직이 없는가 |
| MVVM-2 | ViewModel은 View(Widget/UIView)를 직접 import하지 않는가 |
| MVVM-3 | ViewModel은 상태(State)를 Observable/Stream/StateNotifier 등으로 노출하는가 |
| MVVM-4 | ViewModel이 직접 Repository나 DataSource를 호출하지 않고 UseCase를 통하는가 |
| MVVM-5 | 두 방향 바인딩 또는 단방향 데이터 흐름이 일관성 있게 유지되는가 |

#### MVI 전용 체크리스트

| 항목 | 설명 |
|------|------|
| MVI-1 | Intent가 sealed class / enum으로 정의되어 있고 로직이 없는가 |
| MVI-2 | State가 불변(immutable, copyWith 또는 data class)으로 정의되어 있는가 |
| MVI-3 | View는 State를 구독하고, 사용자 이벤트를 Intent로만 전달하는가 |
| MVI-4 | ViewModel/Processor는 Intent → UseCase → 새 State 흐름을 유지하는가 |
| MVI-5 | Side Effect(네비게이션, 토스트 등)가 별도 채널(SideEffect/Event)로 분리되어 있는가 |

#### TCA 전용 체크리스트

| 항목 | 설명 |
|------|------|
| TCA-1 | State가 값 타입(struct / data class)이고 불변에 가깝게 관리되는가 |
| TCA-2 | Action이 enum으로 정의되어 있고 모든 사용자 이벤트와 Effect 콜백을 포함하는가 |
| TCA-3 | Reducer가 순수 함수에 가깝고, 부작용을 Effect로만 반환하는가 |
| TCA-4 | 의존성(네트워크, DB 등)이 DependencyKey / DependencyValues로 주입되는가 |
| TCA-5 | 자식 Feature가 부모 Feature의 State를 직접 참조하지 않고 Scope/IfLet으로 연결되는가 |

---

### Step 6: 분석 결과 정리 및 사용자 확인

검사 결과를 아래 형식으로 정리하여 사용자에게 보여준다.

```
## 아키텍처 검사 결과 ({패턴명} 기준)

검사 파일 수: {N}개
검사 기준: 클린아키텍처 + {패턴명}

---

### ✅ 준수 항목
- {항목ID}: {간략한 설명}
...

---

### ⚠️ 보완 필요 항목

#### 1. {항목ID} — {항목 제목}
- **파일**: `{파일 경로}:{라인 번호}`
- **문제**: {구체적으로 어떤 코드가 어떤 원칙을 위반했는지}
- **제안**: {어떻게 수정하면 되는지 간략히}

#### 2. ...

---

위 {N}개의 항목을 자동으로 보완할까요? (예/아니오)
일부만 보완하려면 번호를 쉼표로 입력하세요. (예: 1,3)
```

보완 필요 항목이 없으면:
```
✅ 모든 파일이 {패턴명} 기반 클린아키텍처 원칙을 준수하고 있습니다.
```

---

### Step 7: 사용자 승인 후 보완

AskUserQuestion 도구로 보완 여부를 확인한다.

- `예` 또는 `y` → 모든 항목 보완
- 숫자 목록(예: `1,3`) → 해당 항목만 보완
- `아니오` 또는 `n` → 보완하지 않고 종료

승인된 항목에 대해서만 Edit 도구로 파일을 수정한다.

수정 규칙:
- 기존 로직을 최대한 보존하고 최소한으로만 변경한다.
- 파일을 수정하기 전에 반드시 Read 도구로 최신 내용을 확인한다.
- 하나의 항목 수정이 끝나면 다음 항목으로 넘어간다.
- 여러 항목이 같은 파일에 있으면 한 번의 Edit으로 묶어서 처리한다.

보완 완료 후 수정된 파일 목록을 출력한다:
```
✅ 보완 완료:
  - {파일 경로} ({수정된 항목 ID})
  ...
```
