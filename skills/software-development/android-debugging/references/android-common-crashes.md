# Android Common Runtime Crashes — Diagnosis & Fix Reference

## 1. CLEARTEXT Communication Not Permitted

**Error:**
```
Cleartext communication to <host> not permitted by network security policy
```

**Root Cause:** Android 9+ (API 28+) blocks cleartext HTTP by default.

**Fix:**
1. Create `res/xml/network_security_config.xml`:
   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <network-security-config>
       <domain-config cleartextTrafficPermitted="true">
           <domain includeSubdomains="false">YOUR_SERVER_IP_OR_DOMAIN</domain>
       </domain-config>
       <base-config cleartextTrafficPermitted="false">
           <trust-anchors>
               <certificates src="system" />
           </trust-anchors>
       </base-config>
   </network-security-config>
   ```
2. Add to `<application>` in `AndroidManifest.xml`:
   ```xml
   android:networkSecurityConfig="@xml/network_security_config"
   ```

---

## 2. Socket Failed: EPERM (Operation Not Permitted)

**Error:**
```
socket failed: EPERM operation not permitted
```

**Root Cause:** Missing INTERNET permission.

**Fix:** Add before `<application>`:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
```

---

## 3. hiltViewModel() on Non-ViewModel Class

**Error:** App force closes when navigating to a screen.

**Root Cause:** `hiltViewModel<T>()` called on a class that doesn't extend ViewModel.

**Detect:** Search `hiltViewModel<ClassName>()` — if `ClassName` isn't a ViewModel subclass, it will crash.

**Fix:** Inject directly:
```kotlin
// BEFORE (CRASHES):
val sessionManager: SessionManager = hiltViewModel()

// AFTER (WORKS):
val context = LocalContext.current
val sessionManager = remember { SessionManager(context.applicationContext) }
```
Required imports: `LocalContext`, `remember`, and the target class.

---

## 4. Wrong API URL (10.0.2.2 on Real Device)

**Error:** Login times out or fails silently.

**Root Cause:** `10.0.2.2` is emulator-only (maps to host's localhost).

**Fix:** Change to actual server address:
```kotlin
const val DEFAULT_BASE_URL = "http://YOUR_SERVER/api/v1"
```

---

## 5. Password Change Endpoint Missing

**Error:** "Gagal mengganti password" after submitting.

**Root Cause:** Android calls `POST /api/v1/auth/change-password` but backend has no route.

**Fix (backend):**
1. Add route: `Route::post('/change-password', [AuthController::class, 'changePassword'])`
2. Implement: validate password + confirmation, hash, save, clear `password_change_required`

---

## 6. User Info Missing from Compose Screens

**Fix:** Read `SessionManager.activeSession` via `produceState`:
```kotlin
val sessionManager = remember { SessionManager(context.applicationContext) }
val activeSession by produceState<ActiveSession?>(initialValue = null) {
    value = sessionManager.activeSession.first()
}
```
Requires `import kotlinx.coroutines.flow.first`.

---

## 7. Foreign Key Constraint Failed (Code 787 — SQLiteConstraintException)

**Error:**
```
android.database.sqlite.SQLiteConstraintException: FOREIGN KEY constraint failed (code 787)
```

**Root Cause:** Login saves user session to DataStore (via SessionManager) but never inserts a UserEntity into the Room database. When creating a Project or Transaction, the userId foreign key references a user that doesn't exist in the local users table.

**Fix (LoginViewModel):**
1. Inject userDao into LoginViewModel
2. In saveSessionAndRegisterDevice(), insert the user into Room AFTER SessionManager but BEFORE device registration:
   userDao.insertUser(UserEntity(id = response.user.id, name = response.user.name, email = response.user.email, uuid = response.user.uuid))

**Rule of thumb:** Any data referenced by foreign keys from Room entities must exist locally BEFORE those entities are created.

---

## 8. Login Freeze / Hang — HttpClient No Timeout

**Error:** Login button shows spinner forever. App appears frozen but doesn't crash. May show ANR after 5 seconds.

**Root Cause:** Ktor `HttpClient` created WITHOUT `install(HttpTimeout)`. Network request hangs when server unreachable → `isLoading` stays `true` → UI blocked.

**Fix:** In `NetworkModule.provideHttpClient()`:
```kotlin
install(HttpTimeout) {
    requestTimeoutMillis = 15_000
    connectTimeoutMillis = 8_000
    socketTimeoutMillis = 15_000
}
```

Also verify ViewModel has `.onFailure { }` handler to update `isLoading = false`.

See `references/android-login-freeze-ktor-timeout.md` for full diagnosis script.
