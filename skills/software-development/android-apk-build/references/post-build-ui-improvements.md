# Post-Build UI Improvements — FundManager Dashboard & Settings

Reproduction session: `20260630_000100_447baa`

## Context

After fixing connectivity issues (EPERM, CLEARTEXT, URL), users reported three remaining problems:

1. **Dashboard doesn't show who's logged in** — no user name or role displayed
2. **Settings page has no user info** — no profile card showing current user
3. **Logout button is at the very bottom of Settings with no scroll** — users couldn't find it

## Fixes Applied

### Dashboard — Show User Info in TopAppBar

File: `DashboardHomeScreen.kt`

**Goal:** Display `${userName} • ${role}` subtitle under "Dashboard" title.

**Pattern — reading `SessionManager` from a Composable:**

```kotlin
val context = LocalContext.current
val sessionManager = remember { SessionManager(context.applicationContext) }
val activeSession by produceState<ActiveSession?>(initialValue = null) {
    value = sessionManager.activeSession.first()
}
```

Then in the TopAppBar title:
```kotlin
title = {
    Column {
        Text("Dashboard", ...)
        activeSession?.let { session ->
            Text(
                "${session.userName} • ${session.roles.joinToString(", ")}",
                style = MaterialTheme.typography.labelSmall,
                ...
            )
        }
    }
}
```

**Required imports:**
```kotlin
import androidx.compose.runtime.remember
import androidx.compose.runtime.produceState
import androidx.compose.ui.platform.LocalContext
import kotlinx.coroutines.flow.first
import com.example.fundsmanager.data.local.SessionManager
```

### Settings — Add User Info Card + Scroll

File: `SettingsScreen.kt`

**Changes:**
- Added a green profile card at the top showing user name, email, and role
- Wrapped the content Column in `verticalScroll(rememberScrollState())` so all items are reachable
- Added a bottom Spacer so the logout button is visible above the nav bar
- Cleaned up a duplicate `LocalContext` import

**User info card pattern:**
```kotlin
activeSession?.let { session ->
    Card(
        shape = RoundedCornerShape(24.dp),
        colors = CardDefaults.cardColors(containerColor = Color(0xFFF0F7FF)),
        border = BorderStroke(1.dp, Color(0xFFD5E6FF))
    ) {
        Row(...) {
            Box(modifier = Modifier.size(48.dp).background(Color(0xFF238b45), ...)) {
                Icon(Icons.Default.Person, ..., tint = Color.White)
            }
            Column {
                Text(session.userName, fontWeight = FontWeight.ExtraBold)
                Text(session.userEmail, ...)
                Text(session.roles.joinToString(", "), color = Color(0xFF238b45))
            }
        }
    }
}
```

### NavHost — No Changes Needed

`FundsManagerNavHost.kt` does NOT need to pass session info to SettingsScreen anymore — both Dashboard and Settings now read `SessionManager` directly from `LocalContext` using the same `remember { SessionManager(context.applicationContext) }` pattern.

## Key Pattern

For any `@Singleton` class with a simple `@Inject constructor(@ApplicationContext context: Context)`, reading it from a Composable:

```kotlin
val context = LocalContext.current
val instance = remember { MyClass(context.applicationContext) }
```

Works without Hilt injection infrastructure. Use `produceState` to observe flows.
