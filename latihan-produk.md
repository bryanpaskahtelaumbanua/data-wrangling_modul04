# Latihan 4.2 — Duplikat produk dan toko?

| Komponen | Isi |
|---|---|
| Nama | Bryan Paskah Telaumbanua |
| NIM | 124450003 |
| Sel notebook | `m04-l067` (Langkah 1–4), `04_duplikat.ipynb` |
| Berkas keluaran | `keluaran/konsistensi_toko_m04.csv`; baris `K-04-014` pada `log-keputusan.csv` |
| Data | `produk.csv` mentah (850 baris) dan `produk_m04` hasil standardisasi Bagian B |

## 1. Keunikan produk dan kandidat duplikat dekat

| Pemeriksaan | Hasil |
|---|---|
| `kode_produk` unik? | **Ya**, 850 dari 850 |
| `nama_produk` unik? | **Ya**, 850 dari 850 |
| Baris identik pada seluruh kolom | 0 |
| Produk identik pada seluruh kolom selain kode dan nama | **0** (data mentah) dan **0** (setelah format dibakukan) |
| Kandidat lemah: `id_toko`, `kategori`, `harga_katalog_rp`, `berat_gram` sama | 2 baris (1 pasangan) |

- **Keunikan nama bukan bukti tidak ada duplikat.** Nomor di ujung nama sama dengan nomor di kode pada 850 dari 850 produk, dan nomor kode berurutan 1 sampai 850 (lihat nomor 2). Nama unik karena dibangun dengan nomor urut, bukan karena tidak ada produk ganda. Karena itu pemeriksaan dilakukan pada kolom lain.
- **Satu pasangan lemah** (Langkah 1): `LS-MAK-0068` (Makanan Ringan Jambi 68) dan `LS-MAK-0215` (Makanan Ringan Bengkulu 215). Keduanya dari toko T127, kategori Makanan Ringan, harga Rp52.000, berat 200 g. Provinsi berbeda (Jambi dan Bengkulu) dan rating berbeda (4,0 dan 3,9). Karena atribut lain berbeda dan harga hanya memiliki 60–124 nilai berbeda per kategori, kecocokan satu pasangan di antara 850 produk wajar sebagai kebetulan. **Keduanya dipertahankan.**
- **Keputusan:** tidak ada produk dibuang dan tabel duplikat produk kosong. Rekonsiliasi: 850 baris sebelum, 850 sesudah.

## 2. Konsistensi bagian-bagian `nama_produk`

Nama berbentuk `<kategori> <wilayah> <nomor>`. Setiap bagian diuji terhadap kolom lain (Langkah 2).

| Bagian nama | Dibandingkan dengan | Hasil |
|---|---|---|
| Kategori (awalan) | `kategori` kanonik | **850 dari 850** cocok |
| Kategori (awalan) | `kategori` penulisan asli | 823 dari 850 cocok |
| Nomor (akhiran) | nomor pada `kode_produk` | **850 dari 850** sama |
| (pembanding) | tiga huruf kode `LS-KOP-…` vs kategori | 850 dari 850 cocok |
| Wilayah | `provinsi_toko` | 687 baris satu-ke-satu, **163 baris ambigu** |

- **Kategori.** Nama selalu memakai ejaan kanonik, sedangkan kolom `kategori` mentah berbeda pada 27 produk (huruf kapital dan spasi ujung, A-18). Selisih 850 − 823 = 27 sama dengan jumlah produk yang diubah di Bagian B. Ini bukti tambahan bahwa pilihan "ejaan terbanyak" benar.
- **Nomor.** Nomor nama selalu sama dengan nomor kode, sehingga nomor tidak memberi informasi baru.
- **Wilayah.** Ada **9 kata wilayah** untuk **10 provinsi**:
  - 8 kata cocok satu-ke-satu dengan `provinsi_toko`: Aceh (88), Barat (102), Belitung (98), Bengkulu (85), Jambi (86), Lampung (80), Selatan (75), Utara (73), jumlahnya 687 baris. Namun `Barat`, `Selatan`, `Utara` adalah arah tanpa kata "Sumatera", dan `Belitung` tanpa "Bangka". Kata itu hanya dapat dipetakan bila tahu domainnya, tidak lewat kemiripan teks.
  - **`Riau` ambigu pada 163 baris:** 79 berprovinsi **Riau** dan 84 berprovinsi **Kepulauan Riau**. Kata wilayah itu tidak membedakan keduanya.
- **Akibat:** wilayah dalam nama **tidak boleh** dipakai untuk menurunkan atau memeriksa `provinsi_toko`. Sumbernya tetap kolom `provinsi_toko`.

## 3. Konsistensi atribut per `id_toko`

Tabel lengkap ada pada `keluaran/konsistensi_toko_m04.csv` (kolom `id_toko`, `n_produk`, `n_nama_toko`, `n_provinsi_toko`, `rating_min`, `rating_maks`, `n_rating_penanda`, `rentang_rating`). Rentang rating dihitung **tanpa penanda** 0,0 (A-19, 61 produk). Ringkasan (Langkah 3):

| Ukuran | Hasil |
|---|---|
| Banyak toko | **177**; 8 toko hanya berproduk satu; rata-rata 4,8 produk per toko, maksimum 12 |
| `nama_toko` berbeda dalam satu toko | **0 toko**; nama selalu `"Toko "` + nomor `id_toko` |
| `provinsi_toko` berbeda dalam satu toko | **166 dari 177 toko** (166 dari 169 toko yang berproduk ≥ 2); rata-rata 3,7 provinsi per toko, maksimum 9 |
| `rating_toko` berbeda dalam satu toko | **161 dari 176 toko ber-rating**; rentang median 1,1, maksimum 1,8; 1 toko (T146) seluruh ratingnya penanda |

Apakah `provinsi_toko` yang beragam itu kebetulan atau memang acak? Provinsi diacak antarproduk (1.000 kali, seed 2026), lalu dihitung berapa toko yang tampak berprovinsi tunggal:

| | Toko berprovinsi tunggal |
|---|---:|
| Data nyata | **11** |
| Provinsi diacak antarproduk | rata-rata **10,4** (rentang 8–16) |

Data nyata tidak dapat dibedakan dari pengacakan. Artinya `provinsi_toko` praktis **tidak ditentukan oleh `id_toko`**.

## 4. Keputusan: tabel toko tidak diturunkan dari katalog

**Putusan:** tabel toko (satu baris per toko) **tidak dibuat dari katalog produk**. `provinsi_toko` dan `rating_toko` dipertahankan sebagai atribut **tiap produk**, tidak diangkat ke tingkat toko.

Alasan dengan angka:

1. Tabel toko hanya sah bila `id_toko` menentukan setiap atributnya. Pada katalog ini, hanya `nama_toko` yang konsisten (0 toko berbeda), padahal nama itu sudah turunan dari id (`"Toko "` + nomor). Tabel yang terbentuk hanya `id_toko`, `nama_toko` untuk 177 toko dan tidak menambah informasi.
2. `provinsi_toko` berbeda dalam **166 dari 177 toko**, dan sebarannya tidak dapat dibedakan dari acak (11 nyata, 10,4 rata-rata acak).
3. `rating_toko` berbeda dalam **161 dari 176 toko**, dengan rentang hingga 1,8 poin.
4. Memaksa satu nilai per toko berarti **mengarang data**. Memakai provinsi modus mengubah provinsi **544 dari 850 produk (64,0%)**. Merata-ratakan rating menghilangkan variasi yang justru nyata di data.

Alternatif yang dipertimbangkan dan alasan ditolak:

| Alternatif | Mengapa tidak dipilih |
|---|---|
| Ambil provinsi modus per toko | Mengubah 544 dari 850 produk; seri modus dipecahkan hanya oleh urutan kemunculan |
| Rata-rata rating per toko | Menyembunyikan rentang hingga 1,8 poin dan mengikutkan penanda bila tidak hati-hati |
| Tabel toko hanya `id_toko` dan `nama_toko` | Dapat dibuat, tetapi tidak berisi informasi di luar id |

**Dugaan** (belum dikonfirmasi dari kamus data): `provinsi_toko` dan `rating_toko` sebenarnya mencatat sifat produk atau penjual per produk, bukan sifat toko, dan nama kolomnya menyesatkan. Ini sebaiknya diklarifikasi sebelum kolom dipakai di Modul berikutnya.

Keputusan dicatat pada `log-keputusan.csv` sebagai **`K-04-014`** (`id_anomali` A-19 karena rating penanda tidak diikutkan).
