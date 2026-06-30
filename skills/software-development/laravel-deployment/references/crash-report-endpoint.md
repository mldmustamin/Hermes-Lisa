# Server-Side Crash Report Endpoint

Lightweight endpoint for receiving Android crash reports. No authentication required (crashes happen before login).

## Backend Setup

### Route (routes/api.php)
```php
Route::post('/v1/crash-reports', [CrashReportController::class, 'store']);
```
Place OUTSIDE the `auth:sanctum` middleware group — crashes happen when the user is NOT logged in.

### Controller
```php
class CrashReportController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'report' => ['required', 'string', 'max:50000'],
            'device_model' => ['nullable', 'string', 'max:100'],
            'android_version' => ['nullable', 'string', 'max:20'],
            'app_version' => ['nullable', 'string', 'max:50'],
        ]);

        Log::channel('crash')->error('Android Crash Report', [
            'device_model' => $validated['device_model'] ?? 'unknown',
            'android_version' => $validated['android_version'] ?? 'unknown',
            'app_version' => $validated['app_version'] ?? 'unknown',
            'report' => $validated['report'],
        ]);

        return response()->json(['message' => 'Crash report received'], 201);
    }
}
```

### Log Channel (config/logging.php)
```php
'crash' => [
    'driver' => 'single',
    'path' => storage_path('logs/crash-reports.log'),
    'level' => 'debug',
    'replace_placeholders' => true,
],
```

## Android Side (CrashReporter)
Send via Ktor HTTP client when online:
```kotlin
suspend fun sendCrashReport(report: String) {
    try {
        httpClient.post("${ApiConfig.baseUrl}/crash-reports") {
            contentType(ContentType.Application.Json)
            setBody(mapOf(
                "report" to report,
                "device_model" to Build.MODEL,
                "android_version" to Build.VERSION.RELEASE,
                "app_version" to appVersion
            ))
        }
    } catch (_: Exception) {
        // Silent — crash reporting itself shouldn't crash
    }
}
```

## Security Notes
- `max:50000` limits payload size
- No auth required (users may not be logged in when crash occurs)
- Rate limiting recommended in production (Laravel throttle middleware)
- Log file rotation prevents disk exhaustion
