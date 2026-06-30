# Android Login Freeze / Hang Debugging

## Symptom

User taps LOGIN button → loading spinner appears → app appears frozen (spinner shows but no error, no navigation). May eventually force-close after 30-60 seconds.

## Root Causes (check in order)

### 1. Ktor HttpClient has no timeout (MOST COMMON)

Without `HttpTimeout`, network requests hang indefinitely when server is unreachable.

**Check:**
```bash
grep -n "install(HttpTimeout)" app/src/main/java/.../di/NetworkModule.kt
```

**Fix:**
```kotlin
install(HttpTimeout) {
    requestTimeoutMillis = 15_000   // 15 seconds total
    connectTimeoutMillis = 8_000    // 8 seconds to establish TCP
    socketTimeoutMillis = 15_000    // 15 seconds for response
}
```

### 2. Background HTTP calls blocking login flow

The LoginViewModel calls `authRepository.login()` then immediately starts `deviceRepository.registerDevice()`. If device registration runs BEFORE `isLoading = false`, the user sees a spinner for the duration of BOTH HTTP calls.

**Bad pattern:**
```kotlin
.onSuccess { response ->
    saveSessionAndRegisterDevice(response)  // BLOCKS — another HTTP call
    _uiState.update { it.copy(isLoading = false) }
}
```

**Fix:** Move secondary HTTP calls to a separate coroutine:
```kotlin
.onSuccess { response ->
    saveSession(response)  // local only, fast
    _uiState.update { it.copy(isLoading = false) }
    registerDeviceInBackground(response)  // doesn't block UI
}

private fun registerDeviceInBackground(response: LoginResponse) {
    viewModelScope.launch {
        try { deviceRepository.registerDevice(...) }
        catch (_: Exception) { /* non-fatal */ }
    }
}
```

### 3. Database migration failed silently

If Room migration fails, the app may crash on startup or when accessing new tables. Check Room migration debugging reference.

### 4. Network security config blocking HTTP

Android 9+ blocks cleartext HTTP. Need `network_security_config.xml` with domain exemption for the server IP.

### 5. VPN / firewall / network unreachable

Server IP `103.94.11.78` must be reachable from the device. Test in phone browser first.

## Debugging Steps

1. **Check logcat for timeout:**
```bash
adb logcat | grep -i "timeout\|HttpTimeout\|connection.*refused\|unable.*resolve"
```

2. **Check login flow timing:**
```bash
adb logcat | grep "FundsManager" | grep -i "login\|register\|session"
```

3. **Verify timeout is in APK:**
```bash
# Decompile and check
grep -r "HttpTimeout\|requestTimeoutMillis\|connectTimeoutMillis" app/
```

4. **Simulate server unreachable** (turn off WiFi on phone) to verify timeout + error message appears within 15 seconds.
