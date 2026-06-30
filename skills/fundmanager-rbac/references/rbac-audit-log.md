# RBAC Audit Trail

## Session 20260630_000100 — Budget Request Workflow Design

### Initial State
- 6 roles deployed (past sessions)
- Budget approval flow: simplified "SUPERVISOR → submit → OWNER approve"
- No multi-stage workflow, no pagu types, no SUPERVISOR edit capability
- 35 expense categories existed in form but not mapped to pagu system

### Changes Made

#### Fix 1: SUPERVISOR Edit Capability (Opsi B)
- Kordinator sekarang bisa **edit nominal, tambah item, hapus item** dari estimasi FE
- Audit trail: `estimated_amount` (FE) + `revised_amount` (SUP) keduanya tersimpan
- Sebelumnya: SUPERVISOR hanya submit ke Manager (Opsi A pass-through)

#### Fix 2: 7-Stage Budget Workflow
- Budget request bukan 2 form terpisah, tapi **1 form multi-stage**
- Stages: DRAFT → ESTIMASI → FORWARDED → APPROVED → REALISASI → VERIFIED → RECONCILED
- Stage 0: email task dari Kordinator sebagai origination

#### Fix 3: 3 Tipe Pagu — 35 Categories
- FIXED_PAGU (10): PAKET LK, HOTEL, VOUCHER, BURUH, BALLAST, TRANSPORT OJEK, TRANSPORT BENSIN, TRANSPORT LAINNYA, FEE
- TICKET (12): Semua tiket pesawat/ferry/bis/travel/kereta/lainnya — wajib bukti fisik
- MANAGER_APPROVAL (13): Taksi, Material, Lifting, Tarik Kabel, Tebang Pohon, Mounting, Subcont, dll — OWNER tentukan nominal

#### Fix 4: Per-Item Reconciliation
- ADMIN/FINANCE_MANAGER verifikasi per item (bukan per form)
- Tiket tanpa bukti: Finance tentukan nominal saat rekonsiliasi (bukan nominal klaim FE)
- `bill_verified` centang per item

#### Fix 5: Master Data Lokasi
- `master_locations` table untuk historis kunjungan
- OWNER lihat histori budget 5 kunjungan terakhir di lokasi yang sama saat approval
- Di-input oleh ADMIN/OWNER

#### Fix 6: FIELD_ENGINEER Budget Est. Permission
- FE sekarang bisa submit budget estimasi (sebelumnya matrix bilang "tidak bisa budget request")
- FE pilih job type di awal form (berkoordinasi dengan Kordinator via email)

### Files Changed
- `fundmanager-rbac/SKILL.md`: Budget approval flow, permission matrix, SUPERVISOR description
- `fundmanager-rbac/references/role-matrix.md`: Updated role columns + 7-stage table
- `fundmanager-rbac/references/budget-request-workflow.md`: NEW — full simulation, 35 categories, data model

### Open Questions
1. OWNER penolakan: stage balik ke DRAFT atau ada stage REJECTED?
2. Threshold SUPERVISOR approve mandiri (jika upgrade dari Opsi B ke C di masa depan)?
3. Job type authoritative source: email Kordinator atau form FE?

## Past Sessions

### Session 20260630 (Previous)
- RBAC fixes: expand project create, delete PIC+VIEWER, budget approval clarity
- 8 roles → 6 roles final
- 119 tests passed
