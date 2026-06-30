# Full System Test Workflow

After completing a feature or before declaring done, run this 4-stage test pipeline:

## 1. DRY TEST — Does everything compile + pass unit/feature tests?
```bash
# Backend
php artisan test

# Android
./gradlew assembleDebug
```

## 2. SMOKE TEST — Do all API endpoints respond correctly?
- Get tokens for every role (OWNER, FM, FE, SUP, AUDITOR)
- Hit every endpoint category (health, templates, locations, tasks)
- Verify response shape matches expectations

## 3. INTEGRATION TEST — Does the full workflow work end-to-end?
- Run the complete 7-stage workflow via API calls:
  1. FE creates DRAFT
  2. FE submits → ESTIMASI
  3. SUP forwards → FORWARDED
  4. OWNER approves → APPROVED
  5. FE realizes → REALISASI
  6. FM verifies → VERIFIED
  7. FM reconciles → RECONCILED
- Verify audit trail has correct entries
- Verify totals tracking across all stages

## 4. SYSTEM TEST — Does everything work together?
- Health check (public + localhost)
- APK file exists + downloadable
- Database accessible
- Routes registered
- Git status clean
- Crash reporting endpoint

## Report format
```markdown
## FULL TEST REPORT

### 1. DRY TEST
| Suite | Results |
|-------|---------|
| Backend All Tests | N passed, M assertions |
| Android Build | BUILD SUCCESSFUL |

### 2. SMOKE TEST
| Endpoint | Status |
|----------|--------|

### 3. INTEGRATION TEST
| Stage | Result |
|-------|--------|

### 4. SYSTEM TEST
| Check | Result |
|-------|--------|

### Summary
Overall: XX%
```
