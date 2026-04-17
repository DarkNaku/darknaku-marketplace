---
name: phaser-build
description: >
  Phaser + Vite + Capacitor 환경에서 Web/iOS/Android 빌드를 실행하는 스킬.
  사용자가 "빌드해줘", "빌드", "build", "배포 빌드", "릴리즈 빌드",
  "개발 빌드", "디버그 빌드", "웹 빌드", "iOS 빌드", "안드로이드 빌드" 등
  빌드와 관련된 요청을 하면 이 스킬을 사용하라.
---

# Phaser Build

Phaser + Vite + Capacitor 프로젝트의 빌드 스킬.

---

## 전제 조건

- **필수 환경**: Phaser + Vite + Capacitor
- **인풋**: 빌드 플랫폼 (Web / iOS / Android), 빌드 타입 (개발 / 배포)
- **아웃풋**: 빌드 결과물

---

## 워크플로우

### Step 1: 환경 확인

프로젝트가 지원 환경(Phaser + Vite + Capacitor)을 충족하는지 확인한다.

`package.json`의 `dependencies`와 `devDependencies`에서 아래 패키지를 확인한다:

| 패키지 | 확인 대상 |
|--------|-----------|
| `phaser` | dependencies 또는 devDependencies |
| `vite` | devDependencies |
| `@capacitor/core` | dependencies |

**하나라도 없는 경우** — 아래를 출력하고 즉시 종료한다:

```
⚠️ 이 스킬은 Phaser + Vite + Capacitor 환경만 지원합니다.

   누락된 패키지:
   - {누락된 패키지 목록}

   환경을 먼저 구성한 후 다시 시도해 주세요.
```

**모두 존재하는 경우** — Step 2로 진행한다.

---

### Step 2: 빌드 옵션 확정

사용자의 요청에서 **빌드 플랫폼**과 **빌드 타입**을 파악한다.

#### 빌드 플랫폼

| 플랫폼 | 식별 키워드 |
|--------|-------------|
| **Web** | 웹, web, 브라우저, browser |
| **iOS** | ios, 아이폰, iphone, 아이패드, ipad, 애플, apple |
| **Android** | android, 안드로이드, apk, aab |

#### Android 출력 형식 (Android 배포 빌드 시에만)

| 형식 | 식별 키워드 |
|------|-------------|
| **APK** | apk |
| **AAB** | aab, bundle, 번들, 스토어, store |

#### 빌드 타입

| 타입 | 식별 키워드 |
|------|-------------|
| **개발** | 개발, dev, debug, 디버그, 테스트, test |
| **배포** | 배포, release, 릴리즈, 프로덕션, production, 스토어, store, 출시 |

**모든 옵션이 확인된 경우** — 바로 Step 3으로 진행한다.

**확인되지 않은 항목이 있는 경우** — 누락된 항목만 사용자에게 질문한다:

플랫폼이 없는 경우:
```
🔨 어떤 플랫폼으로 빌드할까요?
   1) Web
   2) iOS
   3) Android
```

빌드 타입이 없는 경우:
```
🔨 빌드 타입을 선택해 주세요.
   1) 개발 (debug)
   2) 배포 (release)
```

Android 배포 빌드인데 출력 형식이 확인되지 않은 경우:
```
🔨 Android 출력 형식을 선택해 주세요.
   1) APK — 직접 설치용
   2) AAB (App Bundle) — Play Store 배포용
```

---

### Step 3: 빌드 실행

확정된 플랫폼과 빌드 타입에 따라 해당 섹션의 절차를 실행한다.

---

#### Web 빌드

##### 개발

```bash
npx vite
```

```
🚀 개발 서버 실행 중
   → {출력된 URL}
```

##### 배포

```bash
npx vite build
```

빌드 완료 후 결과를 확인한다:

```bash
ls -la dist/
```

```
✅ Web 배포 빌드 완료
   📁 출력: dist/
```

---

#### iOS 빌드

##### 공통 준비

Vite로 웹 에셋을 빌드한 뒤 Capacitor sync를 실행한다:

```bash
npx vite build
npx cap sync ios
```

##### 개발

```bash
xcodebuild -workspace ios/App/App.xcworkspace \
  -scheme App \
  -configuration Debug \
  -destination 'generic/platform=iOS Simulator' \
  -derivedDataPath build/ios \
  build
```

```
✅ iOS 개발 빌드 완료
   📁 출력: build/ios/
```

##### 배포

배포 빌드에는 서명 정보가 필요하다. 아래 환경 변수를 확인한다:

| 환경 변수 | 설명 |
|-----------|------|
| `IOS_TEAM_ID` | Apple Developer Team ID |
| `IOS_PROVISIONING_PROFILE` | 프로비저닝 프로파일 이름 |
| `IOS_CODE_SIGN_IDENTITY` | 코드 서명 ID (예: `Apple Distribution`) |

**환경 변수가 설정되지 않은 경우** — 아래를 출력하고 종료한다:

```
⚠️ iOS 배포 빌드에 필요한 환경 변수가 설정되지 않았습니다.

   누락된 환경 변수:
   - {누락 목록}

   프로젝트 루트에 .env 파일을 생성하고 아래와 같이 설정해 주세요:

   ── .env ──────────────────────────────────────
   IOS_TEAM_ID=ABC123DEF4
   IOS_PROVISIONING_PROFILE=MyApp Distribution Profile
   IOS_CODE_SIGN_IDENTITY=Apple Distribution
   ──────────────────────────────────────────────

   각 값을 확인하는 방법:
   • IOS_TEAM_ID
     → Apple Developer 사이트 > Membership > Team ID
     → 또는 Xcode > Target > Signing & Capabilities > Team 드롭다운에서 확인
   • IOS_PROVISIONING_PROFILE
     → Apple Developer 사이트 > Certificates, Identifiers & Profiles > Profiles
     → 배포용 프로파일 이름을 그대로 입력 (예: "MyApp Distribution Profile")
   • IOS_CODE_SIGN_IDENTITY
     → App Store 배포: "Apple Distribution"
     → Ad Hoc 배포: "Apple Distribution"
     → 개발용: "Apple Development"

   .env 파일은 반드시 .gitignore에 추가하세요.

   설정 후 다시 시도해 주세요.
```

**환경 변수가 모두 설정된 경우**:

```bash
xcodebuild -workspace ios/App/App.xcworkspace \
  -scheme App \
  -configuration Release \
  -destination 'generic/platform=iOS' \
  -archivePath build/ios/App.xcarchive \
  DEVELOPMENT_TEAM="$IOS_TEAM_ID" \
  PROVISIONING_PROFILE_SPECIFIER="$IOS_PROVISIONING_PROFILE" \
  CODE_SIGN_IDENTITY="$IOS_CODE_SIGN_IDENTITY" \
  archive

xcodebuild -exportArchive \
  -archivePath build/ios/App.xcarchive \
  -exportPath build/ios/release \
  -exportOptionsPlist ios/exportOptions.plist
```

```
✅ iOS 배포 빌드 완료
   📦 출력: build/ios/release/
   🔑 서명: $IOS_CODE_SIGN_IDENTITY
```

---

#### Android 빌드

##### 공통 준비

Vite로 웹 에셋을 빌드한 뒤 Capacitor sync를 실행한다:

```bash
npx vite build
npx cap sync android
```

##### 개발

```bash
cd android && ./gradlew assembleDebug && cd ..
```

```
✅ Android 개발 빌드 완료
   📦 출력: android/app/build/outputs/apk/debug/
```

##### 배포

배포 빌드에는 서명 정보가 필요하다. 아래 환경 변수를 확인한다:

| 환경 변수 | 설명 |
|-----------|------|
| `ANDROID_KEYSTORE_PATH` | 키스토어 파일 경로 |
| `ANDROID_KEYSTORE_PASSWORD` | 키스토어 비밀번호 |
| `ANDROID_KEY_ALIAS` | 키 별칭 |
| `ANDROID_KEY_PASSWORD` | 키 비밀번호 |

**환경 변수가 설정되지 않은 경우** — 아래를 출력하고 종료한다:

```
⚠️ Android 배포 빌드에 필요한 환경 변수가 설정되지 않았습니다.

   누락된 환경 변수:
   - {누락 목록}

   프로젝트 루트에 .env 파일을 생성하고 아래와 같이 설정해 주세요:

   ── .env ──────────────────────────────────────
   ANDROID_KEYSTORE_PATH=./keystore/release.keystore
   ANDROID_KEYSTORE_PASSWORD=your_store_password
   ANDROID_KEY_ALIAS=my-app-key
   ANDROID_KEY_PASSWORD=your_key_password
   ──────────────────────────────────────────────

   키스토어가 없는 경우 아래 명령으로 생성할 수 있습니다:

   keytool -genkeypair \
     -v \
     -storetype PKCS12 \
     -keystore keystore/release.keystore \
     -alias my-app-key \
     -keyalg RSA \
     -keysize 2048 \
     -validity 10000

   각 값의 의미:
   • ANDROID_KEYSTORE_PATH
     → 생성한 키스토어 파일의 경로 (프로젝트 루트 기준 상대 경로)
   • ANDROID_KEYSTORE_PASSWORD
     → keytool 실행 시 입력한 키스토어 비밀번호
   • ANDROID_KEY_ALIAS
     → keytool 실행 시 -alias 옵션에 지정한 이름
   • ANDROID_KEY_PASSWORD
     → keytool 실행 시 입력한 키 비밀번호 (키스토어 비밀번호와 동일하게 설정 가능)

   .env 파일과 키스토어 파일은 반드시 .gitignore에 추가하세요.

   설정 후 다시 시도해 주세요.
```

**환경 변수가 모두 설정된 경우** — 출력 형식에 따라 분기한다:

**AAB (App Bundle)**:

```bash
cd android && ./gradlew bundleRelease \
  -Pandroid.injected.signing.store.file="$ANDROID_KEYSTORE_PATH" \
  -Pandroid.injected.signing.store.password="$ANDROID_KEYSTORE_PASSWORD" \
  -Pandroid.injected.signing.key.alias="$ANDROID_KEY_ALIAS" \
  -Pandroid.injected.signing.key.password="$ANDROID_KEY_PASSWORD" \
  && cd ..
```

```
✅ Android 배포 빌드 완료 (AAB)
   📦 출력: android/app/build/outputs/bundle/release/
   🔑 서명: $ANDROID_KEY_ALIAS
```

**APK**:

```bash
cd android && ./gradlew assembleRelease \
  -Pandroid.injected.signing.store.file="$ANDROID_KEYSTORE_PATH" \
  -Pandroid.injected.signing.store.password="$ANDROID_KEYSTORE_PASSWORD" \
  -Pandroid.injected.signing.key.alias="$ANDROID_KEY_ALIAS" \
  -Pandroid.injected.signing.key.password="$ANDROID_KEY_PASSWORD" \
  && cd ..
```

```
✅ Android 배포 빌드 완료 (APK)
   📦 출력: android/app/build/outputs/apk/release/
   🔑 서명: $ANDROID_KEY_ALIAS
```

---

### Step 4: 완료 보고

```
✅ 빌드 완료

   🎯 플랫폼: {Web / iOS / Android}
   🔨 타입:   {개발 / 배포}
   📁 출력:   {출력 경로}
```

---

## 주의사항

- 이 스킬은 Phaser + Vite + Capacitor 환경만 지원한다. 환경이 다르면 빌드를 시도하지 않는다.
- 배포 빌드 시 서명 환경 변수가 누락되면 빌드를 진행하지 않고 안내한다.
- iOS 배포 빌드에는 `ios/exportOptions.plist` 파일이 필요하다.
- Android 배포 빌드는 AAB(App Bundle) 형식으로 출력한다.
