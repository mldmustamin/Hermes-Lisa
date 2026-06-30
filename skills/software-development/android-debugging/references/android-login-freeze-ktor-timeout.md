# Android Login Freeze — Ktor HttpClient Timeout

## Symptom
Android app freezes (spinner spins forever) after entering username/password and pressing login. No error toast, no crash.

## Root Cause Cascade

### 1. Missing HTTP Timeout (Most Common)
**File:** `di/NetworkModule.kt`
```kotlin
fun provideHttpClient(): HttpClient {
    return HttpClient {
        install(ContentNegotiation) { ... }
        // MISSING: HttpClient hangs forever without timeout
    }
}
```

**Fix:** Add `HttpTimeout` plugin:
```kotlin
import io.ktor.client.plugins.HttpTimeout

fun provideHttpClient(): HttpClient {
    return HttpClient {
        install(ContentNegotiation) { ... }
        install(HttpTimeout) {
            requestTimeoutMillis = 15_000  // 15s total
            connectTimeoutMillis = 8_000   // 8s connect
            socketTimeoutMillis = 15_000   // 15s socket
        }
    }
}
```

### 2. Device Registration Blocks Login Flow
**File:** `ui/screen/auth/LoginViewModel.kt`

The login flow calls `saveSessionAndRegisterDevice()` inside `.onSuccess {}`, which fires another HTTP call (`deviceRepository.registerDevice()`). If this call hangs or fails:

- `isLoading` stays `true` forever
- The UI never updates
- User sees permanent loading spinner

**Fix:** Move device registration to a SEPARATE coroutine that does NOT block login:

```kotlin
.onSuccess { response ->
    // Save session — quick, non-blocking
    sessionManager.saveSession(...)
    
    // Update UI IMMEDIATELY
    _uiState.update { it.copy(isLoading = false, isLoggedIn = true) }
    
    // Device registration in background — won't block login
    registerDeviceInBackground(response)
}
```

### 3. Session Save Errors Not Caught
If `sessionManager.saveSession()` or `userDao.insertUser()` throws, `isLoading` stays `true`.

**Fix:** Wrap in try-catch:
```kotlin
try {
    sessionManager.saveSession(...)
    userDao.insertUser(...)
} catch (e: Exception) {
    appLogger.warning(...) // Non-fatal
}
```

### 4. Generic Error Messages
Without timeout-specific messages, user sees "Unknown error" or nothing.

**Fix:** Add specific messages:
```kotlin
val msg = when {
    error is TimeoutCancellationException -> "Koneksi timeout. Server tidak merespon."
    error.message?.contains("Unable to resolve host") == true -> "Tidak dapat terhubung ke server."
    error.message?.contains("Connection refused") == true -> "Server sedang sibuk. Coba lagi."
    else -> error.message ?: "Gagal login."
}
```

## Debugging Checklist

1. [ ] Check `NetworkModule.kt` has `HttpTimeout` installed
2. [ ] Check `LoginViewModel.login()` sets `isLoading = false` BEFORE device registration
3. [ ] Check `registerDeviceInBackground()` is a separate coroutine
4. [ ] Check `saveSessionAndRegisterDevice` has try-catch
5. [ ] Check error messages include timeout-specific text
6. [ ] Check `network_security_config.xml` allows cleartext to server IP
7. [ ] Check `AndroidManifest.xml` has `INTERNET` permission
8. [ ] Verify server URL in `ApiConfig.kt` is correct and reachable
