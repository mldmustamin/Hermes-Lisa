# HttpClient Timeout — Android Login Freeze

## Symptom
App freezes when user taps Login. Spinner spins forever, no error message, no force close. App appears completely stuck.

## Root Cause
`HttpClient` (Ktor) in `NetworkModule.kt` has **no timeout configured**. When the server is unreachable, slow, or the network is down, the HTTP request hangs indefinitely because there's no deadline set.

## Fix

In `app/src/main/java/com/example/fundsmanager/di/NetworkModule.kt`, add `HttpTimeout`:

```kotlin
import io.ktor.client.plugins.HttpTimeout

install(HttpTimeout) {
    requestTimeoutMillis = 15_000  // 15 detik total
    connectTimeoutMillis = 8_000   // 8 detik untuk connect
    socketTimeoutMillis = 15_000   // 15 detik untuk socket read
}
```

After this fix, requests will fail after the timeout with an exception caught by the ViewModel's `.onFailure` handler, which sets `isLoading = false` and shows an error message to the user.

## Why This Pattern Matters

ALWAYS configure timeouts on HTTP clients in Android apps. Without it:
- Users see indefinite freezes with no feedback
- Appears as a crash/freeze bug when it's actually a missing config
- Affects ALL network calls (login, sync, data fetch)

## Verification

- Build APK after adding timeout
- Test with server unreachable (turn off server or use airplane mode)
- App should show error after 15 seconds, not freeze
