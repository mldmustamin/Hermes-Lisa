---
name: fundmanager-rbac
category: software-development
description: FundManager V2 RBAC — 6-role authorization matrix, budget approval flow, permission enforcement patterns, and how to audit/modify RBAC across backend + Android + tests + docs.
---

# FundManager V2 — Role-Based Access Control

Class-level skill for the FundManager V2 authorization model. Covers the 6-role structure, business rules for budget approval vs reconciliation, permission enforcement patterns (backend + Android), and the systematic method for auditing and modifying RBAC across the full stack.

## When to Load

- User says "perbaiki role", "fix authorization", "RBAC", "siapa yang bisa X", "field engineer tidak bisa Y"
- User asks about budget approval flow or reconciliation authority
- User says "tambah role", "hapus role", "ganti permission role"
- User reports authorization bugs (403/422 errors, user blocked from allowed action, or user allowed to do restricted action)
- Implementing new features that need role-based access (budget request, approval, reconciliation)
- User asks about budget request workflow, estimasi, realisasi, pagu, expense categories, or SUPERVISOR edit capability
- User wants to simulate or design the 7-stage budget workflow

## Reference Files

- `references/role-matrix.md` — Compact 6-role matrix with 7-stage workflow summary
- `references/budget-request-workflow.md` — Full simulation: 35 expense categories, 3 pagu types, 7-stage walkthrough, data model, open questions
- `references/rbac-audit-log.md` — Session-by-session audit trail of all RBAC changes

---

## 6 Roles (Final)

| Role | Name | Core Authority |
|------|------|---------------|
| **OWNER** | Manager | Full control. Satu-satunya yang bisa **approve budget request + set nominal**. Final decision maker. |
| **ADMIN** | Admin | User/device/project admin. **Rekonsiliasi data realisasi** (bersama Finance Manager). Create project. |
| **FINANCE_MANAGER** | Finance | **Pencocokan data realisasi penggunaan dana ke kordinator.** Closing, reconciliation, approve tx harian. Create project. |
| **SUPERVISOR** | Kordinator | **Review + edit estimasi FE, forward ke Manager.** Approve/reject transaksi tim. Create project. |
| **FIELD_ENGINEER** | Field | Mobile capture transaksi/estimasi. **Submit budget estimasi ke Kordinator.** Assigned projects only. **Tidak bisa** create project atau approve. |
| **AUDITOR** | Auditor | Read-only audit dan laporan. Tidak bisa transaksi atau approval. |

---

## Business Rules (CRITICAL — Jangan Dikacaukan)

### Budget Request — 7-Stage Workflow

Satu form (`task_expenses`) mengalir melalui 7 stage:

```
STAGE 0:   Kordinator kirim email task (task_no, VID, job_type)
             ↓
STAGE 1:   DRAFT             FE isi form estimasi (pilih lokasi master data, pilih job type, item-item)
             ↓                 Pagu enforcement: FIXED dicek, TICKET warning wajib bukti, MANAGER_APPROVAL flag
STAGE 2:   ESTIMASI          FE submit → masuk inbox SUPERVISOR
             ↓
STAGE 3:   FORWARDED         SUPERVISOR review + edit (bisa revisi nominal, tambah/hapus item) → forward ke OWNER
             ↓                 Audit trail: estimated_amount (FE) + revised_amount (SUP) tercatat
STAGE 4:   APPROVED          OWNER lihat historis lokasi (master_locations) → approve + final nominal per item
             ↓                 OWNER only — tidak boleh FINANCE_MANAGER atau ADMIN
           ─── PEKERJAAN ───
             ↓
STAGE 5:   REALISASI         FE input realisasi per item + upload bukti kwitansi
             ↓
STAGE 6:   VERIFIED          ADMIN/FINANCE_MANAGER cek bukti per item, cek nominal, flag issue
             ↓
STAGE 7:   RECONCILED        FINANCE_MANAGER crosscheck Kordinator + tentukan tiket tanpa bukti → final close
```

**PITFALL:** Budget approval adalah wewenang eksklusif **OWNER**. FINANCE_MANAGER dan ADMIN **tidak boleh** approve budget — mereka hanya mencocokkan data realisasi dan rekonsiliasi.

**SUPERVISOR edit capability:** Kordinator boleh **edit nominal, tambah item, hapus item** dari estimasi FE sebelum forward ke OWNER. Audit trail menyimpan nilai asli FE (estimated_amount) dan revisi SUP (revised_amount).

### 3 Tipe Pagu — 35 Expense Categories

| Tipe | Jml | Contoh | Rules |
|------|-----|--------|-------|
| **FIXED_PAGU** | 10 | PAKET LK, HOTEL, VOUCHER, BURUH, BALLAST, FEE, OJEK | Nominal absolut dari tabel pagu. > pagu → BLOCK (kecuali HOTEL: boleh lebih dgn bill asli) |
| **TICKET** | 12 | Semua tiket pesawat/ferry/bis/travel/kereta/lainnya | Wajib bukti fisik resmi, tidak boleh dicoret. Tanpa bukti → Finance tentukan nominal saat rekonsiliasi |
| **MANAGER_APPROVAL** | 13 | Taksi, Material, Lifting, Tarik Kabel, Tebang Pohon, Mounting, Subcont, dll | Tidak ada pagu. OWNER tentukan nominal final saat approval |

Lihat `references/budget-request-workflow.md` untuk daftar lengkap 35 kategori, simulasi per-stage, dan aturan detail.

### Reconciliation Flow (per-item)
```
FIELD_ENGINEER     → input realisasi per item + upload bukti
ADMIN/FINANCE      → verifikasi per item: cek bukti (tiket wajib resmi), cek nominal vs approved
FINANCE_MANAGER    → crosscheck ke Kordinator per item
                   → tentukan nominal tiket tanpa bukti
                   → final rekonsiliasi
```

### Transaction Approval Flow
```
FIELD_ENGINEER/SUPERVISOR → submit transaksi (DRAFT → PENDING)
FINANCE_MANAGER/SUPERVISOR → approve/reject transaksi harian tim
FINANCE_MANAGER            → void / correction approved transaction
```

---

## Permission Matrix

| Action | Allowed Roles |
|--------|--------------|
| Create transaction (mobile) | FIELD_ENGINEER, SUPERVISOR (+ project assignment) |
| Submit budget estimasi | FIELD_ENGINEER (+ project assignment) |
| Edit budget estimasi + forward | SUPERVISOR (+ project assignment) |
| **Approve budget + set nominal** | **OWNER only** |
| Approve transaction | FINANCE_MANAGER, SUPERVISOR |
| Void / correction | FINANCE_MANAGER |
| Close period | FINANCE_MANAGER, OWNER |
| Verifikasi realisasi | FINANCE_MANAGER, ADMIN |
| Rekonsiliasi final | FINANCE_MANAGER (crosscheck Kordinator) |
| Create project | OWNER, ADMIN, FINANCE_MANAGER, SUPERVISOR |
| Input master data lokasi (CRUD) | ADMIN, SUPERVISOR |
| Create task assignment (via app) | SUPERVISOR |
| User CRUD | ADMIN, OWNER |
| Device revoke | ADMIN, OWNER |
| Project assignment | ADMIN |
| View audit trail | AUDITOR, FINANCE_MANAGER, ADMIN, OWNER |

---

## Backend Enforcement Pattern

### Controller Authorization (Laravel + Spatie)

```php
private function authorizeCreate(Request $request): void
{
    $user = $request->user();
    if (! $user->hasRole(['OWNER', 'ADMIN', 'FINANCE_MANAGER', 'SUPERVISOR'])) {
        throw ValidationException::withMessages([
            'role' => ['Only OWNER, ADMIN, FINANCE_MANAGER, or SUPERVISOR can create or update projects.'],
        ]);
    }
}
```

**Pattern:** Call `$this->authorizeXxx($request)` at the top of sensitive controller methods. Use `$user->hasRole(array)` for multi-role checks. Throw `ValidationException` with descriptive message (not 403 — mobile clients need the error message in JSON).

### Sync Push Authorization

`SyncPushController` uses `isAssignedOrAdmin()` for project-level access. OWNER and ADMIN bypass project assignment checks. Other roles must have a `project_assignments` row.

```php
private function isAssignedOrAdmin($user, int $projectId): bool
{
    if ($user->hasRole(['OWNER', 'ADMIN'])) {
        return true;
    }
    return ProjectAssignment::where('project_id', $projectId)
        ->where('user_id', $user->id)
        ->exists();
}
```

---

## Android RBAC — Role-Based UI Gating

Use ONE app with role-based UI. The `SessionManager` stores roles from login response (`ActiveSession.roles: List<String>`).

### Pattern (Kotlin/Compose)

**Step 1:** Add a boolean flag to `UiState`:
```kotlin
data class ProjectListUiState(
    val projects: List<ProjectListItem> = emptyList(),
    val showArchived: Boolean = false,
    val error: String? = null,
    val canCreateProject: Boolean = false  // ← RBAC flag
)
```

**Step 2:** Observe session roles in ViewModel:
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

**Step 3:** Conditional UI rendering:
```kotlin
if (uiState.canCreateProject) {
    Button(onClick = { showAddDialog = true }) {
        Icon(Icons.Default.Add, ...)
        Text("Project")
    }
}
```

**Important:** Backend is authoritative. Android UI gating is convenience only — even if bypassed, server rejects unauthorized actions.

---

## Method: Auditing & Modifying RBAC

When modifying RBAC (add/remove roles, change permissions), use this systematic approach:

### Step 1 — Audit All References
```bash
# Find all role references in code
grep -r "ROLE_NAME" backend/ --include="*.php"

# Check docs
grep -r "ROLE_NAME" docs/ --include="*.md"

# Check PRD and Q&A
grep -r "ROLE_NAME" PRD.md OPEN_QNA.md
```

### Step 2 — Fix in Order
1. **RolePermissionSeeder.php** — add/remove roles from array
2. **Controller authorization methods** — update hasRole() arrays
3. **Tests** — update role assignments in test factories
4. **Docs** — update role tables, permission matrices, Q&A entries
5. **PRD.md / OPEN_QNA.md** — update role counts and descriptions

### Step 3 — Deploy
```bash
# Fresh deploy (recommended for role changes)
su - postgres -c "psql -c 'DROP DATABASE fundsmanager_production;'"
su - postgres -c "psql -c 'CREATE DATABASE fundsmanager_production OWNER fundsmanager;'"
php artisan migrate --force
php artisan db:seed --class=StagingSeeder --force

# Verify
php artisan tinker --execute="echo implode(', ', Spatie\Permission\Models\Role::pluck('name')->toArray());"

# Test
php artisan test

# Cache
php artisan config:cache && php artisan route:cache && php artisan view:cache
```

### Step 4 — Verify with Real API Calls
```bash
# Login as each role, test allowed + blocked actions
curl -X POST .../api/v1/auth/login -d '{"login":"10002","password":"password123"}'
curl -X POST .../api/v1/projects -H "Authorization: Bearer TOKEN" -d '{"name":"test"}'
# Expected: FINANCE_MANAGER → 201, FIELD_ENGINEER → 422
```

---

## Staging Test Credentials

| User | Login | Password | Role |
|------|-------|----------|------|
| Super Admin | `admin@fundsmanager.test` or `10001` | `admin` | OWNER |
| Finance | `finance@fundsmanager.test` or `10002` | `password123` | FINANCE_MANAGER |
| Engineer | `engineer@fundsmanager.test` or `10003` | `password123` | FIELD_ENGINEER |
| Supervisor | `supervisor@fundsmanager.test` or `10004` | `password123` | SUPERVISOR |
| Auditor | `auditor@fundsmanager.test` or `10005` | `password123` | AUDITOR |

**Note:** All 6 roles have staging users. Engineer has `password_change_required = true` — login will force password change.

---

## Pitfalls

| Issue | Symptom | Fix |
|-------|---------|-----|
| Too-restrictive project creation | Only OWNER+ADMIN can create — FINANCE_MANAGER blocked | Add FINANCE_MANAGER + SUPERVISOR to authorizeCreate() |
| Budget approver confusion | FINANCE_MANAGER coded as budget approver | OWNER is the ONLY budget approver. FINANCE_MANAGER does reconciliation only. |
| Android UI doesn't hide actions | All roles see "Add Project" button | Add canCreateProject to UiState, observe roles from SessionManager |
| Tests use deleted role | `$user->assignRole('VIEWER')` after VIEWER removed | Replace with AUDITOR for read-only tests |
| RolePermissionSeeder has stale roles | Old roles exist in DB after seed | Fresh DB deploy: drop, recreate, migrate, seed |
| Sync push authorization gap | FIELD_ENGINEER creates project locally, syncs to server | Backend enforces hasRole() at API level. Only trust server-side enforcement. |
| Max 5 drafts exceeded in staging | FIELD_ENGINEER test hits draft limit | Delete test drafts or use DB::table('task_expenses')->where('submitted_by', $userId)->delete() between tests |
| Stale staging credentials in docs | Docs say "only 3 users" but 5 exist | All 6 roles have staging users. Check current StagingSeeder for credentials. |
| Master location CRUD not in matrix | SUPERVISOR can't create locations via API | ADMIN + SUPERVISOR both allowed for master_locations CRUD |
| Hardcoded values scattered | Max drafts, pagination, history limit, rejectable stages hardcoded in controllers | Centralize in config/budget.php. Use config() helper in all controllers. |
| Pagu amounts flat (wrong per job_type) | VOUCHER shows 15rb for all job types, but DISMANTLE/SURVEY should be 5rb | Use pagu_job_type_amounts pivot table. Seed with per-job_type mapping. |
| Stale reference docs after schema change | budget-request-workflow.md missing new tables (pagu_job_type_amounts, config/budget.php) | After every DB schema change, update fundmanager-rbac reference files.
