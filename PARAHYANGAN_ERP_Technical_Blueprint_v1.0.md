# PARAHYANGAN ERP --- Enhanced Edition

## Technical Blueprint v1.0

**Jenis dokumen:** Rancangan teknis awal untuk arsitektur, database,
API, transaksi, keamanan, sinkronisasi, pengujian, dan deployment\
**Status:** Draft teknis --- perlu validasi platform dan keputusan
terbuka\
**Tanggal:** 7 November 2026

> Dokumen ini merupakan pasangan Master Specification v2.1. Ini adalah
> rancangan awal yang cukup rinci untuk memulai desain teknis dan
> proof-of-concept, tetapi bukan klaim bahwa semua nama tabel, endpoint,
> tipe data, atau algoritma sudah final. Hal-hal yang bergantung pada
> platform atau keputusan bisnis ditandai secara eksplisit.

------------------------------------------------------------------------

## 1. Tujuan dan batas teknis

Technical Blueprint menerjemahkan kebutuhan bisnis menjadi rancangan
sistem. Dokumen ini mengatur: - Batas modul dan dependensi. - Model data
konseptual. - Aturan transaksi. - API dan kontrak. - Keamanan dan
permission. - Cache/offline/sinkronisasi. - Biometrik absensi. - Excel
import/export. - Backup, observability, dan deployment. - Pengujian dan
acceptance criteria.

### 1.1 Prinsip desain teknis

1.  Modular monolith terlebih dahulu.
2.  Server sebagai sumber kebenaran transaksi.
3.  Database transaksional dengan constraint.
4.  Semua operasi tulis kritis memiliki idempotency.
5.  Semua perubahan penting dapat diaudit.
6.  Tidak ada hard delete untuk transaksi final.
7.  PII dan biometrik diminimalkan.
8.  Integrasi eksternal terisolasi dari core transaction.
9.  Sistem dapat mengekspor data dalam format portabel.
10. Tidak memilih teknologi sebelum benchmark dan threat review.

## 2. Arsitektur logis

``` text
Desktop Web / Mobile Web
          |
     Presentation
          |
     API / Application
          |
  +-------+---------+----------------+
  |                 |                |
Identity &       Business         Integration
Authorization    Modules          Adapters
  |                 |                |
  +----------- Shared Engines -------+
                  |
        Transaction Boundary
                  |
        Relational Data Store
           |            |
      Object Store   Job/Outbox
           |
       Backup/Export
```

### 2.1 Modul aplikasi

-   Identity & Access.
-   Product Catalog.
-   Inventory.
-   Purchasing & Supplier.
-   Sales/POS.
-   Marketplace & Shipping.
-   Returns & Warranty.
-   Customer & Vehicle.
-   Employee & Attendance.
-   Payroll.
-   Finance.
-   Reporting.
-   Documents.
-   Import/Export.
-   System Administration.

### 2.2 Mesin bersama

-   Transaction Engine.
-   Approval Engine.
-   Audit Engine.
-   Search Engine.
-   Notification/Task Engine.
-   Scanner Adapter.
-   Reconciliation Engine.
-   Sync/Update Engine.
-   Backup/Recovery Engine.
-   Monitoring/Quota Engine.

### 2.3 Batas modul

Modul tidak boleh menulis tabel milik modul lain secara langsung tanpa
melalui service/use case yang ditetapkan. Contoh: Sales meminta
Inventory untuk memvalidasi dan mem-posting mutasi; Sales tidak mengubah
`inventory_balances` sendiri.

## 3. Keputusan platform dan benchmark

### 3.1 Kandidat

-   Cloudflare Workers + D1 + R2 dan layanan pendukung.
-   Supabase/PostgreSQL.
-   Firebase/Firestore.
-   Alternatif lain jika memenuhi kebutuhan.

### 3.2 Uji pembanding wajib

  -----------------------------------------------------------------------
  Kriteria                Metode uji              Bukti yang harus
                                                  disimpan
  ----------------------- ----------------------- -----------------------
  Transaksi atomik        Uji                     Test result
                          penjualan/penerimaan    
                          multi-record            

  Concurrency             Dua transaksi bersamaan Hasil dan log
                          pada stok terakhir      

  Query                   Query daftar stok,      Waktu respons
                          laporan, pencarian      

  Backup/restore          Buat dan pulihkan       Restore report
                          salinan uji             

  Storage                 Upload/download dokumen Biaya/kuota aktual

  Auth/permission         Uji akses lintas role   Security test

  Offline                 Putus-sambung koneksi   Sync test

  Export                  Ekspor workbook besar   Durasi/memori

  Portabilitas            Ekspor dan impor ke     Hasil migrasi
                          lingkungan lain         

  Free tier               Periksa batas resmi     Tanggal dan sumber
                          terkini                 
  -----------------------------------------------------------------------

Tidak boleh memakai angka kuota lama sebagai dasar keputusan final.
Catat tanggal pemeriksaan ketentuan provider. Uji biaya berdasarkan
proyeksi request, database, storage, dan backup.

### 3.3 Kriteria pemilihan

Pilih platform yang paling aman dan dapat dipelihara untuk model data
serta transaksi toko, bukan hanya yang paling mudah dibuat prototipenya.
Jika paket gratis tidak cukup untuk kebutuhan aman, tampilkan batas dan
alternatif; jangan membuat desain yang diam-diam bergantung pada layanan
berbayar.

## 4. Konvensi database

### 4.1 Konvensi umum

-   Nama tabel: snake_case, jamak.
-   Primary key: ID stabil; UUID/ULID disarankan jika cocok dengan
    platform.
-   Waktu: simpan timestamp UTC; tampilkan dalam zona waktu toko
    (`Asia/Jakarta`).
-   Uang: tipe decimal/numeric presisi, bukan floating point.
-   Kuantitas: tipe numeric sesuai kebutuhan satuan.
-   Nomor dokumen bisnis terpisah dari primary key.
-   Data transaksi menyimpan `created_at`, `created_by`, `updated_at`,
    `updated_by` bila relevan.
-   Record penting memiliki `version` untuk optimistic concurrency.
-   `branch_id` disiapkan pada entitas operasional meskipun awalnya
    hanya satu cabang.
-   Status internal menggunakan kode stabil; label UI Bahasa Indonesia.
-   Foreign key dan unique constraints digunakan bila didukung.
-   Jangan menyimpan secret, password, token, atau template wajah dalam
    log umum.

### 4.2 Kolom standar yang disarankan

``` sql
id                PRIMARY KEY
created_at        TIMESTAMP NOT NULL
created_by        ID NULL
updated_at        TIMESTAMP NULL
updated_by        ID NULL
version           INTEGER NOT NULL DEFAULT 1
```

Kolom standar tidak harus ditambahkan ke setiap tabel secara membabi
buta. Tabel event append-only dapat memiliki skema khusus.

### 4.3 Soft delete

-   Master data yang sudah tidak digunakan dapat diarsipkan.
-   Transaksi final tidak dihapus; gunakan reversal/cancel workflow.
-   Audit log bersifat append-only sesuai kebijakan retensi.
-   Dokumen mengikuti kebijakan legal dan retensi.

## 5. Model data konseptual

Nama tabel dan kolom di bawah adalah rancangan domain. Tipe data,
nullability, constraint, dan indeks final harus ditentukan setelah
platform dipilih.

### 5.1 Identity & access

#### `users`

-   `id`
-   `username` UNIQUE
-   `password_hash`
-   `display_name`
-   `status`
-   `last_login_at`
-   `created_at`, `updated_at`

#### `employees`

-   `id`
-   `user_id` FK UNIQUE
-   `employee_code` UNIQUE
-   `full_name`
-   `employment_status`
-   `default_role_id` bila dipakai
-   `joined_at`
-   `ended_at`

#### `roles`, `permissions`, `role_permissions`, `user_roles`

-   Role dan permission many-to-many.
-   Permission berupa kode granular.
-   Hindari hard-code asumsi bahwa satu user hanya punya satu role.

#### `sessions`

-   `id`
-   `user_id`
-   `device_id`
-   `created_at`
-   `last_seen_at`
-   `expires_at`
-   `revoked_at`
-   metadata perangkat yang tidak berlebihan.

#### `login_events`

-   user, waktu, hasil, metode, metadata keamanan minimum.
-   Jangan simpan password/token.

### 5.2 Product & inventory

#### `products`

-   `id`, `sku` UNIQUE, `name`, `brand`, `category`
-   `size`, `load_index`, `speed_rating`
-   `unit`
-   `minimum_stock`
-   `active_status`
-   `description`

#### `product_barcodes`

-   `id`, `product_id` FK, `code` UNIQUE, `code_type`, `active`

#### `product_price_history`

-   `id`, `product_id`, `price_type`, `amount`
-   `effective_from`, `effective_to`
-   `changed_by`, `reason`, `approved_by`

#### `product_batches`

-   `id`, `product_id`, `supplier_id`
-   `dot_code`, `received_at`, `ownership_type`
-   `source_receipt_item_id`
-   metadata batch.

#### `inventory_locations`

-   `id`, `branch_id`, `location_code`, `name`, `active`

#### `inventory_movements`

-   `id`, `branch_id`, `product_id`, `batch_id`, `location_id`
-   `movement_type`
-   `quantity_delta`
-   `source_type`, `source_id`, `source_line_id`
-   `occurred_at`, `posted_at`, `posted_by`
-   `reason`, `reversal_of_id`

Ledger mutasi merupakan dasar penelusuran perubahan stok. Saldo dapat
dimaterialisasi untuk kinerja, tetapi harus dapat direkonsiliasi
terhadap ledger.

#### `inventory_balances`

-   `branch_id`, `product_id`, `batch_id`, `location_id`
-   `physical_qty`, `reserved_qty`, `damaged_qty`, `quarantine_qty`
-   `version`
-   unique composite key sesuai grain saldo.

`available_qty` dihitung dari komponen yang disepakati atau disimpan
sebagai nilai turunan yang dijaga konsistensinya. Definisi final harus
ditetapkan.

#### `inventory_reservations`

-   `id`, `product_id`, `batch_id`, `quantity`
-   `source_type`, `source_id`
-   `status`, `expires_at`, `released_at`

### 5.3 Supplier & purchasing

#### `suppliers`

-   `id`, `supplier_code` UNIQUE, `name`, `contact`, `address`, `status`

#### `purchase_orders`

-   `id`, `document_no` UNIQUE, `supplier_id`, `status`
-   `order_date`, `expected_date`, `branch_id`
-   `subtotal`, `discount_total`, `grand_total`

#### `purchase_order_items`

-   `id`, `purchase_order_id`, `product_id`
-   `ordered_qty`, `unit_cost`, `discount`, `line_total`

#### `goods_receipts`

-   `id`, `document_no` UNIQUE, `purchase_order_id`
-   `supplier_id`, `received_at`, `status`, `received_by`

#### `goods_receipt_items`

-   `id`, `goods_receipt_id`, `product_id`, `batch_id`
-   `received_qty`, `accepted_qty`, `rejected_qty`
-   `unit_cost`, `ownership_type`, `payment_terms_type`
-   `source_purchase_order_item_id`

#### `supplier_invoices`, `supplier_invoice_items`

-   Referensi faktur supplier dan item yang ditagihkan.
-   Simpan nomor faktur, tanggal, jatuh tempo, nilai, dan relasi
    penerimaan.

#### `supplier_payables`

-   Kewajiban per faktur/baris atau unit bisnis yang disepakati.
-   `original_amount`, `paid_amount`, `outstanding_amount`, `status`.

#### `supplier_payments`

-   `id`, `supplier_id`, `payment_date`, `amount`
-   `payment_method`, `reference_no`, `status`, `created_by`.

#### `supplier_payment_allocations`

-   Menghubungkan pembayaran ke satu atau beberapa kewajiban.
-   Mencegah satu pembayaran dihitung dua kali.

#### `consignment_records`

-   Referensi barang titipan, pemilik barang, jumlah, syarat,
    settlement, dan status.
-   Jangan menganggap barang konsinyasi sebagai aset milik toko.

### 5.4 Sales & customer

#### `customers`

-   `id`, `customer_code`, `name`, `phone`, `address`, `status`
-   Simpan hanya data yang diperlukan.

#### `customer_vehicles`

-   `id`, `customer_id`, `vehicle_brand`, `vehicle_model`
-   `plate_number` opsional, `tire_size`, `notes`.

#### `sales`

-   `id`, `document_no` UNIQUE, `branch_id`
-   `customer_id` nullable, `cashier_user_id`
-   `status`, `sold_at`, `subtotal`, `discount_total`, `grand_total`
-   `idempotency_key` UNIQUE dalam scope yang ditentukan.

#### `sale_items`

-   `id`, `sale_id`, `product_id`, `batch_id`
-   `quantity`, `unit_price`, `unit_cost_snapshot`
-   `discount_amount`, `line_total`.

#### `sale_payments`

-   `id`, `sale_id`, `method`, `amount`, `reference`
-   `status`, `received_at`, `verified_by`.

#### `customer_receivables`

-   Dibuat hanya jika penjualan tempo diaktifkan.
-   Menyimpan nilai awal, saldo, jatuh tempo, status.

### 5.5 Marketplace & shipment

#### `sales_channels`

-   Kode kanal dan status integrasi.

#### `marketplace_orders`

-   `id`, `channel_id`, `external_order_id`
-   unique `(channel_id, external_order_id)`
-   status eksternal dan internal, waktu impor, payload reference yang
    aman.

#### `marketplace_order_items`

-   SKU eksternal, product mapping, quantity, price snapshot, validation
    status.

#### `pick_lists`, `packing_sessions`

-   Tugas picking/packing, operator, scan result, discrepancy.

#### `shipments`, `shipment_labels`, `shipment_events`

-   Carrier/channel, resi, label document reference, dispatch status,
    event timeline.
-   Label file disimpan di object storage dengan metadata di database.

### 5.6 Returns & warranty

#### `return_cases`

-   `id`, `case_no` UNIQUE, `return_type`
-   `source_transaction_type`, `source_transaction_id`
-   `status`, `opened_at`, `opened_by`, `decision_at`.

#### `return_items`

-   Produk, batch, quantity, condition, disposition.

#### `return_inspections`

-   Pemeriksa, hasil, catatan, bukti, keputusan.

#### `warranty_claims`, `warranty_events`

-   Garansi mengacu penjualan/item asal dan menyimpan status serta
    komunikasi.

### 5.7 Attendance & payroll

#### `attendance_qr_sessions`

-   `id`, `branch_id`, `session_type` (check-in/check-out)
-   `valid_from`, `valid_until`, `nonce_hash`, `status`, `created_by`.
-   QR harus berumur pendek dan tidak memuat secret permanen.

#### `biometric_enrollments`

-   `id`, `employee_id`, `status`, `enrolled_at`, `enrolled_by`
-   `consent_record_id` atau referensi pemberitahuan/persetujuan.
-   `revoked_at`, `retention_until`.

#### `biometric_templates`

-   `id`, `enrollment_id`, `encrypted_template_ref`
-   `algorithm_version`, `created_at`, `revoked_at`.
-   Template dienkripsi; jangan simpan sebagai kolom yang dapat dibaca
    oleh role umum.
-   Penyimpanan dan pemrosesan harus mengikuti kemampuan platform serta
    evaluasi privasi.

#### `attendance_verification_attempts`

-   `id`, `employee_id`, `qr_session_id`
-   `attempted_at`, `face_result`, `liveness_result`
-   `device_metadata_minimal`, `failure_code`, `review_status`.
-   Hindari menyimpan citra/video mentah sebagai log rutin.

#### `attendance_events`

-   `id`, `employee_id`, `event_type`
-   `server_timestamp`, `verification_attempt_id`
-   `status`, `source`, `approved_correction_id`.
-   Unique/idempotency rule untuk mencegah event ganda.

#### `work_schedules`

-   Pegawai/kelompok, hari, jam mulai/selesai, break, effective dates.

#### `leave_requests`, `attendance_corrections`

-   Status pengajuan, alasan, periode, pemohon, approver, waktu.

#### `pay_periods`

-   `id`, period start/end, status, locked_at, approved_by.

#### `payroll_runs`, `payroll_items`, `payroll_payments`, `payslips`

-   Snapshot komponen gaji dan perhitungan per periode.
-   Pembayaran dapat parsial.
-   Slip hanya tersedia setelah periode disahkan.
-   Perubahan setelah lock memerlukan prosedur koreksi.

### 5.8 Finance

#### `cash_accounts`

-   Jenis akun kas/bank/QRIS clearing sesuai kebutuhan pelaporan.

#### `cash_sessions`

-   Sesi kasir/shift, saldo awal, saldo akhir, counted amount, variance.

#### `cash_movements`

-   Kas masuk/keluar dengan referensi transaksi sumber.

#### `expenses`

-   Kategori, nominal, tanggal, bukti, status, approval.

#### `financial_transactions`

-   Ledger keuangan operasional yang menghubungkan peristiwa bisnis ke
    dampak uang.
-   Jangan mencampur saldo kas dengan laba.

#### `reconciliation_runs`, `reconciliation_items`

-   Periode, sumber, hasil pencocokan, variance, status tindak lanjut.

Jika double-entry diaktifkan, tambahkan `accounts`, `journal_entries`,
`journal_lines`, dan aturan debit/kredit yang tervalidasi. Jangan
membuat jurnal setengah jadi tanpa kebijakan akuntansi yang disetujui.

### 5.9 Approval, documents, imports, audit, system

-   `approval_requests`, `approval_actions`
-   `documents`, `document_versions`
-   `import_jobs`, `import_rows`
-   `export_jobs`
-   `audit_events`
-   `notifications`, `tasks`
-   `backup_jobs`, `restore_tests`
-   `system_settings`, `feature_flags`
-   `schema_migrations`, `integration_jobs`, `system_health_events`

## 6. Relasi inti

``` text
products 1---N product_barcodes
products 1---N product_batches
products 1---N inventory_movements
products 1---N inventory_balances

suppliers 1---N purchase_orders
purchase_orders 1---N purchase_order_items
purchase_orders 1---N goods_receipts
goods_receipts 1---N goods_receipt_items
goods_receipt_items 1---N product_batches (bila batch dibuat)

sales 1---N sale_items
sales 1---N sale_payments
sale_items 1---N return_items (melalui kasus retur dan referensi item asal)

employees 1---N attendance_events
employees 1---N biometric_enrollments
biometric_enrollments 1---N biometric_templates
pay_periods 1---N payroll_runs
payroll_runs 1---N payroll_items
payroll_items 1---N payroll_payments
```

Relasi final harus dinormalisasi dan diuji terhadap kasus parsial,
campuran, pembatalan, dan migrasi.

## 7. Aturan stok dan posting transaksi

### 7.1 Prinsip ledger

`inventory_movements` adalah histori pergerakan. `inventory_balances`
adalah proyeksi saldo untuk query cepat. Setiap posting harus
memperbarui ledger dan saldo dalam satu transaksi database atau pola
konsistensi yang setara.

### 7.2 Contoh dampak

  -----------------------------------------------------------------------
  Peristiwa                           Dampak stok
  ----------------------------------- -----------------------------------
  Penerimaan diterima                 Tambah stok sesuai status
                                      ownership/disposition

  Penjualan final                     Kurangi stok tersedia/fisik sesuai
                                      model

  Reservasi pesanan                   Tambah reserved, kurangi available

  Pesanan batal                       Lepaskan reserved

  Retur pelanggan diterima            Masuk inspection/quarantine dahulu

  Retur supplier dikirim              Kurangi stok yang dapat dikirim

  Barang rusak                        Pindahkan ke damaged

  Opname disahkan                     Adjustment dengan referensi hasil
                                      opname
  -----------------------------------------------------------------------

Definisi apakah konsinyasi masuk stok fisik tetapi bukan stok milik toko
harus dipertahankan secara eksplisit. Laporan harus dapat membedakan
stok fisik dan kepemilikan.

### 7.3 Concurrency stok terakhir

Saat dua transaksi mencoba menjual unit terakhir: 1. Server membaca
saldo terkini dalam transaksi. 2. Mengunci row atau memakai conditional
update/version check sesuai database. 3. Hanya satu transaksi yang
berhasil mem-posting stok. 4. Transaksi lain menerima konflik stok dan
tidak membuat invoice final. 5. Retry tidak boleh menggandakan
transaksi.

## 8. Universal Transaction Engine

### 8.1 Pola posting

Setiap command bisnis: 1. Autentikasi pengguna. 2. Otorisasi permission.
3. Validasi input dan status sumber. 4. Validasi idempotency key. 5.
Buka transaksi database. 6. Periksa versi/saldo/constraint. 7. Simpan
dokumen dan detail. 8. Simpan mutasi stok/keuangan yang relevan. 9.
Simpan audit event. 10. Simpan outbox event untuk pekerjaan lanjutan.
11. Commit. 12. Kirim response berisi status dan ID dokumen.

Pekerjaan eksternal seperti email, notifikasi, atau sinkronisasi
marketplace dilakukan setelah commit melalui job/outbox, bukan di tengah
transaksi database.

### 8.2 Idempotency

Untuk command yang dapat diulang karena timeout: - Client mengirim
idempotency key unik. - Server menyimpan key, actor, endpoint/command,
request fingerprint, dan hasil. - Request identik dengan key sama
mengembalikan hasil sebelumnya. - Key sama dengan payload berbeda
ditolak. - Retensi key ditentukan berdasarkan risiko dan volume.

### 8.3 Reversal

Reversal: - Mengacu ke transaksi asal. - Tidak menghapus record asal. -
Menghasilkan dampak balik yang tepat. - Memerlukan alasan dan approval
sesuai kebijakan. - Tidak boleh me-reverse transaksi dua kali.

## 9. API contract

API berikut adalah rancangan resource/action. Path, metode, pagination,
dan format error final mengikuti platform dan standar yang dipilih.

### 9.1 Konvensi

-   Prefix versi, misalnya `/api/v1`.
-   JSON request/response.
-   Timestamp ISO 8601.
-   Nominal sebagai string decimal atau integer minor-unit sesuai
    kebijakan tunggal; jangan campur format.
-   Pagination cursor untuk daftar besar.
-   Filter dan sort di-whitelist.
-   Error memiliki `code`, `message`, `details`, `request_id`.
-   Jangan mengirim stack trace atau secret kepada client.

### 9.2 Endpoint kelompok

**Identity** - `POST /auth/login` - `POST /auth/logout` -
`POST /auth/refresh` jika token model memerlukannya - `GET /auth/me` -
`GET /sessions` - `DELETE /sessions/{id}`

**Products** - `GET /products` - `POST /products` -
`GET /products/{id}` - `PATCH /products/{id}` -
`GET /products/{id}/price-history` - `GET /products/{id}/stock`

**Inventory** - `GET /inventory/balances` - `GET /inventory/movements` -
`POST /inventory/reservations` -
`POST /inventory/reservations/{id}/release` - `POST /stocktakes` -
`POST /stocktakes/{id}/items` - `POST /stocktakes/{id}/submit` -
`POST /stocktakes/{id}/approve`

**Purchasing** - `POST /purchase-orders` -
`POST /purchase-orders/{id}/submit` -
`POST /purchase-orders/{id}/approve` - `POST /goods-receipts` -
`POST /goods-receipts/{id}/post` - `POST /supplier-payments` -
`POST /supplier-payments/{id}/allocate`

**Sales** - `POST /sales` - `GET /sales` - `GET /sales/{id}` -
`POST /sales/{id}/post` - `POST /sales/{id}/cancel` -
`POST /sales/{id}/reverse` - `POST /sales/{id}/payments`

**Marketplace/shipping** - `POST /marketplace/import-jobs` -
`GET /marketplace/orders` - `POST /marketplace/orders/{id}/validate` -
`POST /packing-sessions` - `POST /shipments/{id}/dispatch`

**Returns** - `POST /returns` - `POST /returns/{id}/inspect` -
`POST /returns/{id}/approve` - `POST /supplier-returns`

**Attendance** - `POST /attendance/qr-sessions` -
`POST /attendance/verify` - `GET /attendance/me` -
`POST /attendance/corrections` - `POST /leave-requests` -
`POST /leave-requests/{id}/decision`

**Payroll** - `POST /payroll-runs` -
`POST /payroll-runs/{id}/calculate` - `POST /payroll-runs/{id}/submit` -
`POST /payroll-runs/{id}/approve` -
`POST /payroll-items/{id}/payments` - `GET /payslips/me`

**Finance/reporting** - `POST /expenses` -
`GET /finance/cash-movements` - `POST /reconciliations` -
`GET /reports/daily` - `GET /reports/inventory` -
`GET /reports/payables`

**Documents/import/export** - `POST /documents` -
`GET /documents/{id}` - `POST /imports` - `GET /imports/{id}/preview` -
`POST /imports/{id}/commit` - `POST /exports` - `GET /exports/{id}`

Endpoint aktual hanya boleh dibangun setelah permission, input schema,
response schema, side effect, dan error behavior ditentukan.

### 9.3 Contoh error response

``` json
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Stok tersedia tidak mencukupi.",
    "details": {
      "product_id": "…",
      "requested": 2,
      "available": 1
    },
    "request_id": "…"
  }
}
```

Jangan mengungkap harga pokok atau data sensitif dalam error kepada role
yang tidak berhak.

## 10. Permission enforcement

Setiap command memiliki permission eksplisit. Contoh:

  Command                Permission
  ---------------------- ------------------------
  Buat penjualan         `sale:create`
  Lihat harga pokok      `cost:read`
  Posting penerimaan     `purchase:receive`
  Koreksi stok           `stock:adjust`
  Setujui koreksi stok   `stock:adjust:approve`
  Ekspor payroll         `payroll:export`
  Enrollment wajah       `biometric:enroll`
  Lihat audit            `audit:read`
  Restore backup         `backup:restore`

Pemeriksaan harus dilakukan pada server untuk resource dan operasi,
termasuk akses berdasarkan kepemilikan data (contoh: slip gaji sendiri).
Deny-by-default diterapkan.

## 11. Approval Engine

### 11.1 Model

`approval_requests`: - entity type/id - action - requested by - reason -
risk/threshold metadata - status - created_at

`approval_actions`: - request_id - approver_id - decision - reason -
decided_at

### 11.2 Aturan

-   Approval terikat pada versi/data yang diminta.
-   Jika data berubah setelah diajukan, approval dapat dibatalkan dan
    perlu diajukan ulang.
-   Pembuat tidak boleh menyetujui tindakan yang mensyaratkan pemisahan
    tugas.
-   Semua keputusan diaudit.
-   Ambang nominal masih perlu dikonfigurasi.

## 12. QR + Face Attendance --- technical requirements

### 12.1 Alur

1.  Pengguna login.
2.  Server menyediakan QR session singkat untuk check-in/check-out.
3.  Client membaca QR dan mengirim token/session proof.
4.  Server memvalidasi masa berlaku, nonce, status, dan konteks.
5.  Client menjalankan capture wajah sesuai rancangan privasi.
6.  Verifikasi wajah dan liveness diproses melalui komponen yang dipilih
    setelah evaluasi keamanan.
7.  Server memvalidasi hasil, policy, dan anti-replay.
8.  Server membuat attendance event dengan server timestamp.
9.  Audit event dibuat.
10. Jika gagal, simpan failure code minimum dan arahkan ke prosedur
    pemeriksaan.

### 12.2 QR security

-   QR tidak boleh berupa kode statis yang dapat dipakai ulang
    selamanya.
-   Gunakan token acak/nonce, masa berlaku pendek, dan status sekali
    pakai atau pembatasan replay.
-   QR session diikat ke cabang/sesi dan jenis event.
-   Server memvalidasi semua klaim; client tidak boleh menentukan
    timestamp final.
-   Jangan masukkan data pribadi sensitif ke payload QR.
-   Screenshot/replay harus menjadi bagian dari test.

### 12.3 Face verification

-   Enrollment oleh admin berwenang.
-   Verifikasi terhadap template terdaftar.
-   Template disimpan terenkripsi atau melalui penyimpanan aman yang
    dipilih.
-   Pisahkan template dari profil pegawai.
-   Versioning algoritma dan template.
-   Re-enrollment/revocation memiliki audit.
-   Threshold false accept/false reject harus diukur pada perangkat dan
    kondisi nyata.
-   Liveness/anti-spoof harus diuji dengan serangan foto, video, replay,
    layar, dan variasi kondisi.
-   Jangan menyatakan aman hanya karena vendor/library menyebut
    liveness.
-   Jangan simpan video/foto mentah kecuali ada kebutuhan yang
    dibenarkan, persetujuan, retensi, dan perlindungan yang jelas.

### 12.4 Fallback

Kegagalan kamera, jaringan, atau model: - Jangan mencatat kehadiran
sebagai berhasil. - Tampilkan status gagal/menunggu pemeriksaan. -
Sediakan koreksi/manual verification oleh pihak berwenang. - Koreksi
memiliki alasan, bukti bila diperlukan, dan audit. - Payroll menggunakan
status yang disahkan, bukan percobaan yang gagal.

### 12.5 GPS

GPS belum menjadi persyaratan final. Jika diaktifkan kelak, harus ada
tujuan yang jelas, pemberitahuan, batas akurasi, perlindungan lokasi,
serta fallback. Jangan mengumpulkan lokasi terus-menerus; jika
diperlukan, ambil hanya pada peristiwa absensi.

### 12.6 Biometric privacy checklist

-   Pemberitahuan dan dasar pemrosesan yang sesuai.
-   Pengumpulan minimum.
-   Enkripsi in transit dan at rest.
-   Akses sangat terbatas.
-   Retensi dan penghapusan terdefinisi.
-   Prosedur insiden.
-   Tidak menggunakan data wajah untuk tujuan lain tanpa dasar dan
    persetujuan yang sesuai.
-   Penilaian legal/privacy sebelum produksi.

## 13. Excel Engine

### 13.1 Arsitektur

-   Workbook generator menghasilkan XLSX.
-   Template registry menyimpan ID, versi, schema, dan kompatibilitas.
-   Export job membuat file berdasarkan filter dan permission.
-   Import job memvalidasi file sebelum commit.
-   File disimpan di object storage dengan signed/authorized download.
-   Formula tidak dijadikan satu-satunya perhitungan kritis.

### 13.2 Template registry

Field konseptual: - `template_id` - `template_name` -
`template_version` - `module` - `schema_version` - `supported_import` -
`supported_export` - `created_at` - `deprecated_at`

### 13.3 Workbook types

**Operational** - `README` - `RINGKASAN` - `PRODUK` - `STOK` -
`MUTASI` - `BARANG_MASUK` - `PENJUALAN` - `STOCK_OPNAME`

**Financial** - `README` - `RINGKASAN` - `KAS_MASUK` - `KAS_KELUAR` -
`UTANG_SUPPLIER` - `PEMBAYARAN` - `PAYROLL`

**Management** - `README` - `DASHBOARD` - `PENJUALAN_PERIODE` - `MARGIN`
(permission terbatas) - `PERSEDIAAN` - `KEWAJIBAN` - `MARKETPLACE`

Sheet aktual dapat dipisah per modul/versi agar workbook tidak terlalu
berat.

### 13.4 Formula specification examples

-   Line total: `=Quantity*UnitPrice`
-   Invoice outstanding: `=InvoiceTotal-SUM(AllocatedPayments)`
-   Stock difference: `=PhysicalQty-SystemQty`
-   Payroll net: `=GrossPay-Deductions`
-   Summary: `=SUM(DetailRange)`

Gunakan structured references jika tabel Excel digunakan. Hindari
formula volatile yang tidak perlu. Formula final harus menyebutkan
definisi bisnis, range, dan penanganan nilai kosong.

### 13.5 Import pipeline

1.  Upload ke staging/quarantine.
2.  Validasi ukuran dan tipe file.
3.  Scan/validasi struktur workbook.
4.  Cek template ID/version.
5.  Parse baris.
6.  Normalisasi data.
7.  Validasi referensi dan permission.
8.  Deteksi duplikasi.
9.  Buat preview valid/invalid.
10. User konfirmasi.
11. Commit dalam batch transaksional yang aman.
12. Catat audit dan hasil.
13. Sediakan file error report.

Import tidak boleh menimpa transaksi final secara massal. Untuk data
besar, gunakan job asynchronous dengan status dan progress.

### 13.6 Export pipeline

1.  Periksa permission dan filter.
2.  Ambil snapshot data yang konsisten.
3.  Buat workbook.
4.  Hitung formula/cached values sesuai library yang dipilih.
5.  Uji format dan formula.
6.  Simpan file dengan akses terbatas.
7.  Berikan tautan unduhan terotorisasi dan masa berlaku.
8.  Audit ekspor data sensitif.

### 13.7 Uji kompatibilitas

Uji dengan Microsoft Excel dan LibreOffice. Verifikasi formula, cached
values, format tanggal, Rupiah, filter, tabel, proteksi, dan ukuran
file. Jangan mengandalkan macro/VBA.

## 14. Offline cache dan synchronization

### 14.1 Data classes

  -----------------------------------------------------------------------
  Kelas                   Contoh                  Kebijakan
  ----------------------- ----------------------- -----------------------
  Reference cache         Produk, supplier        Cache dengan
                          ringkas                 TTL/version

  Read model              Ringkasan stok/laporan  Cache dan invalidasi

  Queued safe write       Draft non-final         Antrean dengan status
                                                  pending

  Online-required         Posting penjualan,      Server wajib
                          pembayaran, approval,   
                          payroll final           
  -----------------------------------------------------------------------

### 14.2 Sync protocol

Setiap perubahan antrean memiliki: - `operation_id` - `entity_type` -
`entity_id` - `base_version` - `payload` - `created_at_client` -
`status` - `retry_count`

Server memvalidasi permission, versi, idempotency, dan aturan bisnis.
`created_at_client` bukan waktu resmi transaksi.

### 14.3 Konflik

-   Data finansial/stok: jangan last-write-wins.
-   Konflik ditampilkan dan diselesaikan oleh alur khusus.
-   Retry dengan idempotency.
-   Kegagalan permanen masuk dead-letter/error queue untuk pemeriksaan.
-   Jangan menghapus antrean sampai server mengakui hasil final.

### 14.4 Update strategy

-   Sync saat masuk aplikasi jika ada perubahan/version mismatch.
-   Tombol Update manual.
-   Invalidation berdasarkan event bila tersedia.
-   Tidak melakukan refresh semua modul setiap perpindahan halaman.
-   Modul tidak aktif tidak polling tanpa kebutuhan.

## 15. Search, scanner, notifications

### 15.1 Search

-   Indeks pada SKU, barcode, nama, ukuran, nomor dokumen, supplier, dan
    resi.
-   Hasil mengikuti permission.
-   Pencarian global tidak boleh membocorkan harga pokok, payroll, atau
    data pribadi.
-   Debounce di client dan batas query di server.

### 15.2 Scanner

-   Adapter kamera/browser.
-   Input manual fallback.
-   Validasi jenis kode dan namespace.
-   Hasil scan harus dikonfirmasi sebelum posting tindakan berdampak.
-   Uji kamera pada perangkat Android target dan browser desktop.

### 15.3 Notifications/tasks

-   Notification record berisi penerima, prioritas, tipe, referensi,
    status dibaca/selesai.
-   Deduplicate notifikasi identik.
-   Task memiliki assignee, due date opsional, status, dan referensi.
-   Jangan menampilkan isi sensitif pada push/lock screen.

## 16. Audit, observability, dan monitoring

### 16.1 Audit event

Minimal: - actor ID - action - entity type/id - timestamp server -
result - reason - request/correlation ID - metadata minimum

Audit bersifat append-only untuk role operasional. Jangan log password,
token, secret, atau biometrik mentah.

### 16.2 System monitoring

-   Error rate.
-   Latency.
-   Failed jobs.
-   Failed sync.
-   Database/storage usage.
-   Request/quota usage.
-   Backup status.
-   Restore test status.
-   Import/export failures.
-   Biometric verification failure trend tanpa menyimpan citra.

### 16.3 Alerting

Alert harus punya severity, waktu, modul, ringkasan, dan tindakan.
Pisahkan alert operasional toko dari alert teknis Developer Control
Center.

## 17. Backup dan disaster recovery

### 17.1 Backup scope

-   Database.
-   Object/file storage.
-   Konfigurasi non-secret.
-   Template Excel dan dokumen sistem.
-   Audit sesuai kebijakan.

### 17.2 Backup controls

-   Jadwal.
-   Enkripsi.
-   Checksum/integrity.
-   Retention.
-   Access control.
-   Backup sebelum migrasi/deployment.
-   Salinan terpisah jika feasible.
-   Restore drill berkala.

### 17.3 RPO/RTO

Belum ditetapkan. Tentukan setelah pola transaksi, kemampuan platform,
biaya, dan toleransi operasional dikaji. Jangan menjanjikan
point-in-time recovery jika provider/konfigurasi tidak mendukungnya.

## 18. Deployment dan lingkungan

### 18.1 Environment

-   Local development.
-   Test/CI.
-   Staging.
-   Production.

Data produksi tidak boleh disalin ke staging tanpa sanitasi dan
otorisasi.

### 18.2 Release process

1.  Review perubahan.
2.  Unit/integration tests.
3.  Security checks.
4.  Backup jika migration berisiko.
5.  Apply schema migration.
6.  Deploy.
7.  Smoke test.
8.  Monitor.
9.  Rollback/mitigation jika gagal.

### 18.3 Secrets

-   Secret disimpan pada secret manager/environment.
-   Tidak berada di repository atau workbook.
-   Rotasi dan pencabutan terdokumentasi.
-   Akses hanya untuk proses yang memerlukan.

## 19. Security threat model awal

  -----------------------------------------------------------------------
  Ancaman                             Kontrol minimum
  ----------------------------------- -----------------------------------
  Akun pegawai disalahgunakan         Session management, revoke, login
                                      audit

  IDOR/data lintas akun               Object-level authorization

  Role escalation                     Server-side permission, test matrix

  Replay QR absensi                   Nonce, expiry, single-use, server
                                      validation

  Spoof wajah                         Liveness test, threshold
                                      validation, fallback audit

  Double submit                       Idempotency key

  Oversell stok                       Transaction/locking/version check

  Import berbahaya                    File validation, size limits,
                                      staging

  Kebocoran export                    Permission, signed link, expiry,
                                      audit

  Secret bocor                        Secret store, rotation

  Backup dicuri                       Encryption, access control

  Sync conflict                       Version checks, explicit resolution
  -----------------------------------------------------------------------

Threat model perlu diperbarui setelah teknologi, API, dan alur final
diketahui.

## 20. Testing strategy

### 20.1 Unit tests

-   Perhitungan total, diskon, saldo, kewajiban, dan payroll.
-   Status transition.
-   Permission policy.
-   Formula mapping.

### 20.2 Integration tests

-   Sales + inventory + payment.
-   Purchase + receiving + payable.
-   Return + inventory + refund.
-   Attendance + payroll.
-   Import + validation + commit.
-   Backup + restore.

### 20.3 Concurrency tests

-   Dua penjualan stok terakhir.
-   Dua pembayaran ke kewajiban yang sama.
-   Dua import dengan key sama.
-   Approval atas record yang berubah.

### 20.4 Security tests

-   Unauthorized endpoint.
-   Cross-user data.
-   Role escalation.
-   Expired/replayed QR.
-   Biometric spoof scenarios.
-   Export access.
-   Session revoke.

### 20.5 Offline tests

-   Putus koneksi saat draft.
-   Timeout setelah commit.
-   Retry request.
-   Conflict versi.
-   Perangkat ganda.

### 20.6 Acceptance criteria utama

**Sales** - Total dihitung benar. - Harga kosong ditolak. - Stok tidak
oversell. - Retry tidak menggandakan transaksi. - Payment status
jelas. - Nota mengacu ke transaksi final.

**Purchasing** - Penerimaan parsial didukung. - Kepemilikan dan skema
pembayaran per baris dapat dibedakan. - Utang sesuai faktur dan
pembayaran.

**Attendance** - QR expired/replayed ditolak. - Face/liveness failure
tidak menghasilkan absensi sukses. - Server timestamp digunakan. -
Manual correction diaudit. - Template hanya dapat dikelola role
berwenang.

**Excel** - Formula sesuai definisi. - Import menampilkan preview. -
Duplikasi terdeteksi. - Data invalid tidak di-commit. - Excel dan
LibreOffice dapat membuka workbook.

**Backup** - File backup diverifikasi. - Restore menghasilkan data yang
dapat dibaca. - Hasil restore test dicatat.

## 21. Data migration technical plan

### 21.1 Migration pipeline

-   Source inventory.
-   Extract.
-   Stage.
-   Normalize.
-   Map.
-   Validate.
-   Deduplicate.
-   Reconcile.
-   Approve.
-   Commit.
-   Post-migration verify.

### 21.2 Reconciliation

Bandingkan: - Jumlah produk. - Jumlah supplier. - Jumlah record per
kategori. - Saldo stok per SKU dan batch. - Total utang supplier. -
Total saldo kas/akun pembukaan bila dimigrasikan. - Jumlah pegawai
aktif. - Referensi dokumen.

Simpan laporan record yang dilewati dan alasan. Jangan menghapus sumber
lama sebelum sign-off.

## 22. Roadmap teknis

1.  Finalisasi open decisions.
2.  Platform benchmark proof-of-concept.
3.  ERD final dan schema review.
4.  API contract dan permission matrix.
5.  Core foundation: auth, audit, transaction, approval.
6.  Product/inventory/purchasing.
7.  Sales/payment/receipt.
8.  Finance/reconciliation.
9.  Attendance QR + face prototype and security evaluation.
10. Payroll and employee mobile.
11. Marketplace/returns/shipping.
12. Excel engine.
13. Offline/cache/sync hardening.
14. Monitoring, backup, restore, Developer Control Center.
15. Migration, pilot, and production launch.

## 23. Open technical decisions

1.  Platform/database final.
2.  GPS attendance.
3.  Biometric processing model/provider/library and legal/privacy
    review.
4.  Face template format, threshold, liveness method, retention.
5.  Single branch vs multi-branch activation.
6.  Double-entry accounting scope.
7.  Payment/receivable policy.
8.  Costing method for inventory and COGS (to be approved by
    business/accounting).
9.  Marketplace API/import formats.
10. Printer/scanner hardware.
11. RPO/RTO and backup retention.
12. Data retention and audit retention.
13. Currency representation and rounding policy.
14. Document numbering rules.

## 24. Artifacts required before implementation

-   ERD diagram.
-   Final SQL/schema migration.
-   Data dictionary with type/nullability/default/constraint/index.
-   API request/response schemas.
-   Permission matrix.
-   State transition diagrams.
-   Sequence diagrams for critical transactions.
-   UI screen specifications for all three experiences.
-   Excel template definitions and formula catalog.
-   Threat model.
-   Backup/restore runbook.
-   Test plan and automated test suite.
-   Platform benchmark report.
-   Migration mapping and reconciliation report.

------------------------------------------------------------------------

**Akhir Technical Blueprint v1.0.** Rancangan ini menjadi dasar diskusi
teknis dan proof-of-concept. Detail yang bergantung pada keputusan
bisnis, platform, keamanan biometrik, dan kebijakan akuntansi harus
diselesaikan sebelum produksi.
