# hiltViewModel() Crash on Non-ViewModel — FundManager Settings Screen

Reproduction session: `20260630_000100_447baa`

## Problem

The Settings screen in FundManager V2 Android app force-closed immediately when the user navigated to it via the bottom navigation bar.

## Investigation

1. Looked at `FundsManagerNavHost.kt` for the `Screen.Settings` composable
2. Found line 182: `val sessionManager: SessionManager = hiltViewModel()`
3. Checked `SessionManager` class — it's a `@Singleton` with `@Inject constructor`, NOT extending `ViewModel`
4. `hiltViewModel()` only works with `ViewModel` subclasses → crash

## Fix

```kotlin
// BEFORE (crashes):
val sessionManager: SessionManager = hiltViewModel()

// AFTER (works):
val context = LocalContext.current
val sessionManager = remember { SessionManager(context.applicationContext) }
```

## Why This Works

`SessionManager` has a simple `@Inject constructor(@ApplicationContext context: Context)`. Since it only takes a Context, we can instantiate it directly in any Composable. The `remember` block ensures it's created once per composition, not on every recomposition.

## If the Class Had Complex Dependencies

If the injected class depended on other Hilt services (Dao, Repository, etc.), direct instantiation wouldn't work. In that case, either:
- Refactor the class into a proper `ViewModel` with `hiltViewModel()`
- Use `EntryPointAccessors.fromApplication()` to access the Hilt graph

But for simple singleton services that only need `Context`, direct instantiation is the cleanest fix.
