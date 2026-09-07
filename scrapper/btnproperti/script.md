# Scraper BTN Properti — Developer List

**Strategi:** Scrape halaman list untuk kumpulkan URL detail → fetch tiap halaman detail untuk ambil kontak.

- List page: JS-rendered → baca DOM di browser
- Detail page: server-rendered → bisa di-`fetch()` langsung
- Total: **1403 halaman list × 14 developer = ±19.642 developer**

---

## Cara Pakai

1. Buka `https://www.btnproperti.co.id/developer?pg=START_PAGE` — sesuaikan dengan nilai `START_PAGE` yang akan kamu set (misal batch 2 → buka `?pg=101`)
2. Tunggu halaman selesai load
3. Buka Console (`F12`)
4. Paste **Step 1** untuk kumpulkan semua URL → lalu **Step 2** untuk fetch detail

> **Penting:** Script navigasi antar halaman dengan klik tombol "Next", bukan lompat langsung. Jadi **browser harus sudah ada di halaman `START_PAGE` saat script dijalankan**.

---

## Step 1 — Kumpulkan URL dari Semua Halaman List

Script ini navigasi halaman per halaman, kumpulkan link detail developer, lalu simpan ke `window._devUrls`.

**Jalankan per batch, misalnya 1–100, lalu 101–200, dst. Data otomatis tergabung di localStorage.**

```javascript
(async function collectDevURLs() {
  'use strict';

  // ═══════════════════════════════════════════
  //   UBAH DUA ANGKA INI SESUAI BATCH KAMU
  const START_PAGE = 1;     // ← mulai dari halaman berapa
  const END_PAGE   = 100;   // ← sampai halaman berapa
  // ═══════════════════════════════════════════
  // Contoh batch:
  //   Batch 1 : START=1,    END=100
  //   Batch 2 : START=101,  END=200
  //   Batch 3 : START=201,  END=300
  //   ... dst sampai 1403

  const DELAY_MIN   = 1800;
  const DELAY_MAX   = 4000;
  const RENDER_WAIT = 3200;
  // ─────────────────────────────────────────

  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const rand  = (a, b) => a + Math.floor(Math.random() * (b - a + 1));

  // Load data batch sebelumnya — TIDAK di-reset, terus ditambah
  window._devUrls = window._devUrls?.length
    ? window._devUrls
    : JSON.parse(localStorage.getItem('btn_dev_urls') || '[]');
  console.log(`📂 Data sebelumnya: ${window._devUrls.length} URL dimuat`);

  async function waitForCards(timeout = 8000) {
    const start = Date.now();
    while (Date.now() - start < timeout) {
      const cards = document.querySelectorAll('.card');
      const devCards = [...cards].filter(c => {
        const a = c.querySelector('a[href*="/developer/detail/"]');
        return !!a;
      });
      if (devCards.length > 0) return devCards;
      await sleep(300);
    }
    return [];
  }

  // Klik tombol Next → lebih reliable daripada pushState di SPA
  async function clickNext() {
    const nextBtn = [...document.querySelectorAll('.page-link, a[aria-label], button[aria-label]')]
      .find(el => {
        const label = (el.getAttribute('aria-label') || el.innerText || '').toLowerCase();
        return /next|›|»|selanjutnya/i.test(label);
      });
    if (nextBtn && !nextBtn.closest('.disabled, [disabled]')) {
      nextBtn.click();
      return true;
    }
    return false;
  }

  async function scrapePage(pg, isFirst) {
    if (!isFirst) {
      // Navigasi ke halaman berikutnya lewat klik Next
      const ok = await clickNext();
      if (!ok) console.warn(`⚠️  Tombol Next tidak ditemukan di hal ${pg}`);
    }

    await sleep(RENDER_WAIT);

    const devCards = await waitForCards();
    if (!devCards.length) {
      console.warn(`⚠️  Hal ${pg}: card tidak ditemukan`);
      return [];
    }

    const urls = devCards.map(card => {
      const a = card.querySelector('a[href*="/developer/detail/"]');
      const name = card.querySelector('h1,h2,h3,h4,h5,[class*="name"],[class*="title"]')?.innerText?.trim() || '';
      return { url: a.href, name, page: pg };
    });

    console.log(`✅ Hal ${pg}: ${urls.length} developer`);
    return urls;
  }

  console.log('🚀 Mulai kumpulkan URL...');
  console.log(`   Halaman ${START_PAGE}–${END_PAGE} (perkiraan ${(END_PAGE - START_PAGE + 1) * 14} developer)`);
  console.log(`   ⚠️  Pastikan browser sudah membuka hal ${START_PAGE} sebelum script ini dijalankan!`);

  // Halaman pertama — sudah terbuka di browser (user wajib buka manual)
  const first = await scrapePage(START_PAGE, true);
  window._devUrls.push(...first);

  for (let pg = START_PAGE + 1; pg <= END_PAGE; pg++) {
    // Simulasi scroll (human behavior)
    window.scrollTo({ top: rand(200, 600), behavior: 'smooth' });

    const delay = rand(DELAY_MIN, DELAY_MAX);
    console.log(`⏳ Hal ${pg}/${END_PAGE} — jeda ${(delay/1000).toFixed(1)}s...`);
    await sleep(delay);

    const data = await scrapePage(pg, false);
    window._devUrls.push(...data);

    // Simpan progress setiap 50 halaman
    if (pg % 50 === 0) {
      console.log(`💾 Progress: ${window._devUrls.length} URL terkumpul`);
      localStorage.setItem('btn_dev_urls', JSON.stringify(window._devUrls));
    }
  }

  // Deduplikasi
  const unique = [...new Map(window._devUrls.map(d => [d.url, d])).values()];
  window._devUrls = unique;
  localStorage.setItem('btn_dev_urls', JSON.stringify(unique));

  console.log(`\n✅ SELESAI — ${unique.length} URL unik tersimpan di window._devUrls`);
  console.log('   Lanjutkan dengan Step 2 untuk fetch detail kontak.');
})();
```

> **Resume jika browser tertutup:**
> ```javascript
> window._devUrls = JSON.parse(localStorage.getItem('btn_dev_urls') || '[]');
> console.log('Resume:', window._devUrls.length, 'URL dimuat');
> ```

---

## Step 2 — Fetch Detail & Ambil Kontak

Fetch tiap halaman detail, parse nama / alamat / telepon / email dengan `DOMParser`.

```javascript
(async function fetchDetails() {
  'use strict';

  // ── CONFIG ────────────────────────────────────────────
  const CONCURRENCY  = 3;     // request paralel bersamaan
  const DELAY_BATCH  = 2200;  // ms jeda antar batch
  const DELAY_JITTER = 1200;  // ms jitter tambahan (acak)
  const START_INDEX  = 0;     // lanjutkan dari index tertentu jika restart
  // ── END CONFIG ───────────────────────────────────────

  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const rand  = (a, b) => a + Math.floor(Math.random() * (b - a));

  // Ambil daftar URL dari Step 1
  const devList = window._devUrls || JSON.parse(localStorage.getItem('btn_dev_urls') || '[]');
  if (!devList.length) {
    console.error('❌ Tidak ada URL. Jalankan Step 1 dulu.');
    return;
  }
  console.log(`📋 ${devList.length} developer akan di-fetch...`);

  // Load progress sebelumnya jika ada
  window._devResults = window._devResults || JSON.parse(localStorage.getItem('btn_dev_results') || '[]');
  const done = new Set(window._devResults.map(r => r.url));
  const todo = devList.slice(START_INDEX).filter(d => !done.has(d.url));
  console.log(`   ${done.size} sudah selesai, ${todo.length} tersisa`);

  // ── Parse halaman detail ──────────────────────────────
  function parseDetail(html, meta) {
    const doc = new DOMParser().parseFromString(html, 'text/html');

    // Nama developer
    const name = doc.querySelector('h1')?.textContent?.trim() || meta.name || '';

    // Cari teks berdasarkan icon SVG tetangga
    function getByIcon(iconFile) {
      const img = doc.querySelector(`img[src*="${iconFile}"]`);
      if (!img) return '';
      const parent = img.closest('div, p, span, li') || img.parentElement;
      if (!parent) return '';
      // Coba text node langsung dulu
      const textNodes = [...parent.childNodes]
        .filter(n => n.nodeType === Node.TEXT_NODE)
        .map(n => n.textContent.trim())
        .filter(Boolean);
      if (textNodes.length) return textNodes.join(' ').trim();
      // Fallback: clone parent, hapus img, ambil textContent
      // (menangani kasus teks terbungkus <span> atau elemen lain)
      const clone = parent.cloneNode(true);
      clone.querySelectorAll('img').forEach(i => i.remove());
      return clone.textContent.trim();
    }

    const address = getByIcon('room_contact');
    const phone   = getByIcon('phone_contact');
    const email   = getByIcon('email_contact');
    const website = getByIcon('website_contact') ||
      [...doc.querySelectorAll('a[href^="http"]')]
        .map(a => a.href)
        .find(h => !h.includes('btnproperti') && !h.includes('facebook') && !h.includes('instagram')) || '';

    // Fallback regex jika icon tidak ditemukan
    const bodyText = doc.body?.textContent || '';
    const phoneRx  = phone || (bodyText.match(/(\+62|0)8\d{8,11}/g) || []).join('; ');
    const emailRx  = email || (bodyText.match(/[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}/g) || []).join('; ');

    // Tags / area layanan
    const tags = [...doc.querySelectorAll('[class*="tag"], [class*="badge"], [class*="area"]')]
      .map(el => el.textContent.trim())
      .filter(t => t.length < 40)
      .join('; ');

    return {
      url:     meta.url,
      slug:    meta.url.split('/developer/detail/')[1] || '',
      name,
      address,
      phone:   phoneRx,
      email:   emailRx,
      website,
      tags,
      page:    meta.page,
    };
  }

  // ── Fetch satu URL dengan retry ───────────────────────
  async function fetchOne(meta, retries = 2) {
    for (let attempt = 0; attempt <= retries; attempt++) {
      try {
        const res = await fetch(meta.url, {
          credentials: 'include',   // kirim cookie sesi
          headers: {
            'Accept': 'text/html,application/xhtml+xml',
            'Accept-Language': 'id-ID,id;q=0.9,en;q=0.8',
            'Referer': 'https://www.btnproperti.co.id/developer',
            // User-Agent tidak bisa di-set dari browser (diabaikan)
          }
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const html = await res.text();
        return parseDetail(html, meta);
      } catch (e) {
        if (attempt < retries) {
          await sleep(rand(3000, 6000)); // backoff sebelum retry
        } else {
          console.warn(`⚠️  Gagal: ${meta.url} — ${e.message}`);
          return { url: meta.url, slug: '', name: meta.name, error: e.message };
        }
      }
    }
  }

  // ── Proses batch paralel ──────────────────────────────
  async function processBatch(batch) {
    // Jitter kecil antar request dalam batch
    const promises = batch.map((meta, i) =>
      sleep(i * rand(300, 700)).then(() => fetchOne(meta))
    );
    return Promise.all(promises);
  }

  // ── MAIN LOOP ─────────────────────────────────────────
  const total  = todo.length;
  let   done2  = 0;

  for (let i = 0; i < total; i += CONCURRENCY) {
    const batch = todo.slice(i, i + CONCURRENCY);
    const results = await processBatch(batch);

    results.forEach(r => {
      if (r) window._devResults.push(r);
    });
    done2 += batch.length;

    const pct = ((done2 / total) * 100).toFixed(1);
    console.log(`📦 ${done2}/${total} (${pct}%) — contoh: "${results[0]?.name}" ${results[0]?.phone}`);

    // Simpan progress setiap 50 batch
    if (Math.floor(i / CONCURRENCY) % 50 === 0) {
      localStorage.setItem('btn_dev_results', JSON.stringify(window._devResults));
      console.log(`💾 Progress disimpan (${window._devResults.length} records)`);
    }

    // Jeda antar batch (human behavior)
    if (i + CONCURRENCY < total) {
      const delay = DELAY_BATCH + rand(0, DELAY_JITTER);
      await sleep(delay);
    }
  }

  // Final save
  localStorage.setItem('btn_dev_results', JSON.stringify(window._devResults));
  console.log(`\n✅ SELESAI — ${window._devResults.length} developer`);
  console.table(window._devResults.slice(0, 5));

  downloadCSV(window._devResults);

  // ── Download CSV ────────────────────────────────────
  function downloadCSV(data) {
    if (!data.length) return;
    const cols = ['no','name','address','phone','email','website','tags','slug','url','page'];
    const esc  = v => `"${String(v ?? '').replace(/"/g,'""')}"`;
    const rows = [
      cols.join(','),
      ...data.map((r, i) => cols.map(c => c === 'no' ? i + 1 : esc(r[c])).join(','))
    ];
    const blob = new Blob(['﻿' + rows.join('\r\n')], { type: 'text/csv;charset=utf-8' });
    const a    = Object.assign(document.createElement('a'), {
      href: URL.createObjectURL(blob),
      download: `btn_developer_${new Date().toISOString().slice(0,10)}.csv`
    });
    document.body.appendChild(a); a.click();
    document.body.removeChild(a);
    console.log('💾 CSV didownload!');
  }
})();
```

---

## Step 2b — Download Hasil yang Sudah Ada

Jika browser ditutup tapi data masih ada di `localStorage`:

```javascript
(function resumeDownload() {
  const data = JSON.parse(localStorage.getItem('btn_dev_results') || '[]');
  if (!data.length) { console.warn('Belum ada hasil.'); return; }
  console.log(`${data.length} records tersimpan`);
  console.table(data.slice(0, 5));

  // Download CSV
  const cols = ['no','name','address','phone','email','website','tags','slug','url'];
  const esc  = v => `"${String(v ?? '').replace(/"/g,'""')}"`;
  const rows = [cols.join(','), ...data.map((r,i) => cols.map(c => c==='no'?i+1:esc(r[c])).join(','))];
  const blob = new Blob(['﻿' + rows.join('\r\n')], { type: 'text/csv;charset=utf-8' });
  const a    = Object.assign(document.createElement('a'), {
    href: URL.createObjectURL(blob),
    download: `btn_developer_${new Date().toISOString().slice(0,10)}.csv`
  });
  document.body.appendChild(a); a.click(); document.body.removeChild(a);
  console.log('💾 Didownload!');
})();
```

---

## Estimasi Waktu

| Scope | Halaman List | Developer | Estimasi Waktu |
|---|---|---|---|
| Test | 10 | ~140 | ~15 menit |
| Sebagian | 100 | ~1.400 | ~2–3 jam |
| Semua | 1.403 | ~19.600 | ~1–2 hari |

> Untuk scope besar, jalankan bertahap. Data tersimpan otomatis di `localStorage` tiap 50 batch — bisa dilanjutkan kapan saja.

---

## Tips Anti-Block

| Teknik | Implementasi di Script |
|---|---|
| Jeda acak | `DELAY_MIN`/`DELAY_MAX` — ubah lebih besar jika mulai lambat |
| Batasi paralel | `CONCURRENCY = 3` — kurangi ke `1` jika kena rate limit |
| Kirim cookie sesi | `credentials: 'include'` — otomatis |
| Referer header | `Referer: /developer` — mirip navigasi normal |
| Retry otomatis | 2x retry + backoff saat error |
| Simpan progress | `localStorage` — aman jika halaman reload |

---

## Output CSV

| Kolom | Contoh |
|---|---|
| name | ARRAYAN BEKASI DEVELOPMENT |
| address | Ruko Cikarang Square Jalan Raya Cikarang |
| phone | 085725625650 |
| email | arrayangroup.official@gmail.com |
| website | https://... |
| tags | Cikarang; Bekasi |
| slug | arrayan-bekasi-development-abdp |
| url | https://www.btnproperti.co.id/developer/detail/... |
