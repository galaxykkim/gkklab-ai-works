---
skill: fastlane-distribution-android
description: Flutter 프로젝트를 Fastlane + Firebase App Distribution으로 Android 배포합니다. build.gradle의 Flavor 정보를 자동 파악하여 빌드 옵션을 제시하고, 릴리즈 노트, 테스터 정보를 대화형으로 입력받아 해당 lane을 실행합니다.
tools: Read, Bash
---

## Overview

Fastlane을 이용해 Firebase App Distribution에 Flutter 앱을 Android 배포합니다.  
빌드 옵션은 `build.gradle` / `build.gradle.kts`에서 Flavor와 buildType을 읽어 자동으로 구성합니다.

---

## Steps

### Step 1 — 프로젝트 루트 확인

`pubspec.yaml`이 있는 디렉터리를 프로젝트 루트로 사용합니다.

```bash
find . -maxdepth 3 -name "pubspec.yaml" ! -path "*/build/*" 2>/dev/null | head -1
```

`pubspec.yaml`이 없으면 사용자에게 경로를 입력받습니다.

---

### Step 2 — fastlane 폴더 확인

프로젝트 루트에 `fastlane/Fastfile`과 `fastlane/.env`가 존재하는지 확인합니다.

```bash
ls fastlane/Fastfile fastlane/.env 2>/dev/null
```

없으면 다음 안내를 출력하고 중단합니다:
```
❌ fastlane 배포 환경이 설정되지 않았습니다.
   먼저 /fastlane-init 스킬을 실행하여 배포 환경을 설정해주세요.
```

---

### Step 3 — Flavor 및 buildType 자동 파악

`android/app/build.gradle` 또는 `android/app/build.gradle.kts` 파일을 읽어 Flavor와 buildType 정보를 추출합니다.

```bash
# Groovy DSL
cat android/app/build.gradle 2>/dev/null
# Kotlin DSL
cat android/app/build.gradle.kts 2>/dev/null
```

#### Flavor 추출 규칙

**Groovy DSL (`build.gradle`)** 에서 아래 블록 내의 항목을 Flavor로 인식합니다:
```
productFlavors {
    dev { ... }
    staging { ... }
    prod { ... }
}
```

**Kotlin DSL (`build.gradle.kts`)** 에서 아래 블록 내의 항목을 Flavor로 인식합니다:
```
productFlavors {
    create("dev") { ... }
    create("staging") { ... }
    create("prod") { ... }
}
```

#### buildType 추출 규칙

`buildTypes` 블록 내의 항목을 buildType으로 인식합니다.  
기본값은 `debug`, `release`이며, 추가로 정의된 타입이 있으면 함께 포함합니다.

#### 빌드 옵션 조합 생성

Flavor가 있는 경우 `{flavor}{BuildType}` 형태로 조합합니다 (첫 글자 대문자):
- 예: `devDebug`, `devRelease`, `stagingDebug`, `stagingRelease`, `prodDebug`, `prodRelease`

Flavor가 없는 경우 buildType만 사용합니다:
- 예: `debug`, `release`

---

### Step 4 — 빌드 옵션 선택

추출한 빌드 옵션 목록을 번호와 함께 출력하고 사용자에게 선택을 요청합니다:

```
빌드 옵션을 선택해주세요:
  1) devDebug
  2) devRelease
  3) stagingDebug
  4) stagingRelease
  5) prodDebug
  6) prodRelease
입력 (번호):
```

- 유효하지 않은 번호면 다시 질문합니다.
- 선택된 값을 `buildVariant`로 저장합니다 (예: `devDebug`).

---

### Step 5 — 릴리즈 노트 입력

배포에 필요한 릴리즈 노트를 사용자에게 질문합니다:

```
릴리즈 노트를 입력해주세요 (필수):
```

입력이 비어있으면 다시 요청합니다.

---

### Step 6 — 테스터 그룹 / 테스터 이메일 입력 (선택)

배포 대상을 선택적으로 입력받습니다. 둘 다 비워두면 옵션 없이 진행합니다.

```
테스터 그룹을 입력해주세요 (없으면 Enter 건너뜀):
예) qa-team,developers
```

```
테스터 이메일을 입력해주세요 (없으면 Enter 건너뜀):
예) tester1@example.com,tester2@example.com
```

---

### Step 7 — Fastlane 명령 구성 및 실행

수집한 값으로 fastlane 명령을 구성합니다.

lane 이름 결정 (`buildVariant` 끝 글자 기준):
- `Debug`로 끝나는 경우 → lane = `distributionDebug`, `flavor:"<buildVariant>"` 추가
  - 예: `devDebug` → `bundle exec fastlane android distributionDebug flavor:"devDebug" note:"..."`
- `Release`로 끝나는 경우 → lane = `distributionRelease`, `flavor:"<buildVariant>"` 추가
  - 예: `prodRelease` → `bundle exec fastlane android distributionRelease flavor:"prodRelease" note:"..."`

옵션 구성:
- flavor는 항상 포함: `flavor:"<buildVariant>"`
- note는 항상 포함: `note:"<릴리즈노트>"`
- groups 입력 시 추가: `groups:"<그룹>"`
- testers 입력 시 추가: `testers:"<테스터>"`

실행 전에 아래와 같이 구성된 명령을 출력하고 사용자에게 확인을 받습니다:

```
아래 명령으로 배포를 진행합니다:

  [Android] bundle exec fastlane android distributionDebug flavor:"devDebug" note:"..." groups:"..." testers:"..."

계속 진행할까요? (y/n):
```

`n` 입력 시 중단합니다.

실행:

```bash
bundle exec fastlane android <lane> flavor:"<buildVariant>" note:"릴리즈노트" groups:"그룹" testers:"테스터"
```

---

## 완료 출력

배포 성공 시:
```
✅ 배포 완료
   플랫폼    : Android
   빌드 옵션 : <buildVariant>
   릴리즈노트: <note>
```

실패 시:
```
❌ 배포 실패: <오류 내용>
```
