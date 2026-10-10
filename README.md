# Seabasst Finance Dashboard

Dashboard keuangan pribadi bergaya Monefy, mobile first, yang membaca dan menulis langsung ke Google Sheets. Bisa dipakai di browser, disimpan ke homescreen, atau dibuka sebagai Telegram Mini App. Seluruh komponen berjalan gratis.

## Arsitektur

```
Telegram Bot (Apps Script, project terpisah)  ──┐
                                                ├──►  Google Sheets (sheet "Finance", "budgeting")
Dashboard (index.html di GitHub Pages)          │
   └── memanggil API ──► Apps Script Web App ───┘
        (Code.gs, project terpisah dari bot)
```

| Komponen | Fungsi | Lokasi |
|---|---|---|
| `index.html` | Antarmuka dashboard (satu file, tanpa build) | GitHub Pages |
| `Code.gs` | API: baca, tambah, edit, hapus transaksi, ambil budget | Project Apps Script khusus dashboard |
| Bot Seabasst | Input via Telegram (tidak diubah oleh proyek ini) | Project Apps Script bot |
| Google Sheets | Database bersama | Sheet `Finance` dan `budgeting` |

Bot dan dashboard adalah dua project terpisah yang memakai spreadsheet yang sama. Kalau salah satunya error, yang lain tetap berjalan.

## Fitur

- Filter periode Hari, Bulan, Tahun. Tap label periode untuk memilih langsung lewat picker. Geser kiri/kanan untuk pindah periode.
- Donut chart pengeluaran per kategori dengan legend. Tap kategori untuk memfilter riwayat, tap lagi atau tap tombol `✕ Kategori` untuk kembali.
- Anggaran per kategori (mode Bulan): lingkaran persen hijau, kuning (mulai 80%), merah (di atas 100%), plus total anggaran.
- Riwayat dengan tiga mode urut: Tanggal, Kategori (collapse per kategori), Nominal (tap sekali menurun, tap lagi menaik).
- Pencarian di keterangan, kategori, dan nominal (menelusuri semua periode).
- Tambah, edit, dan hapus transaksi. Input nominal memakai keypad kalkulator (+ − × ÷ =).
- Analitik (tombol 📊): line chart interaktif, timeframe Harian/Mingguan/Bulanan/Tahunan, toggle Keluar/Masuk/Selisih, filter kategori multi-pilih, dan ringkasan EDA.
- Data di-cache di perangkat sehingga navigasi dan filter instan. Perubahan tampil langsung, sinkron ke sheet berjalan di belakang.

## Struktur Spreadsheet

Sheet `Finance` (satu baris per transaksi):

| Waktu_Moment | Tipe | Nominal | Kategori | Keterangan | Waktu_Entry | UID |
|---|---|---|---|---|---|---|
| 2026-08-23 11:59:49 | Out | 80000 | Daily Meals | Beli makanan | 2026-08-23 11:59:49 | FIN-260823-58A6 |

- `Tipe`: `In` atau `Out`. Investasi adalah kategori (`Invest`) dengan tipe `Out`.
- `Nominal`: angka murni. Tampilan Rp hanya format.
- `UID`: `FIN-yyMMdd-XXXX`, dipakai sebagai kunci edit dan hapus.
- Zona waktu: Asia/Jakarta.

Sheet `budgeting`: kolom `Kategori` dan `Nominal_Budget` (budget bulanan tetap).

Kategori resmi: Kitchen, Daily Meals, Self Reward, Transport, Health, Invest, Papan, Bills, Monthly Expenses, Other.

## Setup dari Nol

### 1. Backend (Apps Script)

1. Buka script.google.com, buat **New project** (misalnya `Seabasst Dashboard API`), tempel isi `Code.gs`.
2. **Project Settings → Script Properties**, tambahkan:
   - `SPREADSHEET_ID`: ID spreadsheet (bagian URL antara `/d/` dan `/edit`).
   - `API_SECRET`: string acak yang panjang. Ini password dashboard, simpan baik-baik dan jangan dibagikan.
3. Jalankan fungsi `testRead` dari editor dan izinkan akses. Cek **Execution log**: harus muncul jumlah transaksi.
4. **Deploy → New deployment → Web app**, isi *Execute as: Me* dan *Who has access: Anyone*. Salin URL Web App.

### 2. Frontend (GitHub Pages)

1. Di `index.html`, pastikan konstanta `API` berisi URL Web App dari langkah di atas.
2. Buat repository GitHub (Public), upload `index.html`.
3. **Settings → Pages**: Branch `main`, folder `/ (root)`, Save.
4. Setelah 1 sampai 2 menit, dashboard bisa dibuka di `https://username.github.io/nama-repo/`.
5. Saat pertama dibuka, masukkan `API_SECRET`. Tersimpan di perangkat itu dan tidak ditanya lagi.

Tes lokal tanpa hosting: buka `index.html` dengan klik dua kali.

### 3. Telegram Mini App

1. Di **@BotFather**: `/mybots` → pilih bot → **Bot Settings → Menu Button → Configure menu button**.
2. Kirim URL GitHub Pages, lalu teks tombol (misalnya `Dashboard`).
3. Buka chat bot, tap tombol **Dashboard**, masukkan `API_SECRET` sekali (penyimpanan Telegram terpisah dari browser).

Opsional, tombol di dalam chat lewat perintah `/dashboard`:

```javascript
function sendDashboardButton(chatId) {
  const token = PropertiesService.getScriptProperties().getProperty("TELEGRAM_TOKEN");
  UrlFetchApp.fetch("https://api.telegram.org/bot" + token + "/sendMessage", {
    method: "post",
    contentType: "application/json",
    payload: JSON.stringify({
      chat_id: chatId,
      text: "Buka dashboard keuangan:",
      reply_markup: { inline_keyboard: [[{ text: "📊 Dashboard", web_app: { url: "URL_GITHUB_PAGES" } }]] }
    })
  });
}
```

## Cara Update

### A. Update tampilan atau fitur (`index.html`)

1. Siapkan `index.html` yang baru. Simpan salinan versi lama dengan nama berbeda (misalnya `index-v2.html`) sebagai cadangan.
2. Di repo GitHub, pilih salah satu cara:
   - **Upload:** Add file → Upload files → drag `index.html` (nama harus persis `index.html`) → Commit changes.
   - **Edit langsung:** buka `index.html` → ikon pensil → pilih semua isi lama, tempel isi baru → Commit changes.
3. Tunggu 1 sampai 2 menit. Verifikasi dengan refresh paksa (`Ctrl+Shift+R`). Di HP atau Telegram, tutup lalu buka lagi aplikasinya.
4. Tidak perlu mengubah apa pun di Telegram atau Apps Script.

### B. Update backend (`Code.gs`)

Menyimpan kode saja **tidak cukup**. Web App harus dideploy ulang agar perubahan aktif:

1. Edit `Code.gs` di project Apps Script, lalu Save.
2. **Deploy → Manage deployments** → klik ikon pensil pada deployment aktif.
3. Di *Version*, pilih **New version**, lalu **Deploy**.
4. URL Web App tetap sama, jadi `index.html` tidak perlu diubah.

### C. Mengubah budget

1. Buka spreadsheet, sheet `budgeting`.
2. Ubah angka di kolom `Nominal_Budget`.
3. Di dashboard, tap **⟳** atau buka ulang aplikasinya. Jangan mengubah nama di kolom `Kategori`.

### D. Mengganti API secret

1. Apps Script → **Project Settings → Script Properties** → ubah nilai `API_SECRET` → Save. Tidak perlu deploy ulang.
2. Perangkat yang memakai secret lama otomatis diminta memasukkan yang baru saat dibuka berikutnya.
3. Lupa secret? Lihat nilainya di halaman Script Properties yang sama.

### E. Menambah atau mengubah kategori

Daftar kategori harus sama di **tiga tempat**:

1. Konfigurasi bot (daftar 10 kategori dan logika klasifikasinya).
2. `Code.gs`: konstanta `CATEGORIES`, lalu deploy ulang (lihat B).
3. `index.html`: objek `CATS` (ikon dan warna), lalu update seperti A.

Sheet `budgeting` juga perlu baris untuk kategori baru.

### F. Kembali ke versi sebelumnya

- **Frontend:** upload ulang salinan lama dengan nama `index.html`. Atau di GitHub buka `index.html` → **History** → pilih commit lama → View file → salin isinya.
- **Backend:** Apps Script → Deploy → Manage deployments → pensil → pilih versi lama pada *Version* → Deploy.

## Referensi API

Satu endpoint (URL Web App), request `POST` dengan `Content-Type: text/plain;charset=utf-8` dan body JSON. Semua request wajib menyertakan `key` (nilai `API_SECRET`).

| action | Parameter | Hasil |
|---|---|---|
| `ping` | - | Waktu server |
| `meta` | - | Daftar kategori dan tipe |
| `list` | `from`, `to` (opsional, `yyyy-MM-dd`) | Transaksi terbaru dulu |
| `budgets` | - | Daftar `{kategori, budget}` |
| `add` | `tipe`, `nominal`, `kategori`, `keterangan`, `waktu` (opsional) | `{uid}` |
| `edit` | `uid` + field yang diubah | `{uid}` |
| `delete` | `uid` | `{uid, deleted}` |

Respons selalu berbentuk `{ ok: true, data: ... }` atau `{ ok: false, error: "..." }`.

## Pemecahan Masalah

| Gejala | Penyebab dan solusi |
|---|---|
| Layar meminta secret terus | Secret salah. Cek `API_SECRET` di Script Properties. |
| Error `Unauthorized` | Sama seperti di atas. |
| Data dari bot belum muncul | Tap **⟳**. Data di-cache dan disegarkan saat dibuka atau setelah 1 menit. |
| Perubahan `index.html` belum tampil | Tunggu 1 sampai 2 menit, refresh paksa, atau tutup lalu buka ulang aplikasi. |
| Perubahan `Code.gs` tidak berpengaruh | Deploy ulang dengan **New version** (lihat bagian B). |
| `Format waktu tidak valid` | Waktu entri harus `yyyy-MM-dd HH:mm(:ss)`. |
| Error `Header kolom tidak ada` | Nama header sheet `Finance` harus persis seperti tabel struktur di atas. |
| Simpan terasa 1 sampai 3 detik di belakang layar | Normal, latensi Apps Script. Tampilan sudah langsung berubah. |

## Keamanan

- `API_SECRET` tidak ada di kode `index.html`, jadi repo Public tetap aman. Jangan menuliskannya di README, chat umum, atau commit.
- URL Web App boleh diketahui orang, tetapi tanpa secret tidak ada data yang bisa dibaca atau diubah.
- Untuk kontrol lebih ketat, ubah repo menjadi Private (GitHub Pages untuk repo Private butuh paket berbayar, jadi pertimbangkan sebelum mengubah).

## Batas Gratis

Apps Script, GitHub Pages, Google Sheets, dan Telegram Bot semuanya gratis untuk pemakaian pribadi. Apps Script punya kuota harian eksekusi, tetapi jauh di atas kebutuhan satu pengguna.
