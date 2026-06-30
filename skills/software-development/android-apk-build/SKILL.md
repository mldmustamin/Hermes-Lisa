---
name: android-apk-build
category: software-development
description: Build Android APK with custom backend URL — install SDK, fix Kotlin compilation, rebuild and distribute.
---

# Android APK Build

Class-level skill for rebuilding an existing Android project APK, especially when the backend API URL changes (e.g. pointing from localhost/emulator to a production server).

## When to Load

- User says "rebuild APK" / "app tidak bisa login" / "APK pointing to wrong server"
- User asks to change backend URL in Android app
- Android project compiles but runtime API calls fail

## Prerequisites

- Android project with `gradlew`, `build.gradle.kts`, Kotlin source
- JDK 17+ (for Gradle 8.x compatibility)
- Internet access to download Android SDK

---

## Step 0 — PRE-BUILD: Verify Against Official Android Architecture

Before writing ANY Android code, verify the implementation plan against the 8 official Android architecture principles. See `references/android-architecture-checklist.md` for the complete checklist.

Key rules:
- **Research FIRST**: Read developer.android.com documentation before coding
- **Check imports exist**: Don't reference classes that aren't in the codebase
- **Follow existing patterns**: New screens must follow the same ViewModel+StateFlow+UiState pattern
- **Single Activity + Compose Navigation**: No new Activities or Fragments

---

## Step 1 — Identify the API Base URL in Source

Most Android apps define the backend URL in a config file. Search for common patterns:

```bash
grep -r "base_url\|BASE_URL\|baseUrl\|10.0.2.2\|localhost\|api_url" app/src/main/java/
```

Common locations:

| Pattern | Typical Value | Context |
|---------|---------------|---------|
| `object ApiConfig` | `http://10.0.2.2:8000/api/v1` | Emulator → host machine |
| `BuildConfig.API_URL` | Set via `buildConfigField` in `build.gradle.kts` | Build-time override |
| Hardcoded string | In repository or service files | Direct URL reference |

**Fix:** Change the URL to the production server:

```kotlin
const val DEFAULT_BASE_URL = "http://YOUR_SERVER_IP/api/v1"
```

---

## Step 2 — Install JDK (if missing)

```bash
# Check current JDK
java -version

# Gradle 8.x requires JDK 17+. Install if missing:
apt-get install -y openjdk-17-jdk   # JDK 17
apt-get install -y openjdk-21-jdk   # JDK 21 (if project requires it)

# Export JAVA_HOME
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64  # adjust path
```

---

## Step 3 — Install Android SDK

The Android project needs `ANDROID_HOME` pointing to an SDK with the correct `compileSdk` platform and `build-tools`.

### 3a — Check what SDK version the project needs

```bash
grep -E "compileSdk|targetSdk|minSdk" app/build.gradle.kts
```

### 3b — Install SDK command-line tools

```bash
mkdir -p /opt/android-sdk/cmdline-tools
cd /opt/android-sdk/cmdline-tools
curl -sL "https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip" -o cmdline-tools.zip
unzip -q cmdline-tools.zip
mv cmdline-tools latest
```

### 3c — Install required platform & build-tools

```bash
export ANDROID_HOME=/opt/android-sdk
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin

yes | sdkmanager --sdk_root=$ANDROID_HOME \
  "platforms;android-36" \
  "build-tools;36.0.0"
```

Replace `android-36` and `36.0.0` with the versions from `build.gradle.kts` (`compileSdk` and latest matching build-tools).

### 3d — Create local.properties

```bash
echo "sdk.dir=/opt/android-sdk" > local.properties
```

---

## Step 4 — Platform Wrapper Check: `gradlew` vs `gradlew-local`

Many Android projects include a platform-specific wrapper (e.g. `gradlew-local` for macOS) alongside the standard `gradlew`. On Linux servers, the platform wrapper will fail because it hardcodes macOS paths:

```bash
# gradlew-local example — hardcoded macOS paths:
JDK17_HOME="$PROJECT_DIR/.tooling/jdk-17.0.19+10/Contents/Home"
SDK_HOME="${ANDROID_SDK_ROOT:-/Users/user/Library/Android/sdk}"
```

**Fix:** Always use the standard `gradlew` (the generic Gradle wrapper) on Linux. Set `JAVA_HOME` and ensure `ANDROID_HOME` is configured via `local.properties`:

```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
./gradlew assembleDebug --no-daemon
```

---

## Step 5 — Fix Common Kotlin Compilation Errors

After changing files or building on a fresh environment, these errors commonly appear:

### 5a — Unresolved reference `jsonObject`

```
e: Unresolved reference 'jsonObject'.
```

**Fix:** Add missing import:

```kotlin
import kotlinx.serialization.json.jsonObject
import kotlinx.serialization.json.JsonObject
```

### 5b — Missing Hilt/Dagger generated classes

```
e: Unresolved reference: Hilt_*
```

**Fix:** Run clean build or enable kapt/ksp in `build.gradle.kts`. Hilt needs `kotlin-kapt` or `google-ksp` plugin:

```kotlin
plugins {
    id("com.google.devtools.ksp") version "2.x.x"
}
```

### 5c — SDK location not found

```
SDK location not found. Define a valid SDK location with an ANDROID_HOME environment variable
```

**Fix:** Either set the env var or create `local.properties`:

```bash
export ANDROID_HOME=/opt/android-sdk
# OR
echo "sdk.dir=/opt/android-sdk" > local.properties
```

---

## Step 6 — Fix Android Manifest Issues (Runtime Failures)

Compilation may succeed but the app crashes or fails at runtime. Two common manifest-related issues when rebuilding an APK for a new server:

### 6a — Missing INTERNET Permission → `socket failed: EPERM`

**Symptom:** App opens but any network call fails with `socket failed: EPERM (operation not permitted)`.

**Root cause:** The `<uses-permission android:name="android.permission.INTERNET" />` is missing from `AndroidManifest.xml`.

**Fix:** Add both permissions before the `<application>` tag:

```xml
<manifest ...>
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <application ...>
```

### 6b — Cleartext HTTP Blocked → `Cleartext HTTP traffic not permitted`

**Symptom:** Network calls fail with `Cleartext HTTP traffic to YOUR_SERVER not permitted by network security policy`.

**Root cause:** Android 9+ (API 28) blocks cleartext (non-HTTPS) traffic by default. The app uses `http://` but the manifest has no exemption.

**Fix A — Network Security Config (recommended, scoped):**

Create `res/xml/network_security_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">YOUR_SERVER_IP</domain>
    </domain-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

Reference it in `AndroidManifest.xml`:

```xml
<application
    ...
    android:networkSecurityConfig="@xml/network_security_config"
    ...>
```

**Fix B — Global flag (quick, less secure):**

```xml
<application
    ...
    android:usesCleartextTraffic="true"
    ...>
```

> **Note:** Fix A is preferred — it only allows HTTP to your specific server and keeps all other cleartext blocked.

---

## Step 7 — Fix Compose Runtime Crash — `hiltViewModel()` on Non-ViewModel

**Symptom:** App force closes (crashes) when navigating to a specific screen, with no useful stack trace in logcat. The crash happens on screen entry, not during a network call or data load.

**Root cause:** A `@Composable` function in the NavHost calls `hiltViewModel<SomeClass>()` where `SomeClass` is **not** a `ViewModel` subclass — it's a plain `@Singleton` service class injected via `@Inject constructor`. Hilt's `hiltViewModel()` only works with actual `ViewModel` subclasses. Calling it on a non-ViewModel throws an unhandled exception during composition.

**Example from FundManager V2 — `FundsManagerNavHost.kt`:**

```kotlin
// ❌ BAD — SessionManager is NOT a ViewModel
composable(Screen.Settings.route) {
    val sessionManager: SessionManager = hiltViewModel()  // CRASH
}
```

Where `SessionManager` is:
```kotlin
@Singleton
class SessionManager @Inject constructor(
    @ApplicationContext private val context: Context
) { ... }
```

### Fix — Instantiate Directly with `LocalContext`

Since `SessionManager` has a standard `@Inject constructor` taking only `@ApplicationContext Context`, you can create it directly from any Composable:

```kotlin
// ✅ GOOD — instantiate directly using @Inject constructor
composable(Screen.Settings.route) {
    val context = LocalContext.current
    val sessionManager = remember { SessionManager(context.applicationContext) }
    val scope = rememberCoroutineScope()
    SettingsScreen(
        ...
        onLogoutClick = {
            scope.launch {
                sessionManager.clearSession()
                navController.navigate(Screen.Login.route) {
                    popUpTo(0) { inclusive = true }
                }
            }
        }
    )
}
```

**Required imports:**
```kotlin
import androidx.compose.runtime.remember
import androidx.compose.ui.platform.LocalContext
```

### Detection Pattern

When investigating a force-close on screen navigation:

1. Look at the NavHost composable for `val x: SomeType = hiltViewModel()`
2. Check if `SomeType` extends `androidx.lifecycle.ViewModel`
3. If not, find the class — if it uses `@Inject constructor` directly:
   - Check if it can be instantiated via `SomeClass(context)` from a Composable
   - Check if it needs to be fetched via Hilt `EntryPointAccessors.fromApplication()`
   - Check if the class should actually be refactored into a proper ViewModel

### When to Use Each Pattern

| Pattern | When | Example |
|---------|------|---------|
| `hiltViewModel()` | Class extends `ViewModel` ✅ | `LoginViewModel`, `DashboardViewModel` |
| Direct instantiation with `remember` | Simple `@Singleton` with `Context`-only constructor | `SessionManager`, `AppLogger` |
| `EntryPointAccessors.fromApplication()` | Complex `@Singleton` with multiple injected dependencies | API clients, repositories (best refactored into ViewModel instead) |

---

## Step 8 — Bump APK Version (BEFORE EVERY BUILD)

Every build MUST increment both `versionCode` and `versionName` in `app/build.gradle.kts`. Room schema export auto-generates version JSON files keyed to `versionCode`. Skipping this causes stale version reporting and migration issues.

**Manual:**
```kotlin
defaultConfig {
    versionCode = 22  // +1 from previous
    versionName = "2.0.0-b22"
}
```

**Auto-script (`bump_version.sh`):**
```bash
#!/bin/bash
cd "$(dirname "$0")"
V=$(grep "versionCode" app/build.gradle.kts | grep -o '[0-9]\+' | head -1)
NV=$((V + 1))
sed -i "s/versionCode = $V/versionCode = $NV/" app/build.gradle.kts
sed -i "s/versionName = \"[^\"]*\"/versionName = \"2.0.0-b$NV\"/" app/build.gradle.kts
echo "$V → $NV"
```

**Run:** `bash bump_version.sh && ./gradlew assembleDebug --no-daemon`

---

## Step 9 — Build the APK

```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
cd /path/to/project
./gradlew assembleDebug --no-daemon
```

| Variant | Command | Use |
|---------|---------|-----|
| **Debug APK** | `./gradlew assembleDebug` | Testing, sideload |
| **Release APK** | `./gradlew assembleRelease` | Production (needs signing config) |
| **Unit tests** | `./gradlew testDebugUnitTest` | Verify before build |

**Output:** `app/build/outputs/apk/debug/app-debug.apk`

---

## Step 10 — Distribute the APK

Option A — Serve via web server for direct download on phone:

```bash
cp app/build/outputs/apk/debug/app-debug.apk /path/to/backend/public/myapp.apk
# User downloads from http://server/myapp.apk via phone browser
```

Option B — Direct download via curl:

```bash
curl -O http://YOUR_SERVER_IP/path/to/app-debug.apk
```

Option C — Sideload via USB:

```bash
adb install app/build/outputs/apk/debug/app-debug.apk
```

---

## Step 10 — Post-Build UI Quick Check

After fixing connectivity and installing on device, verify these UI elements that users commonly expect:

1. **Dashboard shows logged-in user** — Is the user name + role visible in the top bar?
2. **Settings has user info** — Is there a profile card at the top?
3. **Logout is accessible** — Can the user find and tap logout without scrolling excessively?
4. **Data appears after creation** — After creating a project + transaction, does the dashboard reflect it?

If any of these are missing, see `references/post-build-ui-improvements.md` for the implementation pattern.

> **Common oversight:** Adding `SessionManager` to a Composable via `hiltViewModel()` only works for actual `ViewModel` subclasses. For `@Singleton` classes with a `Context`-only constructor, use `remember { SessionManager(context.applicationContext) }` instead. See `references/hilt-viewmodel-crash.md`.

---

## Step 11 — Verify on Device

1. Open Settings → Security → Enable "Install from unknown sources" (browser)
2. Download APK from the URL above
3. Install (overwrites existing app, local data preserved)
4. Login with credentials matching the new backend server

---

## Pitfalls Summary

| Issue | Symptom | Fix |
|-------|---------|-----|
| Gradle requires newer JDK | `Cannot find Java installation matching: Compatible with Java 21` | Install JDK 21: `apt-get install -y openjdk-21-jdk` |
| SDK not found | `SDK location not found` | Install Android SDK or create `local.properties` |
| `jsonObject` unresolved | Compile error in SyncWorker | Import `kotlinx.serialization.json.jsonObject` |
| Missing INTERNET permission | `socket failed: EPERM` at runtime | Add `<uses-permission android:name=\"android.permission.INTERNET\" />` to manifest |
| Cleartext HTTP blocked | `Cleartext HTTP traffic not permitted` | Add `network_security_config.xml` with domain exemption |
| `gradlew-local` fails on Linux | Hardcoded macOS JDK/SDK paths | Use plain `gradlew` instead, set `JAVA_HOME` and `local.properties` |
| APK still uses old URL | Login fails on phone | Double-check `ApiConfig.kt` or `BuildConfig` before build |
| `hiltViewModel()` on non-ViewModel | Force close on screen navigation | Instantiate directly with `LocalContext` + `remember` instead of `hiltViewModel()` |
| Cloud firewall blocks download | APK download URL returns 000 | Use cloud provider panel to open port or serve via port 80 |
| **Writing code before researching official docs** | Unresolved references, wrong patterns, architecture violations | Read `references/android-architecture-checklist.md` FIRST. Check developer.android.com. Only write code AFTER verifying the pattern exists in the codebase. |
| **Referencing non-existent imports** | `Unresolved reference` compilation error | Search codebase for the class with `search_files`. If it doesn't exist, DON'T import it — use existing classes or create the class first. |
| **App freeze/hang on login (Ktor no timeout)** | App opens, login button tap causes indefinite loading spinner, app appears frozen but doesn't crash | Ktor `HttpClient` was created WITHOUT `install(HttpTimeout)`. Without timeout, network requests hang forever when server is unreachable. Fix: always add `install(HttpTimeout) { requestTimeoutMillis = 15_000; connectTimeoutMillis = 8_000 }` in `NetworkModule.provideHttpClient()`. Debug: `grep -rn 'HttpClient' app/` and verify `HttpTimeout` is imported and installed. |\n| **Room DB migration missed (version already at target)** | Tables exist in Entity code but `CREATE TABLE` never executed on device — app crashes with \"Migration didn't properly handle\" | User's DB is already at version N from a previous build that didn't create the tables. Migration N-1→N never runs. Fix: bump DB version + 1, create new migration with `CREATE TABLE IF NOT EXISTS` for every table. Never assume a migration will be re-run — always use IF NOT EXISTS. Also fix column name mismatches (e.g., entity uses `province` but migration used `provinsi`). |\n| **App login freeze (device registration blocks UI)** | Login API call succeeds but app still frozen after login | `saveSessionAndRegisterDevice()` calls `deviceRepository.registerDevice()` synchronously inside `.onSuccess` — another HTTP call that blocks before `isLoading = false`. Fix: set `isLoading = false` BEFORE device registration, move registration to a separate coroutine via `launch {}`. Device registration is non-critical — login should complete instantly after auth succeeds. |
| **New PHP file not readable by PHP-FPM** | API endpoint returns 500 with `Failed to open stream: Permission denied` referencing a controller file | Files created by root have `640` perms (owner-only read). Fix: `chmod 644 /path/to/new/Controller.php`. Prevent: after creating PHP files, run `find dir -name '*.php' -newer reference_file -exec chmod 644 {} \;` |
