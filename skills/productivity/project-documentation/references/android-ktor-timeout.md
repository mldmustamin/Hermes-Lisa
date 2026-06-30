# Ktor HttpClient Timeout — Android Freeze on Login

## Symptom
Android app freezes on login screen. No error message, no loading indicator change. App appears hung.

## Root Cause
Ktor `HttpClient` has NO default timeout. When the server is unreachable or slow to respond, the HTTP request waits indefinitely. Since the login ViewModel sets `isLoading = true` before making the request, the UI stays in loading state forever.

## Fix
Add `HttpTimeout` plugin to HttpClient configuration:

```kotlin
// In NetworkModule.kt or wherever HttpClient is created
import io.ktor.client.plugins.HttpTimeout

fun provideHttpClient(): HttpClient {
    return HttpClient {
        install(HttpTimeout) {
            requestTimeoutMillis = 15_000  // 15 detik total
            connectTimeoutMillis = 8_000   // 8 detik untuk connect
            socketTimeoutMillis = 15_000   // 15 detik socket idle
        }
        // ... ContentNegotiation, etc.
    }
}
```

## Why 15/8/15 seconds?
- **connect (8s)**: Enough for slow networks (EDGE/3G) but not so long the user gives up
- **request (15s)**: Allow server to process complex queries but timeout before ANR (Application Not Responding, ~5s on Android)
- **socket (15s)**: Prevent hanging on slow downloads

## Verification
After fix, when server is unreachable:
1. App shows loading spinner for max 15 seconds
2. `.onFailure` handler catches the timeout exception
3. Error message: "Gagal login. Periksa koneksi dan kredensial."
4. Loading spinner stops, user can retry

## Related
FundManager V2: `app/src/main/java/com/example/fundsmanager/di/NetworkModule.kt`
Commit: `91b4853` — "fix(android): HttpClient timeout + Crash Reporter"
