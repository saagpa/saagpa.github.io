# Panduan Setup — Kelola Alteco

Data Kelola Alteco disimpan di GitHub Gist agar dapat diakses dari semua device.
Push notification dikirim otomatis via OneSignal + GitHub Actions setiap hari.
Viewer cukup buka URL — tidak perlu konfigurasi apapun.
Superadmin perlu melakukan setup sekali berikut ini.

---

## Prasyarat

- Akun GitHub (pemilik repo `dionisiusnp.github.io`)
- Akun OneSignal (gratis) — untuk push notification

---

## Langkah 1 — Buat Gist

1. Buka [gist.github.com](https://gist.github.com)
2. Isi form:
   - **Gist description**: `Alteco Data`
   - **Filename**: `alteco-data.json`
   - **Content**:
     ```json
     {"kelompoks":[],"members":[],"tasks":[],"assets":[],"events":[],"orgName":"Alteco"}
     ```
3. Klik **"Create secret gist"**
4. Salin **Gist ID** dari URL browser:
   ```
   https://gist.github.com/dionisiusnp/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
                                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                        ini Gist ID-nya
   ```

---

## Langkah 2 — Buat Personal Access Token (PAT)

1. Buka [github.com/settings/tokens](https://github.com/settings/tokens)
2. Klik **"Generate new token (classic)"**
3. Isi:
   - **Note**: `alteco-gist`
   - **Expiration**: `No expiration`
   - **Scope**: centang **`gist`** saja
4. Klik **"Generate token"**
5. **Salin token sekarang** — hanya tampil sekali
   ```
   ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

---

## Langkah 3 — Pasang Gist ID ke Kode

Buka file `todolist/index.html`, cari baris:

```javascript
const GIST_ID  = 'GIST_ID_DISINI';
```

Ganti dengan Gist ID dari Langkah 1:

```javascript
const GIST_ID  = '6bf5ebb0397393324168d3dd14c018c8'; // contoh
```

Commit dan push ke GitHub.

> **Catatan**: Jangan taruh PAT di kode — PAT diinput manual saat login (lihat Langkah 4).

---

## Langkah 4 — Input PAT Saat Login Pertama

1. Buka halaman Kelola Alteco
2. Login dengan kode **superadmin**
3. Muncul prompt:
   ```
   Masukkan GitHub PAT untuk sinkronisasi data:
   ```
4. Paste PAT dari Langkah 2 → klik OK
5. PAT tersimpan di `localStorage` browser — tidak perlu input lagi di browser yang sama

---

## Langkah 5 — Setup OneSignal (Push Notification)

### 5a. Buat aplikasi OneSignal

1. Buka [onesignal.com](https://onesignal.com) → daftar akun gratis
2. Buat **New App** → pilih platform **Web**
3. Pilih **Custom Code** sebagai integration type
4. Isi **Site URL**: `https://dionisiusnp.github.io`
5. Selesaikan wizard → catat **App ID**

### 5b. Ambil App ID dan REST API Key

1. Di dashboard OneSignal, buka **Settings → Keys & IDs**
2. Salin:
   - **OneSignal App ID** (format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`)
   - **REST API Key** (format: `os_v2_...`)

### 5c. Pasang App ID ke kode

Buka `todolist/index.html`, cari:

```javascript
const ONESIGNAL_APP_ID = 'ONESIGNAL_APP_ID_DISINI';
```

Ganti dengan App ID dari OneSignal:

```javascript
const ONESIGNAL_APP_ID = 'b03c4a75-c2b0-4de5-ab9c-cb5aa2f58cd2'; // contoh
```

Commit dan push.

### 5d. Service Worker

File `OneSignalSDKWorker.js` sudah ada di root repo. Tidak perlu diubah:

```javascript
importScripts('https://cdn.onesignal.com/sdks/web/v16/OneSignalSDK.sw.js');
```

---

## Langkah 6 — Setup GitHub Secrets

GitHub Actions butuh 4 secrets untuk mengirim notifikasi harian.

1. Buka repo di GitHub → **Settings → Secrets and variables → Actions**
2. Klik **"New repository secret"**, tambahkan satu per satu:

| Secret Name | Nilai |
|---|---|
| `GIST_ID` | Gist ID dari Langkah 1 |
| `GIST_PAT` | PAT dari Langkah 2 |
| `ONESIGNAL_APP_ID` | App ID dari Langkah 5b |
| `ONESIGNAL_REST_KEY` | REST API Key dari Langkah 5b |

---

## Langkah 7 — Aktifkan Push Notification

1. Buka halaman Kelola Alteco → login superadmin
2. Di sidebar, klik **Pengaturan ⚙️**
3. Aktifkan toggle **Push Notification**
4. Set **Waktu Kirim** (jam pengiriman notifikasi, default 07:00 WIB)
5. Set **Notif Mulai H-** (berapa hari sebelum acara sekali notifikasi mulai dikirim, default 1)
6. Klik **Simpan**

---

## Alur Kerja Data

| Aksi | Penjelasan |
|------|-----------|
| Buka halaman | Data dimuat dari Gist (semua device dapat data terbaru) |
| Tambah/edit data (admin) | Tersimpan ke Gist dalam ~600ms setelah perubahan |
| Viewer buka halaman | Baca Gist tanpa PAT — tidak perlu konfigurasi |
| Gist tidak terjangkau | Fallback ke data lokal browser |

---

## Fitur Informasi

Superadmin dapat membuat **pengumuman/acara** di menu **Informasi 🔔**.

### Tipe acara

| Tipe | Penjelasan |
|------|-----------|
| **Sekali** | Acara pada tanggal tertentu. Notifikasi dikirim setiap hari mulai H-X sampai hari H. |
| **Berulang** | Acara rutin. Notifikasi hanya dikirim pada hari H sesuai pola pengulangan. |

### Pola pengulangan (Berulang)

| Pola | Notifikasi dikirim |
|------|-------------------|
| Harian | Setiap hari |
| Mingguan | Hari yang sama dengan tanggal acuan |
| Bulanan | Tanggal yang sama dengan tanggal acuan |
| Tahunan | Tanggal & bulan yang sama dengan tanggal acuan |

### Bell badge

Ikon lonceng di navbar menampilkan jumlah pengumuman baru/diperbarui yang belum dibaca.
Badge hilang setelah membuka halaman Informasi. Badge muncul kembali jika ada pengumuman baru atau yang diedit.

---

## Fitur Pengaturan (Superadmin)

Tersedia di sidebar menu **Pengaturan ⚙️** — hanya terlihat oleh superadmin.

| Field | Penjelasan |
|-------|-----------|
| **Push Notification** | Toggle aktif/nonaktif push notification. Jika nonaktif, notifikasi web biasa saja yang berjalan (browser harus terbuka). |
| **Waktu Kirim** | Jam pengiriman push notification dalam WIB (contoh: `07:00`). |
| **Notif Mulai H-** | Berapa hari sebelum acara *sekali* notifikasi mulai dikirim. Default: 1 hari. Berlaku global untuk semua acara. |
| **OneSignal REST Key** | REST API Key dari OneSignal (dari Langkah 5b). Disimpan di Gist. Diperlukan untuk tombol Kirim Sekarang. |

### Kirim Rangkuman Manual

Tombol **Kirim Sekarang** di bagian *Kirim Manual* mengirim push notification berisi semua informasi yang tanggalnya belum terlewat (`date >= hari ini`) ke seluruh subscriber OneSignal.

- Tidak mengubah `notifSentDate` — status notif harian tidak terganggu
- Acara berulang selalu ikut dirangkum (tidak ada tanggal kadaluarsa)
- Membutuhkan OneSignal REST Key sudah diisi dan disimpan

---

## Jadwal GitHub Actions

File workflow: `.github/workflows/daily-notif.yml`

```
Cron: 1 17 * * *  →  00:01 WIB
```

| Langkah | Penjelasan |
|---------|-----------|
| Baca Gist | Ambil data terbaru termasuk settings |
| Cek `pushNotifEnabled` | Jika false, berhenti |
| Filter acara hari ini | Cek tipe, tanggal, dan `notifSentDate` per acara |
| Kirim 1 bundled push | Semua acara hari ini dalam satu notifikasi |
| `send_after` | OneSignal mengirim pada jam `notifTime` WIB (bukan 00:01) |
| Update `notifSentDate` | Tandai sudah dikirim agar tidak duplikat hari yang sama |

### Dedup per acara

Setiap acara punya field `notifSentDate`. Workflow skip acara jika `notifSentDate === hari ini`.
Saat acara dibuat atau diedit, `notifSentDate` di-reset ke `null` agar notifikasi aktif kembali.

### Trigger manual

Workflow bisa dijalankan manual: **GitHub → Actions → Daily Notification → Run workflow**.

---

## Mode Notifikasi

| Mode | Kondisi | Perilaku |
|------|---------|---------|
| Push ON | `pushNotifEnabled = true`, browser/device sudah allow | Push notification muncul meski halaman tertutup |
| Push OFF | `pushNotifEnabled = false` | Notifikasi web biasa — hanya muncul saat halaman terbuka |

> **iOS Safari**: Push notification web hanya berfungsi jika situs ditambahkan ke Home Screen (Add to Home Screen) dan iOS ≥ 16.4. Browser Safari biasa tidak mendukung.

---

## Reset & Maintenance

### Reset PAT (jika perlu ganti)

```javascript
// Jalankan di browser console
localStorage.removeItem('rak_alteco_pat');
```

Refresh → login ulang sebagai superadmin → input PAT baru.

### Revoke PAT (jika bocor)

1. Buka [github.com/settings/tokens](https://github.com/settings/tokens)
2. Klik **Delete** pada token `alteco-gist`
3. Buat PAT baru (ulangi Langkah 2)
4. Reset PAT di browser (lihat Reset PAT di atas)

### Reset status notifikasi (force kirim ulang)

Edit salah satu acara di Informasi → simpan. `notifSentDate` reset ke `null` → notif akan dikirim di run berikutnya.
