# Android Crash Reporter — Full Pattern

## Overview

A systematic crash reporting system for Android apps that captures uncaught exceptions, stores them locally for review, and provides a share/debug screen.

## Components

### 1. CrashReporter Utility (`util/CrashReporter.kt`)

```kotlin
object CrashReporter {
    fun init(context: Context)
    fun hasPendingCrashes(): Boolean
    fun getCrashFiles(): List<File>
    fun clearCrashes()
    fun deleteCrash(file: File)
}
```

Captures: timestamp, thread name/state, device model, manufacturer, Android version, SDK level, app version, full stack trace, cause chain.

### 2. CrashLogScreen (`ui/screen/settings/CrashLogScreen.kt`)

Compose screen that:
- Lists all crash files sorted by date
- Shows first line preview on collapsed card
- Expands to show full stack trace (3000 chars)
- Share button (via FileProvider or direct text intent)
- Delete per-file
- Clear all with confirmation dialog

### 3. Server Endpoint (`POST /api/v1/crash-reports`)

```php
public function store(Request $request): JsonResponse
{
    $validated = $request->validate([
        'report' => ['required', 'string', 'max:50000'],
        'device_model' => ['nullable', 'string'],
        'android_version' => ['nullable', 'string'],
        'app_version' => ['nullable', 'string'],
    ]);
    Log::channel('crash')->error('Android Crash Report', [...]);
    return response()->json(['message' => 'Crash report received'], 201);
}
```

### 4. Wiring in Application.onCreate()

```kotlin
CrashReporter.init(this)
// Also log via existing FileAppLogger
```

### 5. FileProvider (for share intent)

In `AndroidManifest.xml` (usually already exists for other features):
```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

In `res/xml/file_paths.xml`:
```xml
<files-path name="crash_reports" path="crash_reports/" />
```

## Integration Checklist

- [ ] Create `CrashReporter.kt` in `util/`
- [ ] Call `CrashReporter.init(this)` in `Application.onCreate()`
- [ ] Create `CrashLogScreen.kt` with list/share/delete
- [ ] Add `Screen.CrashLog` route + `composable()` in NavHost
- [ ] Add `files-path` entry for `crash_reports/` in `file_paths.xml`
- [ ] Optionally: add server endpoint for remote crash reports
- [ ] Optionally: add crash log channel to `config/logging.php`
