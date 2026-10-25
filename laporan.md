# Laporan Modul 4 — Menghilangkan Duplikat dan Perbaikan Format Data

| Komponen | Isi |
|---|---|
| Nama |Bryan Paskah Telaumbanua |
| NIM |124450003 |
| Tanggal pengumpulan |25-10-2026|
| Seed dan ukuran data | seed 2026, ukuran kecil |
| Versi Python dan pandas | |
## 1. Ringkasan

Transaksi 24.288 baris menjadi 24.000 setelah 288 baris duplikat dibuang (192 eksak, 96 dekat);
omzet pesanan selesai terhitung ganda 1,24%. Pelanggan 5.760 id sumber disatukan menjadi
5.000 entitas tanpa menghapus satu id pun. Tanpa nomor HP, aturan nama + tanggal daftar WIB + tahun lahir
mencapai presisi 100,0% dan recall 99,7% terhadap acuan perak, sedangkan nama saja hanya berpresisi
1,7% (recall 100,0%).

## 2. Tabel pemetaan format

Setiap pemetaan dicatat pasangan demi pasangan di `keluaran/peta_format_m04.csv` dan `keluaran/peta_wilayah_m04.csv`; nilai yang belum
terpetakan menghentikan notebook (`petakan()`), dan semua peta lulus uji idempoten.

| kolom | penulisan_asli | kanonik | baris_berubah | pasangan_butuh_pengetahuan_domain |
|---|---|---|---|---|
| pelanggan_app.city | 28 | 10 | 1.700 | 11 |
| pelanggan_app.gender | 5 | 2 | 2.488 | 4 |
| pelanggan_web.jenis_kelamin | 7 | 2 | 1.993 | 2 |
| pelanggan_web.kota | 28 | 10 | 2.142 | 11 |
| pelanggan_web.provinsi | 20 | 10 | 1.591 | 7 |
| produk.kategori | 23 | 7 | 27 | 0 |
| produk.perlu_pendingin | 6 | 2 | 850 | 6 |
| transaksi.kota_tujuan | 28 | 10 | 15.941 | 11 |
| transaksi.metode_bayar | 16 | 5 | 17.265 | 6 |
| transaksi.provinsi_tujuan | 20 | 10 | 12.029 | 7 |
| transaksi.status | 8 | 4 | 466 | 0 |

| Pemeriksaan | Hasil | Sumber |
|---|---:|---|
| status: penulisan → kanonik; baris kapital | 8 → 4; 466 | A.2 |
| metode bayar: penulisan → kategori; baris berubah; merek OVO/GoPay | 16 → 5; 17.265; 4.180 | A.2 |
| kota / provinsi: penulisan → kanonik | 28 → 10 / 20 → 10 | A.3 |
| usulan RapidFuzz salah pada penulisan yang tidak terpetakan otomatis | 3 dari 9 | A.3 |
| kota-provinsi tidak konsisten setelah dipetakan (transaksi / web) | 0 / 0 | A.3 |
| waktu di luar masa operasi → `NaT` | 18 ({'1970': 13, '2062': 5}) | A.4 |
| pesanan aplikasi yang berganti tanggal / bulan setelah UTC → WIB | 3.243 dari 11.111 / 102 | A.4 |
| diskon pecahan dikali 100 | 515 baris (semuanya aplikasi) | A.5 |
| berat bertitik desimal; cara naif salah | 158; 158 produk | A.5 |
| kategori produk: penulisan → kanonik; `str.title()` salah | 23 → 7; 107 produk | A.5 |

## 3. Duplikat transaksi dan dampaknya

| Pemeriksaan | Hasil | Sumber |
|---|---:|---|
| salinan eksak (18 kolom identik) | 192 | B.1 |
| salinan dekat (17 kolom identik, id `LS-tttt-9xxxxx`) | 96 (34 dari 96 tanpa `baris_ke` 1) | B.1–B.2 |
| baris dibuang → tersisa | 288 → 24.000 | B.1, D.5 |
| `drop_duplicates("id_pesanan")` akan membuang baris sah | 8.448 | B.2 |
| omzet pesanan selesai dengan / tanpa duplikat (juta Rp) | 8.123,0 / 8.023,2 (+1,24%) | B.3 |
| bulan terdampak terbesar | 2026-06 (+2,23%) | B.3 |
| `total_bayar` tepat +Rp10.000 (ongkir teramati) dikoreksi | 115 (112 setelah duplikat dibuang) | B.4 |
| `total_bayar` tidak dapat diperiksa | 1.112 | B.4 |

Kunci bisnis adalah `(id_pesanan, baris_ke)`. Anomali ikut tersalin: harga ×100 119 → 117,
jumlah negatif 70 → 68, diskon pecahan 515 → 510,
kode yatim 77 → 76; register Modul 1 karena itu melebih-lebihkan setiap anomali (B.3).

## 4. Duplikat pelanggan dan *blocking*

| Pemeriksaan | Hasil | Sumber |
|---|---:|---|
| salinan eksak web → id unik | 100 → 3.160 | C.1 |
| email dipakai lebih dari satu id (berbeda orang) | 20 email, 40 id | C.1 |
| nama berbeda untuk 3.160 pelanggan | 704 | C.1 |
| pasangan tanpa *blocking* / blok `no_hp` / blok `tgl_lahir` | 4.991.220 / 60 / 3.573 | C.2 |
| penanda `1900-01-01` (blok terbesar) | 80 pelanggan | C.2 |
| pasangan web–web diterima (HP sama dan nama cocok) | 60 | C.4 |
| transaksi yang merujuk id pendaftaran ulang | 259 baris, 54 id | C.4 |

*Blocking* mengubah biaya dan mutu sekaligus: blok `no_hp` hanya 60 pasangan kandidat kuat, sedangkan blok `tgl_lahir`
tercemar penanda 1900. Nama berinisial (`irfan w.`) hanya berskor `token_sort_ratio` 35–74; Jaro-Winkler minimum pada blok HP 0,846,
sehingga ambang JW 0,85 dipakai bersama aturan inisial. Pendaftaran ulang tidak dihapus agar transaksinya tidak yatim.

## 5. Resolusi entitas

### 5.1 Web ↔ aplikasi dengan nomor HP (Bagian D)

| tingkat | kandidat | tanggal_daftar_sama | tahun_lahir_bertentangan | didukung | jw_median |
|---|---|---|---|---|---|
| 1 no_hp | 716 | 716 | 0 | 716 | 1.0 |
| 2 email lokal | 69 | 0 | 62 | 0 | 1.0 |
| 3 nama sekota | 2534 | 1 | 2167 | 0 | 0.916 |

Tautan diterima: {'hp_sama': 716}. Hasil: 5.760 id → **5.000 entitas** (tunggal 4.256, gabungan {'app+web': 684, 'web+web': 44, 'app+web+web': 16});
tahun lahir bertentangan di dalam entitas 0. Tanggal daftar sama pada tautan HP: 100,0% dalam WIB,
69,7% bila aplikasi dibiarkan UTC. Sampel D.4: 30/30 pasangan benar, selang Wilson 95% (0.886, 1.0).

### 5.2 Tanpa nomor HP: presisi dan recall terhadap acuan perak (Tugas 1)

Acuan perak = 716 pasangan tautan `no_hp` (`tk1`). Semesta kandidat 42.100 pasangan
(nama sekota JW ≥ 0,80: 41.999; email lokal sama: 814). Tabel lengkap: `keluaran/ambang_entitas_m04.csv`, grafik `keluaran/04_ambang_entitas.png`.

| aturan | ambang | diterima | benar | presisi | recall |
|---|---|---|---|---|---|
| nama_sekota | 0.85 | 23632 | 708 | 0.03 | 0.9888 |
| nama_sekota | 0.9 | 5362 | 605 | 0.1128 | 0.845 |
| nama_sekota | 0.95 | 1981 | 600 | 0.3029 | 0.838 |
| nama_sekota+tgl+lahir | 0.85 | 713 | 708 | 0.993 | 0.9888 |
| nama_sekota+tgl+lahir | 0.9 | 605 | 605 | 1.0 | 0.845 |
| nama_sekota+tgl+lahir | 0.95 | 600 | 600 | 1.0 | 0.838 |
| nama_atau_inisial+tgl+lahir | 0.85 | 720 | 715 | 0.9931 | 0.9986 |
| nama_atau_inisial+tgl+lahir | 0.9 | 714 | 714 | 1.0 | 0.9972 |
| nama_atau_inisial+tgl+lahir | 0.95 | 714 | 714 | 1.0 | 0.9972 |
| email_lokal | 0.0 | 814 | 700 | 0.86 | 0.9777 |
| email_lokal | 0.85 | 807 | 693 | 0.8587 | 0.9679 |
| email_lokal | 0.9 | 697 | 596 | 0.8551 | 0.8324 |
| email_lokal | 0.95 | 692 | 591 | 0.854 | 0.8254 |
| email_lokal+tgl+lahir | 0.0 | 700 | 700 | 1.0 | 0.9777 |
| email_lokal+tgl+lahir | 0.85 | 693 | 693 | 1.0 | 0.9679 |
| email_lokal+tgl+lahir | 0.9 | 596 | 596 | 1.0 | 0.8324 |
| email_lokal+tgl+lahir | 0.95 | 591 | 591 | 1.0 | 0.8254 |

Catatan: `ambang` 0,0 pada aturan email berarti tanpa syarat skor nama; `tgl` = tanggal daftar WIB sama; `lahir` = tahun lahir tidak bertentangan.

- Nama saja berpresisi rendah di semua ambang (≤ 31,1%) karena banyak orang berbagi nama dan kota.
- Tanggal daftar WIB sama dan tahun lahir tidak bertentangan menaikkan presisi menjadi ≈ 100%, tetapi pada JW ≥ 0,90 recall turun ke
  84,5% karena nama berinisial terbuang; aturan inisial memulihkannya ke 99,7%.
- Email lokal saja: presisi 86,0%, recall 97,8%; email lokal sama pada 700 dari 716 pasangan acuan.
- Aturan cadangan yang dipilih: (email lokal ATAU nama sekota JW ≥ 0,90 ATAU inisial) DAN tanggal daftar WIB sama DAN tahun lahir tidak bertentangan.

### 5.3 Sampel berlabel sendiri (Tugas 2)

48 pasangan di sekitar JW 0,90 (strata pita JW × tanggal daftar sama), diberi label tangan tanpa melihat acuan perak
(`keluaran/label_pasangan_m04.csv`): sama 20, beda 28, ragu 0. Label sepakat dengan acuan pada 48/48
(Wilson 95% (0.926, 1.0)). Presisi aturan nama saja pada JW ≥ 0,90 (5.362 pasangan): taksiran dari label
dengan penimbang strata 11,3%, acuan perak 11,3%. Di antara pasangan yang lolos tanggal daftar sama
(n = 24), 20 berlabel sama dan 20 sama menurut acuan.

## 6. Latihan 4.1 dan 4.2

**Latihan 4.1** (`latihan-tanggal.md`, `parser_tanggal_m04.csv`): tiga pola `tgl_daftar`; garis miring bagian 1 maks 31, bagian 2 maks 12 → dd/mm.
`format="mixed"` salah pada 342 baris dan `mixed` + `dayfirst=True` pada 625, tanpa galat atau peringatan. Tanggal daftar sama pada tautan HP:
eksplisit 100,0%, mixed 89,2%, mixed+dayfirst 79,3%, aplikasi UTC 69,7%.

**Latihan 4.2** (`latihan-produk.md`, `konsistensi_toko_m04.csv`): kode dan nama produk unik (850 baris), 0 produk kembar selain kode dan nama; nomor nama = nomor kode pada
850/850; kata wilayah 'Riau' ambigu pada 163 baris. `provinsi_toko` berbeda dalam satu toko pada 166/177 toko
(acak ≈ 10 toko konsisten, nyata 11). Tabel toko tidak diturunkan dari katalog: memaksa satu provinsi per toko mengubah 544 produk.

## 7. Rekonsiliasi baris

Selisih = baris_sebelum − baris_sesudah (positif = baris berkurang).

| tabel | langkah | baris_sebelum | baris_sesudah | selisih |
|---|---|---|---|---|
| transaksi | data mentah -> transaksi_m03 (Modul 3) | 24288 | 24288 | 0 |
| transaksi | buang salinan eksak (18 kolom identik) | 24288 | 24096 | 192 |
| transaksi | buang salinan dekat (17 kolom identik, id 9xxxxx) | 24096 | 24000 | 96 |
| transaksi | transaksi_m03 -> transaksi_m04 | 24288 | 24000 | 288 |
| transaksi | baris ber-pelanggan terdaftar (bukan pembeli tamu) | 23870 | 23589 | 281 |
| pelanggan_web | data mentah -> pelanggan_m04 (sumber web) | 3260 | 3160 | 100 |
| pelanggan_app | data mentah -> pelanggan_m04 (sumber app) | 2600 | 2600 | 0 |
| pelanggan | web + app mentah -> pelanggan_m04 | 5860 | 5760 | 100 |
| entitas | id sumber -> id_entitas | 5760 | 5000 | 760 |
| produk | produk.csv -> produk_m04 | 850 | 850 | 0 |

Setiap selisih dibuktikan oleh berkas lain (assert di sel Tugas 3): `duplikat_transaksi_m04.csv` (288 baris; `baris_m04 + dibuang = baris_m03`), kolom
`salinan_eksak_dibuang` (100), dan `peta_entitas_m04.csv` (760 penyatuan = 716 tautan HP + 60 tautan web–web − 16 sisi berlebih pada komponen tiga id).
Transaksi yatim terhadap `pelanggan_m04`: 0; pembeli tamu 418 → 411 (7 berada di antara duplikat).

## 8. Keterbatasan dan risiko sisa

1. **`total_bayar` pada baris berongkir isian (1.112 baris) tidak dapat diperiksa.** Pada baris berongkir teramati, 115 dari 23.176 (0,50%,
   Wilson 95% 0,40–0,60%) lebih Rp10.000. Bila laju yang sama berlaku, sekitar **5,5 baris** (4,4–6,7) pada kelompok tak terperiksa
   masih salah. Cara memperkecil: koreksi nilai dahulu, imputasi ongkir sesudahnya (pipeline Modul 9); cara memperkirakan: asumsi laju sama diuji dengan sampel baris yang ongkir aslinya diperoleh dari sumber lain.
2. **Recall tautan pelanggan tidak diketahui.** Dari 2.603 kandidat tingkat 2–3 yang ditolak, hanya 1 bertanggal daftar sama (dan tahun lahirnya bertentangan), sehingga
   pasangan benar yang terlewat diperkirakan sangat sedikit. Namun orang yang berganti nomor HP dan email tidak pernah menjadi kandidat. Cara memperkirakan: ambil sampel acak ≥ 100 pasangan
   yang **ditolak** pada JW 0,80–0,90, beri label tangan, lalu recall ≈ benar_diterima / (benar_diterima + dugaan_benar_ditolak), dengan selang Wilson.
3. **Acuan perak tidak independen.** Tautan HP dipakai sebagai kebenaran, tetapi seluruh tautan itu bertanggal daftar sama; aturan yang memakai tanggal daftar karenanya hampir pasti
   berrecall tinggi. Label tangan sepakat pada 48/48 (batas bawah Wilson 0.926), tetapi bukti tanggal daftar dipakai oleh label dan aturan sekaligus. Cara memperkirakan:
   label sampel pasangan tanpa menampilkan tanggal daftar, lalu bandingkan kembali.
4. Tahun 2062 (5 baris) tidak dikoreksi menjadi 2026 dan menjadi `NaT`; bila ingin dipulihkan, buktikan lebih dahulu dengan sampel baris yang urutan `id_pesanan`-nya menempatkannya di 2026.

## 9. Berkas keluaran

`transaksi_m04.parquet`, `pelanggan_m04.parquet`, `produk_m04.parquet`, `peta_format_m04.csv`, `peta_wilayah_m04.csv`, `duplikat_transaksi_m04.csv`, `blocking_m04.csv`,
`pasangan_web_m04.csv`, `peta_entitas_m04.csv`, `sampel_label_m04.csv`, `parser_tanggal_m04.csv`, `konsistensi_toko_m04.csv`, `ambang_entitas_m04.csv`, `label_pasangan_m04.csv`,
`rekonsiliasi_m04.csv`, grafik `01_waktu_format.png`, `02_duplikat_omzet.png`, `03_skor_nama.png`, `04_ambang_entitas.png`, dan `log-keputusan.csv` (salinan).
