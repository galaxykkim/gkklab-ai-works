---
skill: fastlane-distribution-ios
description: Flutter 프로젝트를 Fastlane + Firebase App Distribution으로 iOS 배포합니다. 빌드 타입, 릴리즈 노트, 테스터 정보를 대화형으로 입력받아 해당 lane을 실행합니다.
tools: Read, Bash
---

## Overview

Fastlane을 이용해 Firebase App Distribution에 Flutter 앱을 iOS 배포합니다.  
모든 배포 옵션은 스킬 실행 중 사용자에게 직접 입력받아 진행합니다.

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

### Step 3 — 빌드 타입 입력

사용자에게 빌드 타입을 질문합니다 (하나만 선택):

```
빌드 타입을 선택해주세요:
  1) debug
  2) release
입력 (1 또는 2):
```

- `1` 또는 `debug` 입력 시 → buildType = `debug`
- `2` 또는 `release` 입력 시 → buildType = `release`
- 유효하지 않은 값이면 다시 질문합니다.

---

### Step 4 — 릴리즈 노트 입력

배포에 필요한 릴리즈 노트를 사용자에게 질문합니다:

```
릴리즈 노트를 입력해주세요 (필수):
```

입력이 비어있으면 다시 요청합니다.

---

### Step 5 — 테스터 그룹 / 테스터 이메일 입력 (선택)

배포 대상을 선택적으로 입력받습니다. 둘 다 비워두면 옵션 없이 진행합니다.

```
테스터 그룹을 입력해주세요 (없으면 Enter 건너뜀):
예) nest-mobile, nest-frontend, nest-plan, nest-design, nest-backend
```

```
테스터 이메일을 입력해주세요 (없으면 Enter 건너뜀):
예) tester1@example.com,tester2@example.com
```

---

### Step 6 — Fastlane 명령 구성 및 실행

수집한 값으로 fastlane 명령을 구성합니다.

lane 이름 결정:
- `debug` → `distributionDebug`
- `release` → `distributionRelease`

옵션 구성:
- note는 항상 포함: `note:"<릴리즈노트>"`
- groups 입력 시 추가: `groups:"<그룹>"`
- testers 입력 시 추가: `testers:"<테스터>"`

실행 전에 아래와 같이 구성된 명령을 출력하고 사용자에게 확인을 받습니다:

```
아래 명령으로 배포를 진행합니다:

  [iOS] bundle exec fastlane ios distributionDebug note:"..." groups:"..." testers:"..."

계속 진행할까요? (y/n):
```

`n` 입력 시 중단합니다.

실행:

```bash
bundle exec fastlane ios distributionDebug note:"릴리즈노트" groups:"그룹" testers:"테스터"
```

---

## 완료 출력

배포 성공 시:
```
✅ 배포 완료
   플랫폼  : iOS
   빌드타입: <buildType>
   릴리즈노트: <note>
```

실패 시:
```
❌ 배포 실패: <오류 내용>
```
