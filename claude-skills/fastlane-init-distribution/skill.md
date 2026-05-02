---
skill: fastlane-init-distribution
description: Flutter 프로젝트에 Fastlane + Firebase App Distribution 배포 환경을 설정합니다. Fastfile과 .env를 프로젝트 fastlane 폴더로 복사하고, google-services.json에서 App ID를 자동 추출하여 .env 값을 채웁니다.
tools: Read, Edit, Write, Bash
---

## Overview

Flutter 프로젝트의 `fastlane/` 폴더에 Fastfile과 `.env`를 배치하고, `.env`의 각 값을 자동 추출 또는 안내를 통해 채웁니다.

---

## Instructions

### Step 1 — 프로젝트 루트 확인

현재 디렉터리 또는 부모 디렉터리에서 `pubspec.yaml`을 탐색합니다.

```bash
find . -maxdepth 3 -name "pubspec.yaml" ! -path "*/build/*" 2>/dev/null | head -3
```

`pubspec.yaml`이 없으면 사용자에게 Flutter 프로젝트 루트 경로를 입력받습니다.  
이후 모든 파일 경로는 이 루트를 기준으로 합니다.

---

### Step 2 — fastlane 디렉터리 생성 및 Gemfile 초기화

프로젝트 루트에 `fastlane/` 디렉터리가 없으면 생성합니다.

```bash
mkdir -p fastlane
```

프로젝트 루트에 `Gemfile`이 없으면 `bundle init`을 실행하여 생성합니다.

```bash
[ ! -f Gemfile ] && bundle init
```

이후 `bundle install`을 실행하여 Gemfile.lock을 생성합니다.

```bash
bundle install
```

`bundle install` 실행 결과를 확인합니다. 성공하면 아래 두 플러그인의 사용 가능 여부를 각각 확인합니다.

#### firebase_app_distribution 플러그인 확인

```bash
bundle exec fastlane actions firebase_app_distribution 2>&1 | head -5
```

- 출력에 `firebase_app_distribution` 관련 액션 설명이 나오면 사용 가능 상태입니다.
- `Unknown action` 또는 오류가 출력되면 아래 명령어로 설치합니다.

```bash
bundle exec fastlane add_plugin firebase_app_distribution
```

#### upload_to_play_store 액션 확인

`upload_to_play_store`는 Fastlane에 기본 내장된 액션입니다. 별도 플러그인 설치 없이 아래 명령어로 사용 가능 여부를 확인합니다.

```bash
bundle exec fastlane actions upload_to_play_store 2>&1 | head -5
```

- 출력에 `upload_to_play_store` 관련 액션 설명이 나오면 사용 가능 상태입니다.
- 오류가 출력되면 `bundle install`이 정상적으로 완료되지 않은 것이므로 다시 실행합니다.

두 항목 모두 확인 완료 후 다음 Step으로 진행합니다.

---

### Step 3 — Fastfile 및 .env 복사

스킬 디렉터리(`~/.claude/skills/fastlane-init-distribution/`)에 있는 `Fastfile`과 `.env` 파일을 프로젝트 루트의 `fastlane/` 폴더로 복사합니다.

```bash
SKILL_DIR="$HOME/.claude/skills/fastlane-init-distribution"
cp "$SKILL_DIR/Fastfile" fastlane/Fastfile
cp "$SKILL_DIR/.env"     fastlane/.env
```

복사 후 두 파일이 모두 존재하는지 확인합니다.

```bash
ls -la fastlane/Fastfile fastlane/.env
```

---

### Step 4 — App ID 자동 추출 (google-services.json)

`fastlane/.env`의 `*_ANDROID_APP_ID` 값을 채웁니다.

#### 4-1. Debug App ID

`android/app/src/debug/google-services.json`을 읽어 `mobilesdk_app_id`를 추출합니다.

```bash
python3 - "android/app/src/debug/google-services.json" <<'PYEOF'
import json, sys
try:
    with open(sys.argv[1]) as f:
        d = json.load(f)
    ids = [c['client_info']['mobilesdk_app_id']
           for c in d.get('client', [])
           if 'mobilesdk_app_id' in c.get('client_info', {})]
    print(ids[0] if ids else '')
except Exception as e:
    print(f"ERROR: {e}", file=sys.stderr)
PYEOF
```

추출 성공 시 결과를 `DEBUG_ANDROID_APP_ID`로 저장합니다.

#### 4-2. Release App ID

`android/app/src/release/google-services.json`에서 동일하게 추출합니다.

```bash
python3 - "android/app/src/release/google-services.json" <<'PYEOF'
import json, sys
try:
    with open(sys.argv[1]) as f:
        d = json.load(f)
    ids = [c['client_info']['mobilesdk_app_id']
           for c in d.get('client', [])
           if 'mobilesdk_app_id' in c.get('client_info', {})]
    print(ids[0] if ids else '')
except Exception as e:
    print(f"ERROR: {e}", file=sys.stderr)
PYEOF
```

추출 성공 시 결과를 `RELEASE_ANDROID_APP_ID`로 저장합니다.

#### 4-3. iOS Debug App ID

`ios/Runner/GoogleService-Info-Debug.plist`에서 `GOOGLE_APP_ID`를 추출합니다.

```bash
/usr/libexec/PlistBuddy -c "Print :GOOGLE_APP_ID" "ios/Runner/GoogleService-Info-Debug.plist" 2>/dev/null
```

추출 성공 시 결과를 `DEBUG_IOS_APP_ID`로 저장합니다.  
파일이 없으면 해당 값을 빈 문자열로 유지하고 사용자에게 안내합니다.

#### 4-4. iOS Release App ID

`ios/Runner/GoogleService-Info-Release.plist`에서 `GOOGLE_APP_ID`를 추출합니다.

```bash
/usr/libexec/PlistBuddy -c "Print :GOOGLE_APP_ID" "ios/Runner/GoogleService-Info-Release.plist" 2>/dev/null
```

추출 성공 시 결과를 `RELEASE_IOS_APP_ID`로 저장합니다.  
파일이 없으면 해당 값을 빈 문자열로 유지하고 사용자에게 안내합니다.

#### 4-5. Deploy Package Name

`DEPLOY_PACKAGE_NAME`은 `android/app/src/release/google-services.json`에서 `package_name`을 추출합니다.  
해당 파일이 없으면 사용자에게 직접 입력 받습니다.

```bash
python3 - <<'PYEOF'
import json, sys

def extract_package_name(path):
    try:
        with open(path) as f:
            d = json.load(f)
        names = [
            c['client_info']['android_client_info']['package_name']
            for c in d.get('client', [])
            if 'package_name' in c.get('client_info', {}).get('android_client_info', {})
        ]
        return names[0] if names else ''
    except Exception:
        return ''

name = extract_package_name('android/app/src/release/google-services.json')
print(name)
PYEOF
```

추출 성공 시 결과를 `DEPLOY_PACKAGE_NAME`으로 저장합니다.  
두 파일 모두 없거나 추출에 실패하면 빈 문자열로 유지하고 사용자에게 수동 입력을 안내합니다.

---

### Step 5 — PATH 변수 값 결정

`fastlane` 폴더가 Flutter 프로젝트의 root에 존재하는 경우, 아래 세 경로를 설정합니다.

| 변수 | 값 |
|---|---|
| `DEBUG_APK_FILE_PATH` | `build/app/outputs/flutter-apk/app-debug.apk` |
| `RELEASE_APK_FILE_PATH` | `build/app/outputs/flutter-apk/app-release.apk` |
| `DEPLOY_AAB_FILE_PATH` | `build/app/outputs/bundle/release/app-release.aab` |

---

### Step 6 — .env 파일 업데이트

Step 4·5에서 수집한 값으로 `fastlane/.env`를 업데이트합니다.  
수집 대상에는 `DEPLOY_PACKAGE_NAME`(4-5에서 추출)도 포함됩니다.

Read 도구로 `fastlane/.env`를 읽은 뒤, Edit 도구로 각 항목의 빈 값(`=` 뒤가 비어 있는 줄)을 채웁니다.

- 값을 추출하지 못한 항목(파일 없음 등)은 빈 문자열로 유지합니다.
- 이미 값이 있는 항목은 덮어쓰지 않습니다.

업데이트 후 `fastlane/.env`의 최종 내용을 출력합니다.

---

### Step 7 — 서비스 계정 인증 파일 안내

`*_SERVICE_CREDENTIALS_FILE_PATH` 세 항목은 자동으로 채울 수 없습니다.  
사용자에게 아래 안내를 출력합니다.

```
📌 서비스 계정 인증 파일(JSON) 설정 안내

Firebase App Distribution 및 Google Play Store 배포에는
Google Cloud 서비스 계정 JSON 키 파일이 필요합니다.

─────────────────────────────────────────────────────────
[다운로드 방법]
  1. Google Cloud Console (https://console.cloud.google.com) 접속
  2. IAM 및 관리자 → 서비스 계정 메뉴 이동
  3. 해당 서비스 계정 선택 → [키] 탭 → [키 추가] → JSON 선택
  4. 다운로드된 JSON 파일을 프로젝트 루트의 fastlane/ 폴더로 복사

─────────────────────────────────────────────────────────
[권장 파일명 및 .env 설정 예시]

  Debug 배포용:
    파일: fastlane/firebase-debug-service-account.json
    .env: DEBUG_SERVICE_CREDENTIALS_FILE_PATH=fastlane/firebase-debug-service-account.json

  Release 배포용:
    파일: fastlane/firebase-release-service-account.json
    .env: RELEASE_SERVICE_CREDENTIALS_FILE_PATH=fastlane/firebase-release-service-account.json

  Play Store 배포용:
    파일: fastlane/playstore-service-account.json
    .env: DEPLOY_SERVICE_CREDENTIALS_FILE_PATH=fastlane/playstore-service-account.json

─────────────────────────────────────────────────────────
⚠️  보안 주의
  서비스 계정 JSON 파일과 fastlane/.env 파일은 절대 git에 커밋하지 마세요.
  아래 항목을 .gitignore에 추가하세요:

    fastlane/.env
    fastlane/*-service-account.json
    fastlane/*service_account*.json
```

---

### Step 8 — .gitignore 업데이트

프로젝트 루트의 `.gitignore`에 아래 항목이 없으면 추가합니다.

```gitignore
# Fastlane
fastlane/.env
fastlane/report.xml
fastlane/Preview.html
fastlane/screenshots
fastlane/test_output

# Firebase 서비스 계정 JSON (절대 커밋 금지)
fastlane/*service-account*.json
fastlane/*service_account*.json
```

---

### Step 9 — 완료 요약 출력

```
✅ Fastlane 배포 환경 설정 완료

복사된 파일:
  - fastlane/Fastfile
  - fastlane/.env

자동 설정된 값:
  - DEBUG_ANDROID_APP_ID    : {값 또는 "(추출 실패 — 수동 입력 필요)"}
  - RELEASE_ANDROID_APP_ID  : {값 또는 "(추출 실패 — 수동 입력 필요)"}
  - DEBUG_IOS_APP_ID        : {값 또는 "(추출 실패 — 수동 입력 필요)"}
  - RELEASE_IOS_APP_ID      : {값 또는 "(추출 실패 — 수동 입력 필요)"}
  - DEPLOY_PACKAGE_NAME     : {값 또는 "(추출 실패 — 수동 입력 필요)"}
  - DEBUG_APK_FILE_PATH     : build/app/outputs/flutter-apk/app-debug.apk
  - RELEASE_APK_FILE_PATH   : build/app/outputs/flutter-apk/app-release.apk
  - DEPLOY_AAB_FILE_PATH    : build/app/outputs/bundle/release/app-release.aab

수동 설정 필요:
  - DEBUG_SERVICE_CREDENTIALS_FILE_PATH   (Step 7 안내 참고)
  - RELEASE_SERVICE_CREDENTIALS_FILE_PATH (Step 7 안내 참고)
  - DEPLOY_SERVICE_CREDENTIALS_FILE_PATH  (Step 7 안내 참고)

다음 단계:
  1. fastlane/.env에서 수동 항목을 채우세요.
  2. 배포를 실행하세요 (예: Debug Android):
       bundle exec fastlane android distributionDebug note:"배포 노트"
       bundle exec fastlane android distributionDebug note:"배포 노트" groups:"qa-team"
       bundle exec fastlane android distributionDebug note:"배포 노트" testers:"tester@example.com,tester2@example.com"
```
