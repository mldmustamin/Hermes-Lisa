# Laravel + Android Code Review — Critical Checklist

From: FundManager V2 session 2026-06-30 — 22 issues found, 17 fixed.

## Authorization (FAIL if any found)

- [ ] Every `store()` method has role check (`hasRole('FIELD_ENGINEER')`)
- [ ] Every `forward()` has `hasRole('SUPERVISOR')`
- [ ] Every `approve()` has `hasRole('OWNER')`
- [ ] Every `verify()` has `hasRole('ADMIN') || hasRole('FINANCE_MANAGER')`
- [ ] Every `reconcile()` has `hasRole('FINANCE_MANAGER')`
- [ ] Every `destroy()` on write endpoints (MasterLocation: ADMIN or SUPERVISOR)

## Business Logic (FAIL if any found)

- [ ] Pagu enforcement: FIXED_PAGU items blocked at 422 with violations array
- [ ] Race condition: stage transitions use optimistic locking (`WHERE stage = expected`)
- [ ] Notes appended, not overwritten (SUPERVISOR notes additive to FE notes)
- [ ] verify() items validation is `required`, not `nullable`
- [ ] No dead code (computed variables that are never consumed)

## Data Integrity

- [ ] N+1 queries fixed: `with('template')` before loops accessing `$item->template`
- [ ] DB::transaction() wraps all multi-step mutations
- [ ] Totals recalculated after item changes (`recalculateTotals()`)
- [ ] SoftDeletes consistent across parent/child tables

## Tests (Should Exist)

- [ ] Unauthorized stage transition test (403 Forbidden)
- [ ] Pagu exceeded test (422 with violations array)
- [ ] Hotel pagu exceed test (201 with warnings array)
- [ ] Non-role user cannot create (403)

## Code Quality (Non-blocking)

- [ ] Controller under 500 lines (if not, note for future refactor)
- [ ] No duplicated query patterns (extract to helper method)
- [ ] Values from config, not hardcoded (`config('budget.xxx')`)

## Post-Review Documentation

After code review fixes:
- [ ] Update OPEN_QNA: any new business rule behavior gets a Q&A entry
- [ ] Update ACTION_LOG: entry with issue count, files changed, tests updated
- [ ] Update Obsidian vault _index.md if schema or routes changed
- [ ] Re-run all tests, verify pass count unchanged or increased
