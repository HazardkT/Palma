# Palma Ledger

Aplikasi frontend akuntansi pendapatan dan piutang usaha untuk operasional hospitality. Dibuat dengan HTML, CSS, JavaScript, dan Supabase tanpa framework build.

## Struktur folder

```text
frontend/
	index.html
	css/styles.css
	js/app.js
backend/
	supabase/schema.sql
	supabase/seed.example.sql
	supabase/room_revenue_only.sql
	supabase/single_hotel_only.sql
	supabase/direct_paid_invoice.sql
	supabase/invoice_cancellation.sql
docs/
	erd.md
```

## Alur data

ERD dan alur bisnis ada di [docs/erd.md](docs/erd.md). Data operasional bergerak dari tamu/perusahaan ke reservasi dan folio, lalu diterbitkan sebagai faktur. Penerimaan dialokasikan ke faktur; saldo terbuka dan aging dihitung dari alokasi aktual. RPC `issue_invoice` serta `receive_payment` menulis jurnal debit-kredit secara atomik.

## Menjalankan frontend

Buka [frontend/index.html](frontend/index.html) langsung di browser. Aplikasi mulai dalam mode demo; pembuatan faktur, pencatatan penerimaan, pencarian, filter, dan ekspor CSV dapat dicoba tanpa koneksi Supabase. Font dan ikon memerlukan koneksi internet.

## Menghubungkan Supabase

1. Buat project Supabase dan jalankan seluruh [backend/supabase/schema.sql](backend/supabase/schema.sql) melalui SQL Editor.
2. Untuk database lama, jalankan migrasi yang diperlukan: [room_revenue_only.sql](backend/supabase/room_revenue_only.sql), [single_hotel_only.sql](backend/supabase/single_hotel_only.sql), [direct_paid_invoice.sql](backend/supabase/direct_paid_invoice.sql), dan [invoice_cancellation.sql](backend/supabase/invoice_cancellation.sql). Migrasi hotel tunggal sengaja gagal jika masih ada beberapa record hotel; konsolidasikan data secara eksplisit terlebih dahulu, karena migrasi tidak menghapus data.
3. Buat pengguna pada Supabase Auth, lalu jalankan seed contoh di [backend/supabase/seed.example.sql](backend/supabase/seed.example.sql) setelah mengganti email pemilik jika diperlukan.
4. Isi `url` dan `anonKey` pada konstanta `PALMA_SUPABASE` di [frontend/js/app.js](frontend/js/app.js). Gunakan project URL dan publishable/anon key, bukan service-role key.
5. Buka [frontend/index.html](frontend/index.html), pilih avatar di sidebar, lalu masuk memakai kredensial Supabase Auth tersebut.

Sistem dikonfigurasi untuk satu hotel tanpa pemilih cabang. Tabel `properties` tetap dipakai sebagai identitas hotel dan kunci RLS. Pendapatan hanya mencatat subtotal kamar melalui akun aktif `4101`, sebelum pajak; trigger database menolak akun pendapatan lain. Saat membuat faktur, opsi **Langsung lunas** sekaligus membuat penerimaan dan alokasi dengan metode yang dipilih. RPC faktur memerlukan akun piutang dan pajak keluaran, sementara RPC penerimaan memerlukan akun kas dan piutang. Seed contoh membuat akun awal beserta pemetaan hotel. Semua tabel transaksi memakai RLS berdasarkan keanggotaan `property_members`.

Halaman **Faktur & Piutang** dapat membatalkan faktur terbit dengan alasan. Jika pembayaran sudah diterima, pengguna harus mengonfirmasi refund eksternal terlebih dahulu; sistem lalu mencatat refund dan jurnal pembalik tanpa menghapus jurnal asli. Untuk database lama, jalankan migrasi `invoice_cancellation.sql` sebelum memakai fitur ini.

Halaman **Bagan Akun** menampilkan **Buku Jurnal Umum** dengan filter semua akun, Piutang Usaha (`1201`), dan Pendapatan Kamar (`4101`), serta cetak PDF dan ekspor Excel. Mode demo membentuk jurnal dari faktur dan penerimaan contoh; saat login, laporan membaca jurnal berstatus posted dari Supabase. Faktur langsung lunas mencatat Debit Kas/Bank dan Kredit Pendapatan/Pajak dalam satu entri jurnal; faktur kredit tetap mendebit Piutang Usaha.

## Catatan akuntansi

- Faktur berstatus draft tidak dihitung sebagai piutang.
- Faktur terbit mendebit piutang dan mengkredit akun pendapatan serta pajak keluaran.
- Penerimaan mendebit kas dan mengkredit piutang; alokasi parsial diperbolehkan.
- Jurnal berstatus posted harus seimbang. Total faktur dan saldo tidak disimpan sebagai angka turunan.
- Schema ini adalah fondasi operasional. Sebelum produksi, tinjau pemetaan pajak lokal, kebijakan akses per role, backup, dan proses koreksi/void bersama akuntan hotel.
