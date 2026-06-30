# Android RBAC UI Gating Pattern

## When to Use

Any Android app where different user roles see different UI elements (buttons, menus, screens). The backend enforces authorization at the API level; the Android app should hide unauthorized UI to prevent confusion.

## Architecture

```
Login API response
  └─ roles: ["FIELD_ENGINEER"]
       │
       ▼
  SessionManager.saveSession(..., roles)
       │
       ▼
  DataStore (persisted as comma-separated)
       │
       ▼
  ViewModel observes activeSession.roles
       │
       ▼
  UiState has boolean flags (e.g., canCreateProject)
       │
       ▼
  Composable conditionally renders UI
```

## Step-by-Step Implementation

### 1. SessionManager stores roles

Already done if `ActiveSession` has `roles: List<String>` and `SessionManager.saveSession()` stores them.

```kotlin
data class ActiveSession(
    val userId: Long,
    val userName: String,
    val userEmail: String,
    val userUuid: String,
    val roles: List<String>,  // ← this
    val accessToken: String,
    val deviceUuid: String
)
```

Storage in DataStore:
```kotlin
private object Keys {
    val ROLES = stringPreferencesKey("roles") // JSON array
}

// Save: roles.joinToString(",")
// Read: prefs[Keys.ROLES]?.split(",")?.filter { it.isNotBlank() }
```

### 2. LoginViewModel or AuthRepository saves roles after login

```kotlin
// In LoginViewModel or wherever login response is handled:
sessionManager.saveSession(
    userId = response.user.id,
    userName = response.user.name,
    userEmail = response.user.email,
    userUuid = response.user.uuid,
    roles = response.user.roles,  // ← from API response
    accessToken = response.accessToken,
    deviceUuid = deviceUuid
)
```

### 3. UiState gets boolean flags

```kotlin
data class ProjectListUiState(
    val projects: List<ProjectListItem> = emptyList(),
    val showArchived: Boolean = false,
    val error: String? = null,
    val canCreateProject: Boolean = false  // ← role-gated flag
)
```

### 4. ViewModel observes roles and sets flags

```kotlin
@HiltViewModel
class ProjectListViewModel @Inject constructor(
    private val repository: FundsRepository,
    private val sessionManager: SessionManager,
    private val appLogger: AppLogger
) : ViewModel() {
    private val _uiState = MutableStateFlow(ProjectListUiState())
    val uiState: StateFlow<ProjectListUiState> = _uiState.asStateFlow()

    init {
        observeProjects()
        observeCanCreateProject()  // ← observe role
    }

    private fun observeCanCreateProject() {
        viewModelScope.launch {
            sessionManager.activeSession.collectLatest { session ->
                val roles = session?.roles ?: emptyList()
                val canCreate = roles.any {
                    it in listOf("OWNER", "ADMIN", "FINANCE_MANAGER", "SUPERVISOR")
                }
                _uiState.update { it.copy(canCreateProject = canCreate) }
            }
        }
    }
}
```

### 5. Composable conditionally renders

```kotlin
@Composable
fun ProjectListScreen(
    viewModel: ProjectListViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // Conditional button:
    if (uiState.canCreateProject) {
        Button(onClick = { showAddDialog = true }) {
            Icon(Icons.Default.Add, ...)
            Text("Project")
        }
    }

    // Conditional empty state action:
    EmptyProjectState(
        onCreateClick = { showAddDialog = true },
        canCreate = uiState.canCreateProject  // ← nullable action
    )
}
```

### 6. Backend still enforces (defense in depth)

```php
// ProjectController.php
private function authorizeCreate(Request $request): void
{
    $user = $request->user();
    if (! $user->hasRole(['OWNER', 'ADMIN', 'FINANCE_MANAGER', 'SUPERVISOR'])) {
        throw ValidationException::withMessages([
            'role' => ['Only OWNER, ADMIN, FINANCE_MANAGER, or SUPERVISOR can create projects.'],
        ]);
    }
}
```

## Common Pitfall: UI-only gating

Android UI gating is cosmetic. If the backend doesn't enforce authorization, anyone can craft an API call directly with curl/Postman. Always pair Android UI gating with backend `hasRole()` checks.

## Common Pitfall: Role mismatch between Android and Backend

Android uses roles from login response. If the backend role list changes (e.g., PIC and VIEWER deleted), old sessions with stale roles will still show old UI until the user logs out and back in. After role changes, either:
- Force logout all users
- Or validate roles on each API call (backend already does this)

## Role Matrix Example (FundManager V2)

| Role | Create Project | Create Transaction | Approve | Budget Request |
|------|:---:|:---:|:---:|:---:|
| OWNER | ✅ | ✅ | ✅ | ✅ |
| ADMIN | ✅ | ✅ | ✅ | ✅ |
| FINANCE_MANAGER | ✅ | ✅ | ✅ | ✅ |
| SUPERVISOR | ✅ | ✅ | assigned only | ✅ |
| FIELD_ENGINEER | ❌ | ✅ | ❌ | ❌ |
| AUDITOR | ❌ | ❌ | ❌ | ❌ |
