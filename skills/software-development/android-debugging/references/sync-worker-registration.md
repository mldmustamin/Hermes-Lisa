# SyncWorker Not Registered — Background Sync Never Runs

## Symptom

- Android app runs, login works, device registers
- User creates projects and transactions locally — data appears on dashboard
- But data **never appears on the backend** (sync monitor empty, API list empty)
- No errors or crashes — the app simply never sends data to server

## Root Cause

The `SyncWorker` class is defined in `data/sync/SyncWorker.kt` with all the push/pull logic working correctly, but it is **never registered with WorkManager**. The periodic or one-time sync request is never created, so the sync loop never starts.

This happens because:
1. The developer wrote the sync worker + outbox + API service
2. But forgot the final step: scheduling it via `WorkManager.enqueueUniquePeriodicWork()` in `Application.onCreate()`

## Fix — Two-Part

### Part 1: Periodic Sync in Application.onCreate()

```kotlin
// FundsManagerApp.kt
import androidx.work.Constraints
import androidx.work.ExistingPeriodicWorkPolicy
import androidx.work.NetworkType
import androidx.work.PeriodicWorkRequestBuilder
import androidx.work.WorkManager
import java.util.concurrent.TimeUnit

class FundsManagerApp : Application() {
    override fun onCreate() {
        super.onCreate()
        // ... existing code ...
        scheduleSync()
    }

    private fun scheduleSync() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()

        val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(
            15, TimeUnit.MINUTES
        )
            .setConstraints(constraints)
            .build()

        WorkManager.getInstance(this).enqueueUniquePeriodicWork(
            "fundsmanager_sync",
            ExistingPeriodicWorkPolicy.KEEP,
            syncRequest
        )
    }
}
```

### Part 2: One-Time Sync After Login

```kotlin
// LoginViewModel.kt — inside saveSessionAndRegisterDevice(), after device registration:
import androidx.work.ExistingWorkPolicy
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager

// At the end, after device registration:
val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>().build()
WorkManager.getInstance(application).enqueueUniqueWork(
    "fundsmanager_sync_now",
    ExistingWorkPolicy.REPLACE,
    syncRequest
)
```

This requires injecting `Application` into the ViewModel:
```kotlin
class LoginViewModel @Inject constructor(
    ...
    private val application: Application,
) : ViewModel()
```

## Verification

After fix, sync behavior:
1. App **start** → periodic sync scheduled (every 15 minutes)
2. **Login** success → immediate one-time sync (pushes pending outbox)
3. **Create transaction** → stored in outbox → next sync cycle pushes it
4. **Sync Monitor** on web → shows device + pending/rejected counts

## Detection

Search the codebase for where WorkManager is initialized:

```bash
# Should find at least one of these:
grep -r "WorkManager" app/src/main/java/
grep -r "PeriodicWorkRequestBuilder" app/src/main/java/
grep -r "enqueueUniquePeriodicWork" app/src/main/java/

# If only SyncWorker class exists but no scheduling code above, sync is disabled
```
