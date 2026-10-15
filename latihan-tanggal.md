# Latihan 4.1 — Parser tanggal daftar pelanggan web

| Komponen | Isi |
|---|---|
| Nama | Bryan Paskah Telaumbanua |
| NIM | 124450003 |
| Sel notebook | `m04-l063` (Langkah 1–6), `04_duplikat.ipynb` |
| Berkas keluaran | `keluaran/parser_tanggal_m04.csv`; baris `K-04-013` pada `log-keputusan.csv` |
| Data | `pelanggan_web.csv`: 3.260 baris, 3.160 id unik setelah 100 salinan eksak dibuang (`wu`) |
| Versi | Angka dihitung dengan pandas 3.0.2; jalankan ulang sel untuk memastikan di versi Anda (lihat catatan nomor 3) |

## 1. Pola penulisan `tgl_daftar`

Setiap huruf diganti `a` dan setiap angka diganti `9`, lalu baris dihitung per pola (Langkah 1).

| Pola | Bentuk | Banyak baris | Persen dari 3.160 | Contoh |
|---|---|---:|---:|---|
| `9999-99-99` | ISO `yyyy-mm-dd` | 1.591 | 50,3% | 2026-04-27; 2024-11-10; 2026-03-31 |
| `99/99/9999` | bergaris miring | 964 | 30,5% | 07/07/2024; 05/10/2024; 16/12/2024 |
| `99-aaa-9999` | `dd-Mon-yyyy` | 605 | 19,1% | 10-Nov-2024; 05-May-2025; 23-Aug-2025 |

Pola ISO dan `dd-Mon-yyyy` tidak ambigu. Hanya pola bergaris miring yang dapat dibaca dalam dua urutan.

## 2. Bukti urutan hari/bulan pada pola bergaris miring

Kedua bagian pada 964 baris bergaris miring dipisah dengan `split("/")` (Langkah 2).

| Ukuran | Bagian pertama | Bagian kedua |
|---|---:|---:|
| Nilai maksimum | **31** | **12** |
| Banyak baris yang nilainya > 12 | **593** | **0** |

- Bagian kedua tidak pernah melebihi 12, sedangkan bagian pertama mencapai 31 pada 593 baris. Bila urutannya `mm/dd`, bagian pertama (bulan) tidak mungkin 31. Bila urutannya `dd/mm`, semua nilai masuk akal. Jadi urutannya **hari/bulan (`dd/mm`)**.
- Sisanya, 964 − 593 = **371 baris**, memiliki kedua bagian ≤ 12 sehingga dua bacaan sama-sama sah. Dari 371 baris itu, **29** berhari sama dengan bulan (misalnya `07/07/2024`), jadi tertukar atau tidak hasilnya sama. Yang benar-benar dapat tertukar: 371 − 29 = **342 baris**.

## 3. Parser otomatis versus parser eksplisit

`tgl_web` (parser eksplisit D.1) mencoba format `%Y-%m-%d`, `%d/%m/%Y`, `%d-%b-%Y` secara berurutan dan berhenti bila ada sisa. Ia dijadikan acuan, lalu dibandingkan dengan dua pemanggilan `pd.to_datetime` (Langkah 3).

| Pola | Banyak baris | Beda dari `tgl_web` dengan `format="mixed"` | Beda dengan `format="mixed", dayfirst=True` |
|---|---:|---:|---:|
| `9999-99-99` | 1.591 | 0 | **625** |
| `99/99/9999` | 964 | **342** | 0 |
| `99-aaa-9999` | 605 | 0 | 0 |
| **Jumlah** | 3.160 | **342** | **625** |

Kedua pemanggilan **tidak menghasilkan galat maupun peringatan**. Sel mencetak daftar peringatannya: kosong. Penjelasan pola per pola:

- **`mixed` tanpa `dayfirst`.** Setiap elemen diurai sendiri-sendiri dengan dugaan bulan-dulu. Pada `16/12/2024` dugaan itu mustahil (bulan 16), sehingga parser jatuh ke hari-dulu dan hasilnya benar. Pada `05/10/2024` kedua bacaan sah, sehingga parser memilih bulan-dulu: **10 Mei 2024**, padahal tanggal sebenarnya **5 Oktober 2024**. Itulah 342 baris yang salah. Tidak ada yang gagal, jadi tidak ada galat.
- **`mixed` dengan `dayfirst=True`.** Pola bergaris miring menjadi benar (0 beda), tetapi tanggal **ISO** ikut terkena: `2024-11-10` terbaca `2024-10-11`. Dari 1.591 baris ISO, 668 berhari ≤ 12. Dari 668 itu, 43 berhari sama dengan bulan sehingga aman, dan sisanya **625 baris** tertukar. Hasil keluaran sel mencetak bahwa kedua hitungan (342 dan 625) sama dengan selisih parser (`True`).
- **`dd-Mon-yyyy`** aman pada kedua pemanggilan karena nama bulannya tidak ambigu.
- **Mengapa berbahaya.** Tanggal yang tertukar tetap tanggal sah, sehingga tidak muncul galat. Kesalahan hanya tampak lewat rentang: data eksplisit berada pada 2024-07-01 s.d. 2026-05-31, sedangkan `mixed` berakhir pada 2026-12-04 (43 baris melewati akhir data) dan `mixed` + `dayfirst` pada 2026-12-05 (75 baris). Pemeriksaan rentang hanya menangkap 43 dari 342 (12,6%) dan 75 dari 625 (12,0%) baris yang salah. Sisanya tertukar ke tanggal yang masih masuk akal.

Catatan versi: perilaku `dayfirst=True` terhadap string ISO diamati pada pandas 3.0.2. Angka 625 mengikuti keluaran sel pada versi pandas yang Anda pakai.

## 4. Dampak pada pencocokan Bagian D (tautan tingkat 1)

Tautan tingkat 1 adalah **716 pasangan** web–aplikasi yang nomor HP E.164-nya sama (Langkah 4). Tabel menunjukkan persen pasangan yang tanggal daftarnya sama. Tahun lahir bertentangan pada tingkat ini 0, sehingga pasangan diterima persis bila tanggal daftarnya sama.

| Parser tanggal web | Aplikasi dalam WIB | Aplikasi dibiarkan UTC |
|---|---:|---:|
| Eksplisit (`tgl_web`) | **716 dari 716 = 100,0%** | 499 dari 716 = 69,7% |
| `mixed` | 639 dari 716 = 89,2% | 446 dari 716 = 62,3% |
| `mixed, dayfirst=True` | 568 dari 716 = 79,3% | 400 dari 716 = 55,9% |

- Pasangan yang sebenarnya orang yang sama ditolak karena tanggal salah: **77** pasangan karena `mixed`, **148** karena `mixed` + `dayfirst`, dan **217** karena aplikasi dibiarkan UTC (pendaftaran pukul 00.00–07.00 WIB jatuh pada tanggal UTC sebelumnya).
- Dua kesalahan ini bertumpuk: kombinasi terburuk (`dayfirst` + UTC) hanya menyetujui 55,9% pasangan, padahal semuanya adalah pasangan yang benar (nomor HP sama dan tidak ada tahun lahir bertentangan).
- Pada kaskade D, pasangan yang diterima turun dari 716 menjadi 639 atau 568, sehingga lebih banyak orang yang sama menjadi entitas terpisah. Jumlah entitas akan melampaui 5.000. Besarnya tidak diukur pada latihan ini.

## 5. Keputusan

- Tanggal daftar web **diurai per pola dengan format eksplisit** (`%Y-%m-%d`, `%d/%m/%Y`, `%d-%b-%Y`) lewat `urai_tanggal`, dengan urutan hari/bulan yang dibuktikan pada nomor 2. `format="mixed"` dan `dayfirst=True` **tidak dipakai** untuk kolom ini.
- Teks asli tetap tersimpan pada `tanggal_daftar_asli`. Tabel `keluaran/parser_tanggal_m04.csv` berisi satu baris per pola dengan kolom `pola`, `n_baris`, `contoh`, `maks_bagian_1`, `maks_bagian_2`, `beda_mixed`, `beda_mixed_dayfirst`. Kolom `maks_bagian_*` hanya diisi untuk pola bergaris miring.
- Keputusan dicatat pada `log-keputusan.csv` sebagai **`K-04-013`**.
- Alternatif yang dipertimbangkan: `mixed` (ditolak karena menukar 342 tanggal tanpa galat) dan `mixed, dayfirst=True` (ditolak karena menukar 625 tanggal ISO).
