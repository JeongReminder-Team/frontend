# 네이티브 모바일 기반 구축 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 기존 Flutter 앱을 보존하면서 SwiftUI 및 Jetpack Compose 네이티브 앱의 실행 가능한 셸, 내비게이션, 테스트 기반을 구축한다.

**Architecture:** 저장소 루트의 기존 `frontend/`를 `flutter-legacy/`로 이동하고 `ios-swift/`, `android-kotlin/`을 독립 앱으로 추가한다. 이번 계획은 앱 셸과 공통 화면 상태의 경계까지만 구현하며, 인증·게시판·프로필·팀·알림 기능은 후속 기능별 계획에서 TDD로 구현한다.

**Tech Stack:** Swift 6, SwiftUI, Observation, Swift Testing, XCTest/XCUIAutomation, Kotlin 2.2.10, Jetpack Compose BOM 2026.09.00, Navigation 3 1.1.7, AGP 9.0.1, Gradle 9.1.0, JDK 17

**Spec:** `docs/superpowers/specs/2026-10-01-native-ios-android-migration-design.md`

## Global Constraints

- 기존 Flutter 앱은 기능 동등성과 별도 삭제 승인이 확인될 때까지 삭제하지 않는다.
- iOS는 Swift 6 strict concurrency와 iOS 17 이상을 사용한다.
- Android는 `minSdk 23`, `compileSdk 36`, `targetSdk 36`을 사용한다.
- Android UI 상태는 UDF, `StateFlow`, `collectAsStateWithLifecycle`을 사용한다.
- iOS 상태는 Observation의 `@Observable`, 비동기는 `async/await`를 사용한다.
- Android 내비게이션은 안정 버전 Navigation 3 `1.1.7`을 사용한다.
- iOS 단위·통합 테스트는 Swift Testing, UI 테스트는 XCTest/XCUIAutomation을 사용한다.
- Android 인증정보는 향후 Android Keystore 기반 저장소에 보관하며 `EncryptedSharedPreferences`를 사용하지 않는다.
- 앱 식별자는 기존 배포 값인 iOS `com.sungkyul.reminderApp`, Android `com.sungkyul.reminder`를 유지한다.
- 모든 커밋 메시지는 한글로 작성하고 `git push`는 실행하지 않는다.
- 현재 작업 환경에는 Flutter, Android SDK/Gradle, Swift/Xcode가 없으므로 빌드 성공을 주장하지 않는다. Android 검증에는 JDK 17과 Android SDK 36이 필요하고, iOS 검증에는 macOS와 Xcode가 필요하다.

## 계획 분할

이 설계는 여러 독립 기능을 포함하므로 다음 계획으로 나눈다. 이번 문서는 첫 번째 항목만 구현한다.

1. 네이티브 앱 셸과 테스트 기반 — 이 문서
2. 인증·보안 토큰·자동 갱신
3. 게시판·파일·추천·투표
4. 프로필·기술 스택·프로젝트 경험
5. 팀 모집·지원·승인
6. 알림·딥 링크·관리자 기능
7. 기능 동등성 검증·배포 전환

## 파일 구조

```text
flutter-legacy/                         기존 Flutter 앱 전체
android-kotlin/
├─ settings.gradle.kts                 플러그인과 모듈 등록
├─ build.gradle.kts                    루트 플러그인 버전
├─ gradle.properties                   AndroidX와 Kotlin 빌드 설정
├─ gradle/libs.versions.toml            의존성 버전 카탈로그
└─ app/
   ├─ build.gradle.kts                 SDK, Compose, 테스트 설정
   └─ src/
      ├─ main/AndroidManifest.xml       앱 진입점
      ├─ main/kotlin/com/sungkyul/reminder/
      │  ├─ MainActivity.kt             단일 Activity 진입점
      │  ├─ app/ReminderApp.kt          테마와 루트 내비게이션
      │  ├─ navigation/AppRoute.kt      타입 안전 화면 키
      │  ├─ navigation/AppNavigator.kt  Navigation 3 백스택 소유자
      │  └─ ui/theme/ReminderTheme.kt   앱 테마
      └─ test/kotlin/com/sungkyul/reminder/navigation/
         └─ AppNavigatorTest.kt         초기 경로와 이동 테스트
ios-swift/
├─ project.yml                          XcodeGen 프로젝트 명세
├─ Config/Debug.xcconfig                개발 빌드 설정
├─ Config/Release.xcconfig              운영 빌드 설정
├─ ReminderApp/
│  ├─ ReminderApp.swift                 SwiftUI 앱 진입점
│  ├─ App/AppContainer.swift            의존성 조립
│  ├─ Navigation/AppRoute.swift         화면 경로
│  ├─ Navigation/AppRouter.swift        Observation 기반 경로 상태
│  └─ UI/RootView.swift                 루트 화면
└─ ReminderAppTests/
   └─ AppRouterTests.swift              초기 경로와 이동 테스트
docs/migration/native-feature-parity.md  플랫폼별 기능 동등성 표
README.md                               새 저장소 구조와 실행 방법
```

---

### Task 1: 기존 Flutter 앱 보존 및 저장소 구조 정리

**Files:**
- Move: `frontend/` → `flutter-legacy/`
- Modify: `README.md`
- Create: `docs/migration/native-feature-parity.md`

**Interfaces:**
- Consumes: 현재 Flutter 앱의 모든 추적 파일과 기존 루트 README
- Produces: `flutter-legacy/pubspec.yaml`, 네이티브 앱이 따라야 할 기능 동등성 표

- [ ] **Step 1: 이동 전 기준 파일을 확인한다**

Run:

```powershell
git ls-files frontend/pubspec.yaml frontend/lib/main.dart frontend/test/widget_test.dart
```

Expected: 세 경로가 모두 한 줄씩 출력된다.

- [ ] **Step 2: Git 이력을 보존해 Flutter 앱을 이동한다**

Run:

```powershell
git mv frontend flutter-legacy
```

Expected: `git status --short`에서 Flutter 파일이 rename으로 표시되고 `flutter-legacy/pubspec.yaml`이 존재한다.

- [ ] **Step 3: 기능 동등성 체크리스트를 작성한다**

Create `docs/migration/native-feature-parity.md`:

```markdown
# 네이티브 기능 동등성

| 기능 | Flutter 기준 | iOS | Android |
|---|---|---|---|
| 앱 실행 및 루트 화면 | `flutter-legacy/lib/main.dart` | 진행 중 | 진행 중 |
| 로그인·로그아웃 | `flutter-legacy/lib/services/login_services.dart` | 미착수 | 미착수 |
| 토큰 갱신 | `flutter-legacy/lib/services/login_services.dart` | 미착수 | 미착수 |
| 게시판·투표 | `flutter-legacy/lib/providers/` | 미착수 | 미착수 |
| 프로필·경험 | `flutter-legacy/lib/services/profile_service.dart` | 미착수 | 미착수 |
| 팀 모집·지원 | `flutter-legacy/lib/services/makeTeam_service.dart` | 미착수 | 미착수 |
| 알림·딥 링크 | `flutter-legacy/lib/main.dart` | 미착수 | 미착수 |
| 관리자 기능 | `flutter-legacy/lib/admin/` | 미착수 | 미착수 |
```

- [ ] **Step 4: README의 구조와 검증 명령을 갱신한다**

Add to `README.md`:

````markdown
## 모바일 클라이언트

- `flutter-legacy/`: 전환 완료 전까지 유지하는 기존 Flutter 앱
- `ios-swift/`: SwiftUI 기반 iOS 앱
- `android-kotlin/`: Jetpack Compose 기반 Android 앱

### 검증

```bash
cd flutter-legacy && flutter test
cd android-kotlin && ./gradlew test
cd ios-swift && xcodegen generate && xcodebuild test -scheme ReminderApp -destination 'platform=iOS Simulator,name=iPhone 17'
```
````

- [ ] **Step 5: 이동 결과를 정적으로 검증한다**

Run:

```powershell
Test-Path flutter-legacy/pubspec.yaml
Test-Path flutter-legacy/lib/main.dart
Test-Path frontend
git diff --check
```

Expected: 앞의 두 값은 `True`, 세 번째 값은 `False`, `git diff --check`는 출력 없이 종료한다.

- [ ] **Step 6: Flutter 도구가 있는 환경에서 회귀 테스트를 실행한다**

Run:

```powershell
Set-Location flutter-legacy
flutter pub get
flutter test
```

Expected: 패키지 해석과 모든 Flutter 테스트가 통과한다. 현재 작업 환경에서는 Flutter가 없으므로 이 검증은 도구가 있는 환경에서 수행하고 결과를 기능 동등성 문서에 기록한다.

- [ ] **Step 7: 구조 변경을 커밋한다**

```powershell
git add -- README.md docs/migration/native-feature-parity.md flutter-legacy
git commit -m "구조: 기존 Flutter 앱을 레거시 디렉터리로 이동"
```

---

### Task 2: Android Compose 프로젝트와 라우터 테스트 구축

**Files:**
- Create: `android-kotlin/settings.gradle.kts`
- Create: `android-kotlin/build.gradle.kts`
- Create: `android-kotlin/gradle.properties`
- Create: `android-kotlin/gradle/libs.versions.toml`
- Create: `android-kotlin/app/build.gradle.kts`
- Create: `android-kotlin/app/src/main/AndroidManifest.xml`
- Create: `android-kotlin/app/src/main/kotlin/com/sungkyul/reminder/navigation/AppRoute.kt`
- Create: `android-kotlin/app/src/main/kotlin/com/sungkyul/reminder/navigation/AppNavigator.kt`
- Test: `android-kotlin/app/src/test/kotlin/com/sungkyul/reminder/navigation/AppNavigatorTest.kt`

**Interfaces:**
- Consumes: Android SDK 36, JDK 17, Gradle 9.1.0
- Produces: `AppRoute : NavKey`, `AppNavigator.backStack: List<AppRoute>`, `navigate(AppRoute)`, `back(): Boolean`

- [ ] **Step 1: Android 빌드 설정을 생성한다**

Create `settings.gradle.kts`:

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "ReminderAndroid"
include(":app")
```

Create root `build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.compose) apply false
    alias(libs.plugins.kotlin.serialization) apply false
}
```

Create `gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
org.gradle.configuration-cache=true
```

Use these exact stable versions in `gradle/libs.versions.toml`:

```toml
[versions]
agp = "9.0.1"
kotlin = "2.2.10"
composeBom = "2026.09.00"
navigation3 = "1.1.7"
lifecycle = "2.10.0"
coroutines = "1.10.2"
activity = "1.12.1"
junit = "4.13.2"

[libraries]
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "composeBom" }
compose-ui = { module = "androidx.compose.ui:ui" }
compose-material3 = { module = "androidx.compose.material3:material3" }
compose-ui-tooling-preview = { module = "androidx.compose.ui:ui-tooling-preview" }
compose-ui-tooling = { module = "androidx.compose.ui:ui-tooling" }
lifecycle-runtime-compose = { module = "androidx.lifecycle:lifecycle-runtime-compose", version.ref = "lifecycle" }
lifecycle-viewmodel-compose = { module = "androidx.lifecycle:lifecycle-viewmodel-compose", version.ref = "lifecycle" }
navigation3-runtime = { module = "androidx.navigation3:navigation3-runtime", version.ref = "navigation3" }
navigation3-ui = { module = "androidx.navigation3:navigation3-ui", version.ref = "navigation3" }
coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
activity-compose = { module = "androidx.activity:activity-compose", version.ref = "activity" }
junit = { module = "junit:junit", version.ref = "junit" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

Create `app/build.gradle.kts` with the following configuration:

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.kotlin.serialization)
}

android {
    namespace = "com.sungkyul.reminder"
    compileSdk = 36

    defaultConfig {
        applicationId = "com.sungkyul.reminder"
        minSdk = 23
        targetSdk = 36
        versionCode = 24
        versionName = "1.0.1"
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    buildFeatures { compose = true }
}

kotlin { jvmToolchain(17) }

dependencies {
    implementation(platform(libs.compose.bom))
    androidTestImplementation(platform(libs.compose.bom))
    implementation(libs.compose.ui)
    implementation(libs.compose.material3)
    implementation(libs.compose.ui.tooling.preview)
    debugImplementation(libs.compose.ui.tooling)
    implementation(libs.lifecycle.runtime.compose)
    implementation(libs.lifecycle.viewmodel.compose)
    implementation(libs.navigation3.runtime)
    implementation(libs.navigation3.ui)
    implementation(libs.coroutines.android)
    implementation(libs.activity.compose)
    testImplementation(libs.junit)
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
    androidTestImplementation("androidx.test.ext:junit:1.3.0")
    androidTestImplementation("androidx.test:runner:1.7.0")
}
```

Create a minimal manifest:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application android:label="정리마인더" />
</manifest>
```

Generate the wrapper:

```powershell
Set-Location android-kotlin
gradle wrapper --gradle-version 9.1.0
```

Expected: `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`, and `gradle-wrapper.properties` are created.

- [ ] **Step 2: 실패하는 라우터 테스트를 작성한다**

Create `AppNavigatorTest.kt`:

```kotlin
package com.sungkyul.reminder.navigation

import org.junit.Assert.assertEquals
import org.junit.Assert.assertFalse
import org.junit.Assert.assertTrue
import org.junit.Test

class AppNavigatorTest {
    @Test
    fun `초기 화면은 로그인이다`() {
        val navigator = AppNavigator()
        assertEquals(listOf(AppRoute.Login), navigator.backStack)
    }

    @Test
    fun `화면 이동과 뒤로 가기를 순서대로 처리한다`() {
        val navigator = AppNavigator()
        navigator.navigate(AppRoute.Home)

        assertEquals(listOf(AppRoute.Login, AppRoute.Home), navigator.backStack)
        assertTrue(navigator.back())
        assertEquals(listOf(AppRoute.Login), navigator.backStack)
        assertFalse(navigator.back())
    }
}
```

- [ ] **Step 3: 테스트가 올바른 이유로 실패하는지 확인한다**

Run:

```powershell
Set-Location android-kotlin
.\gradlew.bat testDebugUnitTest --tests "com.sungkyul.reminder.navigation.AppNavigatorTest"
```

Expected: `AppRoute`와 `AppNavigator`가 없어 컴파일이 실패한다.

- [ ] **Step 4: 최소 라우터를 구현한다**

Create `AppRoute.kt`:

```kotlin
package com.sungkyul.reminder.navigation

import androidx.navigation3.runtime.NavKey
import kotlinx.serialization.Serializable

sealed interface AppRoute : NavKey {
    @Serializable data object Login : AppRoute
    @Serializable data object Home : AppRoute
}
```

Create `AppNavigator.kt`:

```kotlin
package com.sungkyul.reminder.navigation

import androidx.compose.runtime.mutableStateListOf

class AppNavigator(initialRoute: AppRoute = AppRoute.Login) {
    private val mutableBackStack = mutableStateListOf(initialRoute)
    val backStack: List<AppRoute> get() = mutableBackStack

    fun navigate(route: AppRoute) {
        mutableBackStack.add(route)
    }

    fun back(): Boolean {
        if (mutableBackStack.size == 1) return false
        mutableBackStack.removeAt(mutableBackStack.lastIndex)
        return true
    }
}
```

- [ ] **Step 5: 라우터 테스트를 통과시킨다**

Run:

```powershell
.\gradlew.bat testDebugUnitTest --tests "com.sungkyul.reminder.navigation.AppNavigatorTest"
```

Expected: 두 테스트가 모두 통과한다.

- [ ] **Step 6: Android 기반을 커밋한다**

```powershell
git add -- android-kotlin
git commit -m "기반: Android Compose 프로젝트와 라우터 추가"
```

---

### Task 3: Android 앱 셸과 Compose 화면 테스트 구축

**Files:**
- Create: `android-kotlin/app/src/main/kotlin/com/sungkyul/reminder/MainActivity.kt`
- Create: `android-kotlin/app/src/main/kotlin/com/sungkyul/reminder/app/ReminderApp.kt`
- Create: `android-kotlin/app/src/main/kotlin/com/sungkyul/reminder/ui/theme/ReminderTheme.kt`
- Test: `android-kotlin/app/src/androidTest/kotlin/com/sungkyul/reminder/app/ReminderAppTest.kt`

**Interfaces:**
- Consumes: Task 2의 `AppNavigator`와 `AppRoute`
- Produces: `ReminderApp(navigator: AppNavigator)`, 테스트 태그 `login-screen`, `home-screen`

- [ ] **Step 1: 실패하는 Compose 화면 테스트를 작성한다**

Create `ReminderAppTest.kt`:

```kotlin
package com.sungkyul.reminder.app

import androidx.compose.ui.test.assertIsDisplayed
import androidx.compose.ui.test.junit4.createComposeRule
import androidx.compose.ui.test.onNodeWithTag
import com.sungkyul.reminder.navigation.AppNavigator
import com.sungkyul.reminder.navigation.AppRoute
import org.junit.Rule
import org.junit.Test

class ReminderAppTest {
    @get:Rule val composeRule = createComposeRule()

    @Test
    fun 초기_로그인_화면을_표시한다() {
        composeRule.setContent { ReminderApp(navigator = AppNavigator()) }
        composeRule.onNodeWithTag("login-screen").assertIsDisplayed()
    }

    @Test
    fun 홈_경로를_표시한다() {
        val navigator = AppNavigator(AppRoute.Home)
        composeRule.setContent { ReminderApp(navigator = navigator) }
        composeRule.onNodeWithTag("home-screen").assertIsDisplayed()
    }
}
```

- [ ] **Step 2: 테스트가 `ReminderApp` 부재로 실패하는지 확인한다**

Run:

```powershell
.\gradlew.bat connectedDebugAndroidTest -Pandroid.testInstrumentationRunnerArguments.class=com.sungkyul.reminder.app.ReminderAppTest
```

Expected: `ReminderApp`을 찾지 못해 컴파일이 실패한다.

- [ ] **Step 3: 최소 앱 셸을 구현한다**

Create `ReminderApp.kt`:

```kotlin
package com.sungkyul.reminder.app

import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.testTag
import androidx.navigation3.ui.NavDisplay
import androidx.navigation3.runtime.entry
import androidx.navigation3.runtime.entryProvider
import com.sungkyul.reminder.navigation.AppNavigator
import com.sungkyul.reminder.navigation.AppRoute

@Composable
fun ReminderApp(navigator: AppNavigator) {
    NavDisplay(
        backStack = navigator.backStack,
        onBack = { navigator.back() },
        entryProvider = entryProvider {
            entry<AppRoute.Login> {
                Box(
                    modifier = Modifier.fillMaxSize().testTag("login-screen"),
                    contentAlignment = Alignment.Center,
                ) { Text("로그인") }
            }
            entry<AppRoute.Home> {
                Box(
                    modifier = Modifier.fillMaxSize().testTag("home-screen"),
                    contentAlignment = Alignment.Center,
                ) { Text("홈") }
            }
        },
    )
}
```

Create `MainActivity.kt`:

```kotlin
package com.sungkyul.reminder

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import com.sungkyul.reminder.app.ReminderApp
import com.sungkyul.reminder.navigation.AppNavigator
import com.sungkyul.reminder.ui.theme.ReminderTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { ReminderTheme { ReminderApp(AppNavigator()) } }
    }
}
```

Create `ReminderTheme.kt`:

```kotlin
package com.sungkyul.reminder.ui.theme

import androidx.compose.material3.MaterialTheme
import androidx.compose.runtime.Composable

@Composable
fun ReminderTheme(content: @Composable () -> Unit) {
    MaterialTheme(content = content)
}
```

Declare `MainActivity` as the exported launcher activity in the manifest:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application android:label="정리마인더" android:theme="@android:style/Theme.Material.Light.NoActionBar">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

- [ ] **Step 4: Compose 화면 테스트와 단위 테스트를 실행한다**

Run:

```powershell
.\gradlew.bat testDebugUnitTest connectedDebugAndroidTest
```

Expected: Task 2 단위 테스트와 두 Compose UI 테스트가 모두 통과한다.

- [ ] **Step 5: Android 앱 셸을 커밋한다**

```powershell
git add -- android-kotlin
git commit -m "기반: Android 앱 셸과 화면 상태 추가"
```

---

### Task 4: iOS 프로젝트 명세와 라우터 테스트 구축

**Files:**
- Create: `ios-swift/project.yml`
- Create: `ios-swift/Config/Debug.xcconfig`
- Create: `ios-swift/Config/Release.xcconfig`
- Create: `ios-swift/ReminderApp/Navigation/AppRoute.swift`
- Create: `ios-swift/ReminderApp/Navigation/AppRouter.swift`
- Test: `ios-swift/ReminderAppTests/AppRouterTests.swift`

**Interfaces:**
- Consumes: macOS, Xcode, XcodeGen
- Produces: `AppRoute`, `@Observable @MainActor AppRouter`, `path: [AppRoute]`, `navigate(to:)`, `back() -> Bool`

- [ ] **Step 1: XcodeGen 프로젝트 명세를 작성한다**

Create `project.yml`:

```yaml
name: ReminderApp
options:
  deploymentTarget:
    iOS: "17.0"
configs:
  Debug: debug
  Release: release
settings:
  base:
    SWIFT_VERSION: "6.0"
    SWIFT_STRICT_CONCURRENCY: complete
targets:
  ReminderApp:
    type: application
    platform: iOS
    sources: [ReminderApp]
    configFiles:
      Debug: Config/Debug.xcconfig
      Release: Config/Release.xcconfig
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.sungkyul.reminderApp
        GENERATE_INFOPLIST_FILE: YES
        INFOPLIST_KEY_CFBundleDisplayName: 정리마인더
        INFOPLIST_KEY_UILaunchScreen_Generation: YES
  ReminderAppTests:
    type: bundle.unit-test
    platform: iOS
    sources: [ReminderAppTests]
    dependencies:
      - target: ReminderApp
  ReminderAppUITests:
    type: bundle.ui-testing
    platform: iOS
    sources: [ReminderAppUITests]
    dependencies:
      - target: ReminderApp
schemes:
  ReminderApp:
    build:
      targets:
        ReminderApp: all
    test:
      targets:
        - ReminderAppTests
        - ReminderAppUITests
```

Create `Debug.xcconfig` and `Release.xcconfig` with `CODE_SIGN_STYLE = Automatic` and no development team identifier.

- [ ] **Step 2: 실패하는 Swift 라우터 테스트를 작성한다**

Create `AppRouterTests.swift`:

```swift
import Testing
@testable import ReminderApp

@MainActor
struct AppRouterTests {
    @Test("초기 화면은 로그인이다")
    func initialRouteIsLogin() {
        let router = AppRouter()
        #expect(router.path == [.login])
    }

    @Test("화면 이동과 뒤로 가기를 순서대로 처리한다")
    func navigateAndBack() {
        let router = AppRouter()
        router.navigate(to: .home)

        #expect(router.path == [.login, .home])
        #expect(router.back())
        #expect(router.path == [.login])
        #expect(!router.back())
    }
}
```

- [ ] **Step 3: 테스트가 타입 부재로 실패하는지 macOS에서 확인한다**

Run:

```bash
cd ios-swift
xcodegen generate
xcodebuild test -scheme ReminderApp -destination 'platform=iOS Simulator,name=iPhone 17' -only-testing:ReminderAppTests/AppRouterTests
```

Expected: `AppRouter`와 `AppRoute`가 없어 Swift 컴파일이 실패한다.

- [ ] **Step 4: 최소 Swift 라우터를 구현한다**

Create `AppRoute.swift`:

```swift
enum AppRoute: Hashable, Sendable {
    case login
    case home
}
```

Create `AppRouter.swift`:

```swift
import Observation

@Observable
@MainActor
final class AppRouter {
    private(set) var path: [AppRoute]

    init(initialRoute: AppRoute = .login) {
        path = [initialRoute]
    }

    func navigate(to route: AppRoute) {
        path.append(route)
    }

    @discardableResult
    func back() -> Bool {
        guard path.count > 1 else { return false }
        path.removeLast()
        return true
    }
}
```

- [ ] **Step 5: Swift 라우터 테스트를 통과시킨다**

Run the same `xcodebuild test` command from Step 3.

Expected: 두 Swift Testing 테스트가 모두 통과한다.

- [ ] **Step 6: iOS 라우터 기반을 커밋한다**

```bash
git add -- ios-swift
git commit -m "기반: iOS SwiftUI 프로젝트와 라우터 추가"
```

---

### Task 5: iOS 앱 셸과 UI 테스트 구축

**Files:**
- Create: `ios-swift/ReminderApp/ReminderApp.swift`
- Create: `ios-swift/ReminderApp/App/AppContainer.swift`
- Create: `ios-swift/ReminderApp/UI/RootView.swift`
- Test: `ios-swift/ReminderAppUITests/ReminderAppUITests.swift`

**Interfaces:**
- Consumes: Task 4의 `AppRouter`와 `AppRoute`
- Produces: `RootView(router:)`, 접근성 식별자 `login-screen`, `home-screen`

- [ ] **Step 1: 실패하는 UI 테스트를 작성한다**

Create `ReminderAppUITests.swift`:

```swift
import XCTest

final class ReminderAppUITests: XCTestCase {
    func testLaunchShowsLoginScreen() {
        let app = XCUIApplication()
        app.launch()
        XCTAssertTrue(app.otherElements["login-screen"].waitForExistence(timeout: 2))
    }
}
```

- [ ] **Step 2: 테스트가 앱 진입점 부재로 실패하는지 확인한다**

Run:

```bash
cd ios-swift
xcodegen generate
xcodebuild test -scheme ReminderApp -destination 'platform=iOS Simulator,name=iPhone 17' -only-testing:ReminderAppUITests
```

Expected: 앱 진입점 또는 `login-screen` 요소가 없어 실패한다.

- [ ] **Step 3: 최소 SwiftUI 앱 셸을 구현한다**

Create `ReminderApp.swift`:

```swift
import SwiftUI

@main
struct ReminderApp: App {
    @State private var router = AppRouter()

    var body: some Scene {
        WindowGroup {
            RootView(router: router)
        }
    }
}
```

Create `RootView.swift`:

```swift
import SwiftUI

struct RootView: View {
    let router: AppRouter

    var body: some View {
        Group {
            switch router.path.last {
            case .login, .none:
                Text("로그인")
                    .frame(maxWidth: .infinity, maxHeight: .infinity)
                    .accessibilityIdentifier("login-screen")
            case .home:
                Text("홈")
                    .frame(maxWidth: .infinity, maxHeight: .infinity)
                    .accessibilityIdentifier("home-screen")
            }
        }
    }
}
```

Create `AppContainer.swift` as a concrete empty dependency root for this milestone:

```swift
struct AppContainer: Sendable {}
```

- [ ] **Step 4: 단위 및 UI 테스트를 모두 실행한다**

Run:

```bash
xcodebuild test -scheme ReminderApp -destination 'platform=iOS Simulator,name=iPhone 17'
```

Expected: 라우터 단위 테스트와 로그인 시작 화면 UI 테스트가 모두 통과한다.

- [ ] **Step 5: iOS 앱 셸을 커밋한다**

```bash
git add -- ios-swift
git commit -m "기반: iOS 앱 셸과 화면 상태 추가"
```

---

### Task 6: 기반 기능 동등성 확인과 문서 마감

**Files:**
- Modify: `docs/migration/native-feature-parity.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: Task 1~5의 검증 결과
- Produces: 재현 가능한 실행 명령과 기반 단계 완료 상태

- [ ] **Step 1: 검증 명령을 실행한다**

Android:

```powershell
Set-Location android-kotlin
.\gradlew.bat testDebugUnitTest connectedDebugAndroidTest
```

iOS on macOS:

```bash
cd ios-swift
xcodegen generate
xcodebuild test -scheme ReminderApp -destination 'platform=iOS Simulator,name=iPhone 17'
```

Flutter where available:

```bash
cd flutter-legacy
flutter test
```

Expected: 사용 가능한 각 플랫폼의 전체 테스트가 통과한다. 실행할 수 없는 플랫폼은 성공으로 표시하지 않고 실행 환경과 미검증 사유를 기록한다.

- [ ] **Step 2: 기능 동등성 표를 실제 결과로 갱신한다**

Change only `앱 실행 및 루트 화면` to `완료` for a platform after its build and tests pass. Keep every unimplemented feature as `미착수`.

- [ ] **Step 3: 전체 변경을 점검한다**

Run:

```powershell
git diff --check
git status --short
git log --oneline -8
```

Expected: 공백 오류가 없고, 의도하지 않은 파일이 없으며, 구현 커밋이 작업별로 분리되어 있다.

- [ ] **Step 4: 문서 마감을 커밋한다**

```powershell
git add -- README.md docs/migration/native-feature-parity.md
git commit -m "문서: 네이티브 앱 기반 검증 결과 기록"
```

## 다음 계획의 시작 조건

- Android 기반 단계는 `testDebugUnitTest`와 `connectedDebugAndroidTest`가 통과해야 한다.
- iOS 기반 단계는 macOS/Xcode에서 단위 및 UI 테스트가 통과해야 한다.
- 두 조건이 충족된 뒤 `로그인`, `토큰 갱신`, `로그아웃`, `비밀번호 변경`을 다루는 인증 계획을 작성한다.
- Flutter 앱의 기존 API 계약을 먼저 고정하는 계약 테스트를 작성한 후 플랫폼별 인증 구현을 시작한다.
