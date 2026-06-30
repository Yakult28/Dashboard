# Dashboard Direktorat Trantibum — Otorita Ibu Kota Nusantara

Dashboard internal untuk memantau seluruh aktivitas operasional Direktorat Ketenteraman dan Ketertiban Umum (Trantibum) OIKN, mengintegrasikan data dari tiga unit kerja: **Satuan Polisi Pamong Praja (Pol PP)**, **Polisi Kehutanan (Polhut)**, dan **Pemadam Kebakaran (Damkar)**.

🔗 **Live Dashboard**: *(isi link GitHub Pages setelah deploy)*

> ⚠️ **Catatan keamanan**: repository ini bersifat publik agar GitHub Pages dapat diaktifkan tanpa biaya tambahan. Seluruh ID Spreadsheet, ID Folder Drive, dan URL Web App API yang sesungguhnya **TIDAK dicantumkan** di dokumen ini — nilai-nilai tersebut dikelola secara terpisah dan hanya diketahui oleh tim internal Direktorat Trantibum. Halaman ini hanya menjelaskan struktur dan arsitektur sistem secara umum.

---

## Ringkasan Sistem

Dashboard ini terdiri dari tiga modul utama, masing-masing dengan sumber data dan alur kerja sendiri:

| Modul | Fungsi | Sumber Data |
|---|---|---|
| **Pelaporan** | Rekap kegiatan rutin 6 sumber laporan | Sheet Master (agregasi otomatis tiap jam) |
| **Laporan Real-time** | Laporan lapangan bergeolokasi dari petugas, per bidang (Pol PP/Polhut/Damkar) | Form HP petugas → 3 tab Sheet terpisah |
| **Rekapan Surat Teguran** | Rekap surat penindakan pelanggaran (STBP/BAP/BAT) | 3 Spreadsheet terpisah, dibaca langsung real-time |

---

## Arsitektur Teknis

```
┌─────────────────────────┐      ┌──────────────────────────┐
│   FORM PELAPORAN HP      │      │   6 SUMBER LAPORAN ASLI    │
│  (repo terpisah)          │      │  (Rapat, Satpol PP,        │
│                            │      │   Kriminal, Patroli,       │
│                            │      │   Damkar, Polhut)          │
└───────────┬───────────────┘      └────────────┬───────────────┘
            │ submit                              │ dibaca tiap 1 jam
            ▼                                      ▼
┌─────────────────────────┐      ┌──────────────────────────┐
│  Google Sheet "Laporan    │      │   Apps Script Agregator    │
│  Real-time" — 3 tab:       │      │   → tulis ke Sheet Master   │
│  Pol PP / Polhut / Damkar │      │                              │
└───────────┬───────────────┘      └────────────┬───────────────┘
            │                                      │
            └──────────────┬───────────────────────┘
                            ▼
              ┌──────────────────────────┐
              │   Web App API               │
              │   doGet — seluruh endpoint  │
              └────────────┬───────────────┘
                            ▼
              ┌──────────────────────────┐
              │   Dashboard HTML            │
              │   (repo ini, GitHub Pages)  │
              └──────────────────────────┘
```

---

## Sumber Data (Struktur Umum)

> ID Spreadsheet, ID Folder Drive, dan URL API sesungguhnya disimpan terpisah secara internal — tidak dicantumkan di sini.

### 1. Sheet Master (Pelaporan — 6 sumber)

Mengagregasi 6 sumber laporan rutin secara otomatis tiap 1 jam, hanya membaca tab bernama Januari–Desember (+ tab "VERY IMPORTANT" khusus Polhut):

- Laporan Kegiatan Rapat
- Laporan Kegiatan Satpol PP
- Laporan Kejadian Kriminal di Wilayah IKN
- Laporan Patroli Malam
- Laporan Damkar
- Laporan Polhut

**Sinkronisasi cerdas**: agregator tidak hanya menambah baris baru, tapi juga **mendeteksi perubahan** pada baris yang sudah pernah masuk (misal koreksi kategori oleh petugas) dan memperbaruinya otomatis di Sheet Master.

### 2. Laporan Real-time (3 bidang)

Spreadsheet tunggal dengan 3 tab terpisah (`Pol PP`, `Polhut`, `Damkar`), masing-masing 11 kolom identik. Setiap bidang punya **folder Google Drive foto sendiri** (terpisah total, tidak bersarang).

**Klasifikasi Jenis Kejadian** (multi-select, disimpan sebagai teks dipisah koma):
- **Pol PP**: 15 kategori (Tempat hiburan, Jalan, Angkutan jalan, dst.)
- **Polhut**: 7 kategori (Monitoring dan Evaluasi, Patroli Kawasan, dst.)
- **Damkar**: 16 kategori (Pemadaman Kebakaran, Evakuasi Ular, dst.)

### 3. Rekapan Surat Teguran (3 jenis)

Tiga spreadsheet terpisah untuk **STBP** (Surat Tanda Bukti Pelanggaran), **BAP** (Berita Acara Pembongkaran), dan **BAT** (Berita Acara Teguran Tertulis), dengan 10 klasifikasi pelanggaran baku. Saat ini hanya aktif untuk Pol PP; Polhut dan Damkar berstatus "Segera" (UI sudah siap, menunggu data).

---

## Backend (Google Apps Script)

Dua project Apps Script terpisah:

### Project "API Dashboard"
- **`Code.gs`** — agregator 6 sumber Pelaporan, berjalan otomatis tiap 1 jam via trigger.
- **`WebAppAPI.gs`** — seluruh endpoint `doGet` yang dipanggil dashboard:
  - `ringkasanPelaporan`, `detailPelaporan`
  - `ringkasanRealtime&bidang=...`, `detailRealtime&bidang=...`, `detailRealtimeGabungan`
  - `ringkasanSurat`, `detailSurat`
  - `trenBulananPelaporan`, `trenBulananSurat`

### Project "API Trantibum" (terpisah, untuk form HP)
- **`doPost`** — menerima submit dari form pelaporan lapangan, menulis ke tab bidang yang sesuai + upload foto ke folder Drive yang sesuai.

> ⚠️ **Catatan deployment**: "Edit Deployment" pada Apps Script beberapa kali terbukti tidak konsisten menerapkan kode terbaru. **Gunakan selalu "New Deployment"** saat memperbarui backend, lalu update URL di frontend (lihat catatan internal terpisah untuk konfigurasi URL terkini).

---

## Frontend (Dashboard HTML)

File tunggal `dashboard_v3_pelaporan.html` — HTML, CSS, dan JavaScript digabung dalam satu file untuk kemudahan deploy ke GitHub Pages.

**Library eksternal** (CDN jsdelivr — `cdnjs.cloudflare.com` diblokir jaringan internal OIKN):
- Chart.js `4.4.4`
- Leaflet `1.9.4`

**Fitur utama**:
- Landing page dengan 3 kartu modul + sub-kartu pilihan bidang (Pol PP/Polhut/Damkar)
- Drill-down 3 tingkat untuk Pelaporan (Sumber → Kategori → Sub-kategori)
- Laporan Real-time: halaman antara (peta gabungan 3 bidang) → halaman detail per bidang (chart + peta GPS)
- Grafik tren bulanan (stacked bar chart) untuk Pelaporan dan Surat Teguran
- Mode terang/gelap
- Modal detail laporan saat klik baris tabel manapun

---

## Konfigurasi (Diisi Terpisah, Tidak di Repository)

Sebelum dashboard ini dapat berfungsi, beberapa nilai konfigurasi perlu diisi langsung di kode oleh tim internal (tidak disimpan di repository publik ini):

| Variabel | Lokasi | Keterangan |
|---|---|---|
| `API_BASE_URL` | `dashboard_v3_pelaporan.html` | URL Web App project "API Dashboard" |
| `SCRIPT_URL` | Form pelaporan (repo terpisah) | URL Web App project "API Trantibum" |
| ID Spreadsheet Sheet Master | `Code.gs` | Lihat catatan konfigurasi internal |
| ID Spreadsheet 6 sumber Pelaporan | `Code.gs` | Lihat catatan konfigurasi internal |
| ID Spreadsheet Laporan Real-time | `WebAppAPI.gs`, `doPost.gs` | Lihat catatan konfigurasi internal |
| ID Folder Drive (3 bidang) | `doPost.gs` | Lihat catatan konfigurasi internal |
| ID Spreadsheet Surat Teguran (3 jenis) | `WebAppAPI.gs` | Lihat catatan konfigurasi internal |

---

## Pelajaran Teknis Penting

Sejumlah isu nyata yang ditemukan dan diperbaiki sepanjang pengembangan — dicatat agar tidak terulang:

1. **CDN cdnjs.cloudflare.com diblokir** jaringan kantor OIKN → selalu pakai jsdelivr.
2. **Koordinat GPS form lapangan kadang kehilangan tanda desimal** (mis. `-9748170` alih-alih `-0,9748170`) — backend punya fungsi rekonstruksi otomatis dengan validasi rentang geografis wilayah IKN sebagai jaring pengaman.
3. **Baris "sampah" di Sheet Master**: beberapa sumber sempat menyisipkan baris berisi nomor urut kolom atau baris kosong total yang ikut tertarik ke Sheet Master. Backend kini memvalidasi kolom Kategori Kegiatan sebagai penentu tunggal, dengan tiga lapis pertahanan berbeda.
4. **Akses folder Drive**: gunakan `DriveApp.getFolderById()` (ID langsung), bukan `getFoldersByName()` — yang terbukti rawan gagal otorisasi.
5. **Inisialisasi peta Leaflet**: container harus sudah punya tinggi CSS valid (bukan `0px`) sebelum `L.map()` dipanggil; gunakan `invalidateSize()` dengan jeda setelah view berpindah dari `display:none`.
6. **GitHub Pages untuk repository privat** memerlukan akun berbayar (Pro/Team/Enterprise) — repository ini dipertahankan publik dengan seluruh kredensial/ID disimpan terpisah demi tetap bisa memakai GitHub Pages gratis.

---

## Struktur Repository

```
.
├── dashboard_v3_pelaporan.html   # Dashboard utama (single-file)
├── logo-oikn.png                  # Logo landing page
├── Batang_Banyu.png                # Motif dekoratif landing page
└── README.md                       # Dokumen ini
```

---

## Riwayat Pengembangan Singkat

| Tahap | Status |
|---|---|
| 1–3: Audit sumber, desain Sheet Master, agregator otomatis | Selesai |
| 4: Geocoding | Tidak diperlukan (GPS langsung dari form) |
| 5: Web App API | Selesai, live |
| 6: Dashboard HTML lengkap | Selesai |
| 7: Deploy GitHub Pages | *(lihat status di bagian atas)* |
| Revisi form pelaporan multi-bidang (Pol PP/Polhut/Damkar + multi-select) | Selesai |
| Laporan Real-time multi-bidang dengan halaman antara + peta gabungan | Selesai |

---

*Dokumen ini dikelola oleh Direktorat Trantibum, Otorita Ibu Kota Nusantara. Untuk pertanyaan teknis atau akses ke konfigurasi internal, hubungi pengelola sistem.*
