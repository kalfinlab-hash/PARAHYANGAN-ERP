# PARAHYANGAN ERP --- Enhanced Edition

## Master Specification v2.1

**Jenis dokumen:** Dokumen induk kebutuhan dan ruang lingkup\
**Status:** Baseline perencanaan --- bukan spesifikasi implementasi
final\
**Tanggal:** 7 November 2026\
**Bisnis:** Toko Ban Parahyangan Lembang

------------------------------------------------------------------------

## 1. Tujuan dokumen

Dokumen ini menetapkan visi, ruang lingkup, prinsip, kebutuhan bisnis,
modul, keputusan yang sudah disetujui, dan keputusan yang masih terbuka
untuk PARAHYANGAN ERP --- Enhanced Edition.

Dokumen ini menjadi sumber induk. Detail implementasi teknis berada di
dokumen kedua: **Technical Blueprint v1.0**. Jika detail teknis belum
diputuskan, implementasi harus menunggu keputusan atau menggunakan
rancangan sementara yang diberi label jelas.

## 2. Identitas dan strategi proyek

  -----------------------------------------------------------------------
  Area                                Ketetapan
  ----------------------------------- -----------------------------------
  Nama                                PARAHYANGAN ERP --- Enhanced
                                      Edition

  Strategi                            New Build; aplikasi dan arsitektur
                                      baru

  Aplikasi lama                       Referensi alur, fitur, keputusan,
                                      dan data yang masih berguna

  Migrasi                             Selektif; data uji, duplikat, dan
                                      data tidak valid dikecualikan

  UI                                  Redesain penuh dengan
                                      mempertahankan alur bisnis yang
                                      baik

  Platform                            Dievaluasi melalui benchmark
                                      sebelum dipilih

  Biaya                               Free-first; tidak ada
                                      ketergantungan berbayar tanpa
                                      persetujuan

  Excel                               XLSX dengan formula dan template
                                      yang berfungsi adalah kebutuhan
                                      wajib

  Absensi                             QR Code dan verifikasi wajah
                                      digunakan bersama

  Cabang                              Rekomendasi satu cabang aktif pada
                                      awalnya; kesiapan multi-cabang

  Akuntansi                           Rekomendasi keuangan operasional
                                      dahulu; siapkan evolusi ke
                                      double-entry
  -----------------------------------------------------------------------

## 3. Konteks bisnis

-   Toko Ban Parahyangan Lembang menjual ban mobil.
-   Pengguna terdiri dari pemilik, admin, dan sekitar 2--5 pegawai.
-   Sebagian pegawai membutuhkan antarmuka yang sederhana dan mudah
    dipahami.
-   Pemilik terbiasa menggunakan Excel.
-   Admin dapat diberi akses operasional luas, tetapi tetap melalui
    permission dan audit.
-   Sistem harus sederhana untuk operasional harian dan cukup kuat untuk
    berkembang.

## 4. Sasaran bisnis

1.  Akurasi persediaan dan mutasi barang.
2.  Ketepatan penjualan, penerimaan pembayaran, dan dokumen.
3.  Pelacakan barang milik toko, barang konsinyasi, dan barang dengan
    pembayaran tertunda.
4.  Kontrol utang supplier dan pembayaran cicilan.
5.  Penanganan pesanan marketplace, packing, pengiriman, dan retur.
6.  Absensi dan payroll yang dapat diaudit.
7.  Laporan pemilik yang dapat ditelusuri ke transaksi sumber.
8.  Ekspor/impor Excel yang dapat digunakan dalam pekerjaan nyata.
9.  Keamanan, backup, pemulihan, dan portabilitas data.
10. Penggunaan infrastruktur yang hemat dan terukur.

## 5. Prinsip sistem

-   Akurasi data lebih penting daripada kemudahan mengubah angka secara
    langsung.
-   Setiap transaksi penting memiliki identitas, status, waktu server,
    pelaku, dan audit.
-   Transaksi stok dan uang harus konsisten.
-   Pembatalan transaksi final dilakukan melalui reversal/koreksi
    terkontrol, bukan penghapusan diam-diam.
-   Permission diperiksa di server.
-   Cache lokal mempercepat pembacaan, bukan menggantikan sumber
    kebenaran server.
-   Perpindahan halaman tidak memicu refresh penuh tanpa alasan.
-   Fitur yang tidak sedang digunakan tidak boleh menghasilkan
    request/polling yang tidak perlu.
-   Kegagalan jaringan tidak boleh membuat status transaksi seolah
    berhasil.
-   Sistem menyediakan backup dan proses restore yang diuji.
-   Data pribadi dan biometrik diminimalkan serta dibatasi aksesnya.

## 6. Pengguna dan kewenangan

### 6.1 Pemilik

-   Pemantauan seluruh operasional.
-   Laporan penjualan, margin/laba, stok, kewajiban, dan payroll.
-   Approval tindakan berisiko.
-   Pengelolaan pengguna, konfigurasi, audit, dan backup.
-   Akses penuh sesuai kebijakan keamanan.

### 6.2 Admin

-   Menjalankan operasional sesuai permission.
-   Mengelola produk, supplier, pembelian, penerimaan, penjualan,
    pegawai, dan laporan yang diizinkan.
-   Tidak otomatis mendapat akses terhadap semua data sensitif atau
    fungsi teknis.

### 6.3 Pegawai

-   Melihat dan menjalankan tugas yang diberikan.
-   Absensi, izin, riwayat pribadi, slip gaji pribadi setelah disahkan.
-   Kasir, packing, penerimaan, atau stock opname jika diberi izin.

Role dan permission harus dipisahkan. Hak akses minimum diterapkan.
Penyembunyian tombol di antarmuka bukan pengamanan yang cukup.

## 7. Ruang lingkup modul

### 7.1 Dashboard dan pemantauan

-   Ringkasan penjualan dan transaksi.
-   Pembayaran menurut metode.
-   Stok menipis/habis.
-   Barang masuk.
-   Pesanan marketplace bermasalah.
-   Absensi.
-   Utang supplier dan cicilan.
-   Retur.
-   Pengeluaran.
-   Payroll.
-   Notifikasi dan approval.

### 7.2 Master produk ban

-   SKU/kode internal.
-   Nama dan merek.
-   Ukuran ban.
-   Tipe/kategori.
-   Indeks beban/kecepatan bila relevan.
-   DOT/batch jika tersedia.
-   Barcode/QR.
-   Harga beli dan histori.
-   Harga jual dan histori.
-   Stok minimum.
-   Lokasi rak.
-   Status aktif/nonaktif.
-   Foto/catatan bila diperlukan.

### 7.3 Persediaan

-   Stok fisik, tersedia, reserved, rusak, karantina, dan konsinyasi
    dibedakan.
-   Setiap mutasi memiliki sumber transaksi dan alasan.
-   Tidak ada perubahan saldo langsung tanpa mekanisme adjustment.
-   Stock opname Grand dan Partial.
-   Hasil opname diverifikasi dan disetujui sebelum adjustment.

### 7.4 Supplier dan pembelian

-   Supplier dan kontak.
-   Purchase Order.
-   Penerimaan barang.
-   Faktur supplier.
-   Pembayaran penuh, cicilan, pembayaran setelah barang terjual, dan
    konsinyasi.
-   Skema dapat berbeda antarbaris/lot dalam satu penerimaan.
-   Utang dan pembayaran supplier dapat direkonsiliasi.

### 7.5 Penjualan dan POS

-   Scan/pilih produk.
-   Validasi harga dan stok.
-   Diskon sesuai permission.
-   Pembayaran tunai, transfer, QRIS.
-   Simpan dan Simpan & Cetak.
-   Nota profesional.
-   Produk tanpa harga valid tidak otomatis dapat dijual.
-   Penjualan tempo hanya jika kebijakan toko mengizinkan.

### 7.6 Marketplace dan pengiriman

-   Kanal yang dapat disiapkan: Shopee, Tokopedia, TikTok Shop, Lazada.
-   Import CSV/Excel atau format resmi.
-   API hanya jika akses dan ketentuan tersedia.
-   Validasi produk, kuantitas, stok, alamat, status.
-   Picking, packing, scan label/resi, dispatch.
-   Pesanan tanpa stok masuk daftar penanganan.
-   Tanggal kirim menunggu konfirmasi jika belum pasti.
-   Modul marketplace tidak boleh membebani modul kasir ketika tidak
    digunakan.

### 7.7 Retur dan garansi

Pisahkan retur pelanggan, retur marketplace, retur supplier, barang
rusak, dan klaim garansi. Setiap kasus mengacu transaksi asal, menyimpan
hasil inspeksi, keputusan, bukti, dan dampak stok/keuangan.

### 7.8 Pelanggan dan kendaraan

Simpan data secukupnya: nama/kontak, histori transaksi, kendaraan,
ukuran ban yang relevan. Batasi akses dan hindari pengumpulan data yang
tidak diperlukan.

### 7.9 Pegawai, absensi, payroll

-   Data pegawai dan jadwal.
-   QR Code + verifikasi wajah secara bersamaan.
-   Enrollment/re-enrollment oleh admin berwenang.
-   Kamera depan.
-   Liveness/anti-spoofing harus diuji, bukan sekadar diklaim.
-   Timestamp absensi ditetapkan server.
-   Izin, koreksi, approval, dan histori.
-   Payroll: Draft → Diperiksa → Disahkan/terkunci → Pembayaran
    penuh/sebagian → Slip gaji.
-   Pegawai hanya melihat slip miliknya.
-   GPS belum ditetapkan sebagai kewajiban.

Baseline jam kerja yang pernah dibahas: enam hari per minggu, sekitar
09.00--17.00 dengan istirahat satu jam, dan payroll bulanan. Semua tetap
dapat dikonfigurasi.

### 7.10 Keuangan operasional

-   Kas masuk/keluar.
-   Penerimaan penjualan.
-   Pembayaran supplier.
-   Utang/piutang.
-   Pengeluaran operasional.
-   Payroll.
-   Refund.
-   Rekonsiliasi.
-   Laba kotor dan estimasi laba bersih.

Omzet, laba, saldo kas, dan arus kas harus dibedakan. Double-entry
accounting masih perlu keputusan.

### 7.11 Stock opname

-   Grand dan Partial.
-   Penugasan ke semua/sebagian pegawai atau admin.
-   Pegawai tidak melihat stok sistem saat menghitung.
-   Mode kertas menyediakan kolom hitung kosong.
-   QR dapat membuka laporan digital.
-   Adjustment hanya setelah pemeriksaan dan approval.

### 7.12 Dokumen

Nota, invoice, PO, faktur, surat jalan, bukti pembayaran, retur,
garansi, label pengiriman, payroll, dan laporan.

## 8. Antarmuka

### 8.1 Desktop owner/admin

-   Workspace/worksheet untuk data tabular.
-   Tabel dapat diedit sesuai izin.
-   Scroll vertikal/horizontal.
-   Resize area kerja dan reflow.
-   Zoom workspace dengan Ctrl + mouse wheel bila kompatibel.
-   Sidebar dapat disembunyikan.
-   Font tetap terbaca.
-   Enam tema visual.
-   Tidak ada judul ganda, logo berlebihan, atau kartu dekoratif yang
    menghabiskan ruang.

### 8.2 Mobile owner/admin

Antarmuka mobile-native terpisah. Tidak menampilkan worksheet. Gunakan
kartu, daftar, form, detail, tombol aksi, dan navigasi bawah.

### 8.3 Mobile pegawai

Antarmuka ringkas untuk login, absensi, izin, histori pribadi, slip
gaji, dan tugas yang diberikan. Mode kasir hanya jika diizinkan.

## 9. Mesin bersama

### Business engines

1.  Universal Transaction Engine.
2.  Universal Search Engine.
3.  Notification & Task Center.
4.  Financial Control Engine.
5.  Document Management Engine.
6.  Barcode & QR Universal Scanner.
7.  Data Integrity & Reconciliation Engine.
8.  Approval Engine.

### System engines

1.  Authentication & Permission.
2.  Audit.
3.  Sync & Update.
4.  Backup & Recovery.
5.  Security & Privacy.
6.  System Health & Quota Monitoring.
7.  Testing & Release.
8.  Developer Control Center.

## 10. Excel Engine --- wajib

XLSX adalah format workbook utama. CSV/JSON merupakan format pertukaran
data dan tidak mempertahankan formula/format workbook.

Kelompok workbook: - Operational: produk, stok, barang masuk, penjualan,
stock opname, supplier. - Financial: kas, pembayaran, utang, cicilan,
pengeluaran, payroll. - Management: ringkasan, margin/laba sesuai izin,
perputaran stok, kewajiban, marketplace.

Standar: - Formula otomatis dan saling terhubung. - Sheet ringkasan dan
detail. - Filter/tabel. - Format Rupiah dan tanggal. - Sel input
dibedakan dari sel formula. - Formula dilindungi dari perubahan tidak
sengaja. - Tidak bergantung pada macro/VBA. - Template memiliki versi. -
Import melalui validasi, preview, konfirmasi, dan log. - Formula diuji
di Microsoft Excel dan LibreOffice. - Perhitungan server tetap menjadi
sumber kebenaran transaksi.

## 11. Offline, cache, dan sync

-   Cache untuk data referensi dan tampilan yang aman.
-   Sinkronisasi saat masuk, perubahan terdeteksi, atau manual Update.
-   Hindari refresh penuh saat berpindah halaman.
-   Transaksi kritis harus mendapat validasi server.
-   Antrean offline hanya untuk tindakan yang memiliki aturan konflik
    dan idempotensi aman.
-   Konflik stok/uang tidak boleh diselesaikan dengan last-write-wins.
-   UI harus membedakan pending, confirmed, failed, dan rejected.

## 12. Keamanan dan privasi

-   HTTPS/TLS.
-   Password hashing yang sesuai.
-   Session/token aman.
-   Server-side authorization.
-   Rate limit.
-   Validasi input server.
-   Audit tindakan kritis.
-   Secret tidak ditanam di source code.
-   Enkripsi data sensitif.
-   Pengelolaan sesi/perangkat.
-   Kebijakan retensi dan penghapusan.
-   Biometrik dipisahkan dari data pegawai umum.
-   Minimalkan penyimpanan foto mentah.
-   Sediakan prosedur kegagalan verifikasi wajah.

## 13. Backup dan pemulihan

-   Backup terjadwal.
-   Backup sebelum migrasi dan perubahan berisiko.
-   Backup database dan dokumen.
-   Verifikasi integritas.
-   Riwayat backup.
-   Restore test berkala.
-   Ekspor data untuk mengurangi vendor lock-in.
-   Tentukan RPO/RTO setelah platform dipilih.

## 14. Platform dan biaya

Belum diputuskan. Kandidat yang dievaluasi: Cloudflare Workers/D1/R2,
Supabase/PostgreSQL, Firebase/Firestore, dan alternatif relevan.

Benchmark mencakup biaya/kuota gratis terkini, konsistensi transaksi,
query, backup, storage, keamanan, portabilitas, latensi Indonesia, dan
beban realistis. Jangan menjanjikan layanan gratis selamanya.

## 15. Migrasi selektif

1.  Inventarisasi data lama.
2.  Klasifikasi migrasi/arsip/abaikan/perlu pemeriksaan.
3.  Mapping dan normalisasi.
4.  Deteksi duplikasi.
5.  Dry run staging.
6.  Rekonsiliasi jumlah record dan nominal.
7.  Pemeriksaan sampel.
8.  Approval.
9.  Migrasi produksi.
10. Verifikasi pascamigrasi.

Aplikasi lama tetap dipertahankan sampai validasi selesai. Data uji
tidak masuk produksi. Saldo pembukaan harus diverifikasi agar transaksi
historis tidak dihitung dua kali.

## 16. Prioritas

  Prioritas   Area
  ----------- -----------------------------------------------------
  P0          Database, autentikasi, permission, audit, transaksi
  P0          Produk, stok, pembelian, supplier
  P0          Penjualan, pembayaran, nota
  P0          Excel XLSX berformula
  P0          Backup, restore, data integrity
  P0          QR + verifikasi wajah; lulus uji keamanan
  P1          Keuangan dan rekonsiliasi
  P1          Payroll, izin, koreksi absensi
  P1          Stock opname
  P1          Marketplace, packing, pengiriman
  P1          Retur dan garansi
  P1          Search, scanner, notification
  P2          Developer Control Center lengkap
  P2          Analitik lanjutan
  P2          Multi-cabang aktif

## 17. Keputusan terbuka

1.  GPS absensi: QR + wajah saja atau ditambah validasi lokasi?
2.  Cabang: satu aktif dengan kesiapan multi-cabang atau multi-cabang
    sejak v1?
3.  Akuntansi: keuangan operasional atau double-entry sejak awal?
4.  Platform final berdasarkan benchmark.
5.  Retensi, consent, dan penghapusan biometrik.
6.  Batas margin dan kewenangan pengecualian.
7.  Apakah piutang pelanggan/penjualan tempo diperbolehkan?
8.  Batas approval diskon, retur, pembelian, pembatalan, dan adjustment.
9.  Printer nota/label dan perangkat scanner.
10. Marketplace prioritas pertama dan format data yang tersedia.

## 18. Tata kelola proyek

Pelihara: - Requirement Registry. - Feature Registry. - Architecture
Decision Records. - Change Log. - Known Issues. - Testing Results. -
Deployment History. - Migration Records. - Release Checklist. - Decision
Register.

Perubahan kebutuhan yang sudah disetujui harus dicatat dan dampaknya
dijelaskan.

## 19. Dokumen turunan

1.  Technical Blueprint & Database Design.
2.  API Contract.
3.  Permission Matrix.
4.  Business Process Flow.
5.  UI/UX Specification desktop.
6.  UI/UX Specification mobile owner/admin.
7.  UI/UX Specification mobile employee.
8.  Attendance & Biometric Security Specification.
9.  Excel Workbook & Formula Specification.
10. Import/Export Specification.
11. Backup & Disaster Recovery Plan.
12. Security Threat Model.
13. Test & Acceptance Plan.
14. Migration Plan.
15. Deployment Runbook.
16. Developer Control Center Specification.

## 20. Definition of Done

Modul selesai jika requirement, alur, UI perangkat target, validasi
server, permission, konsistensi data, audit, penanganan error,
integrasi, test, dokumentasi, dan backup sudah diverifikasi. Tampilan
yang berhasil dirender saja tidak cukup.

------------------------------------------------------------------------

**Akhir Master Specification v2.1.** Dokumen induk ini menetapkan ruang
lingkup dan prinsip. Detail teknis harus mengikuti Technical Blueprint
dan keputusan final yang dicatat.
