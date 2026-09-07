# Rak Perpustakaan

Portal multi-halaman berisi alat kerja tim + **Kelola Alteco** — aplikasi manajemen organisasi berbasis web (single-file, GitHub Pages, tanpa server).

---

## Struktur Repo

```
/
├── index.html              ← Homepage (nav ke semua rak)
├── proyekan/index.html     ← Referensi command & AI tools
├── pengajar/index.html     ← Sektor industri + modul ERP
├── pemasaran/index.html    ← Siklus penawaran
├── mitra/index.html        ← Logo & info mitra
├── todolist/index.html     ← Kelola Alteco (app utama)
├── OneSignalSDKWorker.js   ← Service worker push notif
└── .github/workflows/
    └── daily-notif.yml     ← Cron harian kirim push notif
```

---

## Fork & Deploy ke Repo Baru

### 1. Fork / Clone

```bash
git clone https://github.com/dionisiusnp/dionisiusnp.github.io.git
cd dionisiusnp.github.io
```

Atau fork via GitHub UI → rename repo sesuai username: `<username>.github.io`

### 2. Aktifkan GitHub Pages

GitHub → repo → **Settings → Pages → Source: Deploy from branch → branch: main → / (root)**

Situs live di: `https://<username>.github.io`

---

## Setup Layanan Eksternal

Kelola Alteco butuh 3 layanan eksternal: **GitHub Gist** (data), **OneSignal** (push notif), dan **GitHub Actions** (cron). Firebase Hosting opsional sebagai alternatif GitHub Pages.

---

## A — GitHub Gist (Penyimpanan Data)

Data organisasi disimpan di Gist agar semua device dapat data terbaru tanpa backend server.

### A1. Buat Gist

1. Buka [gist.github.com](https://gist.github.com)
2. Isi:
   - **Description**: nama organisasimu
   - **Filename**: `alteco-data.json`
   - **Content**:
     ```json
     {"kelompoks":[],"members":[],"tasks":[],"assets":[],"events":[],"projects":[],"cultures":[],"orgName":"Nama Organisasi"}
     ```
3. Klik **Create secret gist**
4. Salin **Gist ID** dari URL:
   ```
   https://gist.github.com/<username>/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
                                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   ```

### A2. Buat GitHub Personal Access Token (PAT)

1. Buka [github.com/settings/tokens](https://github.com/settings/tokens)
2. **Generate new token (classic)**
3. Isi:
   - **Note**: `kelola-gist`
   - **Expiration**: No expiration
   - **Scope**: centang `gist` saja
4. Generate → **salin token sekarang** (hanya tampil sekali)

### A3. Pasang Gist ID ke Kode

Buka `todolist/index.html`, baris ~596:

```javascript
const GIST_ID = 'GIST_ID_DISINI';
```

Ganti dengan Gist ID dari A1. Commit & push.

> PAT **jangan** ditaruh di kode. Diinput manual saat login pertama — tersimpan di `localStorage` browser.

---

## B — OneSignal (Push Notification)

### B1. Daftar & Buat App

1. Buka [onesignal.com](https://onesignal.com) → daftar akun gratis
2. **New App** → beri nama → pilih platform **Web**
3. Setup Web:
   - **Integration**: `Custom Code`
   - **Site URL**: `https://<username>.github.io` (atau domain Firebase jika pakai hosting Firebase)
   - **Default Icon URL**: URL ikon organisasimu (opsional)
4. Selesaikan wizard

### B2. Ambil Credentials

Di dashboard OneSignal → **Settings → Keys & IDs**:

| Key | Format |
|-----|--------|
| **App ID** | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| **REST API Key** | `os_v2_...` |

### B3. Pasang App ID ke Kode

Buka `todolist/index.html`, baris ~593:

```javascript
const ONESIGNAL_APP_ID = 'ONESIGNAL_APP_ID_DISINI';
```

Ganti dengan App ID dari B2. Commit & push.

### B4. Service Worker

File `OneSignalSDKWorker.js` di root repo sudah siap — tidak perlu diubah:

```javascript
importScripts('https://cdn.onesignal.com/sdks/web/v16/OneSignalSDK.sw.js');
```

> **iOS Safari**: Push notif web hanya berfungsi jika situs di-*Add to Home Screen* dan iOS ≥ 16.4.

---

## C — Firebase Hosting (Opsional — Alternatif GitHub Pages)

Gunakan jika butuh custom domain lebih mudah atau ingin deploy di luar GitHub Pages.

### C1. Buat Project Firebase

1. Buka [console.firebase.google.com](https://console.firebase.google.com)
2. **Add project** → beri nama → (opsional) nonaktifkan Google Analytics
3. Tunggu project dibuat

### C2. Install Firebase CLI

```bash
npm install -g firebase-tools
firebase login
```

### C3. Init Hosting

Di root repo:

```bash
firebase init hosting
```

Pilih:
- **Project**: pilih project yang dibuat di C1
- **Public directory**: `.` (root — karena `index.html` ada di root)
- **Single-page app**: `No`
- **GitHub automatic deploys**: opsional

Firebase akan membuat `firebase.json` dan `.firebaserc`.

### C4. Konfigurasi `firebase.json`

```json
{
  "hosting": {
    "public": ".",
    "ignore": ["firebase.json", ".firebaserc", ".git/**", "README.md"],
    "headers": [
      {
        "source": "/OneSignalSDKWorker.js",
        "headers": [{ "key": "Service-Worker-Allowed", "value": "/" }]
      }
    ]
  }
}
```

> Header `Service-Worker-Allowed` wajib agar OneSignal service worker bisa berjalan dari root scope.

### C5. Deploy

```bash
firebase deploy --only hosting
```

Situs live di: `https://<project-id>.web.app`

Update **Site URL** di OneSignal (B1) ke domain Firebase ini.

---

## D — GitHub Actions (Cron Notifikasi Harian)

Workflow di `.github/workflows/daily-notif.yml` mengirim push notif setiap hari pukul 00:01 WIB.

### D1. Pasang Repository Secrets

GitHub → repo → **Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Nilai | Dari |
|--------|-------|------|
| `GIST_ID` | Gist ID | Langkah A1 |
| `GIST_PAT` | GitHub PAT | Langkah A2 |
| `ONESIGNAL_APP_ID` | OneSignal App ID | Langkah B2 |
| `ONESIGNAL_REST_KEY` | OneSignal REST API Key | Langkah B2 |

### D2. Aktifkan Push Notification di App

1. Buka app → login superadmin
2. **Pengaturan** → aktifkan **Push Notification**
3. Set **Waktu Kirim** (default 07:00 WIB)
4. Set **Notif Mulai H-** (berapa hari sebelum acara mulai notif dikirim)
5. Simpan

### D3. Trigger Manual

GitHub → **Actions → Daily Notification → Run workflow**

Centang `Force send` untuk kirim ulang semua acara aktif tanpa update `notifSentDate`.

---

## E — Konfigurasi Kode Akses

Password login ada di `todolist/index.html` baris ~589:

```javascript
const PASS_VIEW  = 'GANTI_KODE_VIEWER';   // akses baca semua data
const PASS_ADMIN = 'GANTI_KODE_ADMIN';    // akses penuh + CRUD
```

Ganti sesuai kebutuhan, commit & push.

> Ini bukan auth yang aman untuk data sensitif — cukup untuk use case internal komunitas.

---

## F — Login Pertama (Input PAT)

1. Buka situs → masukkan kode admin
2. Muncul prompt: *"Masukkan GitHub PAT untuk sinkronisasi data"*
3. Paste PAT dari A2 → OK
4. PAT tersimpan di `localStorage` — tidak perlu input ulang di browser yang sama

Reset PAT jika perlu ganti:
```javascript
// Di browser console
localStorage.removeItem('rak_alteco_pat');
```

---

## Ringkasan Checklist Deploy

- [ ] Fork repo & aktifkan GitHub Pages (atau setup Firebase Hosting)
- [ ] Buat GitHub Gist → salin Gist ID
- [ ] Buat GitHub PAT (scope: `gist`)
- [ ] Pasang `GIST_ID` ke `todolist/index.html`
- [ ] Daftar OneSignal → pasang `ONESIGNAL_APP_ID` ke `todolist/index.html`
- [ ] Commit & push
- [ ] Pasang 4 secrets ke GitHub Actions
- [ ] Buka app → login admin → input PAT → aktifkan push notif
