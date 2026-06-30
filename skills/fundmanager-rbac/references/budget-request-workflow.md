# FundManager V2 — Budget Request Workflow (Full Simulation)

> Session 20260630_000100: Designed with user (FIELD_ENGINEER perspective). 7-stage single-form workflow, 35 expense categories with 3 pagu types, per-item reconciliation.

---

## 35 Expense Categories — 3 Pagu Types

### FIXED_PAGU (10 items — nominal absolut, bervariasi per job type)

| # | Kategori | Pagu Table | Nominal (per job type) |
|---|----------|-----------|------------------------|
| 1 | Akomodasi Paket (Luar Homebase) | PAKET LK | 120.000 (INSTALASI/RELOKASI/PMCM/DISMANTLE/SURVEY) |
| 2 | Akomodasi Hotel (Luar Homebase) | HOTEL LK | 200.000 / 275.000 (Papua) |
| 3 | Biaya Lain-Voucher | VOUCHER HP | 15.000 (INSTALASI/RELOKASI/PMCM), 5.000 (DISMANTLE/SURVEY) |
| 4 | Biaya Lain-Buruh | BURUH | 120.000 (INSTALASI/RELOKASI) — koordinasi Manager. PMCM/DISMANTLE/SURVEY: - |
| 5 | Biaya Lain-Ballast/Pondasi | BALLAST/COR | 200.000 (INSTALASI/RELOKASI). PMCM/DISMANTLE/SURVEY: - |
| 6 | Transport Ojek Berangkat | TRANSPORT HB | Sesuai Gojek / bensin 20rb/hari |
| 7 | Transport Ojek Pulang | TRANSPORT HB | Sesuai Gojek / bensin 20rb/hari |
| 8 | Transport Bensin | TRANSPORT HB | 20.000/hari |
| 9 | Transport Lainnya | TRANSPORT LK | Travel/Bus/Kapal |
| 10 | Biaya Lain-Fee Pekerjaan | FEE PEKERJAAN | 40.000 (INSTALASI), 75.000 (RELOKASI), 15.000 (PMCM/DISMANTLE), - (SURVEY) |

**Pagu enforcement:** FE input > pagu → BLOCK (tidak bisa submit). KECUALI HOTEL: boleh > pagu tapi wajib bill asli.

### TICKET (12 items — wajib bukti fisik, tidak boleh dicoret)

| # | Kategori |
|---|----------|
| 1 | Tiket Pesawat Berangkat |
| 2 | Tiket Pesawat Pulang |
| 3 | Tiket Ferry Berangkat |
| 4 | Tiket Ferry Pulang |
| 5 | Tiket Bis Berangkat |
| 6 | Tiket Bis Pulang |
| 7 | Tiket Travel Berangkat |
| 8 | Tiket Travel Pulang |
| 9 | Tiket Kereta Berangkat |
| 10 | Tiket Kereta Pulang |
| 11 | Tiket Lainnya |

**Rules:**
- Tiket fisik WAJIB (tidak boleh dicoret-coret)
- FE upload bukti tiket di stage REALISASI
- Finance verifikasi bukti tiket di stage VERIFIED
- **Jika tidak ada bukti tiket:** Finance tentukan nominal saat rekonsiliasi (stage RECONCILED) — bukan nominal klaim FE

### MANAGER_APPROVAL (13 items — OWNER tentukan nominal saat approval)

| # | Kategori |
|---|----------|
| 1 | Transport Taksi Berangkat |
| 2 | Transport Taksi Pulang |
| 3 | Transport Ojek Beli Material |
| 4 | Akomodasi Uang Makan (Homebase) |
| 5 | Akomodasi Lainnya |
| 6 | Biaya Lain-Material |
| 7 | Biaya Lain-Lifting |
| 8 | Biaya Lain-Tarik Kabel |
| 9 | Biaya Lain-Tebang Pohon |
| 10 | Biaya Lainnya |
| 11 | Pengembalian Dana |
| 12 | Biaya Mounting (Modifikasi/Baru) |
| 13 | Jasa Subcont |

**Rules:** FE estimasi nominal sebagai referensi. OWNER menentukan nominal final saat approval. Tidak ada batas pagu.

---

## 5 Job Types

| Job Type | Keterangan |
|----------|-----------|
| INSTALASI | Pemasangan baru |
| RELOKASI / REPOSISI | Pemindahan |
| PMCM | Preventive Maintenance Corrective Maintenance |
| DISMANTLE | Pembongkaran |
| SURVEY | Survey lokasi |

Job type dipilih oleh FIELD_ENGINEER di awal form (berkoordinasi dengan Kordinator via email task). Job type menentukan nominal pagu untuk kategori FIXED_PAGU dan FEE.

---

## 7-Stage Workflow (Simulation)

### STAGE 0 — Email Task dari Kordinator

Kordinator kirim email ke FE: task_no, VID, job_type, lokasi, list estimasi awal. FE gunakan sebagai referensi saat buka form di app.

### STAGE 1 — DRAFT (FIELD_ENGINEER)

FE buka form di Android (offline-capable):
1. Isi task_no, VID (dari email)
2. Pilih job_type (INSTALASI/RELOKASI/PMCM/DISMANTLE/SURVEY)
3. Pilih lokasi dari master data (`master_locations`)
4. Tambah item budget:
   - Pilih kategori dari 35 kategori (dropdown)
   - Isi estimasi nominal
   - Tanggal per item
   - Catatan per item
5. Pagu enforcement:
   - FIXED_PAGU: nominal > pagu → BLOCK (kecuali HOTEL)
   - TICKET: ⚠ warning "wajib bukti fisik"
   - MANAGER_APPROVAL: 🔷 flag "OWNER tentukan"
6. Simpan draft (bisa resume kapan saja)
7. Klik "KIRIM KE KORDINATOR" → stage = ESTIMASI

### STAGE 2 — ESTIMASI (FE → SUPERVISOR Inbox)

Form masuk inbox SUPERVISOR.

### STAGE 3 — FORWARDED (SUPERVISOR Edit + Forward)

SUPERVISOR bisa:
- **Edit nominal per item** (revisi estimasi FE)
- **Tambah item baru**
- **Hapus item**
- **Tambah catatan Kordinator**

Audit trail: `estimated_amount` (asli FE) + `revised_amount` (revisi SUP) keduanya tersimpan.

SUPERVISOR klik "TERUSKAN KE MANAGER" → stage = FORWARDED.

**Tidak ada stage REJECTED terpisah.** Jika SUPERVISOR tidak setuju, bisa langsung edit sendiri (sesuai Opsi B wewenang Kordinator). Jika perlu koordinasi, bisa "KEMBALIKAN KE FE" → stage balik ke DRAFT.

### STAGE 4 — APPROVED (OWNER/Manager)

OWNER lihat:
1. **Historis kunjungan terakhir di lokasi yang sama** (dari `task_expenses` sebelumnya, stage >= APPROVED)
2. Estimasi asli FE + revisi SUPERVISOR
3. Pagu enforcement per item
4. Kategori MANAGER_APPROVAL: OWNER isi nominal final

OWNER bisa **adjust nominal per item** (approved_amount) — bisa berbeda dari estimasi FE maupun revisi SUP.

OWNER klik "APPROVE" → stage = APPROVED. Dana tersedia muncul di project.

**Jika OWNER tolak:** klik "TOLAK" → stage balik ke DRAFT dengan catatan penolakan.

### STAGE 5 — REALISASI (FIELD_ENGINEER)

Setelah pekerjaan selesai, FE buka form yang sama:
1. Setiap item sekarang muncul kolom "Realisasi"
2. FE isi `realization_amount` per item
3. Upload bukti kwitansi per item
4. Sistem auto-hitung selisih (approved - realisasi)

FE klik "KIRIM REALISASI" → stage = REALISASI.

### STAGE 6 — VERIFIED (ADMIN + FINANCE_MANAGER)

ADMIN/FINANCE_MANAGER verifikasi per item:
1. Cek bukti tiket: wajib resmi, tidak dicoret
2. Cek bukti hotel: jika > pagu, wajib bill asli
3. Cek nominal realisasi vs approved
4. Flag item yang tidak ada bukti / mencurigakan
5. `bill_verified` centang per item

### STAGE 7 — RECONCILED (FINANCE_MANAGER)

FINANCE_MANAGER:
1. Crosscheck ke Kordinator per item
2. **Tiket tanpa bukti:** Finance tentukan nominal (bisa berbeda dari klaim FE)
3. Final close → stage = RECONCILED

---

## Data Model (Planned)

### master_locations
```
id, project_id FK, remote_name, address, latitude, longitude, timestamps
```
Di-input oleh ADMIN/OWNER. Digunakan untuk historis lokasi saat OWNER approval.

### task_expenses
```
id, uuid, project_id FK, task_no, vid, task_name, remote_name, location_id FK,
job_type ENUM, submitted_by FK, forwarded_by FK, approved_by FK, verified_by FK,
reconciled_by FK, stage ENUM(7 values), total_estimated, total_approved,
total_realization, total_saving, notes, timestamps
```

### expense_items
```
id, task_expense_id FK, category_id FK, tanggal,
estimated_amount (FE), revised_amount (SUP, nullable),
approved_amount (OWNER), realization_amount (FE),
note, bukti_path, requires_bill, bill_verified,
item_status, rejection_reason
```

### budget_item_templates
```
id, category_name, category_group, pagu_type ENUM(FIXED_PAGU/TICKET/MANAGER_APPROVAL),
pagu_amount (nullable, fallback), requires_bill, is_active
```

### pagu_job_type_amounts (pivot — per-job_type pagu)
```
id, template_id FK, job_type ENUM(INSTALASI/RELOKASI/PMCM/DISMANTLE/SURVEY),
amount (nullable — null = tidak berlaku untuk job type ini)
```

**Pagu varies by job_type in this table:**
- VOUCHER: 15rb(INSTALASI/RELOKASI/PMCM) / 5rb(DISMANTLE/SURVEY)
- BURUH: 120rb(INSTALASI/RELOKASI) / null(PMCM/DISMANTLE/SURVEY)
- BALLAST: 200rb(INSTALASI/RELOKASI) / null(PMCM/DISMANTLE/SURVEY)
- FEE: 40rb(INSTALASI) / 75rb(RELOKASI) / 15rb(PMCM/DISMANTLE) / null(SURVEY)
- PAKET_LK, HOTEL_LK, HOTEL_PAPUA: same across all job types

### config/budget.php
All hardcoded parameters centralized here. Key keys:
```
BUDGET_MAX_DRAFTS=5, BUDGET_PAGINATION=20, BUDGET_HISTORY_LIMIT=10
Stages: DRAFT→ESTIMASI→FORWARDED→APPROVED→REALISASI→VERIFIED→RECONCILED
Rejectable stages: ESTIMASI, FORWARDED
```

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **1 form, multi-stage** (bukan 2 form) | 1 source of truth, FE tidak input ulang, audit trail linear, no data duplication |
| **SUPERVISOR edit (Opsi B)** | Kordinator paling tahu kondisi lapangan, perlu wewenang koreksi sebelum OWNER review |
| **Per-item reconciliation** | Setiap item diverifikasi terpisah (tiket vs akomodasi vs fee punya aturan beda) |
| **Tiket: Finance tentukan nominal** | Jika tidak ada bukti fisik, Finance punya otoritas tentukan nominal saat rekonsiliasi |
| **Master data lokasi** | Untuk query historis yang akurat (bukan free text) saat OWNER approval |
| **Offline-capable** | FE di lapangan tanpa internet — form + master data di-cache Room DB, sync via outbox |

---

## Open Questions (Resolved & Open)

✅ **OWNER tolak:** Stage balik ke DRAFT dengan catatan rejection. SUPERVISOR juga bisa reject (ESTIMASI stage) → cascade ke DRAFT.
✅ **SUPERVISOR edit (Opsi B):** Kordinator boleh edit nominal, tambah/hapus item sebelum forward ke OWNER. Audit trail: estimated_amount (FE) + revised_amount (SUP) keduanya tersimpan.
✅ **Job type:** Ditentukan Kordinator via task assignment di app. FE bisa pilih sendiri dengan berkoordinasi ke Kordinator.
✅ **Master data lokasi:** Di-input ADMIN + SUPERVISOR via API CRUD. OWNER approval pakai historis dari lokasi ini.
✅ **Pagu per job_type:** Di-implement via `pagu_job_type_amounts` pivot table. VOUCHER, FEE, BURUH, BALLAST beda nominal per job type.
✅ **Realisasi:** Per-item (bukan total). ADMIN/FINANCE verifikasi per item, rekonsiliasi per item.
✅ **Offline:** Android handle via Room DB cache + outbox pattern (existing). 5x retry. Max 5 draft per FE.
✅ **Foto:** 19 foto wajib SCM (di-master-kan di `master_equipment_options`, field_key=FOTO_WAJIB_SCM). Foto pekerjaan berbeda dengan foto bukti kwitansi.
✅ **Laporan Pekerjaan:** Form teknis terpisah (`laporan_pekerjaan` table), diisi FE bersamaan atau paralel dengan realisasi biaya. Diverifikasi oleh SUPERVISOR (Kordinator).
✅ **Task assignment:** Via app (bukan email). SUPERVISOR assign task ke FE dengan task_no, VID, job_type, lokasi. Muncul di "My Tasks" FE.

### Still Open
- Threshold SUPERVISOR approve mandiri? (Saat ini pakai Opsi B: edit+forward, bukan approve langsung) 
