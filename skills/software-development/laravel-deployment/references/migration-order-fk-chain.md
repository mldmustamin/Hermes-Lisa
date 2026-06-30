# Laravel Migration FK Chain Error — Reproduction & Fix

## Symptom

Running `php artisan migrate --force` fails with:

```
Illuminate\Database\QueryException
  SQLSTATE[42P01]: Undefined table: 7 ERROR:  relation "transactions" does not exist
  (Connection: pgsql, SQL: alter table "attachments" add constraint
  "attachments_transaction_id_foreign" foreign key ("transaction_id")
  references "transactions" ("id") on delete cascade)
```

## Root Cause

Multiple migration files share the same timestamp `YYYY_MM_DD_HHIISS`. Laravel runs them in alphabetical order, but the FK dependency chain is out of order.

**Example from FundManager V2:**

```
2026_06_27_142227_create_attachments_table.php     # FK → transactions (but runs FIRST!)
2026_06_27_142227_create_audit_events_table.php    # FK → users      OK
2026_06_27_142227_create_projects_table.php        # FK → users      OK
2026_06_27_142227_create_transactions_table.php    # FK → users, projects, accounts, categories
```

`attachments` (alphabetically first) tries to create FK to `transactions` before `transactions` exists.

## Dependency Graph Discovery

Search all migrations with the same timestamp for `foreign` or `constrained`:

```bash
for f in database/migrations/2026_06_27_142227_*.php; do
  echo "=== $(basename $f) ==="
  grep -n 'foreign\|constrained\|references' "$f"
done
```

## Fix

Rename files to assign unique timestamps in dependency order (base tables first, dependent tables later):

```bash
# Files with no FK to custom tables → keep or assign earliest
mv audit_events_table.php  2026_06_27_142226_create_audit_events_table.php
mv projects_table.php       2026_06_27_142227_create_projects_table.php

# Lookup tables (FK only to users) → middle
mv accounts_table.php       2026_06_27_142228_create_accounts_table.php
mv categories_table.php     2026_06_27_142228_create_categories_table.php

# Tables that reference multiple others → later
mv transactions_table.php   2026_06_27_142229_create_transactions_table.php

# Tables that reference the above → last
mv attachments_table.php    2026_06_27_142230_create_attachments_table.php
```

## Related: `transactions` FK to `accounts`/`categories`

Same pattern, different timestamp group:

```
2026_06_27_142227_create_transactions_table.php  # FK → accounts (but accounts is in 142228!)
2026_06_27_142228_create_accounts_table.php
```

`accounts` and `categories` must be created BEFORE `transactions`. Renumber transactions to `142229` and accounts/categories stay at `142228`.

## Verification

After fixing and resetting the DB:

```bash
su - postgres -c "psql -c \"DROP DATABASE myapp_production;\""
su - postgres -c "psql -c \"CREATE DATABASE myapp_production OWNER myapp;\""
php artisan migrate --force
# Expect: all migrations pass with "DONE" status, no errors
```
