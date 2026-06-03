# PRD — Dashboard Monitoring Pengadaan Barang/Jasa Daerah
**Versi:** 2.2 | **Tanggal:** 1 Juni 2026 | **Status:** Final (Acuan Development)

> **Pemilik Produk:** Bagian Pengadaan Barang dan Jasa (BPBJ) Kota Surakarta
> **Platform:** Web Application — Single Page Application (SPA)
> **Arsitektur:** Google Sheets + Google Apps Script + HTML Statis

---

## Daftar Isi

1. [Executive Summary](#1-executive-summary)
2. [Business Problem](#2-business-problem)
3. [Product Goals](#3-product-goals)
4. [Stakeholders & User Personas](#4-stakeholders--user-personas)
5. [User Journey](#5-user-journey)
6. [Struktur Data Aktual](#6-struktur-data-aktual)
7. [Aturan Normalisasi Data (ETL Rules)](#7-aturan-normalisasi-data-etl-rules)
8. [Matching Engine](#8-matching-engine)
9. [Kalkulasi Efisiensi](#9-kalkulasi-efisiensi)
10. [Database Schema](#10-database-schema)
11. [Functional Requirements](#11-functional-requirements)
12. [API Specification](#12-api-specification)
13. [Komponen Dashboard](#13-komponen-dashboard)
14. [UI/UX Requirements](#14-uiux-requirements)
15. [Non-Functional Requirements](#15-non-functional-requirements)
16. [Security Requirements](#16-security-requirements)
17. [Acceptance Criteria](#17-acceptance-criteria)
18. [Technology Stack](#18-technology-stack)
19. [Deployment Architecture](#19-deployment-architecture)
20. [Risiko & Mitigasi](#20-risiko--mitigasi)
21. [Future Roadmap](#21-future-roadmap)

---

## 1. Executive Summary

BPBJ Kota Surakarta mengelola **±15.000 paket pengadaan per tahun** dari **160 satuan kerja (satker)** aktif. Dashboard ini dibangun untuk memfasilitasi pemantauan terpusat seluruh OPD dengan arsitektur **zero-cost serverless** berbasis ekosistem Google Workspace.

**Alur kerja inti:**
Setiap bulan, Administrator mengunduh 2 file CSV dari SIRUP/SPSE → mengupload ke sistem → backend memproses ETL & matching → dashboard terupdate otomatis untuk pelaporan ke pimpinan.

**Arsitektur teknis:**
```
[CSV SIRUP/SPSE] → [Frontend Upload] → [Apps Script API] → [Google Sheets DB]
                                                    ↓
                              [Dashboard SPA] ← [Apps Script API]
```

---

## 2. Business Problem

| # | Masalah | Dampak |
|---|---------|--------|
| B-01 | Visibilitas terbatas — pimpinan sulit mendapat gambaran utuh dari 160 satker | Keputusan strategis terhambat |
| B-02 | Matching manual antara 15.281 paket RUP dan 3.874 transaksi realisasi rentan error | Data efisiensi tidak akurat |
| B-03 | **46% baris realisasi tidak memiliki Kode RUP** — tidak bisa di-link langsung ke RUP | Potensi under-reporting efisiensi |
| B-04 | 9 varian status paket & 9 varian jenis pengadaan yang tidak seragam antara RUP dan Realisasi | Analisis distribusi tidak bisa dilakukan |
| B-05 | Tidak ada repositori historis per bulan untuk analisis tren | Tidak bisa membandingkan progres antar periode |

---

## 3. Product Goals

- **G-01** Menampilkan ringkasan eksekutif: total pagu, total kontrak, efisiensi, dan jumlah paket seluruh OPD
- **G-02** Menampilkan komparasi nilai pagu RUP vs realisasi kontrak secara dinamis per OPD/periode
- **G-03** Menampilkan distribusi paket berdasarkan metode, jenis, sumber dana, dan status
- **G-04** Menyediakan analisis efisiensi yang mengikuti filter analitik aktif
- **G-05** Menampilkan tren bulanan dan perbandingan antar periode
- **G-06** Menjadi pusat monitoring pengadaan daerah untuk pengambilan keputusan strategis

---

## 4. Stakeholders & User Personas

### Stakeholders

| Pemangku Kepentingan | Peran |
|----------------------|-------|
| Wali Kota & Wakil Wali Kota | Pengguna eksekutif; butuh KPI tingkat tinggi |
| Sekretaris Daerah | Memantau kepatuhan OPD |
| Kepala & Staf BPBJ | Tim teknis; pengambil keputusan manajerial |
| APIP (Auditor) | Pengawasan; butuh filter mendalam dan ekspor data |
| OPD (160 Satker) | Entitas pelaksana; tidak mengakses sistem langsung |
| Administrator Sistem | Staf BPBJ yang menjalankan upload data bulanan |

### User Personas

**Bapak Eksekutif (Pimpinan)**
- Akses: Dashboard publik tanpa login
- Butuh: Visualisasi KPI yang jelas, grafik mudah dibaca, tidak perlu konfigurasi

**Ibu Admin (Staf Data BPBJ)**
- Akses: Modul upload dengan token autentikasi
- Butuh: Antarmuka upload yang andal, notifikasi sukses/gagal yang jelas, laporan kualitas data

**Bapak Analis (APIP)**
- Akses: Dashboard publik
- Butuh: Filter mendalam per OPD/periode, fitur ekspor Excel dan PDF

---

## 5. User Journey

### Journey Administrator (Bulanan)

```
1. Unduh CSV dari SIRUP → data_rup_YYYYMMDD.csv
2. Unduh CSV dari SPSE  → data_realisasi_YYYYMMDD.csv
3. Buka website → Masuk Modul Upload (masukkan token)
4. Pilih Tahun & Bulan periode
5. Upload data_rup.csv → sistem validasi → notifikasi sukses
6. Upload data_realisasi.csv → sistem validasi + jalankan matching engine
7. Cek Laporan Kualitas Data → konfirmasi data sudah benar
```

### Journey Pimpinan / Publik

```
1. Akses URL website
2. Sistem tampilkan Executive Summary (periode terbaru otomatis)
3. Gunakan filter Tahun, Bulan, OPD untuk drill-down
4. Baca grafik tren & perbandingan OPD
5. Export PDF untuk bahan rapat (opsional)
```

---

## 6. Struktur Data Aktual

> **Sumber data:** File CSV yang diunduh dari SIRUP (RUP) dan SPSE (Realisasi) Kota Surakarta.
> Data diupload **2 file per bulan** oleh Administrator.

### 6.1 File RUP — `data_rup_YYYYMMDD.csv`

**Statistik Mei 2026:** 15.281 paket | 160 satker unik | Total Pagu: **Rp 937.957.995.621**

| # | Nama Kolom CSV | Tipe | Nullable | Catatan |
|---|----------------|------|----------|---------|
| 1 | `Nama Instansi` | String | ✗ | Selalu `KOTA SURAKARTA` |
| 2 | `Nama Satuan Kerja` | String | ✗ | Format: `NAMA SATKER - KODE SATKER`<br>Contoh: `DINAS KESEHATAN - 1.02.0.00.0.00.01.0000` |
| 3 | `Tahun Anggaran` | Integer | ✗ | Contoh: `2026` |
| 4 | `Cara Pengadaan` | Enum | ✗ | `Swakelola` \| `Penyedia` |
| 5 | `Metode Pengadaan` | Enum | ✗ | `-` (jika Swakelola) \| `Pengadaan Langsung` \| `E-Purchasing` \| `Dikecualikan` \| `Seleksi` \| `Tender` \| `Penunjukan Langsung` |
| 6 | `Jenis Pengadaan` | Enum | ✗ | `-` (jika Swakelola) \| `Barang` \| `Jasa Lainnya` \| `Jasa Konsultansi` \| `Pekerjaan Konstruksi` \| `Terintegrasi` |
| 7 | `Nama Paket` | String | ✗ | Kunci matching sekunder |
| 8 | `Kode RUP` | Integer | ✗ | **Primary key matching**. Total unik: 15.281 |
| 9 | `Sumber Dana` | Enum | ✗ | `APBD` \| `APBDP` \| `BLUD` \| `APBD;APBDP` |
| 10 | `Produk Dalam Negeri` | Enum | ✗ | `Ya` \| `Tidak` |
| 11 | `Total Nilai (Rp)` | Integer | ✗ | **Nilai Pagu RUP** dalam Rupiah |

> **Mapping ke kolom DB:** `Total Nilai (Rp)` → `pagu`

---

### 6.2 File Realisasi — `data_realisasi_YYYYMMDD.csv`

**Statistik Mei 2026:** 3.874 transaksi | 143 satker unik | Total Kontrak: **Rp 241.889.523.233**
⚠️ **Kode RUP kosong: 1.786 baris (46%)** — kritis untuk matching engine

| # | Nama Kolom CSV | Tipe | Nullable | Catatan |
|---|----------------|------|----------|---------|
| 1 | `Nama Instansi` | String | ✗ | Selalu `KOTA SURAKARTA` |
| 2 | `Nama Satuan Kerja` | String | ✗ | Format sama: `NAMA SATKER - KODE SATKER` |
| 3 | `Kode Paket` | String | ✗ | ID unik transaksi dari SPSE<br>Contoh: `01KKVNTDP0V3E6032PYYY1P9FC` |
| 4 | `Kode RUP` | Float | ✅ | **NULLABLE — 46% kosong.** Kunci matching utama jika ada |
| 5 | `Tahun Anggaran` | Integer | ✗ | Contoh: `2026` |
| 6 | `Sumber Transaksi` | Enum | ✗ | `Pencatatan` \| `E-Katalog 6.0` \| `Tokodaring` \| `Non Tender` \| `Swakelola` \| `Tender` |
| 7 | `Sumber Dana` | Enum | ✅ | Nullable. `APBD` \| `APBDP` \| `BLUD` \| `APBD,APBDP` \| `APBD;APBDP` |
| 8 | `Nama Penyedia` | String | ✅ | Nullable (1.894 kosong). Nama vendor/penyedia jasa |
| 9 | `Metode Pengadaan` | Enum | ✅ | Nullable (98 kosong). Varian lebih beragam dari RUP |
| 10 | `Jenis Pengadaan` | Enum | ✅ | Nullable. Lihat normalisasi §7.3 |
| 11 | `Nama Paket` | String | ✅ | Nullable (1.786 kosong). Kunci matching sekunder |
| 12 | `Status Paket` | Enum | ✅ | Nullable. **9 varian** — lihat normalisasi §7.2 |
| 13 | `Total Nilai (Rp)` | Integer | ✗ | **Nilai Kontrak Realisasi** |
| 14 | `Nilai PDN (Rp)` | Integer | ✗ | Nilai Produk Dalam Negeri |

> **Mapping ke kolom DB:** `Total Nilai (Rp)` → `nilai_kontrak`, `Nilai PDN (Rp)` → `nilai_pdn`

---

## 7. Aturan Normalisasi Data (ETL Rules)

Semua normalisasi dijalankan di sisi **Backend (Google Apps Script)** setelah data diterima.

### 7.1 Parsing Satuan Kerja

Berlaku untuk kedua file (RUP & Realisasi):

```
Input  : "DINAS KESEHATAN - 1.02.0.00.0.00.01.0000"
Output : nama_satker  = "DINAS KESEHATAN"
         kode_satker  = "1.02.0.00.0.00.01.0000"
Aturan : Split pada delimiter " - " (spasi-hubung-spasi).
         Segmen terakhir = kode_satker. Sisa = nama_satker.
```

---

### 7.2 Mapping Status Paket (Realisasi)

| Nilai Asli di CSV | Nilai Standar (`status_paket`) |
|-------------------|-------------------------------|
| `COMPLETED` | `Selesai` |
| `SELESAI` | `Selesai` |
| `PAKET SELESAI` | `Selesai` |
| `PAYMENT OUTSIDE SYSTEM` | `Selesai` |
| `PAKET SEDANG BERJALAN` | `Sedang Berjalan` |
| `ON PROCESS` | `Sedang Berjalan` |
| `BERLANGSUNG` | `Sedang Berjalan` |
| `ON ADDENDUM` | `Sedang Berjalan` |
| `NULL` / kosong | `Belum Berproses` |

> Nilai asli tetap disimpan di kolom `status_paket_asli` untuk audit.

---

### 7.3 Mapping Jenis Pengadaan (Normalisasi Realisasi → standar RUP)

| Nilai Asli di CSV (Realisasi) | Nilai Standar (`jenis_pengadaan`) |
|-------------------------------|-----------------------------------|
| `Barang`, `Pengadaan Barang` | `Barang` |
| `Jasa Konsultansi Badan Usaha Konstruksi` | `Jasa Konsultansi` |
| `Jasa Konsultansi Badan Usaha Non Konstruksi` | `Jasa Konsultansi` |
| `Jasa Konsultansi Perorangan Konstruksi` | `Jasa Konsultansi` |
| `Jasa Konsultansi` | `Jasa Konsultansi` |
| `Jasa Lainnya` | `Jasa Lainnya` |
| `Pekerjaan Konstruksi` | `Pekerjaan Konstruksi` |
| `NULL` / kosong | `Tidak Terklasifikasi` |

---

### 7.4 Normalisasi Sumber Dana

| Nilai Asli | Nilai Standar |
|------------|---------------|
| `APBD;APBDP` | `APBD+APBDP` |
| `APBD,APBDP` | `APBD+APBDP` |
| `NULL` (Realisasi) | `Tidak Diketahui` |

---

### 7.5 Override Metode Pengadaan (RUP — Swakelola)

```
JIKA  Cara Pengadaan = "Swakelola"
MAKA  Metode Pengadaan = "Swakelola"  (menggantikan "-")
      Jenis Pengadaan  = "Swakelola"  (menggantikan "-")
```

Berlaku **hanya untuk RUP**. Pada Realisasi, Swakelola diidentifikasi dari `Sumber Transaksi = "Swakelola"`.

---

## 8. Matching Engine

Mesin pencocokan menghubungkan paket **RUP** dengan transaksi **Realisasi** per periode.
Berjalan **otomatis** setiap kali upload Realisasi selesai.
Scope: **per `periode` + `kode_satker`** — tidak cross-satker, tidak cross-periode.

### Urutan Prioritas

```
INPUT: Satu baris Realisasi
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  CEK: apakah kode_rup Realisasi tersedia (tidak NULL)?          │
└──────────────┬──────────────────────────┬───────────────────────┘
               │ YA                       │ TIDAK (kode_rup = NULL)
               ▼                          ▼
┌──────────────────────────┐   ┌──────────────────────────────────┐
│  PRIORITAS 1             │   │  PRIORITAS 2                     │
│  Exact Kode RUP Match    │   │  Nama Paket Match                │
│                          │   │                                  │
│  Kondisi:                │   │  Kondisi:                        │
│  kode_rup Realisasi =    │   │  kode_satker sama                │
│  kode_rup RUP            │   │  DAN similarity(nama_paket) ≥75% │
│                          │   │  (Jaro-Winkler / Levenshtein)    │
│  Hasil: MATCHED          │   │                                  │
│  metode: EXACT_ID        │   │  Hasil: MATCHED                  │
└──────────────┬───────────┘   │  metode: NAMA_PAKET              │
               │ tidak cocok   └────────────────┬─────────────────┘
               │                                │ tidak cocok /
               │                                │ nama_paket NULL
               ▼                                ▼
┌──────────────────────────────────────────────────────────────────┐
│  PRIORITAS 3 — Tidak Tercocokkan                                 │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  SUB-KASUS A: kode_rup Realisasi ADA tapi tidak         │    │
│  │  ditemukan di data RUP periode yang sama                │    │
│  │  → status_matching = "UNMATCHED"                        │    │
│  │  → masuk rekap Unmatched RUP biasa                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  SUB-KASUS B: kode_rup Realisasi NULL DAN nama_paket    │    │
│  │  tidak cocok / nama_paket juga NULL                     │    │
│  │  → status_matching = "ORPHAN"          ← TANDA KHUSUS  │    │
│  │  → masuk rekap TERPISAH di tabel Orphan Realisasi       │    │
│  │  → TIDAK masuk kalkulasi efisiensi                      │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### Penjelasan Status Matching

| Status | Kondisi | Masuk Efisiensi | Tampil di |
|--------|---------|-----------------|-----------|
| `MATCHED` (EXACT_ID) | kode_rup cocok | ✅ Ya | Dashboard utama |
| `MATCHED` (NAMA_PAKET) | kode_rup NULL, nama_paket cocok di satker sama | ✅ Ya | Dashboard utama + flag kuning |
| `UNMATCHED` | kode_rup ada tapi tidak ditemukan pasangan di RUP | ❌ Tidak | Rekap Kualitas Data |
| `ORPHAN` | kode_rup NULL + nama_paket tidak cocok / NULL | ❌ Tidak | **Rekap ORPHAN tersendiri** |

> ⚠️ **Catatan ORPHAN:** Transaksi berstatus `ORPHAN` adalah realisasi yang **sama sekali tidak dapat dihubungkan** ke paket RUP manapun di satker yang sama. Ditampilkan di rekap terpisah agar admin dapat menginvestigasi apakah transaksi ini perlu dikoreksi di sumber data (SPSE).

### Parameter Matching

| Parameter | Nilai | Keterangan |
|-----------|-------|-----------|
| `similarity_threshold` | `0.75` (75%) | Dapat dikalibrasi admin jika false positive tinggi |
| `scope` | Per `periode` + `kode_satker` | Tidak cross-satker, tidak cross-periode |
| `relasi` | One-to-one | Satu paket RUP dipasangkan dengan tepat satu transaksi Realisasi. Jika satu RUP memiliki >1 kandidat Realisasi, ambil yang similarity tertinggi; kandidat lain masuk UNMATCHED/ORPHAN |

---

## 9. Kalkulasi Efisiensi

```
Efisiensi (Rp)    = Pagu RUP - Nilai Kontrak Realisasi
Efisiensi (%)     = (Efisiensi Rp / Total Pagu MATCHED) × 100%

Syarat            : Hanya dari pasangan berstatus MATCHED
Scope kalkulasi   : Mengikuti filter aktif di Frontend
                    (misal: filter OPD tertentu → efisiensi hanya OPD tsb)
```

> **Penting:** Kalkulasi efisiensi **tidak menggunakan rumus Google Sheets**, melainkan dihitung melalui iterasi array di dalam Apps Script. Snapshot hasil disimpan ke tabel `DASHBOARD_AGGREGATE`.

---

## 10. Database Schema

Semua tabel berada dalam **satu dokumen Google Sheets** yang terhubung ke Apps Script.

### 10.1 `UPLOAD_LOG`

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `upload_id` | String (UUID) | Primary key, auto-generated |
| `periode` | String | Format `YYYY-MM`, contoh: `2026-05` |
| `bulan` | Integer | 1–12 |
| `tahun` | Integer | Contoh: 2026 |
| `jenis_data` | Enum | `RUP` \| `REALISASI` |
| `nama_file` | String | Nama file asli yang diunggah |
| `jumlah_record` | Integer | Jumlah baris yang berhasil diproses |
| `uploaded_by` | String | Token/identitas pengunggah |
| `uploaded_at` | DateTime | Timestamp upload |
| `status` | Enum | `SUCCESS` \| `PARTIAL` \| `FAILED` |
| `catatan` | String | Pesan error atau info proses ETL |

---

### 10.2 `RUP_DATA`

| Kolom | Tipe | Sumber Kolom CSV |
|-------|------|-----------------|
| `upload_id` | String | — (FK ke UPLOAD_LOG) |
| `periode` | String | — (derived) |
| `bulan` | Integer | — (derived) |
| `tahun` | Integer | `Tahun Anggaran` |
| `nama_satker` | String | Parsed dari `Nama Satuan Kerja` |
| `kode_satker` | String | Parsed dari `Nama Satuan Kerja` |
| `cara_pengadaan` | String | `Cara Pengadaan` |
| `metode_pengadaan` | String | `Metode Pengadaan` (setelah override Swakelola) |
| `jenis_pengadaan` | String | `Jenis Pengadaan` (setelah override Swakelola) |
| `nama_paket` | String | `Nama Paket` |
| `kode_rup` | String | `Kode RUP` |
| `sumber_dana` | String | `Sumber Dana` (setelah normalisasi) |
| `produk_dalam_negeri` | String | `Produk Dalam Negeri` |
| `pagu` | Integer | `Total Nilai (Rp)` |

---

### 10.3 `REALISASI_DATA`

| Kolom | Tipe | Sumber Kolom CSV |
|-------|------|-----------------|
| `upload_id` | String | — (FK ke UPLOAD_LOG) |
| `periode` | String | — (derived) |
| `bulan` | Integer | — (derived) |
| `tahun` | Integer | `Tahun Anggaran` |
| `nama_satker` | String | Parsed dari `Nama Satuan Kerja` |
| `kode_satker` | String | Parsed dari `Nama Satuan Kerja` |
| `kode_paket` | String | `Kode Paket` |
| `kode_rup` | String \| NULL | `Kode RUP` (nullable) |
| `sumber_transaksi` | String | `Sumber Transaksi` |
| `sumber_dana` | String | `Sumber Dana` (setelah normalisasi) |
| `nama_penyedia` | String \| NULL | `Nama Penyedia` |
| `metode_pengadaan` | String \| NULL | `Metode Pengadaan` |
| `jenis_pengadaan` | String | `Jenis Pengadaan` (setelah normalisasi §7.3) |
| `nama_paket` | String \| NULL | `Nama Paket` |
| `status_paket_asli` | String \| NULL | `Status Paket` (nilai mentah dari CSV) |
| `status_paket` | String | `Status Paket` (setelah normalisasi §7.2) |
| `nilai_kontrak` | Integer | `Total Nilai (Rp)` |
| `nilai_pdn` | Integer | `Nilai PDN (Rp)` |

---

### 10.4 `PACKAGE_MATCHING`

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `matching_id` | String (UUID) | Primary key |
| `periode` | String | Periode matching dijalankan |
| `kode_satker` | String | Scope matching |
| `nama_satker` | String | |
| `kode_rup` | String \| NULL | Dari RUP |
| `kode_paket` | String \| NULL | Dari Realisasi |
| `nama_paket_rup` | String \| NULL | |
| `nama_paket_realisasi` | String \| NULL | |
| `metode_matching` | Enum | `EXACT_ID` \| `HEURISTIC` \| `UNMATCHED` |
| `similarity_score` | Float \| NULL | 0.0–1.0; diisi jika metode = HEURISTIC |
| `pagu_rup` | Integer \| NULL | |
| `nilai_kontrak` | Integer \| NULL | |
| `efisiensi` | Integer \| NULL | `pagu_rup - nilai_kontrak` |
| `persen_efisiensi` | Float \| NULL | |
| `status_matching` | Enum | `MATCHED` \| `UNMATCHED` |
| `created_at` | DateTime | Timestamp proses matching |

---

### 10.5 `DASHBOARD_AGGREGATE`

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `aggregate_id` | String (UUID) | Primary key |
| `periode` | String | |
| `bulan` | Integer | |
| `tahun` | Integer | |
| `total_pagu` | Integer | Sum pagu RUP seluruh satker |
| `total_kontrak` | Integer | Sum nilai kontrak MATCHED |
| `total_efisiensi` | Integer | `total_pagu - total_kontrak` (MATCHED only) |
| `persen_efisiensi` | Float | |
| `jumlah_paket_rup` | Integer | Total paket di RUP |
| `jumlah_paket_realisasi` | Integer | Total transaksi di Realisasi |
| `jumlah_matched` | Integer | |
| `jumlah_unmatched_rup` | Integer | Paket RUP tanpa pasangan |
| `jumlah_unmatched_realisasi` | Integer | Transaksi tanpa pasangan |
| `jumlah_selesai` | Integer | Status = Selesai |
| `jumlah_sedang_berjalan` | Integer | Status = Sedang Berjalan |
| `jumlah_belum_berproses` | Integer | Status = Belum Berproses |
| `total_nilai_pdn` | Integer | Sum `nilai_pdn` dari seluruh transaksi Realisasi periode ini |
| `persen_pdn` | Float | `(total_nilai_pdn / total_kontrak) × 100%` |
| `created_at` | DateTime | Timestamp snapshot dibuat |

---

## 11. Functional Requirements

### FR-01 — Manajemen Periode

- Sistem mendukung kombinasi **Tahun** dan **Bulan** sebagai identifier periode (format `YYYY-MM`)
- Data tiap periode adalah **snapshot independen**, tidak menimpa periode lain
- Administrator dapat **menghapus data periode** tertentu jika ditemukan kesalahan upload
- Frontend menampilkan daftar periode yang tersedia secara dinamis dari data di Sheets

---

### FR-02 — Upload Data Bulanan

- Administrator mengupload **2 file CSV per bulan**: `data_rup_*.csv` dan `data_realisasi_*.csv`
- Upload dilakukan secara **berurutan**: RUP terlebih dahulu, baru Realisasi
- Matching engine **otomatis berjalan** setelah upload Realisasi selesai
- **Batas ukuran file:** 10 MB per file
- **Batas baris:** 20.000 baris per file
- File dengan **>5.000 baris** → Frontend memecah menjadi chunk 500 baris, dikirim ke API secara berurutan

---

### FR-03 — Validasi & Notifikasi

**Validasi di Frontend (sebelum upload):**
- Ekstensi file harus `.csv`
- Ukuran file ≤ 10 MB
- Header baris pertama harus mengandung semua kolom wajib (lihat §6)

**Validasi di Backend (Apps Script):**
- Tipe data per kolom
- Nilai enum pada kolom terbatas
- Konsistensi format `Nama Satuan Kerja`

**Notifikasi** menggunakan **SweetAlert2** untuk:
- ✅ Sukses upload (dengan jumlah record yang diproses)
- ❌ Error validasi (dengan keterangan kolom yang tidak sesuai)
- ⚠️ Konfirmasi hapus data periode
- ℹ️ Progress bar untuk upload chunked

---

### FR-04 — Autentikasi Administrator

- Halaman upload dilindungi **token akses** sederhana
- Token dikonfigurasi sebagai konstanta di Apps Script (**tidak disimpan di Frontend**)
- Token dikirim dalam setiap request body dan divalidasi di baris pertama `doPost()`
- Pengguna publik mengakses **dashboard tanpa autentikasi**
- Request tanpa token atau token salah → respons `{ status: "error", code: 401 }`

---

### FR-05 — Global Filtering & SPA Behavior

- Filter global mencakup: **Tahun**, **Bulan**, dan **OPD/Satker**
- Perubahan filter memperbarui **seluruh metrik, grafik, dan tabel** tanpa reload halaman
- Daftar OPD di filter bersifat dinamis sesuai data periode yang dipilih
- Filter OPD menampilkan `nama_satker` (bukan kode) yang diurutkan secara alfabetis
- State filter dipertahankan selama sesi (tidak hilang saat pindah tab dashboard)

---

### FR-06 — Kalkulasi Efisiensi

- Efisiensi dihitung sesuai rumus di §9
- Kalkulasi mengikuti **filter yang sedang aktif** di Frontend
- Efisiensi negatif (nilai kontrak > pagu) tetap ditampilkan dengan warna merah sebagai alert

---

### FR-07 — Export Data

- **Export Excel (.xlsx):** Menggunakan library **SheetJS** (client-side). Data mengikuti filter aktif.
- **Export PDF:** Menggunakan library **jsPDF** yang dirender langsung di browser. Mencakup KPI cards dan grafik utama.

---

### FR-08 — Laporan Kualitas Data

- Tabel **Unmatched RUP**: paket RUP yang tidak memiliki pasangan Realisasi
- Tabel **Unmatched Realisasi**: transaksi yang tidak memiliki pasangan RUP
- **Ringkasan**: persentase data tercocokkan per periode
- Akses terbatas: **hanya tampil jika token admin aktif** di sesi

---

## 12. API Specification

Backend API menggunakan **Google Apps Script (Code.gs)** sebagai Web App.
Semua request dikendalikan melalui fungsi tunggal `doPost(e)`.

### Konfigurasi CORS

```javascript
// Di Frontend: gunakan Content-Type text/plain untuk bypass CORS preflight
fetch(SCRIPT_URL, {
  method: 'POST',
  headers: { 'Content-Type': 'text/plain' },
  body: JSON.stringify(payload)
})

// Di Backend: parse manual
const data = JSON.parse(e.postData.contents);
```

### Struktur Request

```json
{
  "action": "namaAction",
  "token": "xxx",         // wajib untuk endpoint admin
  "periode": "2026-05",   // opsional, sesuai kebutuhan action
  "kode_satker": "...",   // opsional, untuk filter OPD
  "data": []              // untuk upload
}
```

### Struktur Response

```json
{
  "status": "success" | "error",
  "code": 200 | 400 | 401 | 500,
  "message": "...",
  "data": {}  // payload respons
}
```

**Contoh response `kpiDashboard`:**
```json
{
  "status": "success",
  "code": 200,
  "data": {
    "periode": "2026-05",
    "total_pagu": 937957995621,
    "total_kontrak": 241889523233,
    "total_efisiensi": 696068472388,
    "persen_efisiensi": 74.21,
    "jumlah_paket_rup": 15281,
    "jumlah_paket_realisasi": 3874,
    "jumlah_matched": 1189,
    "jumlah_unmatched_rup": 14092,
    "jumlah_unmatched_realisasi": 2685,
    "jumlah_selesai": 1240,
    "jumlah_sedang_berjalan": 834,
    "jumlah_belum_berproses": 1786,
    "total_nilai_pdn": 198450000000,
    "persen_pdn": 82.04
  }
}
```

### Endpoint Routes

| Action | Akses | Deskripsi |
|--------|-------|-----------|
| `uploadRUP` | Admin | Upload & ETL data RUP. Body: `{ token, periode, data: [] }` |
| `uploadRealisasi` | Admin | Upload & ETL Realisasi + jalankan matching engine |
| `deletePeriode` | Admin | Hapus semua data pada periode tertentu |
| `unmatchedData` | Admin | Laporan data tidak tercocokkan |
| `kpiDashboard` | Publik | Ambil KPI agregat per periode + filter OPD |
| `dashboardOPD` | Publik | Performa seluruh OPD untuk periode terpilih |
| `dashboardMetode` | Publik | Distribusi metode & jenis pengadaan |
| `dashboardStatus` | Publik | Distribusi status paket |
| `dashboardSumber` | Publik | Distribusi sumber dana & sumber transaksi |
| `dashboardPenyedia` | Publik | Top vendor berdasarkan nilai kontrak |
| `dashboardEfisiensi` | Publik | Analisis efisiensi per OPD |
| `dashboardTren` | Publik | Tren bulanan dari semua periode tersedia |
| `detailPaket` | Publik | Tabel detail dengan paginasi & pencarian |
| `listPeriode` | Publik | Daftar periode yang tersedia di database |

### doGet — Safety Handler

```javascript
// Wajib diimplementasi agar URL tidak error jika dibuka di browser
function doGet(e) {
  return ContentService.createTextOutput(
    JSON.stringify({ status: "ok", message: "BPBJ Surakarta API" })
  ).setMimeType(ContentService.MimeType.JSON);
}
```

---

## 13. Komponen Dashboard

| Dashboard | Komponen Visual | Data Sumber |
|-----------|-----------------|-------------|
| **Eksekutif (Home)** | KPI Cards: Total Pagu, Total Kontrak, Efisiensi (Rp & %), Jumlah Paket RUP, Jumlah Transaksi Realisasi, % Nilai PDN. Donut chart distribusi metode. | `DASHBOARD_AGGREGATE` |
| **Performa OPD** | Tabel sortable: nama satker, pagu, kontrak, efisiensi, % realisasi, jumlah paket. Bar chart horizontal top 10 OPD. | `PACKAGE_MATCHING` GROUP BY satker |
| **Metode & Jenis** | Bar chart metode pengadaan. Bar chart jenis pengadaan. Tabel distribusi. | `RUP_DATA` + `REALISASI_DATA` |
| **Sumber Dana & Transaksi** | Donut chart sumber dana. Bar chart sumber transaksi realisasi (Pencatatan, E-Katalog, Tokodaring, Non Tender, Swakelola, Tender). | `REALISASI_DATA` |
| **Status Paket** | Pie chart Selesai vs Sedang Berjalan vs Belum Berproses. KPI jumlah per status. | `REALISASI_DATA` |
| **Realisasi & PDN** | KPI: Total nilai kontrak, total nilai PDN, % PDN dari kontrak. Bar chart nilai PDN per satker. Tabel: satker, nilai kontrak, nilai PDN, % PDN. | `REALISASI_DATA` |
| **Top Penyedia** | Tabel 10 vendor teratas: nama penyedia, jumlah transaksi, total nilai kontrak. | `REALISASI_DATA` GROUP BY nama_penyedia |
| **Efisiensi** | Horizontal bar chart efisiensi per OPD. Highlight tertinggi (hijau) & terendah/negatif (merah). | `PACKAGE_MATCHING` MATCHED only |
| **Tren Bulanan** | Line chart pagu & kontrak dari semua periode. Perbandingan bulan ke bulan (delta %). | `DASHBOARD_AGGREGATE` semua periode |
| **Detail Paket** | Tabel lengkap dengan search, pagination 50 per halaman, filter kolom. Kolom: satker, nama paket, pagu, kontrak, efisiensi, status, metode. | `PACKAGE_MATCHING` JOIN RUP + Realisasi |
| **Kualitas Data** *(admin)* | Tabel unmatched RUP. Tabel unmatched Realisasi. Summary % matching rate per periode. | `PACKAGE_MATCHING` UNMATCHED |

---

## 14. UI/UX Requirements

### Framework & Library

| Kebutuhan | Library | CDN |
|-----------|---------|-----|
| Layout & Style | Tailwind CSS | `cdn.tailwindcss.com` |
| Grafik | Chart.js | `cdn.jsdelivr.net` |
| Notifikasi | SweetAlert2 | `cdn.jsdelivr.net` |
| Export Excel | SheetJS (xlsx.js) | `cdn.sheetjs.com` |
| Export PDF | jsPDF | `cdn.jsdelivr.net` |

### SPA Navigation

- Seluruh konten berada dalam **satu file `index.html`**
- Navigasi antar dashboard menggunakan **show/hide section** (tidak ada page load)
- URL hash (`#eksekutif`, `#opd`, dll.) dapat digunakan untuk deep-link

### Responsivitas

| Breakpoint | Layout | Navigasi |
|------------|--------|---------|
| Desktop ≥ 1024px | Multi-kolom, sidebar permanen | Sidebar kiri |
| Tablet 768–1023px | 2 kolom, sidebar lipat | Sidebar collapsible |
| Mobile < 768px | 1 kolom | Hamburger menu |

### Panduan Tampilan

- **Warna utama:** Biru pemerintahan (`#1E3A5F` / `#0070C0`)
- **Status warna:** Hijau = Selesai, Kuning = Sedang Berjalan, Abu = Belum Berproses, Merah = Efisiensi Negatif
- **Angka Rupiah:** Format `Rp 937.957.995.621` atau singkat `Rp 937,9 M` untuk KPI cards
- **Tabel:** Zebra stripe, sticky header, sortable kolom

---

## 15. Non-Functional Requirements

### Performa

| Metrik | Target |
|--------|--------|
| Respons API query dashboard | ≤ 5 detik |
| Upload 15.000 baris RUP (chunked) | ≤ 60 detik total |
| Transisi antar tab dashboard | Tanpa reload, < 2 detik |
| Ukuran file `index.html` | ≤ 500 KB (tanpa aset eksternal) |

### Ketersediaan

- Bergantung pada **Google Apps Script SLA** (99.9% Google Workspace)
- Frontend statis memiliki uptime **independen** dari Backend

### Browser Support

Chrome ≥ 90, Firefox ≥ 88, Edge ≥ 90, Safari ≥ 14

---

## 16. Security Requirements

### Validasi Dua Lapis

```
Layer 1 (Frontend) : cek ekstensi, ukuran, dan header kolom CSV sebelum kirim
Layer 2 (Backend)  : validasi tipe data, enum, dan format sebelum insert ke Sheets
```

### Token Admin

- Token disimpan sebagai **konstanta di Apps Script** (bukan di kode Frontend)
- Token dikirim via **request body** (bukan URL parameter / header)
- Jika token salah → respons generik `401`, tanpa detail error spesifik

### Audit Trail

- Seluruh aktivitas upload dicatat di sheet `UPLOAD_LOG` (upload_id, waktu, status, catatan)
- Aksi hapus periode juga dicatat dengan kolom `catatan = "DELETED by [token]"`

### Endpoint Protection

- Endpoint admin (`uploadRUP`, `uploadRealisasi`, `deletePeriode`, `unmatchedData`) **wajib validasi token**
- Endpoint publik tidak memerlukan token tetapi **tidak mengekspos data sensitif** (nama penyedia disensor untuk tampilan publik jika diperlukan)

---

## 17. Acceptance Criteria

| ID | JIKA | MAKA |
|----|------|------|
| AC-01 | Admin upload `data_rup.csv` dengan 11 kolom benar dan token valid | Backend proses ETL, simpan ke `RUP_DATA`, respons sukses dengan jumlah record |
| AC-02 | Admin upload `data_realisasi.csv` setelah RUP bulan yang sama tersedia | Matching engine berjalan otomatis, `PACKAGE_MATCHING` terisi, `DASHBOARD_AGGREGATE` diperbarui |
| AC-03 | Admin upload file CSV dengan kolom tidak sesuai | Backend tolak, kembalikan pesan error spesifik menyebut kolom yang tidak sesuai |
| AC-04 | Admin upload file > 10 MB | Frontend tolak sebelum upload, tampilkan SweetAlert error |
| AC-05 | Pengguna mengubah filter OPD di dashboard | Seluruh KPI, grafik, dan tabel berubah tanpa reload halaman dalam < 2 detik |
| AC-06 | URL Backend Apps Script dibuka langsung di browser | `doGet()` mengembalikan pesan aman `{ status: "ok" }`, tidak error |
| AC-07 | Pengguna klik export PDF | Browser menghasilkan file PDF dengan data sesuai filter aktif |
| AC-08 | Request tanpa token atau token salah ke endpoint admin | Backend kembalikan `{ status: "error", code: 401 }`, tidak ada data diproses |
| AC-09 | Efisiensi sebuah OPD bernilai negatif | Nilai ditampilkan dengan warna merah di tabel dan chart efisiensi |
| AC-10 | Dashboard diakses dari mobile (< 768px) | Layout 1 kolom, semua konten terbaca tanpa horizontal scroll |

---

## 18. Technology Stack

| Layer | Teknologi | Versi / CDN | Fungsi |
|-------|-----------|-------------|--------|
| Frontend UI | HTML + Tailwind CSS | via CDN | Antarmuka SPA responsif |
| Grafik | Chart.js | latest via CDN | Bar, donut, line, pie chart |
| Notifikasi | SweetAlert2 | latest via CDN | Pop-up interaktif |
| Export Excel | SheetJS (xlsx.js) | latest via CDN | Generate file XLSX client-side |
| Export PDF | jsPDF | latest via CDN | Generate file PDF di browser |
| Backend API | Google Apps Script | — | ETL, matching engine, routing API |
| Database | Google Sheets | — | Flat-table structured storage |
| Hosting Frontend | Netlify / Vercel / GitHub Pages | — | Hosting file statis gratis |

---

## 19. Deployment Architecture

### Frontend

```
File: index.html (satu file)
Host: Netlify / Vercel / GitHub Pages
URL : https://bpbj-surakarta.netlify.app (contoh)

Deploy: drag & drop folder ke dashboard layanan
Env var: konstanta SCRIPT_URL diisi URL Apps Script di bagian atas index.html
```

### Backend

```
File  : Code.gs
Host  : Google Apps Script Web App
Akses : "Anyone" (Anyone, even anonymous)
URL   : https://script.google.com/macros/s/[ID]/exec

Deploy: Setiap perubahan Code.gs wajib Deploy ulang sebagai versi baru
```

### Konfigurasi Koneksi

```javascript
// Di index.html — satu-satunya konfigurasi yang perlu disesuaikan
const SCRIPT_URL = "https://script.google.com/macros/s/[SCRIPT_ID]/exec";
const ADMIN_TOKEN = "";  // kosong untuk publik; admin isi via input di UI
```

---

## 20. Risiko & Mitigasi

| Risiko | Tingkat | Mitigasi |
|--------|---------|---------|
| 46% data Realisasi tidak memiliki Kode RUP — matching bergantung nama paket | 🔴 Tinggi | Implementasi heuristic matching threshold 75%. Administrator cek laporan Kualitas Data tiap bulan dan kalibrasikan threshold jika perlu |
| Batas eksekusi Apps Script 6 menit terlampaui saat upload data besar | 🟡 Sedang | Frontend memecah upload menjadi chunk 500 baris. Matching engine dijalankan sebagai proses terpisah setelah semua chunk selesai |
| CORS rejection dari browser modern | 🟢 Rendah | Gunakan `Content-Type: text/plain` + `JSON.parse` manual di backend — metode ini terbukti stabil untuk Apps Script |
| SIRUP/SPSE mengubah struktur kolom CSV | 🟡 Sedang | Validasi header kolom wajib di Frontend sebelum upload. Backend kembalikan error spesifik per kolom. Perubahan cukup update mapping di ETL Rules |
| Efisiensi negatif karena data anomali (nilai kontrak > pagu) | 🟢 Rendah | Tetap ditampilkan dengan warna merah sebagai alert. Administrator dapat investigasi via tabel Detail Paket |
| Volume data kumulatif melampaui kapasitas Google Sheets | 🟢 Rendah (jangka pendek) | Lihat §21 roadmap migrasi database |

---

## 21. Future Roadmap

| Fase | Fitur | Kondisi Trigger |
|------|-------|----------------|
| **Fase 1** *(saat ini)* | Arsitektur Google Sheets + Apps Script | Volume < 20.000 baris/bulan |
| **Fase 2** | Notifikasi email otomatis ke Kepala BPBJ jika realisasi OPD < threshold % pagu | Permintaan manajemen |
| **Fase 3** | Integrasi API langsung dengan SIRUP/SPSE untuk eliminasi proses upload manual | API SIRUP/SPSE tersedia publik |
| **Fase 4** | Migrasi database ke PostgreSQL/MySQL dengan Node.js backend. Frontend tidak perlu diubah, hanya `SCRIPT_URL` diganti. | Volume kumulatif > 100.000 baris |

---

## Appendix: Daftar Nilai Enum

### Enum RUP

```
Cara Pengadaan    : Swakelola | Penyedia
Metode Pengadaan  : - | Pengadaan Langsung | E-Purchasing | Dikecualikan |
                    Seleksi | Tender | Penunjukan Langsung
Jenis Pengadaan   : - | Barang | Jasa Lainnya | Jasa Konsultansi |
                    Pekerjaan Konstruksi | Terintegrasi
Sumber Dana       : APBD | APBDP | BLUD | APBD;APBDP
Produk DN         : Ya | Tidak
```

### Enum Realisasi (sebelum normalisasi)

```
Sumber Transaksi  : Pencatatan | E-Katalog 6.0 | Tokodaring | Non Tender |
                    Swakelola | Tender
Status Paket      : COMPLETED | SELESAI | PAKET SELESAI | PAYMENT OUTSIDE SYSTEM |
                    PAKET SEDANG BERJALAN | ON PROCESS | BERLANGSUNG | ON ADDENDUM |
                    NULL
Jenis Pengadaan   : Barang | Pengadaan Barang | Jasa Lainnya |
                    Jasa Konsultansi Badan Usaha Konstruksi |
                    Jasa Konsultansi Badan Usaha Non Konstruksi |
                    Jasa Konsultansi Perorangan Konstruksi |
                    Jasa Konsultansi | Pekerjaan Konstruksi | NULL
```

### Enum Standar (setelah normalisasi)

```
status_paket      : Selesai | Sedang Berjalan | Belum Berproses
jenis_pengadaan   : Barang | Jasa Lainnya | Jasa Konsultansi |
                    Pekerjaan Konstruksi | Terintegrasi | Swakelola |
                    Tidak Terklasifikasi
sumber_dana       : APBD | APBDP | BLUD | APBD+APBDP | Tidak Diketahui
metode_matching   : EXACT_ID | HEURISTIC | UNMATCHED
status_matching   : MATCHED | UNMATCHED
```

---

*Dokumen ini adalah acuan utama development. Setiap perubahan requirement wajib diupdate di sini sebelum diimplementasikan.*

*PRD v2.2 — BPBJ Kota Surakarta — 1 Juni 2026*

> **Changelog v2.2:** Perbaikan relasi matching (Many-to-one → One-to-one), penambahan kolom `total_nilai_pdn` & `persen_pdn` di `DASHBOARD_AGGREGATE`, penambahan komponen **Dashboard Realisasi & PDN**, dan contoh response shape `kpiDashboard`.
