# RBAC Authorization Audit — Post-Deploy Recipe

Trigger: user reports "logic app kacau", "seharusnya X role bisa/tidak bisa Y", or any role-permission confusion after deployment.

## Backend Audit

### Step 1 — Find all inline role checks

```bash
cd backend
grep -rn 'hasRole' app/Http/Controllers/ --include='*.php'
```

Example output:
```
app/Http/Controllers/Api/ProjectController.php:174:        if (! $user->hasRole(['OWNER', 'ADMIN'])) {
app/Http/Controllers/Api/TransactionController.php:447:        if (! $user->hasRole(['OWNER', 'ADMIN', 'FIELD_ENGINEER'])) {
app/Http/Controllers/Api/SyncPushController.php:369:        if ($user->hasRole(['OWNER', 'ADMIN'])) {
```

### Step 2 — For each match, compare against business requirements

| Controller method | Current allowed roles | Business requirement | Gap? |
|---|---|---|---|
| `ProjectController::authorizeCreate()` | OWNER, ADMIN | OWNER, ADMIN, FINANCE_MANAGER, SUPERVISOR | Missing FINANCE_MANAGER + SUPERVISOR |
| `TransactionController::authorizeCreate()` | OWNER, ADMIN, FIELD_ENGINEER | FIELD_ENGINEER should create transactions | ✅ OK |
| `SyncPushController::isAssignedOrAdmin()` | OWNER, ADMIN | Admin roles bypass assignment | ✅ OK |

### Step 3 — Fix gaps

```php
// Too restrictive → add roles
if (! $user->hasRole(['OWNER', 'ADMIN', 'FINANCE_MANAGER', 'SUPERVISOR'])) {

// Too permissive → remove roles
if (! $user->hasRole(['OWNER', 'ADMIN'])) {  // was: ['OWNER', 'ADMIN', 'FIELD_ENGINEER']
```

### Step 4 — Rebuild caches & re-test

```bash
php artisan config:cache && php artisan route:cache && php artisan view:cache
php artisan test
```

---

## Android Cross-Check

For every backend role-restricted endpoint, verify the Android app hides the corresponding UI:

### Pattern — role-based UI gating

**1. UiState data class — add boolean flag:**
```kotlin
data class ProjectListUiState(
    val projects: List<ProjectListItem> = emptyList(),
    val canCreateProject: Boolean = false,
    // ...
)
```

**2. ViewModel — observe session roles:**
```kotlin
private fun observeCanCreateProject() {
    viewModelScope.launch {
        sessionManager.activeSession.collectLatest { session ->
            val roles = session?.roles ?: emptyList()
            val canCreate = roles.any { it in listOf("OWNER", "ADMIN", "FINANCE_MANAGER", "SUPERVISOR") }
            _uiState.update { it.copy(canCreateProject = canCreate) }
        }
    }
}
```

**3. Screen composable — conditional rendering:**
```kotlin
if (uiState.canCreateProject) {
    Button(onClick = { showAddDialog = true }) {
        Icon(Icons.Default.Add, ...)
        Text("Project")
    }
}
```

### Find all Android UI elements that trigger restricted actions

```bash
grep -rn 'insertProject\|createProject\|addProject' app/src/main/java/ --include='*.kt'
grep -rn 'deleteProject\|approve\|reject\|void' app/src/main/java/ --include='*.kt'
```

For each match, verify the ViewModel has role observation and the Screen has conditional rendering.

---

## Smoke Test Role Boundaries

```bash
# Login as each test role
ADMIN_TOKEN=$(curl -s -X POST :8080/api/v1/auth/login \
  -d '{"login":"10001","password":"admin"}' | jq -r .access_token)

FM_TOKEN=$(curl -s -X POST :8080/api/v1/auth/login \
  -d '{"login":"10002","password":"password123"}' | jq -r .access_token)

FE_TOKEN=$(curl -s -X POST :8080/api/v1/auth/login \
  -d '{"login":"10003","password":"password123"}' | jq -r .access_token)

# Test project creation boundary
for token in "$ADMIN_TOKEN" "$FM_TOKEN" "$FE_TOKEN"; do
  echo "---"
  curl -s -X POST :8080/api/v1/projects \
    -H "Authorization: Bearer $token" \
    -d '{"name":"test"}' -w "\nHTTP: %{http_code}"
done
# Expected: ADMIN=201, FM=201, FE=422
```

## FundManager V2 Role Matrix (Reference)

| Role | Create Project | Create Transaction | Approve | Void | Close Period |
|------|:---:|:---:|:---:|:---:|:---:|
| OWNER | ✅ | ✅ | ✅ | ✅ | ✅ |
| ADMIN | ✅ | ✅ | ✅ | ❌ | ❌ |
| FINANCE_MANAGER | ✅ | ✅ | ✅ | ✅ | ✅ |
| SUPERVISOR | ✅ | ✅ | ✅ | ❌ | ❌ |
| PIC | ❌ | ✅ | ❌ | ❌ | ❌ |
| FIELD_ENGINEER | ❌ | ✅ | ❌ | ❌ | ❌ |
| AUDITOR | ❌ | ❌ | ❌ | ❌ | ❌ |
| VIEWER | ❌ | ❌ | ❌ | ❌ | ❌ |
