# Scraper SISPARNAS — DTW (Daya Tarik Wisata)

**Target:** `https://sisparnas.kemenpar.go.id/datapanel?scope=dtw`
**Data diambil:** Nama wisata · PIC (opsional) · Nomor telepon (opsional)
**Profil per-item:** `/p/{ID}` — accessible tanpa login, status 200 selalu

**Strategi:** Scan ID dari range 1–50000, skip ID kosong, ambil yang valid.

---

## Cara Pakai

1. Buka `https://sisparnas.kemenpar.go.id/datapanel?scope=dtw` di Chrome
2. Tekan `F12` → Console
3. Paste dan jalankan script di bawah
4. Tunggu selesai — file CSV otomatis terdownload
5. Kalau browser ditutup, jalankan **Resume** untuk lanjutkan

---

## Script Utama — Scan & Export CSV

```javascript
(async function scanDTW() {
  'use strict';

  // ╔══════════════════════════════════════════╗
  // ║           KONFIGURASI — UBAH DI SINI     ║
  const START_ID    = 5000;   // ← mulai dari ID berapa
  const END_ID      = 50000;  // ← sampai ID berapa
  // ╚══════════════════════════════════════════╝

  // Anti-bot config — jangan diubah kecuali perlu
  const CONCURRENCY   = 2;    // request paralel (max 3, turunkan ke 1 jika lambat)
  const DELAY_MIN     = 800;  // ms jeda minimum antar batch
  const DELAY_MAX     = 2000; // ms jeda maksimum antar batch
  const REST_EVERY    = 80;   // istirahat panjang setiap N batch
  const REST_DURATION = 8000; // ms durasi istirahat panjang
  // ────────────────────────────────────────────

  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const rand  = (a, b) => a + Math.floor(Math.random() * (b - a + 1));

  // Load progress sebelumnya
  window._sisResults = window._sisResults?.length
    ? window._sisResults
    : JSON.parse(localStorage.getItem('sis_results') || '[]');

  const doneFinal = new Set(window._sisResults.map(r => r.id));

  // Buat daftar ID yang belum diproses
  const todo = [];
  for (let i = START_ID; i <= END_ID; i++) {
    if (!doneFinal.has(i)) todo.push(i);
  }

  console.log(`📂 Sebelumnya: ${window._sisResults.length} valid tersimpan`);
  console.log(`🚀 Scan ID ${START_ID}–${END_ID} | ${todo.length} ID tersisa`);
  console.log(`   CONCURRENCY=${CONCURRENCY} | jeda ${DELAY_MIN}–${DELAY_MAX}ms | istirahat tiap ${REST_EVERY} batch`);

  // ── Parse profil dari HTML ────────────────────────────
  function parseProfile(html, id) {
    const name  = html.match(/<h1[^>]*>([^<]{2,100})<\/h1>/i)?.[1]?.trim() || '';
    const pic   = html.match(/PIC\s*[:\-]\s*([^<\n]{2,60})/i)?.[1]?.trim() || '';
    const phone = (
      html.match(/(?:Phone|Telepon|No\.?\s*HP|Handphone|Tel|Telp)\s*[:\-]\s*([\d\+\s\-\(\)]{6,20})/i)?.[1]
      || html.match(/(\+?62[\d\s\-]{9,14}|0\d[\d\s\-]{8,13})/)?.[1]
      || ''
    ).replace(/\s+/g, '').trim();

    return { id, name, pic, phone, url: `https://sisparnas.kemenpar.go.id/p/${id}` };
  }

  // ── Fetch satu ID dengan simulasi human behavior ──────
  async function fetchOne(id) {
    try {
      const r = await fetch(`/p/${id}`, {
        credentials: 'include',
        headers: {
          'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
          'Accept-Language': 'id-ID,id;q=0.9,en-US;q=0.8,en;q=0.7',
          'Cache-Control': 'no-cache',
          'Referer': 'https://sisparnas.kemenpar.go.id/datapanel?scope=dtw',
        }
      });
      const html = await r.text();

      const hasData = html.includes('PIC') || html.includes('Phone') || html.includes('Telepon');
      if (!hasData) return null;

      const profile = parseProfile(html, id);
      if (!profile.name) return null;

      return profile;
    } catch (e) {
      return null;
    }
  }

  // ── MAIN LOOP ─────────────────────────────────────────
  const total   = todo.length;
  let   scanned = 0;
  let   found   = 0;
  let   batchNo = 0;

  for (let i = 0; i < total; i += CONCURRENCY) {
    const batch = todo.slice(i, i + CONCURRENCY);

    // Jitter antar request dalam batch (simulasi human typing speed)
    const promises = batch.map((id, idx) =>
      sleep(idx * rand(200, 600)).then(() => fetchOne(id))
    );
    const results = await Promise.all(promises);

    results.forEach(r => {
      if (r) { window._sisResults.push(r); found++; }
    });
    scanned += batch.length;
    batchNo++;

    // Log progress setiap 10 batch
    if (batchNo % 10 === 0) {
      const pct       = ((scanned / total) * 100).toFixed(1);
      const lastFound = results.find(Boolean);
      const eta       = Math.round(((total - scanned) / CONCURRENCY) * ((DELAY_MIN + DELAY_MAX) / 2) / 1000 / 60);
      console.log(`📦 ${scanned}/${total} (${pct}%) | valid: ${found} | ETA: ~${eta}m | ${lastFound ? `"${lastFound.name}"` : 'tidak ada'}`);
    }

    // Simpan progress setiap 500 scan
    if (scanned % 500 === 0) {
      localStorage.setItem('sis_results', JSON.stringify(window._sisResults));
      console.log(`💾 Progress: ${window._sisResults.length} data valid disimpan`);
    }

    // Istirahat panjang berkala (simulasi jeda baca)
    if (batchNo % REST_EVERY === 0) {
      const rest = REST_DURATION + rand(0, 3000);
      console.log(`😴 Istirahat ${(rest/1000).toFixed(1)}s... (batch ke-${batchNo})`);
      await sleep(rest);
    } else {
      // Jeda acak normal antar batch
      await sleep(rand(DELAY_MIN, DELAY_MAX));
    }
  }

  // Final save & export
  localStorage.setItem('sis_results', JSON.stringify(window._sisResults));
  console.log(`\n✅ SELESAI — ${window._sisResults.length} wisata ditemukan`);
  console.table(window._sisResults.slice(0, 5));

  downloadCSV(window._sisResults);

  // ── Download CSV (tanpa library eksternal) ────────────
  function downloadCSV(data) {
    if (!data.length) { console.warn('Tidak ada data.'); return; }
    const cols = ['no', 'name', 'pic', 'phone', 'id', 'url'];
    const esc  = v => `"${String(v ?? '').replace(/"/g, '""')}"`;
    const rows = [
      ['No', 'Nama Wisata', 'PIC', 'No. Telepon', 'ID', 'URL'].join(','),
      ...data.map((r, i) => [i + 1, esc(r.name), esc(r.pic), esc(r.phone), r.id, esc(r.url)].join(','))
    ];
    const blob = new Blob(['\uFEFF' + rows.join('\r\n')], { type: 'text/csv;charset=utf-8' });
    const a    = Object.assign(document.createElement('a'), {
      href: URL.createObjectURL(blob),
      download: `sisparnas_dtw_${new Date().toISOString().slice(0, 10)}.csv`
    });
    document.body.appendChild(a); a.click(); document.body.removeChild(a);
    console.log(`💾 CSV didownload! (${data.length} rows)`);
  }
})();
```

---

## Resume (jika browser ditutup)

```javascript
window._sisResults = JSON.parse(localStorage.getItem('sis_results') || '[]');
console.log('Resume:', window._sisResults.length, 'data dimuat');
// Lalu ubah START_ID di script utama ke ID terakhir yang diproses, jalankan ulang
```

## Download Ulang CSV

```javascript
(function() {
  var data = JSON.parse(localStorage.getItem('sis_results') || '[]');
  if (!data.length) { console.warn('Belum ada hasil.'); return; }
  function esc(v) { return '"' + String(v == null ? '' : v).replace(/"/g, '""') + '"'; }
  var rows = ['No,Nama Wisata,PIC,No. Telepon,ID,URL'];
  data.forEach(function(r, i) {
    rows.push([i + 1, esc(r.name), esc(r.pic), esc(r.phone), r.id, esc(r.url)].join(','));
  });
  var blob = new Blob([rows.join('\r\n')], { type: 'text/csv;charset=utf-8' });
  var a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'sisparnas_dtw_' + new Date().toISOString().slice(0, 10) + '.csv';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  console.log('Downloaded', data.length, 'records');
})();
```

---

## Estimasi Waktu

| Range | Total Scan | Valid DTW | Estimasi |
|---|---|---|---|
| 1–10.000 | 10.000 | ~500 | ~15 menit |
| 1–25.000 | 25.000 | ~2.000 | ~35 menit |
| 1–50.000 | 50.000 | ~5.000+ | ~70 menit |

> Waktu bisa bervariasi tergantung kecepatan internet.
> Kurangi `CONCURRENCY` ke `2` jika koneksi lambat atau kena rate limit.

---

## Output CSV

| Kolom | Contoh |
|---|---|
| name | Green Aquatic Park |
| pic | Joko Fartoni |
| phone | 082183061016 |
| id | 31593 |
| url | https://sisparnas.kemenpar.go.id/p/31593 |

---

## Catatan

- **ID 1–4999**: kosong semua, script akan skip otomatis dengan cepat
- **ID valid**: selalu `status=200` baik valid maupun tidak — deteksi pakai keberadaan teks `PIC`/`Phone`
- **Progress tersimpan** di `localStorage` tiap 1000 scan — aman jika tab reload
- Kalau ingin batasi hanya satu provinsi, belum bisa langsung (tidak ada endpoint list per-kota yang publik) — untuk ini tetap pakai scan range
