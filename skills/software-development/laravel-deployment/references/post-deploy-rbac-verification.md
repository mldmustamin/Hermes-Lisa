# Post-Deploy RBAC Verification

After running migrations + seeders, verify that roles are correctly deployed and authorization works as expected.

## Verify Role Count and Names

```bash
cd backend
php artisan tinker --execute="echo implode(', ', Spatie\Permission\Models\Role::pluck('name')->toArray());"
```

Expected output for FundManager V2:
```
OWNER, ADMIN, FINANCE_MANAGER, SUPERVISOR, FIELD_ENGINEER, AUDITOR
```

## Smoke Test Authorization

```bash
# 1. Login as Finance Manager
TOKEN=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"login":"10002","password":"password123"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# 2. Allowed action (should 201)
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:8080/api/v1/projects \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"RBAC Test"}'
# Expected: 201

# 3. Blocked action (should 422)
TOKEN_FE=$(curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"login":"10003","password":"password123"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:8080/api/v1/projects \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $TOKEN_FE" \
  -d '{"name":"Should Fail"}'
# Expected: 422
```

## Run RBAC-Related Tests

```bash
php artisan test --filter=Project
php artisan test --filter=Transaction
php artisan test --filter=Approval
php artisan test --filter=Correction
```

All should pass. If any fail with "role" validation errors, check the authorization methods in the relevant controller.
