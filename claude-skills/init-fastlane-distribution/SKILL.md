---
name: init-fastlane-distribution
description: Fastlane과 Firebase App Distribution을 이용한 앱 배포 설정 스킬. 플랫폼(Android/iOS)과 배포타입(release/debug)을 입력받아 필수 항목을 검사한 뒤 Fastlane 설정 파일을 생성한다.
tools: AskUserQuestion, Read, Edit, Write, Bash
---

# Init Fastlane Distribution

Fastlane과 Firebase App Distribution을 이용한 앱 배포 환경을 설정한다.
누락된 파일이나 설정이 없는지 사전 검사 후 설정 파일을 생성한다.

---

## 실행 절차

### Step 1: 플랫폼 입력받기

AskUserQuestion 도구로 플랫폼을 물어본다.

```
배포할 플랫폼을 선택하세요.

1) Android
2) iOS

번호 또는 이름을 입력하세요:
```

입력값을 정규화한다.
- `1` 또는 `android` (대소문자 무관) → `Android`
- `2` 또는 `ios` (대소문자 무관) → `iOS`
- 그 외 → 다시 물어본다.

정규화된 값을 `PLATFORM`으로 저장한다.

---

### Step 2: 배포 타입 입력받기

AskUserQuestion 도구로 배포 타입을 물어본다.

```
배포 타입을 선택하세요.

1) release
2) debug

번호 또는 이름을 입력하세요:
```

입력값을 정규화한다.
- `1` 또는 `release` (대소문자 무관) → `release`
- `2` 또는 `debug` (대소문자 무관) → `debug`
- 그 외 → 다시 물어본다.

정규화된 값을 `BUILD_TYPE`으로 저장한다.

---

### Step 3: 필수 항목 사전 검사

`PLATFORM`과 `BUILD_TYPE` 조합에 따라 아래 항목을 Bash/Read 도구로 확인한다.

각 항목의 결과를 `CHECKLIST`(항목명 → `OK` / `MISSING` / `WARNING`)로 관리한다.

---

#### 공통 검사 항목

**[C-1] Fastlane 설치 여부**
```bash
which fastlane 2>/dev/null || gem list fastlane 2>/dev/null | grep fastlane
```
없으면 `MISSING` 처리.

**[C-2] fastlane/ 디렉토리**
```bash
ls fastlane/ 2>/dev/null
```
없으면 `MISSING` (설정 시 생성).

**[C-3] Gemfile**
```bash
ls Gemfile 2>/dev/null
```
없으면 `MISSING` (설정 시 생성).

**[C-4] firebase_app_distribution 플러그인**
```bash
cat fastlane/Pluginfile 2>/dev/null | grep firebase_app_distribution
```
없으면 `MISSING` (설정 시 추가).

**[C-5] Firebase 인증 정보**
```bash
# 서비스 계정 JSON 파일 탐색 (루트 또는 fastlane/ 하위)
find . -maxdepth 3 -name "*.json" ! -path "*/node_modules/*" ! -path "*/build/*" \
  -exec grep -l '"type": "service_account"' {} \; 2>/dev/null | head -3
# FIREBASE_TOKEN 환경변수 확인
echo "${FIREBASE_TOKEN:+set}"
```
서비스 계정 JSON도 없고 `FIREBASE_TOKEN`도 비어 있으면 `WARNING` (배포 실행 시 필요).

---

#### Android 전용 검사 항목 (PLATFORM = Android)

**[A-1] android/ 디렉토리**
```bash
ls android/ 2>/dev/null
```
없으면 `MISSING` (Android 프로젝트 확인 필요).

**[A-2] google-services.json**
```bash
find . -name "google-services.json" ! -path "*/build/*" 2>/dev/null
```
없으면 `MISSING`.

**[A-3] Firebase App ID (Android)**

`google-services.json`에서 추출한다.
`find` 결과를 변수에 저장한 뒤 python3에 인수로 전달하여 경로 인젝션을 방지한다:

```bash
GOOGLE_SERVICES_PATH=$(find . -name "google-services.json" ! -path "*/build/*" 2>/dev/null | head -1)
if [ -n "$GOOGLE_SERVICES_PATH" ]; then
  python3 - "$GOOGLE_SERVICES_PATH" <<'PYEOF'
import json, sys
try:
    with open(sys.argv[1]) as f:
        d = json.load(f)
    ids = [c['client_info']['mobilesdk_app_id']
           for c in d.get('client', [])
           if 'mobilesdk_app_id' in c.get('client_info', {})]
    print(ids[0] if ids else '')
except Exception:
    pass
PYEOF
fi
```
추출 성공 시 `FIREBASE_APP_ID`에 저장하고 `OK`, 실패 시 `MISSING`.

**[A-4] 패키지명 (applicationId)**

`android/app/build.gradle` 또는 `android/app/build.gradle.kts`에서 추출:
```bash
grep -E "applicationId|namespace" android/app/build.gradle android/app/build.gradle.kts 2>/dev/null | head -3
```
찾은 값을 `APP_PACKAGE_NAME`에 저장.

**[A-5] 서명 설정 (BUILD_TYPE = release 일 때만)**

keystore 비밀번호 노출을 막기 위해 **파일/설정의 존재 여부만** 확인하고 내용은 출력하지 않는다:

```bash
# keystore 파일 존재 여부 (경로만 확인, 내용 출력 금지)
find . \( -name "*.jks" -o -name "*.keystore" \) ! -path "*/build/*" 2>/dev/null | head -3
# keystore.properties 존재 여부
ls android/keystore.properties 2>/dev/null
# signingConfigs 정의 여부만 확인 (grep -l: 파일명만 출력, 비밀번호 값 노출 금지)
grep -rl "signingConfigs" android/app/build.gradle android/app/build.gradle.kts 2>/dev/null
```
keystore 파일이 없거나 signingConfigs 정의가 없으면 `WARNING` (서명 없이 배포 불가).

---

#### iOS 전용 검사 항목 (PLATFORM = iOS)

**[I-1] ios/ 디렉토리**
```bash
ls ios/ 2>/dev/null
```
없으면 `MISSING`.

**[I-2] Xcode 프로젝트**
```bash
find ios/ -maxdepth 2 \( -name "*.xcworkspace" -o -name "*.xcodeproj" \) 2>/dev/null | head -3
```
없으면 `MISSING`. 찾은 workspace/project 경로를 `XCODE_WORKSPACE`에 저장.

**[I-3] GoogleService-Info.plist**
```bash
find . -name "GoogleService-Info.plist" ! -path "*/build/*" 2>/dev/null
```
없으면 `MISSING`.

**[I-4] Firebase App ID (iOS)**

`GoogleService-Info.plist`에서 추출한다.
경로를 변수에 분리한 뒤 인수로 전달하여 경로 인젝션을 방지한다:

```bash
PLIST_PATH=$(find . -name "GoogleService-Info.plist" ! -path "*/build/*" 2>/dev/null | head -1)
if [ -n "$PLIST_PATH" ]; then
  /usr/libexec/PlistBuddy -c "Print :GOOGLE_APP_ID" "$PLIST_PATH" 2>/dev/null
fi
```
추출 성공 시 `FIREBASE_APP_ID`에 저장하고 `OK`, 실패 시 `MISSING`.

> `GoogleService-Info.plist`에는 `API_KEY`(iOS API 키) 필드도 포함되어 있다.
> 이 스킬은 `GOOGLE_APP_ID`만 추출하며 `API_KEY`는 읽거나 출력하지 않는다.
> 파일 자체가 git에 커밋되지 않도록 Step 5-6에서 `.gitignore` 추가를 안내한다.

**[I-5] Bundle Identifier**

경로를 변수에 분리하여 인젝션을 방지한다:

```bash
INFO_PLIST_PATH=$(find ios/ -name "Info.plist" ! -path "*/build/*" ! -path "*Tests*" 2>/dev/null | head -1)
if [ -n "$INFO_PLIST_PATH" ]; then
  /usr/libexec/PlistBuddy -c "Print :CFBundleIdentifier" "$INFO_PLIST_PATH" 2>/dev/null
fi
```
찾은 값을 `BUNDLE_ID`에 저장.

**[I-6] 코드 서명 설정 (BUILD_TYPE = release 일 때만)**
```bash
# 인증서 확인
security find-identity -v -p codesigning 2>/dev/null | grep -i "distribution\|developer" | head -5
# provisioning profile 확인
ls ~/Library/MobileDevice/Provisioning\ Profiles/ 2>/dev/null | wc -l
```
Distribution 인증서나 provisioning profile이 없으면 `WARNING`.

---

### Step 4: 검사 결과 보고

검사 결과를 아래 형식으로 출력한다.

```
## 사전 검사 결과 ({PLATFORM} · {BUILD_TYPE})

### ✅ 확인된 항목
- [C-1] Fastlane 설치: {버전}
- [A-3] Firebase App ID: {FIREBASE_APP_ID}
- ...

### ⚠️ 경고 항목 (배포 실행 전 필요)
- [C-5] Firebase 인증 정보 없음
  → 서비스 계정 JSON 파일 경로를 fastlane/.env에 설정하거나
    FIREBASE_TOKEN 환경변수를 설정하세요.
- ...

### ❌ 누락 항목 (설정 진행 불가)
- [A-2] google-services.json 없음
  → Firebase 콘솔에서 다운로드 후 android/app/ 에 추가하세요.
- ...
```

`MISSING` 항목이 하나라도 있으면 AskUserQuestion 도구로 확인한다:

```
❌ 위 누락 항목을 해결해야 설정을 완료할 수 있습니다.
누락 항목을 직접 해결한 뒤 다시 실행하거나,
경고 항목만 있는 경우 설정 파일 생성을 계속 진행할 수 있습니다.

계속 진행하시겠습니까? (예/아니오)
```

- `아니오` → 종료
- `예` (MISSING 항목이 없는 경우만) → Step 5 진행

---

### Step 5: Fastlane 설정 파일 생성

아래 파일들을 생성 또는 업데이트한다.
파일이 이미 존재하면 Read 도구로 내용을 확인한 뒤 병합 필요 여부를 판단한다.

---

#### 5-1. Gemfile

`Gemfile`이 없으면 아래 내용으로 생성한다.
이미 있으면 `fastlane` gem이 포함되어 있는지 확인하고, 없으면 추가한다.

```ruby
source "https://rubygems.org"

gem "fastlane"
```

---

#### 5-2. fastlane/Pluginfile

`fastlane/Pluginfile`이 없으면 생성한다.
이미 있으면 `firebase_app_distribution` 플러그인이 없을 때만 추가한다.

```ruby
# Autogenerated by fastlane
#
# Ensure this file is checked in to source control!

gem 'fastlane-plugin-firebase_app_distribution'
```

---

#### 5-3. fastlane/Appfile

**Android의 경우:**
```ruby
json_key_file("")  # 서비스 계정 JSON 경로 (선택)
package_name("{APP_PACKAGE_NAME}")
```

**iOS의 경우:**
```ruby
app_identifier("{BUNDLE_ID}")
apple_id("")       # Apple ID (선택)
team_id("")        # Apple Team ID (선택)
```

`APP_PACKAGE_NAME` 또는 `BUNDLE_ID`를 Step 3에서 추출한 값으로 대체한다.
값이 없으면 빈 문자열 또는 플레이스홀더로 남긴다.

---

#### 5-4. fastlane/.env

환경 변수 템플릿 파일을 생성한다. 이미 있으면 기존 파일을 유지하고 누락된 항목만 주석으로 안내한다.

**Android의 경우:**
```dotenv
# Firebase App Distribution 설정
FIREBASE_APP_ID_ANDROID={FIREBASE_APP_ID}

# Firebase 인증 (둘 중 하나 설정)
# FIREBASE_TOKEN=<firebase login --reauth 로 발급>
# GOOGLE_APPLICATION_CREDENTIALS=<서비스 계정 JSON 경로>

# 배포 대상 테스터 그룹 (Firebase Console에서 생성)
FIREBASE_TESTER_GROUPS=testers

# 릴리즈 노트 (선택)
RELEASE_NOTES=New build
```

**iOS의 경우:**
```dotenv
# Firebase App Distribution 설정
FIREBASE_APP_ID_IOS={FIREBASE_APP_ID}

# Firebase 인증 (둘 중 하나 설정)
# FIREBASE_TOKEN=<firebase login --reauth 로 발급>
# GOOGLE_APPLICATION_CREDENTIALS=<서비스 계정 JSON 경로>

# 배포 대상 테스터 그룹 (Firebase Console에서 생성)
FIREBASE_TESTER_GROUPS=testers

# iOS 빌드 설정
IOS_SCHEME={앱_스킴명}

# 릴리즈 노트 (선택)
RELEASE_NOTES=New build
```

`FIREBASE_APP_ID`는 Step 3에서 추출한 값으로 대체한다.

---

#### 5-5. fastlane/Fastfile

`PLATFORM`과 `BUILD_TYPE` 조합에 따라 아래 템플릿으로 생성한다.
이미 존재하면 `distribute` lane이 없을 때만 추가하고, 있으면 덮어쓰기 전 사용자 확인을 받는다.

**Android + release:**
```ruby
default_platform(:android)

platform :android do
  desc "Firebase App Distribution 배포 (Release)"
  lane :distribute do
    gradle(
      task: "bundle",
      build_type: "Release",
      project_dir: "android/"
    )
    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID_ANDROID"],
      android_artifact_type: "AAB",
      groups: ENV["FIREBASE_TESTER_GROUPS"] || "testers",
      release_notes: ENV["RELEASE_NOTES"] || "New build",
      service_credentials_file: ENV["GOOGLE_APPLICATION_CREDENTIALS"]
    )
  end
end
```

**Android + debug:**
```ruby
default_platform(:android)

platform :android do
  desc "Firebase App Distribution 배포 (Debug)"
  lane :distribute do
    gradle(
      task: "assemble",
      build_type: "Debug",
      project_dir: "android/"
    )
    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID_ANDROID"],
      android_artifact_type: "APK",
      groups: ENV["FIREBASE_TESTER_GROUPS"] || "testers",
      release_notes: ENV["RELEASE_NOTES"] || "New debug build",
      service_credentials_file: ENV["GOOGLE_APPLICATION_CREDENTIALS"]
    )
  end
end
```

**iOS + release:**
```ruby
default_platform(:ios)

platform :ios do
  desc "Firebase App Distribution 배포 (Release)"
  lane :distribute do
    build_app(
      workspace: "ios/Runner.xcworkspace",
      scheme: ENV["IOS_SCHEME"] || "Runner",
      configuration: "Release",
      export_method: "ad-hoc",
      output_directory: "build/ios"
    )
    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID_IOS"],
      groups: ENV["FIREBASE_TESTER_GROUPS"] || "testers",
      release_notes: ENV["RELEASE_NOTES"] || "New build",
      service_credentials_file: ENV["GOOGLE_APPLICATION_CREDENTIALS"]
    )
  end
end
```

**iOS + debug:**
```ruby
default_platform(:ios)

platform :ios do
  desc "Firebase App Distribution 배포 (Debug)"
  lane :distribute do
    build_app(
      workspace: "ios/Runner.xcworkspace",
      scheme: ENV["IOS_SCHEME"] || "Runner",
      configuration: "Debug",
      export_method: "development",
      output_directory: "build/ios"
    )
    firebase_app_distribution(
      app: ENV["FIREBASE_APP_ID_IOS"],
      groups: ENV["FIREBASE_TESTER_GROUPS"] || "testers",
      release_notes: ENV["RELEASE_NOTES"] || "New debug build",
      service_credentials_file: ENV["GOOGLE_APPLICATION_CREDENTIALS"]
    )
  end
end
```

iOS의 경우 `XCODE_WORKSPACE`에서 추출한 실제 workspace 경로로 `ios/Runner.xcworkspace`를 대체한다.

---

#### 5-6. .gitignore 업데이트

프로젝트 루트의 `.gitignore`에 아래 항목이 없으면 추가한다.

```gitignore
# Fastlane
fastlane/.env
fastlane/report.xml
fastlane/Preview.html
fastlane/screenshots
fastlane/test_output

# Firebase 인증 (서비스 계정 JSON은 절대 커밋하지 않음)
*service-account*.json
*service_account*.json

# Firebase 앱 설정 파일 (API 키 포함 — 팀 정책에 따라 결정)
# google-services.json
# GoogleService-Info.plist
```

`google-services.json` / `GoogleService-Info.plist`는 빌드 시 필요하므로 커밋 여부는 팀 정책에 따르되,
**이 두 파일이 이미 git에 추적 중인지** 아래 명령으로 확인하고 결과를 사용자에게 보여준다:

```bash
git ls-files android/app/google-services.json ios/GoogleService-Info.plist 2>/dev/null
```

추적 중이라면 아래 경고를 출력한다:
```
⚠️ 보안 주의: 아래 파일이 git에 추적되고 있습니다.
   - {파일 경로}
   이 파일에는 API_KEY 등 민감한 값이 포함될 수 있습니다.
   원격 저장소가 공개(public)라면 .gitignore에 추가하고 git rm --cached로 추적을 해제하세요.
```

---

### Step 6: 설정 완료 안내

생성된 파일 목록과 다음 단계를 출력한다.

```
✅ Fastlane 배포 설정이 완료되었습니다.

생성된 파일:
  - Gemfile
  - fastlane/Appfile
  - fastlane/Fastfile  (lane: distribute)
  - fastlane/Pluginfile
  - fastlane/.env      ← Firebase App ID 및 인증 정보 입력 필요

다음 단계:
  1. fastlane/.env 파일에서 빈 값을 채우세요.
     - FIREBASE_TOKEN 또는 GOOGLE_APPLICATION_CREDENTIALS 중 하나 필수
     - FIREBASE_TESTER_GROUPS: Firebase Console에서 테스터 그룹 이름 확인

  2. 플러그인을 설치하세요:
     $ bundle install
     $ bundle exec fastlane install_plugins

  3. 배포를 실행하세요:
     $ bundle exec fastlane distribute
```

`WARNING` 항목이 있으면 아래를 추가로 출력한다:

```
⚠️ 배포 실행 전 해결이 필요한 항목:
  - {WARNING 항목 설명 및 해결 방법}
```
