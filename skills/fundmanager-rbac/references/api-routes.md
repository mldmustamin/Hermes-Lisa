# Budget Request API Routes (Phase 4)

Built: 2026-06-30. See also `references/budget-request-workflow.md` for the full 7-stage simulation with per-role UI mockups, and `references/role-matrix.md` for the compact authorization matrix.

## Routes (21 endpoints)

```
GET    /api/v1/budget-templates                   BudgetItemTemplateController@index
GET    /api/v1/equipment-options                  MasterEquipmentOptionController@index
                                                  ?field_key=JENIS_ANTENNA (optional filter)
GET    /api/v1/projects/{project}/locations       MasterLocationController@index
POST   /api/v1/projects/{project}/locations       MasterLocationController@store
GET    /api/v1/locations/{location}               MasterLocationController@show
PUT    /api/v1/locations/{location}               MasterLocationController@update
DELETE /api/v1/locations/{location}               MasterLocationController@destroy
GET    /api/v1/locations/{location}/history       MasterLocationController@history
GET    /api/v1/task-expenses                      TaskExpenseController@index
POST   /api/v1/task-expenses                      TaskExpenseController@store
GET    /api/v1/task-expenses/{taskExpense}         TaskExpenseController@show
PUT    /api/v1/task-expenses/{taskExpense}         TaskExpenseController@update
DELETE /api/v1/task-expenses/{taskExpense}         TaskExpenseController@destroy
POST   /api/v1/task-expenses/{taskExpense}/submit  TaskExpenseController@submit
POST   /api/v1/task-expenses/{taskExpense}/forward TaskExpenseController@forward
POST   /api/v1/task-expenses/{taskExpense}/approve TaskExpenseController@approve
POST   /api/v1/task-expenses/{taskExpense}/reject  TaskExpenseController@reject
POST   /api/v1/task-expenses/{taskExpense}/realize TaskExpenseController@realize
POST   /api/v1/task-expenses/{taskExpense}/verify  TaskExpenseController@verify
POST   /api/v1/task-expenses/{taskExpense}/reconcile TaskExpenseController@reconcile
GET    /api/v1/task-expenses/{taskExpense}/histories TaskExpenseController@histories
```

## Stage Transition Authorization

| Transition | Allowed Role | From Stage | To Stage |
|-----------|-------------|------------|----------|
| submit | FIELD_ENGINEER (own form) | DRAFT | ESTIMASI |
| forward | SUPERVISOR | ESTIMASI | FORWARDED |
| approve | OWNER | FORWARDED | APPROVED |
| reject | SUPERVISOR or OWNER | ESTIMASI or FORWARDED | DRAFT |
| realize | FIELD_ENGINEER (own form) | APPROVED | REALISASI |
| verify | ADMIN or FINANCE_MANAGER | REALISASI | VERIFIED |
| reconcile | FINANCE_MANAGER | VERIFIED | RECONCILED |

## Rejection Cascade Rules

When any role rejects (SUPERVISOR or OWNER):
- Stage resets to DRAFT
- All SUPERVISOR revisions cleared (revised_amount → null)
- All OWNER approvals cleared (approved_amount → null)
- Item status reset to DRAFT
- rejection_reason stored for FE to see
- Audit history records who rejected and why

## Max Drafts Enforcement

- FIELD_ENGINEER max 5 drafts simultaneously (stage=DRAFT)
- Counted in TaskExpense::draftCountForUser()
- 422 error if exceeded: "Maksimal 5 draft. Harap hapus atau selesaikan draft yang ada."

## Database Tables Created (Phase 1)

| Table | Purpose | Key Fields |
|-------|---------|------------|
| budget_item_templates | 35 expense categories + pagu rules | category_group, pagu_type, pagu_amount, requires_bill |
| master_locations | Location master data | remote_name, address, provinsi, kota_kab |
| task_expenses | Single form, 7-stage workflow | task_no, vid, job_type, stage, submitted_by |
| expense_items | Items within task_expense | estimated_amount, revised_amount, approved_amount, realization_amount |
| task_expense_histories | Stage transition audit trail | action, old_stage, new_stage, actor_id |
| laporan_pekerjaan | Technical work report | perangkat info, parameters, timestamps |
| perangkat_terpasang | Installed equipment (VSAT/M2M) | SN numbers, equipment types |
| perangkat_rusak | Damaged/replaced equipment | Old SN numbers |
| laporan_pekerjaan_foto | Work photos (separate from kwitansi) | file_path, description |
| master_equipment_options | Dropdown options (one table) | field_key, label (antenna, mounting, modem, gangguan, foto) |
