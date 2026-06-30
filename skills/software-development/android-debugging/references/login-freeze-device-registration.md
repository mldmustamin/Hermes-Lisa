# Login Freeze — Device Registration Blocking

## Symptom

Android app freezes on login AFTER adding HttpClient timeout. Login button pressed → spinner spins for 15-30 seconds → either errors or eventually succeeds. No force close, just UI stuck.

## Root Cause

Device registration HTTP call blocks the login `onSuccess` handler:

```kotlin
// ❌ OLD — device registration BLOCKS login completion
authRepository.login(login, password)
    .onSuccess { response ->
        saveSessionAndRegisterDevice(response)  // <-- suspend fun, HTTP call inside
        _uiState.update { it.copy(isLoading = false, isLoggedIn = true) }
    }
```

`saveSessionAndRegisterDevice()` calls `deviceRepository.registerDevice()` which makes another HTTP request via the same HttpClient. Even with timeout, the user sees:

1. Login request (max 15s timeout)
2. Device registration request (max 15s timeout)  
3. `isLoading = false` only runs AFTER step 2

Total freeze: up to 30 seconds.

## Fix (3 changes)

### 1. Set isLoading=false BEFORE device registration

```kotlin
authRepository.login(login, password)
    .onSuccess { response ->
        // Save session FIRST (fast, local)
        try {
            sessionManager.saveSession(...)
            userDao.insertUser(...)
        } catch (e: Exception) { /* non-fatal */ }

        // Unblock UI immediately
        _uiState.update { it.copy(isLoading = false, isLoggedIn = true) }

        // Device registration in BACKGROUND
        registerDeviceInBackground(response)
    }
```

### 2. Move device registration to separate coroutine

```kotlin
private fun registerDeviceInBackground(response: LoginResponse) {
    viewModelScope.launch {
        try {
            deviceRepository.registerDevice(...)
                .onSuccess { ... }
                .onFailure { appLogger.warning(...) }
        } catch (_: Exception) { /* non-fatal */ }
    }
}
```

### 3. Better error messages for network failures

```kotlin
.onFailure { error ->
    val msg = when {
        error is TimeoutCancellationException -> "Koneksi timeout. Server tidak merespon."
        error.message?.contains("Unable to resolve host") == true -> "Tidak dapat terhubung ke server."
        error.message?.contains("Connection refused") == true -> "Server sedang sibuk. Coba lagi."
        else -> error.message ?: "Gagal login."
    }
    _uiState.update { it.copy(isLoading = false, error = msg) }
}
```

## Result

- Login completes in <1 second after auth success
- Device registration runs silently in background
- If device registration fails, login still succeeds
- User sees specific error messages (not generic "error")
