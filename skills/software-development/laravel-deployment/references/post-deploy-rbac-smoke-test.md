# Post-Deploy RBAC Smoke Test

After every production/staging deployment, verify RBAC is working by testing API endpoints as different roles.

## Test Pattern

For each role, test an action they SHOULD be able to do (expect 200/201) and an action they SHOULD NOT be able to do (expect 403/422).

```bash
BASE="http://localhost:8080"

# 1) Finance Manager: should CREATE project
TOKEN=$(curl -s -X POST $BASE/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"login":"10002","password":"password123"}' | jq -r '.access_token')

HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST $BASE/api/v1/projects \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"name":"RBAC Test"}')
echo "FM create project: $HTTP (expected 201)"

# 2) Field Engineer: should NOT create project
TOKEN=$(curl -s -X POST $BASE/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"login":"10003","password":"password123"}' | jq -r '.access_token')

HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST $BASE/api/v1/projects \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"name":"Should Fail"}')
echo "FE create project: $HTTP (expected 422)"

# 3) Field Engineer: SHOULD create transaction
PROJ_UUID=$(curl -s $BASE/api/v1/projects \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Accept: application/json' | jq -r '.projects[0].uuid')

HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
  -X POST $BASE/api/v1/transactions \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{\"type\":\"FUND_IN\",\"project_uuid\":\"$PROJ_UUID\",\"date\":\"$(date +%F)\",\"reported_amount\":100000,\"real_amount\":100000}")
echo "FE create tx: $HTTP (expected 201)"
```

## Quick Matrix to Verify

| Role | Create Project | Create Transaction | Approve Tx | Close Period |
|------|:---:|:---:|:---:|:---:|
| OWNER | 201 | 201 | 200 | 200 |
| ADMIN | 201 | 201 | 422 | 422 |
| FINANCE_MANAGER | 201 | 201 | 200 | 200 |
| SUPERVISOR | 201 | 201 | 200 (assigned) | 422 |
| FIELD_ENGINEER | 422 | 201 | 422 | 422 |
| AUDITOR | 422 | 422 | 422 | 422 |

## Role Cleanup After DB Reset

If roles are deleted from `RolePermissionSeeder`, the database must be dropped and recreated — existing roles in the `roles` table persist across re-seeds:

```bash
su - postgres -c "psql -c 'DROP DATABASE mydb;'"
su - postgres -c "psql -c 'CREATE DATABASE mydb OWNER myuser;'"
php artisan migrate --force
php artisan db:seed --class=StagingSeeder --force
```

Verify only expected roles exist:
```bash
php artisan tinker --execute="echo implode(', ', Spatie\Permission\Models\Role::pluck('name')->toArray());"
```
