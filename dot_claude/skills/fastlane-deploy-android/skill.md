---
skill: fastlane-deploy-android
description: Flutter 프로젝트를 Fastlane으로 Google Play Store에 Android 배포합니다. build.gradle의 Flavor 정보를 자동 파악하여 빌드 옵션을 제시하고, 릴리즈 노트, 테스터 정보를 대화형으로 입력받아 deploy lane을 실행합니다.
tools: Read, Bash
---

## Overview

Fastlane을 이용해 Google Play Store에 Flutter 앱을 Android 배포합니다.  
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

Flavor가 없는 경우 'release'만 사용합니다.

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

### Step 5 — Fastlane 명령 구성 및 실행

수집한 값으로 fastlane 명령을 구성합니다.

lane 이름: 항상 `deploy`를 사용합니다.

옵션 구성:
- flavor는 항상 포함: `flavor:"<buildVariant>"`

실행 전에 아래와 같이 구성된 명령을 출력하고 사용자에게 확인을 받습니다:

```
아래 명령으로 배포를 진행합니다:

  [Android] bundle exec fastlane android deploy flavor:"devRelease"

계속 진행할까요? (y/n):
```

`n` 입력 시 중단합니다.

실행:

```bash
bundle exec fastlane android deploy flavor:"<buildVariant>"
```

---

## 완료 출력

배포 성공 시:
```
✅ 배포 완료
   플랫폼    : Android
   빌드 옵션 : <buildVariant>
```

실패 시:
```
❌ 배포 실패: <오류 내용>
```
