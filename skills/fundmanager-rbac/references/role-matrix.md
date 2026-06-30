# FundManager V2 — Compact Role Matrix

## Roles

| # | Role | Nama | Budget Approve? | Create Project? | Create Tx? | Submit Est.? | Edit Est.? |
|---|------|------|:---:|:---:|:---:|:---:|:---:|
| 1 | OWNER | Manager | ✅ (only) | ✅ | ✅ | ❌ | ❌ |
| 2 | ADMIN | Admin | ❌ | ✅ | ✅ | ❌ | ❌ |
| 3 | FINANCE_MANAGER | Finance | ❌ | ✅ | ✅ | ❌ | ❌ |
| 4 | SUPERVISOR | Kordinator | ❌ | ✅ | ✅ | ❌ | ✅ (edit+forward) |
| 5 | FIELD_ENGINEER | Field | ❌ | ❌ | ✅ | ✅ (submit) | ❌ |
| 6 | AUDITOR | Auditor | ❌ | ❌ | ❌ | ❌ | ❌ |

## Key Rules

1. **Budget approval = OWNER only.** No exceptions.
2. **Budget estimasi = FIELD_ENGINEER submit → SUPERVISOR edit + forward → OWNER approve.**
3. **Reconciliation = FINANCE_MANAGER + ADMIN** (per-item verification).
4. **Rekonsiliasi final = FINANCE_MANAGER** (crosscheck Kordinator, tentukan nominal tiket tanpa bukti).
5. **Project creation = OWNER, ADMIN, FINANCE_MANAGER, SUPERVISOR.**
6. **Transaction creation = FIELD_ENGINEER, SUPERVISOR** (mobile). Everyone via API.
7. **Master data lokasi = ADMIN, SUPERVISOR.**
8. **AUDITOR = read-only everything.**
9. **Verifikasi realisasi = ADMIN, FINANCE_MANAGER.**
10. **Rekonsiliasi final = FINANCE_MANAGER.**
11. **Task assignment (assign FE ke task) = SUPERVISOR.**
12. **Laporan pekerjaan (teknis) = FE input, SUPERVISOR verifikasi.**
13. **Config parameters** in `config/budget.php`: BUDGET_MAX_DRAFTS=5, BUDGET_PAGINATION=20, BUDGET_HISTORY_LIMIT=10.

## 7-Stage Budget Workflow (1 Form)

| Stage | Actor | Action |
|-------|-------|--------|
| DRAFT | FE | Isi form estimasi (lokasi master data, job type, item + nominal) |
| ESTIMASI | FE→SUP | Submit ke inbox Kordinator |
| FORWARDED | SUP | Review, edit nominal/item, forward ke OWNER |
| APPROVED | OWNER | Lihat historis lokasi, approve + final nominal |
| REALISASI | FE | Input realisasi per item + upload bukti |
| VERIFIED | ADMIN/FM | Cek bukti per item, flag issue |
| RECONCILED | FM | Crosscheck Kordinator, tentukan tiket tanpa bukti → final |
