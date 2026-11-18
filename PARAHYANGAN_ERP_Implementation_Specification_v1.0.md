# PARAHYANGAN ERP — Enhanced Edition
# Implementation Specification v1.0

**Jenis dokumen:** Dokumen implementasi dan kriteria penerimaan  
**Tanggal:** 7 November 2026  
**Status:** Draf implementasi terstruktur — wajib divalidasi sebelum produksi  
**Dokumen terkait:**  
1. `PARAHYANGAN_ERP_Master_Specification_v2.1.md` — acuan kebutuhan dan ruang lingkup.
2. `PARAHYANGAN_ERP_Technical_Blueprint_v1.0.md` — acuan rancangan arsitektur dan teknis.
3. `PARAHYANGAN_ERP_Implementation_Specification_v1.0.md` — dokumen ini.

> Dokumen ini menerjemahkan kebutuhan dan rancangan menjadi perilaku fitur, urutan implementasi, validasi, kondisi gagal, dan kriteria penerimaan. Dokumen ini bukan bukti bahwa fitur sudah dibangun atau diuji. Semua keputusan yang masih terbuka harus dicatat dan disetujui sebelum menjadi konfigurasi produksi.

---

## 0. Cara membaca dan status ketentuan

Setiap ketentuan menggunakan salah satu label berikut:

- **WAJIB** — harus tersedia sebelum modul dinyatakan selesai.
- **DIKONFIGURASI** — nilainya dapat diubah oleh pemilik/admin yang berwenang.
- **KEPUTUSAN TERBUKA** — belum boleh diasumsikan sebagai keputusan final.
- **REKOMENDASI** — pilihan awal yang perlu disetujui.
- **NANTI** — dapat dijadwalkan setelah fungsi inti stabil.

### 0.1 Prinsip implementasi lintas modul

1. Satu transaksi bisnis memiliki ID unik yang tidak berubah.
2. Waktu penting dicatat oleh server, bukan dipercaya dari jam perangkat.
3. Perubahan data penting harus memiliki pelaku, waktu, alasan, nilai sebelum/sesudah, dan referensi transaksi.
4. Transaksi yang sudah diposting tidak dihapus diam-diam. Koreksi dilakukan melalui pembatalan, reversal, atau transaksi koreksi yang terlacak.
5. Perhitungan stok, utang, pembayaran, laba, dan penggajian harus berasal dari catatan transaksi yang dapat direkonsiliasi.
6. Permintaan simpan harus idempoten: pengiriman ulang akibat jaringan tidak boleh menggandakan transaksi.
7. Otorisasi diperiksa di server untuk setiap operasi, bukan hanya menyembunyikan tombol di antarmuka.
8. UI harus menampilkan status proses yang jelas: belum disimpan, sedang diproses, berhasil, gagal, perlu tindakan, atau menunggu persetujuan.
9. Kegagalan jaringan tidak boleh ditampilkan sebagai keberhasilan.
10. Data contoh/pengujian harus diberi penanda dan tidak boleh bercampur dengan data produksi.

---

# BAGIAN A — STANDAR APLIKASI

## 1. Pola halaman dan navigasi

### 1.1 Desktop pemilik/admin

**WAJIB**
- Menggunakan tata letak desktop yang efisien dan terasa seperti aplikasi kerja.
- Modul tabel operasional dapat memakai worksheet/grid yang dapat diedit sesuai izin.
- Mendukung scroll vertikal dan horizontal tanpa memotong kolom penting.
- Kolom, panel, dan area kerja dapat disesuaikan ukurannya jika relevan.
- Sidebar dapat disembunyikan dan dipanggil kembali.
- Ukuran teks tetap terbaca; zoom tidak boleh merusak layout atau menutupi kontrol.
- Status stok rendah dan stok kosong dibedakan secara visual; warna tidak menjadi satu-satunya penanda.
- Hindari judul halaman ganda, kartu berlebihan, logo berulang, dan informasi dekoratif yang tidak membantu pekerjaan.

**Kriteria penerimaan**
- Pengguna dapat mengakses fungsi utama dengan navigasi konsisten.
- Tabel besar tetap dapat di-scroll pada dua arah.
- Mengubah lebar panel tidak membuat tombol utama hilang atau menimpa data.
- Hak akses menentukan kolom dan tindakan yang dapat dilakukan.

### 1.2 Smartphone pemilik/admin

**WAJIB**
- Antarmuka mobile-native yang terpisah dari desktop.
- Tidak menampilkan worksheet desktop.
- Mengutamakan kartu, daftar, formulir ringkas, detail, tombol aksi, dan navigasi bawah.
- Konten penting dapat digunakan dengan satu tangan sejauh memungkinkan.
- Form panjang dibagi menjadi bagian logis dan mempertahankan input ketika validasi gagal.

### 1.3 Smartphone pegawai

**WAJIB**
- Antarmuka lebih sederhana daripada admin.
- Hanya menampilkan fungsi yang diberikan: absensi, riwayat, pengajuan izin/koreksi, slip gaji yang sudah diterbitkan, dan mode kasir jika diizinkan.
- Tidak menampilkan data pegawai lain, laporan keuangan, atau data sensitif yang tidak diperlukan.
- Label dan instruksi mudah dipahami oleh pegawai dengan kemampuan digital beragam.

### 1.4 Login dan sesi

**WAJIB**
- Login aman untuk seluruh peran.
- Tidak menampilkan alur “lupa password” jika kebijakan akun internal memang tidak menggunakannya; pemulihan dilakukan admin/pemilik melalui prosedur aman.
- Sesi dapat dicabut oleh pemilik/admin berwenang.
- Perangkat/sesi aktif dapat ditinjau dan dicabut.
- Percobaan login gagal dibatasi dan dicatat.
- Pesan kesalahan tidak membocorkan apakah username tertentu terdaftar.

**Pengujian**
- Login valid dan tidak valid.
- Akun nonaktif.
- Sesi kedaluwarsa.
- Pencabutan sesi dari perangkat lain.
- Pengguna mencoba membuka URL modul tanpa hak akses.

---

# BAGIAN B — IDENTITAS, HAK AKSES, DAN AUDIT

## 2. Pengguna, peran, dan permission

### 2.1 Peran dasar

- **Pemilik:** akses penuh terhadap data bisnis, konfigurasi, persetujuan, laporan, dan pengelolaan akun.
- **Admin:** akses operasional sesuai kewenangan yang diberikan.
- **Pegawai:** hanya fungsi dan data yang ditugaskan.
- **Developer/operator teknis:** akses teknis terpisah dari akun operasional toko; tidak otomatis memiliki hak keuangan atau data pegawai.

Peran bukan pengganti permission. Sistem harus mendukung permission granular.

### 2.2 Aturan otorisasi

**WAJIB**
- Periksa permission pada server untuk setiap baca, buat, ubah, hapus/arsip, ekspor, impor, persetujuan, dan reversal.
- Terapkan prinsip least privilege.
- Pegawai hanya dapat melihat data pribadinya kecuali ada penugasan khusus.
- Perubahan role/permission harus diaudit.
- Aksi berisiko tinggi dapat memerlukan persetujuan pemilik atau autentikasi ulang.

### 2.3 Matriks permission minimum

| Area | Pemilik | Admin | Pegawai |
|---|---|---|---|
| Dashboard pemilik | Penuh | Sesuai izin | Tidak |
| Master produk | Penuh | Kelola sesuai izin | Lihat sesuai tugas |
| Stok | Penuh | Operasional | Input/lihat tugas |
| Pembelian dan utang | Penuh | Sesuai izin | Tidak, kecuali ditugaskan |
| Penjualan/kasir | Penuh | Kelola | Kasir jika diizinkan |
| Keuangan/laba | Penuh | Terbatas | Tidak |
| Absensi pribadi | Penuh | Kelola | Milik sendiri |
| Data pegawai | Penuh | Sesuai izin | Milik sendiri |
| Penggajian | Penuh | Sesuai izin | Slip sendiri setelah disahkan |
| Pengaturan sistem | Penuh | Terbatas | Tidak |
| Audit log | Penuh | Terbatas | Tidak |

Matriks final harus dikonfigurasi berdasarkan keputusan pemilik.

## 3. Audit trail

Setiap aksi sensitif minimal mencatat:
- ID audit.
- Pengguna dan peran saat aksi.
- Jenis aksi dan modul.
- Jenis serta ID objek.
- Timestamp server.
- Nilai sebelum dan sesudah, atau ringkasan perubahan.
- Alasan untuk koreksi/pembatalan.
- ID request/trace dan perangkat/sesi jika tersedia.

**Kriteria penerimaan**
- Pengguna tanpa izin tidak dapat mengubah atau menghapus audit.
- Koreksi transaksi menautkan transaksi asal.
- Audit dapat difilter berdasarkan waktu, pengguna, modul, dan ID transaksi.
- Data audit tidak memuat password, token rahasia, atau template biometrik mentah.

---

# BAGIAN C — MASTER DATA

## 4. Master produk ban mobil

### 4.1 Data minimum produk

- ID produk internal.
- SKU unik.
- Nama produk.
- Merek.
- Kategori/jenis ban.
- Ukuran ban.
- Indeks beban/kecepatan bila digunakan.
- Barcode/QR produk jika tersedia.
- Harga beli standar dan harga jual aktif.
- Satuan.
- Batas stok minimum.
- Lokasi rak.
- Status aktif/nonaktif.
- Catatan.

DOT, batch, dan tanggal produksi dicatat pada tingkat lot/unit penerimaan bila dibutuhkan; jangan memaksa satu nilai DOT untuk semua stok SKU yang sama.

### 4.2 Validasi

- SKU wajib unik.
- Harga dan kuantitas tidak boleh negatif.
- Produk nonaktif tidak dapat dipilih untuk transaksi baru, tetapi tetap muncul pada riwayat.
- Produk tanpa harga jual valid diberi status belum siap dijual.
- Ukuran dan merek menggunakan master pilihan jika diperlukan agar pencarian konsisten.
- Perubahan harga dicatat sebagai riwayat, bukan menimpa histori transaksi.

### 4.3 Kriteria penerimaan
- Admin dapat membuat, mengubah, menonaktifkan, dan mencari produk sesuai izin.
- SKU duplikat ditolak dengan pesan yang jelas.
- Riwayat penjualan lama tetap menampilkan harga saat transaksi, bukan harga terbaru.

## 5. Master pemasok dan pelanggan

### 5.1 Pemasok
Simpan identitas, kontak, alamat, status, syarat pembayaran, dan catatan. Data rekening/identitas sensitif dibatasi.

### 5.2 Pelanggan dan kendaraan
Simpan nama/kontak pelanggan dan informasi kendaraan seperlunya, misalnya nomor polisi, merek/model, serta catatan ukuran ban. Pengumpulan data harus proporsional dan akses dibatasi.

### 5.3 Validasi
- Nomor kontak dinormalisasi.
- Duplikasi ditandai untuk pemeriksaan, bukan otomatis digabung.
- Penghapusan data yang telah dipakai transaksi dilakukan sebagai penonaktifan/arsip.

---

# BAGIAN D — INVENTORY DAN STOCK LEDGER

## 6. Model stok

Sistem membedakan:
- **Stok fisik:** barang yang secara fisik ada.
- **Stok tersedia:** stok yang dapat dijual setelah memperhitungkan reservasi dan status lain.
- **Stok reserved:** dialokasikan untuk pesanan.
- **Stok rusak/karantina:** tidak dapat dijual.
- **Stok retur:** menunggu inspeksi/keputusan.
- **Stok konsinyasi:** barang titipan dengan status kepemilikan terpisah.
- **Stok milik toko:** barang yang telah menjadi milik toko.

Rumus ketersediaan final harus mengikuti model stok yang disetujui. Jangan menjumlahkan barang konsinyasi sebagai milik toko tanpa memperhitungkan status kepemilikan.

## 7. Pergerakan stok

Setiap perubahan kuantitas wajib berasal dari transaksi sumber, misalnya:
- Penerimaan pembelian.
- Penjualan.
- Reservasi/pembatalan reservasi.
- Pengiriman marketplace.
- Retur pelanggan/marketplace.
- Retur pemasok.
- Barang rusak.
- Stock opname yang telah disetujui.
- Transfer lokasi jika kelak ada cabang.

**DILARANG:** mengubah saldo stok langsung tanpa catatan movement.

### 7.1 Aturan posting
1. Validasi status transaksi.
2. Validasi ketersediaan stok dan ownership.
3. Buat movement dengan ID sumber.
4. Perbarui saldo materialized/cache secara atomik atau melalui proses konsisten.
5. Catat audit.
6. Jika langkah gagal, transaksi tidak boleh berada dalam status sukses parsial.

### 7.2 Peringatan stok
- Stok rendah: tampilkan penanda kuning.
- Stok kosong: tampilkan penanda merah.
- Jangan hanya mengandalkan tulisan “menipis/habis”; tetap sediakan angka stok.
- Peringatan tidak boleh menghitung stok reserved sebagai stok bebas.

## 8. Stock opname

### 8.1 Jenis
- **Grand:** seluruh produk/lokasi yang dipilih.
- **Partial:** produk tertentu.

### 8.2 Alur
1. Admin membuat sesi opname.
2. Pilih cakupan produk/lokasi.
3. Tetapkan petugas: semua pegawai tertentu, sebagian, atau admin.
4. Sistem membuat daftar kerja.
5. Petugas mengisi jumlah fisik.
6. Untuk pegawai, jumlah stok sistem/ekspektasi tidak ditampilkan.
7. Petugas mengirim hasil.
8. Admin memeriksa selisih dan bukti/catatan.
9. Pemilik/admin berwenang menyetujui.
10. Sistem membuat adjustment movement dan laporan selisih.

### 8.3 Form kertas
- Menampilkan identitas produk dan ruang kosong untuk jumlah fisik.
- Tidak mencetak stok ekspektasi untuk petugas.
- Tidak menambah kolom waktu/paraf jika tidak dibutuhkan.
- QR kecil dapat membuka detail sesi/laporan digital.

### 8.4 Kriteria penerimaan
- Pegawai tidak dapat melihat expected quantity.
- Draft opname tidak mengubah saldo stok.
- Hanya sesi disetujui yang membuat adjustment.
- Selisih memiliki alasan dan jejak persetujuan.
- Pengiriman ulang hasil tidak membuat duplikasi.

---

# BAGIAN E — PEMBELIAN, PENERIMAAN, DAN UTANG PEMASOK

## 9. Penerimaan barang

### 9.1 Alur
1. Buat draft pesanan/penerimaan.
2. Pilih pemasok.
3. Tambahkan item, SKU, kuantitas, harga, diskon, pajak jika berlaku, DOT/batch.
4. Tentukan kepemilikan dan skema pembayaran per item/lot.
5. Periksa jumlah fisik yang diterima.
6. Catat selisih/kerusakan.
7. Minta persetujuan jika melewati batas kewenangan.
8. Posting penerimaan.
9. Buat stock movement dan kewajiban finansial sesuai skema.

### 9.2 Skema pembayaran wajib didukung
- Lunas di muka/pada penerimaan.
- Cicilan.
- Bayar setelah barang terjual.
- Konsinyasi.
- Campuran beberapa skema pada satu penerimaan, termasuk per item/lot.

### 9.3 Data kewajiban per item/lot
- Nilai pokok.
- Nilai sudah dibayar.
- Sisa kewajiban.
- Jadwal/jatuh tempo bila ada.
- Jenis skema.
- Pemasok dan referensi penerimaan.
- Kuantitas diterima, terjual, dikembalikan, atau tersisa.
- Status kepemilikan.

### 9.4 Pembayaran pemasok
- Pembayaran dapat dialokasikan ke satu atau beberapa kewajiban.
- Pembayaran parsial diperbolehkan.
- Total alokasi tidak boleh melebihi dana pembayaran atau saldo kewajiban.
- Pembatalan pembayaran harus melalui reversal.
- Sistem menampilkan saldo sebelum/sesudah dan riwayat alokasi.

### 9.5 Kriteria penerimaan
- Barang konsinyasi tidak otomatis dianggap aset milik toko.
- Barang bayar-setelah-terjual mengurangi kewajiban sesuai unit/kuantitas yang terjual dan aturan kontrak.
- Cicilan parsial memperbarui saldo dengan benar.
- Satu penerimaan dapat berisi item dengan skema berbeda.
- Laporan utang dapat direkonsiliasi dengan pembayaran.

---

# BAGIAN F — PENJUALAN DAN POS

## 10. Alur kasir

1. Buka transaksi baru.
2. Cari/pindai produk.
3. Validasi produk aktif, harga, stok tersedia, dan kepemilikan.
4. Masukkan kuantitas.
5. Hitung subtotal, diskon, biaya, dan total.
6. Pilih metode pembayaran.
7. Konfirmasi pembayaran.
8. Simpan transaksi dan posting stock movement.
9. Buat nomor nota.
10. Tawarkan cetak nota jika dipilih.

### 10.1 Aksi simpan
- **Simpan:** menyelesaikan transaksi dan menyimpan bukti.
- **Simpan dan Cetak:** menyelesaikan transaksi lalu membuka alur cetak.

Kedua aksi tidak boleh membuat dua transaksi bila tombol ditekan berulang atau jaringan lambat.

### 10.2 Metode pembayaran
- Tunai.
- Transfer bank.
- QRIS.

Aplikasi hanya mencatat metode dan status pembayaran; dana masuk langsung ke rekening/kas pemilik/toko. Aplikasi bukan dompet elektronik.

### 10.3 Aturan harga dan stok
- Produk tanpa harga jual valid tidak dapat dijual sampai diperbaiki atau diberi override berizin dengan alasan.
- Penjualan tidak boleh membuat stok negatif kecuali kebijakan khusus disetujui dan dikonfigurasi.
- Harga transaksi disimpan sebagai snapshot.
- Perubahan master harga tidak mengubah nota lama.

### 10.4 Pembatalan dan pengembalian
- Transaksi draft dapat dibatalkan sesuai izin.
- Transaksi posted tidak dihapus.
- Pembatalan posted membuat reversal yang terhubung ke transaksi asal.
- Jika barang telah dikirim/diterima pelanggan, gunakan proses retur, bukan sekadar pembatalan.

### 10.5 Nota
Nota minimal memuat nomor, tanggal/waktu, identitas toko, item, ukuran/merek, kuantitas, harga, diskon, total, metode/status pembayaran, dan petugas. Informasi pajak hanya dicetak sesuai status/kewajiban usaha yang berlaku.

### 10.6 Kriteria penerimaan
- Satu klik atau pengiriman ulang tidak menggandakan penjualan.
- Total dihitung server-side.
- Stok berkurang tepat satu kali setelah transaksi posted.
- Nota dapat dicetak ulang tanpa membuat transaksi baru.
- Kasir tidak dapat melihat laporan keuangan di luar izin.

---

# BAGIAN G — MARKETPLACE DAN PENGIRIMAN

## 11. Kanal penjualan online

Dukungan kanal seperti Shopee, Tokopedia, TikTok Shop, dan Lazada bergantung pada akses resmi, format ekspor, serta izin integrasi yang tersedia. Jangan menganggap API gratis atau akses otomatis selalu tersedia.

### 11.1 Impor pesanan
1. Pilih kanal dan file/format.
2. Unggah file.
3. Parse ke staging.
4. Validasi struktur, nomor pesanan, SKU, variasi, kuantitas, alamat/status yang diperlukan.
5. Tampilkan preview dan error.
6. Deteksi duplikasi berdasarkan kanal + ID pesanan + item.
7. Admin mengonfirmasi.
8. Sistem mengimpor secara idempoten dan mencatat batch impor.

### 11.2 Validasi stok
- Cocokkan SKU marketplace ke master produk.
- Tampilkan pesanan yang stoknya tidak cukup.
- Jangan otomatis menjanjikan barang yang tidak tersedia.
- Reservasi stok mengikuti status pesanan yang disepakati.

### 11.3 Picking dan packing
- Tampilkan daftar barang per pesanan.
- Pindai SKU/barcode untuk validasi item dan jumlah.
- Ketidaksesuaian harus menghentikan proses sampai dikoreksi.
- Pisahkan status belum diproses, picking, packing, siap kirim, dan dikirim.

### 11.4 Label dan resi
- Izinkan unggah/scan label atau resi bila format mendukung.
- Validasi resi terhadap pesanan yang benar.
- Pemindaian resi menandai proses yang sesuai; tidak boleh otomatis dianggap barang telah diserahkan ke kurir tanpa bukti/status yang ditentukan.
- Tanggal kirim yang belum pasti dapat disimpan sebagai menunggu konfirmasi dan diperbarui setelah komunikasi marketplace.

### 11.5 Kinerja
- Modul POS tidak boleh melakukan polling marketplace yang tidak diperlukan.
- Sinkronisasi dilakukan atas tindakan eksplisit, jadwal yang disetujui, atau event resmi yang tersedia.
- Catat batas API, kegagalan, dan waktu sinkronisasi terakhir.

### 11.6 Kriteria penerimaan
- Impor file yang sama dua kali tidak menggandakan pesanan.
- Pesanan dengan SKU tak dikenal masuk daftar pengecualian.
- Pesanan stok kurang tidak dapat ditandai siap kirim tanpa penyelesaian.
- Label/resi tidak tertaut ke pesanan yang salah.

---

# BAGIAN H — RETUR, KERUSAKAN, DAN GARANSI

## 12. Jenis kasus

Pisahkan:
1. Retur pelanggan.
2. Retur marketplace.
3. Retur ke pemasok.
4. Barang rusak.
5. Klaim garansi.

### 12.1 Alur kasus
1. Buat case ID.
2. Hubungkan ke transaksi/pesanan/penerimaan asal.
3. Catat item, kuantitas, alasan, tanggal, pelapor, dan bukti.
4. Inspeksi kondisi.
5. Tentukan hasil: kembali ke stok tersedia, karantina, rusak, dikirim ke pemasok, diganti, atau refund.
6. Persetujuan sesuai batas kewenangan.
7. Posting dampak stok dan finansial.
8. Tutup kasus dengan catatan.

### 12.2 Kriteria penerimaan
- Retur tidak otomatis menambah stok tersedia sebelum inspeksi.
- Refund dan penggantian terhubung ke transaksi asal.
- Retur pemasok mengurangi stok/utang sesuai kondisi dan skema kepemilikan.
- Kasus garansi dapat dilacak dari pembukaan sampai penyelesaian.

---

# BAGIAN I — ABSENSI QR DAN VERIFIKASI WAJAH

## 13. Ketentuan wajib

**WAJIB:** absensi menggunakan **QR Code sekaligus verifikasi wajah**. QR saja tidak cukup dan verifikasi wajah saja tidak cukup.

Fitur GPS/lokasi masih **KEPUTUSAN TERBUKA** sampai disetujui pemilik. Jangan mengaktifkan atau menyatakan GPS wajib sebelum keputusan tersebut dibuat.

### 13.1 Pendaftaran wajah (enrollment)
- Hanya admin/pemilik dengan permission khusus yang dapat mendaftarkan atau mendaftarkan ulang wajah pegawai.
- Verifikasi identitas pegawai dilakukan sebelum enrollment.
- Gunakan kamera depan.
- Tampilkan pemberitahuan tujuan pemrosesan biometrik dan kebijakan penyimpanan.
- Ambil sampel sesuai panduan kualitas.
- Tolak gambar buram, terlalu gelap, wajah terhalang, atau lebih dari satu wajah bila sistem mensyaratkan satu wajah.
- Simpan representasi/template biometrik terlindungi, bukan foto mentah secara default.
- Catat siapa yang mendaftarkan, kapan, versi model/algoritma, dan status enrollment.
- Re-enrollment menonaktifkan template lama secara terkendali dan tercatat.

### 13.2 Alur absensi masuk/keluar
1. Pegawai login atau teridentifikasi sesuai desain keamanan.
2. Pegawai memindai QR yang sedang berlaku.
3. Server memvalidasi QR: tanda tangan/token, masa berlaku, status, konteks, dan pencegahan replay.
4. Aplikasi membuka kamera depan.
5. Pegawai menjalani pemeriksaan kualitas dan liveness/anti-spoofing.
6. Sistem membandingkan wajah dengan template pegawai yang terdaftar.
7. Server memvalidasi jadwal, status pegawai, dan aturan absensi.
8. Server menetapkan timestamp.
9. Catatan absensi disimpan dengan hasil verifikasi dan referensi audit.
10. Aplikasi menampilkan hasil yang jelas.

### 13.3 Keamanan QR
- QR harus dinamis atau memiliki mekanisme token berumur pendek.
- Token memiliki signature/validasi server.
- Token sekali pakai atau mekanisme anti-replay.
- Jangan memakai QR statis yang dapat difoto dan dipakai dari jauh.
- Rotasi dan masa berlaku ditentukan melalui konfigurasi.
- QR tidak memuat data pribadi pegawai dalam bentuk terbuka.

### 13.4 Liveness dan anti-spoofing
- Sistem harus memiliki pengujian terhadap foto cetak, foto pada layar, video replay, dan percobaan injeksi kamera bila lingkungan memungkinkan.
- Liveness tidak boleh hanya diasumsikan dari deteksi wajah.
- Hasil liveness, confidence, dan alasan penolakan dicatat secara aman tanpa mengekspos data biometrik.
- Threshold harus ditentukan melalui pengujian pada perangkat dan kondisi toko sebenarnya.
- Tidak boleh mengklaim anti-spoofing sempurna.

### 13.5 Kegagalan dan fallback
Kegagalan kamera, jaringan, QR, liveness, atau pencocokan wajah tidak boleh otomatis membuat absensi sukses.

Alur pengecualian:
1. Tampilkan alasan umum yang tidak membocorkan data biometrik.
2. Izinkan percobaan ulang terbatas.
3. Jika tetap gagal, pegawai mengajukan koreksi/absensi manual.
4. Admin memeriksa bukti dan memberi keputusan sesuai permission.
5. Semua override wajib memiliki alasan, pelaku, waktu, dan status persetujuan.
6. Sistem membedakan absensi terverifikasi otomatis dan absensi hasil koreksi manual.

### 13.6 Privasi biometrik
- Batasi akses template wajah hanya ke layanan yang memerlukan.
- Enkripsi saat transit dan saat tersimpan.
- Jangan tampilkan template atau foto enrollment pada halaman umum.
- Tetapkan tujuan, retensi, penghapusan, dan prosedur pencabutan.
- Jangan gunakan biometrik untuk tujuan lain tanpa persetujuan dan kajian terpisah.
- Hindari menyimpan foto/video mentah lebih lama dari yang diperlukan.
- Sediakan prosedur penghapusan ketika pegawai tidak lagi bekerja, sesuai kewajiban hukum dan kebutuhan pembuktian.
- Evaluasi dasar pemrosesan dan pemberitahuan privasi sesuai hukum Indonesia sebelum produksi.

### 13.7 Kriteria penerimaan absensi
- Absensi normal mensyaratkan QR valid dan wajah cocok serta liveness lolos.
- QR kedaluwarsa/di-replay ditolak.
- Wajah tidak cocok ditolak.
- Foto cetak/layar dan video replay diuji.
- Timestamp berasal dari server.
- Percobaan gagal tercatat tanpa membuat kehadiran sukses.
- Override manual memerlukan otorisasi dan alasan.
- Pegawai hanya dapat melihat riwayat absensinya sendiri.

---

# BAGIAN J — HR DAN PENGGAJIAN

## 14. Data pegawai dan jadwal

Baseline operasional yang pernah ditetapkan:
- 6 hari kerja per minggu.
- Jam kerja 09.00–17.00.
- Istirahat bersama 1 jam.
- Penggajian bulanan.
- Ketentuan dapat dikonfigurasi.

Hari libur, toleransi keterlambatan, lembur, potongan, dan komponen lain harus menjadi konfigurasi yang disetujui, bukan angka hard-coded.

## 15. Pengajuan izin dan koreksi

Status minimum:
- Draft.
- Diajukan.
- Disetujui.
- Ditolak.
- Dibatalkan.

Aturan:
- Pegawai hanya mengajukan untuk dirinya sendiri.
- Persetujuan dilakukan pihak berwenang.
- Koreksi absensi menyimpan nilai awal dan nilai koreksi.
- Perubahan setelah periode payroll disahkan mengikuti prosedur koreksi khusus.

## 16. Payroll

### 16.1 Siklus
1. Buat periode payroll.
2. Ambil data absensi dan komponen gaji.
3. Hitung draft.
4. Tampilkan rincian per pegawai.
5. Validasi anomali dan pengecualian.
6. Pemilik/admin berwenang mengesahkan.
7. Periode berstatus disahkan/terkunci.
8. Catat pembayaran penuh atau parsial.
9. Terbitkan slip gaji setelah disahkan.

### 16.2 Aturan
- Status: Draft → Disahkan/Terkunci.
- Periode yang disahkan tidak diedit langsung.
- Pembayaran parsial diperbolehkan setelah disahkan.
- Slip hanya tersedia setelah periode disahkan.
- Perubahan setelah pengesahan harus melalui adjustment/periode koreksi yang diaudit.
- Formula payroll dan angka final harus dihitung/validasi di server.

### 16.3 Kriteria penerimaan
- Username/kode pegawai unik.
- Timestamp absensi menggunakan server.
- Draft dapat dihitung ulang sebelum pengesahan.
- Periode disahkan tidak dapat diubah tanpa prosedur khusus.
- Pembayaran parsial menghasilkan saldo yang benar.
- Pegawai hanya melihat slip miliknya.

---

# BAGIAN K — KEUANGAN DAN REKONSILIASI

## 17. Pemisahan nilai keuangan

Sistem harus memisahkan:
- Penjualan bruto.
- Diskon/retur.
- Penjualan bersih.
- Harga pokok penjualan (HPP).
- Laba kotor.
- Beban operasional.
- Laba bersih sesuai definisi laporan.
- Penerimaan kas/bank.
- Arus kas.
- Utang pemasok.
- Piutang pelanggan bila fitur kredit diaktifkan.
- Kewajiban barang konsinyasi/bayar setelah terjual.

Jangan menyamakan omzet, laba, saldo kas, dan arus kas.

## 18. Rekonsiliasi kas dan pembayaran

- Setiap pembayaran terhubung ke transaksi sumber.
- Catat metode, nominal, waktu, referensi, petugas, dan status.
- Selisih kas dicatat sebagai kejadian dengan alasan.
- Transfer/QRIS dapat memiliki status menunggu konfirmasi bila bukti belum tersedia.
- Rekonsiliasi tidak boleh mengubah transaksi asal secara diam-diam.

## 19. Kontrol margin dan harga

- Simpan riwayat harga beli dan jual.
- Tampilkan margin perkiraan sebelum transaksi jika diizinkan.
- Peringatan jika harga jual di bawah batas margin yang dikonfigurasi.
- Override memerlukan alasan dan permission.
- Nilai HPP mengikuti metode yang disetujui dan konsisten.

**KEPUTUSAN TERBUKA:** metode costing final (misalnya FIFO atau weighted average), batas margin, penjualan kredit, dan struktur akuntansi penuh. Jangan mengimplementasikan asumsi akuntansi final sebelum keputusan disetujui.

---

# BAGIAN L — EXCEL ENGINE

## 20. Kewajiban XLSX

**WAJIB:** ekspor Excel `.xlsx` dengan formula yang berfungsi dan template yang dapat digunakan kembali. CSV/JSON tidak menggantikan XLSX.

### 20.1 Jenis workbook
1. **Operational Workbook:** produk, stok, penerimaan, penjualan, opname, retur, pesanan online.
2. **Financial Workbook:** pembayaran, utang pemasok, kas, HPP, laba, payroll.
3. **Management Workbook:** ringkasan KPI, tren, anomali, dan laporan pemilik.

### 20.2 Aturan workbook
- Gunakan nama sheet, header, format tanggal, mata uang Rupiah, dan satuan yang konsisten.
- Sertakan metadata periode, waktu ekspor, versi template, dan filter.
- Formula harus berupa formula spreadsheet asli, bukan sekadar angka hasil.
- Hindari macro/VBA sebagai ketergantungan.
- Beri proteksi/peringatan pada sel formula agar tidak mudah tertimpa.
- Sertakan petunjuk input dan definisi kolom.
- Data ekspor harus konsisten dengan filter/periode yang dipilih.
- Rumus penting harus diuji di Microsoft Excel dan LibreOffice.

### 20.3 Formula minimum
Contoh:
- Total baris = kuantitas × harga satuan − diskon.
- Total dokumen = penjumlahan total baris + biaya yang relevan.
- Saldo utang = nilai kewajiban − total alokasi pembayaran − kredit/retur yang sah.
- Selisih opname = jumlah fisik − jumlah sistem.
- Sisa payroll = gaji bersih disahkan − pembayaran tercatat.
- Margin = penjualan bersih − HPP.
- Rata-rata dan total sesuai filter/periode.

Formula final mengikuti definisi akuntansi dan skema data yang disetujui. Nilai authoritative tetap dihitung dan divalidasi server-side.

### 20.4 Impor XLSX
1. Unduh template versi aktif.
2. Pengguna mengisi file.
3. Unggah file.
4. Sistem memeriksa format, versi template, sheet, header, tipe, dan nilai.
5. Tampilkan preview.
6. Tandai error, duplikat, dan konflik.
7. Pengguna memilih koreksi/merge sesuai aturan.
8. Konfirmasi impor.
9. Commit dalam batch yang dapat dilacak.
10. Tampilkan ringkasan sukses/gagal dan ID batch.

### 20.5 Kriteria penerimaan
- File dapat dibuka tanpa peringatan kerusakan.
- Formula dapat dihitung di Excel dan LibreOffice.
- Template salah versi ditolak atau diproses melalui migrasi template yang jelas.
- Impor tidak menggandakan data ketika file yang sama dikirim ulang.
- Baris gagal tidak diam-diam dianggap sukses.
- Nilai formula penting dibandingkan dengan hasil perhitungan server pada test fixture.

---

# BAGIAN M — PENCARIAN, NOTIFIKASI, DAN DOKUMEN

## 21. Universal Search

Pencarian lintas modul minimal dapat menemukan:
- SKU/nama/ukuran ban.
- Nomor nota.
- ID pesanan marketplace.
- Nomor resi.
- Pemasok.
- Pelanggan.
- Pegawai sesuai permission.
- ID retur/garansi.
- Dokumen.

Hasil pencarian harus menerapkan permission yang sama dengan halaman asal. Jangan membocorkan nama atau ringkasan objek yang tidak boleh dilihat pengguna.

## 22. Notification & Task Center

Jenis notifikasi:
- Stok rendah/kosong.
- Pesanan online bermasalah.
- Persetujuan menunggu.
- Utang/jatuh tempo.
- Absensi gagal/perlu koreksi.
- Payroll perlu diperiksa.
- Backup gagal.
- Integrasi atau sinkronisasi gagal.
- Anomali rekonsiliasi.

Notifikasi harus memiliki status terbaca, belum terbaca, selesai, atau ditugaskan. Hindari duplikasi notifikasi untuk kejadian yang sama.

## 23. Document Management

- Simpan dokumen dengan ID, kategori, relasi transaksi, metadata, uploader, waktu, dan checksum.
- Batasi tipe dan ukuran file.
- Validasi file upload; jangan percaya ekstensi saja.
- Hak akses dokumen mengikuti objek dan permission.
- File sensitif tidak boleh tersedia melalui URL publik permanen.
- Dokumen yang dihapus mengikuti kebijakan retensi dan audit.

---

# BAGIAN N — SCANNER, OFFLINE, DAN SINKRONISASI

## 24. Universal scanner

Satu komponen scanner dapat membaca barcode/QR, tetapi setiap kode memiliki namespace dan tujuan berbeda:
- Produk.
- Absensi.
- Pesanan.
- Resi/label.
- Dokumen atau tautan internal.

Sistem harus mengidentifikasi jenis kode sebelum menjalankan aksi. QR absensi tidak boleh diproses sebagai QR produk.

## 25. Offline/cache/sync

### 25.1 Prinsip
- Cache lokal boleh digunakan untuk data referensi dan tampilan yang aman.
- Tindakan yang mengubah stok, uang, payroll, atau absensi harus memiliki kebijakan sinkronisasi dan validasi server.
- Jangan mengklaim transaksi final berhasil ketika belum dikonfirmasi server.
- Tampilkan status pending/offline secara eksplisit.

### 25.2 Konflik
- Setiap perubahan membawa versi/revision atau mekanisme concurrency.
- Konflik tidak boleh diselesaikan dengan last-write-wins untuk transaksi finansial/stok.
- Sediakan layar konflik dan proses rekonsiliasi.
- Simpan antrean lokal dengan ID idempoten bila offline action memang diizinkan.

### 25.3 Refresh
- Hindari refresh otomatis berulang saat pengguna berpindah halaman.
- Muat data saat masuk awal, ketika diperlukan, setelah aksi simpan yang relevan, atau atas permintaan refresh pengguna.
- Sediakan indikator kapan data terakhir diperbarui.
- Modul marketplace tidak boleh membebani POS dengan request yang tidak perlu.

---

# BAGIAN O — KEAMANAN, BACKUP, DAN KESEHATAN SISTEM

## 26. Keamanan aplikasi

Checklist wajib:
- TLS untuk komunikasi.
- Password disimpan dengan password hashing yang sesuai.
- Rahasia dan token tidak ditanam di frontend.
- Validasi input di server.
- Proteksi CSRF/XSS/SQL injection sesuai arsitektur.
- Rate limit untuk login, endpoint sensitif, scanner, dan ekspor.
- Otorisasi object-level untuk mencegah IDOR.
- Log aman tanpa kredensial/biometrik mentah.
- Dependensi dipindai dan diperbarui secara terkontrol.
- Prosedur pencabutan akun dan sesi.
- Review keamanan sebelum rilis.

## 27. Backup dan disaster recovery

**WAJIB**
- Backup database dan dokumen.
- Backup terjadwal sesuai kemampuan platform.
- Backup sebelum migrasi dan rilis berisiko.
- Enkripsi backup.
- Checksum/integrity check.
- Retensi dan lokasi backup yang terdokumentasi.
- Uji pemulihan berkala.
- Prosedur restore dan siapa yang berwenang.

Backup dianggap berhasil hanya jika dapat diverifikasi dan uji restore membuktikan data dapat dipulihkan.

## 28. System health dan quota monitoring

Pantau:
- Pemakaian database/storage.
- Batas request/CPU/egress atau batas platform terkait.
- Error rate dan latency.
- Kegagalan job/sinkronisasi.
- Umur backup terakhir.
- Ruang penyimpanan dokumen.
- Kegagalan ekspor/impor.
- Kesehatan layanan biometrik.

Ambang peringatan harus dapat dikonfigurasi. Jika batas gratis mendekati limit, sistem memberi peringatan dan prosedur mitigasi; jangan diam-diam berpindah ke layanan berbayar.

## 29. Prinsip free-first

- Prioritaskan opsi gratis yang aman dan layak.
- Verifikasi kuota aktual sebelum pemilihan platform dan sebelum produksi.
- Jangan mengandalkan fitur trial yang akan berbayar otomatis.
- Jangan menambahkan dependensi berbayar tanpa persetujuan eksplisit pemilik.
- Jika suatu kemampuan (terutama biometrik/liveness, backup, atau integrasi marketplace) tidak layak dilakukan gratis dengan tingkat keamanan yang dibutuhkan, dokumentasikan pilihan, risiko, dan konsekuensinya untuk keputusan pemilik.

---

# BAGIAN P — DEVELOPER CONTROL CENTER DAN GOVERNANCE

## 30. Pemisahan panel

Developer Control Center adalah alat teknis terpisah dari aplikasi operasional toko. Panel ini tidak boleh menjadi jalur pintas untuk melewati permission bisnis.

Fungsi yang direncanakan:
- Status aplikasi dan layanan.
- Versi/build.
- Riwayat deployment.
- Health check.
- Error dan audit teknis.
- Status backup/restore.
- Pemakaian kuota.
- Status migrasi.
- Hasil pengujian.
- Manajemen konfigurasi yang aman.
- Rollback sesuai prosedur.

## 31. Universal Audit & Update Engine

- Mendeteksi versi aplikasi dan perubahan skema.
- Menampilkan ringkasan perubahan sebelum update.
- Menjalankan migrasi skema yang terversi.
- Menyediakan backup/restore point sebelum perubahan berisiko.
- Mencatat hasil update.
- Menyediakan rollback aplikasi; rollback database harus melalui strategi kompatibilitas/migrasi balik yang teruji.
- Tidak menjalankan perubahan destruktif tanpa persetujuan dan backup terverifikasi.

## 32. Project Governance

Setiap perubahan harus memiliki:
- Requirement ID.
- Feature ID.
- Change request.
- Keputusan arsitektur bila relevan.
- Dampak modul/data/API.
- Risiko.
- Test case.
- Hasil pengujian.
- Persetujuan.
- Versi rilis dan catatan deployment.

Dokumen governance minimal: Requirement Registry, Feature Registry, Decision Register/ADR, Change Log, Known Issues, Test Results, Deployment History, Migration Records, dan Release Checklist.

---

# BAGIAN Q — MIGRASI DATA DARI APLIKASI LAMA

## 33. Prinsip migrasi selektif

- Aplikasi lama tetap tersedia sampai migrasi tervalidasi.
- Hanya data yang berguna, sah, dan dapat dipetakan yang dipindahkan.
- Data uji, duplikat, rusak, atau tidak valid dikecualikan dan dicatat.
- Jangan menghapus aplikasi/data lama sebelum pemilik menyetujui hasil migrasi.

### 33.1 Tahapan
1. Inventarisasi sumber data.
2. Backup sumber.
3. Profiling kualitas data.
4. Mapping field dan ID.
5. Tentukan aturan deduplikasi.
6. Import ke staging.
7. Validasi jumlah dan relasi.
8. Rekonsiliasi stok, transaksi, utang, dan pegawai.
9. Uji alur operasional.
10. Persetujuan pemilik.
11. Cutover.
12. Pantau hasil dan simpan rollback plan.

### 33.2 Kriteria penerimaan
- Jumlah record sumber, diterima, ditolak, dan duplikat dapat dijelaskan.
- Relasi transaksi tidak putus.
- Saldo stok dan utang direkonsiliasi.
- Data lama tetap tersedia sampai sign-off.
- Ada bukti backup dan hasil uji restore.

---

# BAGIAN R — PENGUJIAN DAN ACCEPTANCE

## 34. Jenis pengujian wajib

1. Unit test untuk aturan hitung dan validasi.
2. Integration test untuk alur antar-modul.
3. API contract test.
4. Permission/authorization test.
5. Security test.
6. UI test desktop dan smartphone.
7. Compatibility test browser/perangkat.
8. Import/export test XLSX.
9. Concurrency/idempotency test.
10. Offline/retry/conflict test.
11. Backup/restore drill.
12. Performance dan quota test.
13. Biometric liveness/spoof test.
14. User acceptance test oleh pemilik/admin/pegawai.

## 35. Skenario uji lintas modul

| ID | Skenario | Hasil yang diharapkan |
|---|---|---|
| T-001 | Buat produk dengan SKU baru | Produk tersimpan dan dapat dicari |
| T-002 | Buat SKU duplikat | Ditolak, data lama tidak berubah |
| T-003 | Jual satu item | Nota tercipta dan stok berkurang sekali |
| T-004 | Klik simpan berulang | Hanya satu transaksi tercipta |
| T-005 | Jual produk tanpa harga | Ditolak atau override berizin dengan alasan |
| T-006 | Terima barang konsinyasi | Kepemilikan tetap ditandai konsinyasi |
| T-007 | Cicil pembayaran pemasok | Saldo kewajiban berkurang sesuai alokasi |
| T-008 | Bayar setelah barang terjual | Kewajiban unit terkait diperbarui |
| T-009 | Impor pesanan sama dua kali | Tidak ada duplikasi |
| T-010 | Scan resi untuk pesanan lain | Ditolak |
| T-011 | Retur pelanggan | Masuk inspeksi, belum otomatis tersedia |
| T-012 | Stock opname draft | Saldo stok tidak berubah |
| T-013 | Setujui selisih opname | Adjustment tercatat dan dapat diaudit |
| T-014 | QR absensi kedaluwarsa | Absensi ditolak |
| T-015 | QR valid + wajah tidak cocok | Absensi ditolak |
| T-016 | QR valid + wajah cocok + liveness lolos | Absensi tersimpan dengan waktu server |
| T-017 | Foto/video replay | Ditolak pada test anti-spoof yang didukung |
| T-018 | Kamera/jaringan gagal | Tidak membuat absensi sukses otomatis |
| T-019 | Payroll draft | Dapat diperiksa dan dihitung ulang |
| T-020 | Payroll disahkan | Periode terkunci |
| T-021 | Pembayaran payroll parsial | Saldo sisa benar |
| T-022 | Pegawai membuka data pegawai lain | Akses ditolak |
| T-023 | XLSX export | File terbuka dan formula berfungsi |
| T-024 | XLSX import dengan duplikat | Konflik ditampilkan sebelum commit |
| T-025 | Reversal transaksi posted | Transaksi asal tetap ada dan reversal tertaut |
| T-026 | Restore backup | Data dan dokumen dapat dipulihkan |
| T-027 | Pengguna tanpa permission memanggil API | Server menolak |
| T-028 | Kuota mendekati batas | Peringatan muncul dan mitigasi tersedia |

## 36. Definition of Done

Sebuah fitur hanya boleh dinyatakan selesai bila:
- Kebutuhan dan acceptance criteria terpenuhi.
- UI desktop/mobile sesuai peran.
- Validasi client dan server berjalan.
- Permission server-side teruji.
- Audit tersedia untuk aksi sensitif.
- Kondisi gagal dan retry ditangani.
- Tidak ada duplikasi transaksi pada retry.
- Unit/integration/security test relevan lulus.
- Dokumentasi diperbarui.
- Tidak ada bug blocker/critical yang terbuka.
- Pemilik/admin menyetujui UAT bila fitur berdampak operasional.

---

# BAGIAN S — URUTAN IMPLEMENTASI

## 37. Tahapan pembangunan

### Fase 0 — Validasi dan keputusan
- Konfirmasi master requirement.
- Putuskan platform setelah benchmark biaya/kuota, keamanan, dan kemampuan.
- Putuskan kebijakan GPS absensi.
- Putuskan metode costing, aturan kredit, approval threshold, dan retensi biometrik.
- Buat threat model dan data classification.

**Gerbang:** tidak ada keputusan kritis yang tersembunyi sebagai asumsi.

### Fase 1 — Fondasi
- Repo, environment, CI/build.
- Database migration framework.
- Login, sesi, role, permission.
- Audit trail.
- Error handling dan observability.
- Backup dasar dan health check.

**Gerbang:** akses dan audit lolos pengujian.

### Fase 2 — Master data dan inventory
- Produk, pemasok, pelanggan.
- Stock ledger dan saldo.
- Penerimaan barang.
- Stock opname.

**Gerbang:** seluruh mutasi stok dapat ditelusuri dan direkonsiliasi.

### Fase 3 — POS dan keuangan inti
- Kasir, pembayaran, nota.
- Reversal/retur.
- Kewajiban pemasok dan alokasi pembayaran.
- Laporan penjualan, HPP, margin, kas.

**Gerbang:** skenario jual-beli dan utang lolos rekonsiliasi.

### Fase 4 — HR, absensi, payroll
- Data pegawai dan jadwal.
- QR dinamis.
- Enrollment wajah, liveness, matching, fallback.
- Izin/koreksi.
- Payroll dan slip.

**Gerbang:** uji privasi, spoofing, kegagalan perangkat, dan payroll disetujui.

### Fase 5 — Marketplace dan scanner
- Impor pesanan.
- Validasi stok.
- Picking/packing.
- Label/resi.
- Retur marketplace.

**Gerbang:** integrasi hanya memakai akses resmi/format yang tersedia dan idempotent.

### Fase 6 — Excel Engine dan laporan lanjutan
- Operational, Financial, Management workbook.
- Template impor.
- Formula dan validasi lintas aplikasi spreadsheet.

**Gerbang:** semua workbook lolos test fixture Excel/LibreOffice.

### Fase 7 — Developer Control Center dan automation
- Monitoring kuota.
- Update/migration orchestration.
- Backup/restore dashboard.
- Governance registry.
- Notification/task center dan universal search.

**Gerbang:** tidak ada operasi destruktif tanpa kontrol, backup, dan audit.

### Fase 8 — Migrasi, UAT, dan go-live
- Migrasi selektif.
- Rekonsiliasi.
- Pelatihan pengguna.
- UAT.
- Cutover bertahap.
- Monitoring intensif.
- Persetujuan final pemilik.

**Gerbang:** sign-off tertulis dan rollback plan tersedia.

---

# BAGIAN T — KEPUTUSAN YANG HARUS DIKUNCI

## 38. Daftar keputusan terbuka

| ID | Keputusan | Status | Rekomendasi awal |
|---|---|---|---|
| D-001 | Platform hosting/database | Terbuka | Benchmark beberapa opsi gratis sebelum memilih |
| D-002 | GPS pada absensi | Terbuka | Jangan aktifkan sebelum disetujui; QR + wajah tetap wajib |
| D-003 | Metode HPP/costing | Terbuka | Pilih satu metode konsisten setelah simulasi data |
| D-004 | Akuntansi double-entry | Terbuka | Mulai dari kontrol keuangan operasional dengan kesiapan ekspansi |
| D-005 | Penjualan kredit pelanggan | Terbuka | Nonaktif secara default sampai aturan piutang disetujui |
| D-006 | Batas approval transaksi | Terbuka | Konfigurasi oleh pemilik |
| D-007 | Threshold margin minimum | Terbuka | Konfigurasi per kategori/produk |
| D-008 | Retensi data biometrik | Terbuka | Minimasi data dan tetapkan jadwal penghapusan |
| D-009 | Kanal marketplace prioritas | Terbuka | Prioritaskan berdasarkan volume dan akses ekspor/API |
| D-010 | Printer/scanner perangkat | Terbuka | Uji perangkat nyata sebelum finalisasi |
| D-011 | Jumlah cabang aktif | Terbuka | Satu lokasi operasional awal, skema siap ekspansi |
| D-012 | Aturan lembur/telat/insentif | Terbuka | Konfigurasi setelah aturan kerja disahkan |

Tidak boleh mengubah status terbuka menjadi final tanpa keputusan pemilik yang tercatat.

---

# LAMPIRAN 1 — STATUS TRANSAKSI UMUM

Status berikut adalah pola awal; tiap modul dapat memiliki status khusus:
- Draft.
- Submitted.
- Pending Approval.
- Approved.
- Posted.
- Partially Paid.
- Paid.
- Rejected.
- Cancelled.
- Reversed.
- Closed.

Transisi status harus divalidasi server-side. Status terminal tidak boleh dibuka kembali tanpa prosedur khusus.

# LAMPIRAN 2 — FORMAT NOMOR DOKUMEN

Format yang direkomendasikan:
- `SALE-YYYYMMDD-XXXX`
- `PUR-YYYYMMDD-XXXX`
- `RCV-YYYYMMDD-XXXX`
- `RET-YYYYMMDD-XXXX`
- `SO-YYYYMMDD-XXXX`
- `PAY-YYYYMM-XXXX`
- `ATT-YYYYMMDD-XXXX`

Nomor dokumen bukan primary key database. ID internal harus tetap unik meskipun nomor tampilan berubah format.

# LAMPIRAN 3 — CHECKLIST RILIS

- [ ] Requirement dan perubahan disetujui.
- [ ] Migration database diuji.
- [ ] Backup sebelum rilis terverifikasi.
- [ ] Unit/integration/security test lulus.
- [ ] Permission dan audit diperiksa.
- [ ] Desktop dan smartphone diuji terpisah.
- [ ] XLSX diuji di Excel dan LibreOffice.
- [ ] Absensi QR + wajah diuji termasuk spoof/failure.
- [ ] Kuota dan biaya potensial diperiksa.
- [ ] Monitoring dan alert aktif.
- [ ] Rollback plan tersedia.
- [ ] UAT pemilik/admin selesai.
- [ ] Release notes dan deployment history diperbarui.

---

## Catatan final

Dokumen ini adalah spesifikasi implementasi tingkat fitur dan acceptance criteria. Sebelum coding produksi, dokumen ini harus diturunkan lagi menjadi task teknis yang dapat dikerjakan (per endpoint, migration, komponen UI, test case otomatis, dan deployment pipeline) setelah Technical Blueprint disepakati. Bila terjadi konflik, catat konflik tersebut dan selesaikan melalui Decision Register; jangan memilih salah satu secara diam-diam.

**Acuan keputusan penting:** absensi wajib QR + verifikasi wajah; XLSX dengan formula yang berfungsi wajib; antarmuka smartphone pegawai dan pemilik/admin terpisah dari worksheet desktop; sistem free-first tanpa dependensi berbayar yang ditambahkan tanpa persetujuan.
