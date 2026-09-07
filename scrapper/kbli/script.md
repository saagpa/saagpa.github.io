# Scraper KBLI — Klasifikasi Baku Lapangan Usaha Indonesia

**Sumber:** [kbli.co.id](https://kbli.co.id)

**Strategi:** Site pakai Next.js SSR — data tertanam di HTML (bukan client-side). Bisa `fetch()` langsung dari browser console tanpa Playwright/Puppeteer.

- List kode: embed JSON di `/id/classifications`
- Detail: embed JSON RSC payload di `/id/{kode}`
- Total: **1.559 kode KBLI 5-digit** (versi 2025)

---

## Cara Pakai

1. Buka `https://kbli.co.id/id/classifications` di browser
2. Buka Console (`F12`)
3. Paste **Step 1** untuk extract semua kode dari halaman list
4. Paste **Step 2** untuk fetch detail tiap kode → download CSV

> Tidak perlu login. Tidak perlu navigasi antar halaman.

---

## Step 1 — Extract Semua Kode dari Halaman Classifications

Fetch halaman list, parse RSC payload, simpan 1.559 kode ke `localStorage`.

```javascript
(async function extractKbliCodes() {
  'use strict';

  console.log('📋 Mengambil daftar kode KBLI...');

  const res = await fetch('https://kbli.co.id/id/classifications', {
    headers: {
      'Accept': 'text/html',
      'Accept-Language': 'id-ID,id;q=0.9',
    }
  });
  const html = await res.text();

  // Data ada di RSC payload — cari blok yang berisi categories
  const marker = '"categories":[{"code":"A"';
  const escaped = '\\"categories\\":[{\\"code\\":\\"A\\"';

  let chunk = '';
  let idx = html.indexOf(escaped);
  if (idx !== -1) {
    // Escaped form (dalam JS string)
    const raw = html.slice(idx).replace(/\\"/g, '"').replace(/\\\\/g, '\\');
    chunk = raw.slice(raw.indexOf(marker) + '"categories":'.length);
  } else {
    idx = html.indexOf(marker);
    if (idx !== -1) {
      chunk = html.slice(idx + '"categories":'.length);
    }
  }

  if (!chunk) {
    console.error('❌ Tidak menemukan data categories di halaman.');
    return;
  }

  // Extract semua kode 5-digit (leaf node = KBLI detail)
  const codes5 = [...chunk.matchAll(/"code":"(\d{5})"/g)].map(m => m[1]);
  const unique5 = [...new Set(codes5)];

  // Extract juga hierarchy untuk referensi cepat
  // Format: [{ code, nameEn, nameId, maxForeignOwnership }]
  const allClasses = [];
  const classRx = /"code":"(\d{5})","nameEn":"([^"]+)","nameId":"([^"]+)","maxForeignOwnership":(\d+)/g;
  let match;
  while ((match = classRx.exec(chunk)) !== null) {
    allClasses.push({
      code: match[1],
      nameEn: match[2],
      nameId: match[3],
      maxForeignOwnership: parseInt(match[4]),
    });
  }

  localStorage.setItem('kbli_codes', JSON.stringify(unique5));
  localStorage.setItem('kbli_classes_meta', JSON.stringify(allClasses));

  console.log(`✅ ${unique5.length} kode KBLI tersimpan`);
  console.log(`   Contoh: ${unique5.slice(0, 5).join(', ')} ... ${unique5.slice(-3).join(', ')}`);
  console.log('   Lanjutkan dengan Step 2.');
})();
```

---

## Step 2 — Fetch Detail & Download CSV

Fetch tiap halaman detail, ekstrak deskripsi + hierarki + info PMA dari RSC payload.

```javascript
(async function fetchKbliDetails() {
  'use strict';

  // ── CONFIG ────────────────────────────────────────────
  const CONCURRENCY   = 4;     // request paralel
  const DELAY_BATCH   = 1500;  // ms jeda antar batch
  const DELAY_JITTER  = 800;   // ms jitter tambahan
  const START_INDEX   = 0;     // ubah jika lanjut setelah restart
  const LANG          = 'id';  // 'id' = Bahasa Indonesia, 'en' = English
  // ── END CONFIG ───────────────────────────────────────

  const sleep = ms => new Promise(r => setTimeout(r, ms));
  const rand  = (a, b) => a + Math.floor(Math.random() * (b - a));

  // Load daftar kode
  const codes = JSON.parse(localStorage.getItem('kbli_codes') || '[]');
  if (!codes.length) {
    console.error('❌ Kode KBLI kosong. Jalankan Step 1 dulu.');
    return;
  }

  // Load meta (nameEn/nameId/maxForeignOwnership dari list page)
  const metaArr = JSON.parse(localStorage.getItem('kbli_classes_meta') || '[]');
  const metaMap = Object.fromEntries(metaArr.map(m => [m.code, m]));

  // Load progress sebelumnya
  window._kbliResults = window._kbliResults
    || JSON.parse(localStorage.getItem('kbli_results') || '[]');
  const done = new Set(window._kbliResults.map(r => r.code));
  const todo = codes.slice(START_INDEX).filter(c => !done.has(c));
  console.log(`📦 ${done.size} selesai, ${todo.length} tersisa dari ${codes.length} total`);

  // ── Parse RSC payload ─────────────────────────────────
  function parseDetailHtml(html, code) {
    // Data embed di RSC payload dalam bentuk escaped JSON string
    // Cari "detail":{"code":"XXXXX",...}
    const needle = `\\"detail\\":{\\"code\\":\\"${code}\\"`;
    let idx = html.indexOf(needle);

    let rawDetail = null;

    if (idx !== -1) {
      // Escaped — extract chunk lalu unescape
      const chunk = html.slice(idx, idx + 8000);
      const unesc = chunk.replace(/\\"/g, '"').replace(/\\\\/g, '\\').replace(/\\n/g, ' ');
      const detailIdx = unesc.indexOf('"detail":');
      if (detailIdx !== -1) {
        rawDetail = unesc.slice(detailIdx + '"detail":'.length);
      }
    } else {
      // Mungkin tidak di-escape (edge case)
      const needle2 = `"detail":{"code":"${code}"`;
      idx = html.indexOf(needle2);
      if (idx !== -1) {
        rawDetail = html.slice(idx + '"detail":'.length, idx + 8000);
      }
    }

    if (!rawDetail) return null;

    // Ekstrak field satu per satu pakai regex (aman untuk nested JSON parsial)
    const field = (key) => {
      const rx = new RegExp(`"${key}":"((?:[^"\\\\]|\\\\.)*)"`);
      const m = rawDetail.match(rx);
      return m ? m[1].replace(/\\n/g, ' ').replace(/\\"/g, '"').trim() : '';
    };
    const boolField = (key) => {
      const rx = new RegExp(`"${key}":(true|false)`);
      const m = rawDetail.match(rx);
      return m ? m[1] === 'true' : null;
    };
    const numField = (key) => {
      const rx = new RegExp(`"${key}":(\\d+)`);
      const m = rawDetail.match(rx);
      return m ? parseInt(m[1]) : null;
    };

    // Hierarchy
    const hierarchyChunk = rawDetail.slice(
      rawDetail.indexOf('"hierarchy":'),
      rawDetail.indexOf('"isDeprecated"')
    );
    const hierField = (level, subkey) => {
      const rx = new RegExp(`"${level}":{[^}]*"${subkey}":"([^"]+)"`);
      const m = hierarchyChunk.match(rx);
      return m ? m[1] : '';
    };

    return {
      code:                   field('code') || code,
      version:                field('version'),
      ossId:                  field('ossId'),
      nameEn:                 field('nameEn'),
      nameId:                 field('nameId'),
      descriptionEn:          field('descriptionEn'),
      descriptionId:          field('descriptionId'),
      isDeprecated:           boolField('isDeprecated'),
      existsInBothVersions:   boolField('existsInBothVersions'),
      predecessorCode:        field('predecessorCode'),
      successorCode:          field('successorCode'),
      maxForeignOwnership:    metaMap[code]?.maxForeignOwnership ?? numField('maxForeignOwnership'),
      // Hierarchy
      categoryCode:           hierField('category', 'code'),
      categoryNameEn:         hierField('category', 'nameEn'),
      categoryNameId:         hierField('category', 'nameId'),
      majorGroupCode:         hierField('majorGroup', 'code'),
      majorGroupNameEn:       hierField('majorGroup', 'nameEn'),
      majorGroupNameId:       hierField('majorGroup', 'nameId'),
      groupCode:              hierField('group', 'code'),
      groupNameEn:            hierField('group', 'nameEn'),
      groupNameId:            hierField('group', 'nameId'),
      subgroupCode:           hierField('subgroup', 'code'),
      subgroupNameEn:         hierField('subgroup', 'nameEn'),
      subgroupNameId:         hierField('subgroup', 'nameId'),
    };
  }

  // ── Fetch satu kode ───────────────────────────────────
  async function fetchOne(code, retries = 2) {
    const url = `https://kbli.co.id/${LANG}/${code}`;
    for (let attempt = 0; attempt <= retries; attempt++) {
      try {
        const res = await fetch(url, {
          headers: {
            'Accept': 'text/html,application/xhtml+xml',
            'Accept-Language': 'id-ID,id;q=0.9,en;q=0.8',
            'Referer': 'https://kbli.co.id/id/classifications',
          }
        });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const html = await res.text();
        const data = parseDetailHtml(html, code);
        if (!data) throw new Error('parse failed');
        return data;
      } catch (e) {
        if (attempt < retries) {
          await sleep(rand(2000, 4000));
        } else {
          console.warn(`⚠️  Gagal: ${code} — ${e.message}`);
          return { code, error: e.message };
        }
      }
    }
  }

  // ── Batch paralel ────────────────────────────────────
  async function processBatch(batch) {
    const promises = batch.map((code, i) =>
      sleep(i * rand(200, 500)).then(() => fetchOne(code))
    );
    return Promise.all(promises);
  }

  // ── MAIN LOOP ─────────────────────────────────────────
  const total = todo.length;
  let done2 = 0;

  for (let i = 0; i < total; i += CONCURRENCY) {
    const batch = todo.slice(i, i + CONCURRENCY);
    const results = await processBatch(batch);

    results.forEach(r => { if (r) window._kbliResults.push(r); });
    done2 += batch.length;

    const pct = ((done2 / total) * 100).toFixed(1);
    const sample = results.find(r => r && !r.error);
    console.log(`📦 ${done2}/${total} (${pct}%) — ${sample?.code}: ${sample?.nameId || sample?.nameEn || '?'}`);

    // Simpan progress setiap 100 batch
    if (Math.floor(i / CONCURRENCY) % 100 === 0) {
      localStorage.setItem('kbli_results', JSON.stringify(window._kbliResults));
      console.log(`💾 Progress: ${window._kbliResults.length} records tersimpan`);
    }

    if (i + CONCURRENCY < total) {
      await sleep(DELAY_BATCH + rand(0, DELAY_JITTER));
    }
  }

  // Final save
  localStorage.setItem('kbli_results', JSON.stringify(window._kbliResults));
  console.log(`\n✅ SELESAI — ${window._kbliResults.length} kode KBLI`);
  console.table(window._kbliResults.slice(0, 3));

  downloadCSV(window._kbliResults);

  // ── Download CSV ─────────────────────────────────────
  function downloadCSV(data) {
    if (!data.length) return;
    const cols = [
      'no', 'code', 'version', 'ossId',
      'nameId', 'nameEn',
      'descriptionId', 'descriptionEn',
      'maxForeignOwnership',
      'isDeprecated', 'existsInBothVersions',
      'predecessorCode', 'successorCode',
      'categoryCode', 'categoryNameId', 'categoryNameEn',
      'majorGroupCode', 'majorGroupNameId', 'majorGroupNameEn',
      'groupCode', 'groupNameId', 'groupNameEn',
      'subgroupCode', 'subgroupNameId', 'subgroupNameEn',
      'error'
    ];
    const esc = v => `"${String(v ?? '').replace(/"/g, '""')}"`;
    const rows = [
      cols.join(','),
      ...data.map((r, i) => cols.map(c => c === 'no' ? i + 1 : esc(r[c])).join(','))
    ];
    const blob = new Blob(['﻿' + rows.join('\r\n')], { type: 'text/csv;charset=utf-8' });
    const a = Object.assign(document.createElement('a'), {
      href: URL.createObjectURL(blob),
      download: `kbli_${new Date().toISOString().slice(0, 10)}.csv`,
    });
    document.body.appendChild(a); a.click(); document.body.removeChild(a);
    console.log('💾 CSV didownload!');
  }
})();
```

---

## Download Hasil yang Sudah Ada

Jika browser ditutup tapi data masih di `localStorage`:

```javascript
(function resumeDownload() {
  const data = JSON.parse(localStorage.getItem('kbli_results') || '[]');
  if (!data.length) { console.warn('Belum ada hasil.'); return; }
  console.log(`${data.length} records di localStorage`);

  const cols = [
    'no','code','version','ossId',
    'nameId','nameEn','descriptionId','descriptionEn',
    'maxForeignOwnership','isDeprecated','existsInBothVersions',
    'predecessorCode','successorCode',
    'categoryCode','categoryNameId','categoryNameEn',
    'majorGroupCode','majorGroupNameId','majorGroupNameEn',
    'groupCode','groupNameId','groupNameEn',
    'subgroupCode','subgroupNameId','subgroupNameEn',
    'error'
  ];
  const esc = v => `"${String(v ?? '').replace(/"/g, '""')}"`;
  const rows = [cols.join(','), ...data.map((r,i) => cols.map(c => c==='no'?i+1:esc(r[c])).join(','))];
  const blob = new Blob(['﻿' + rows.join('\r\n')], { type: 'text/csv;charset=utf-8' });
  const a = Object.assign(document.createElement('a'), {
    href: URL.createObjectURL(blob),
    download: `kbli_${new Date().toISOString().slice(0,10)}.csv`,
  });
  document.body.appendChild(a); a.click(); document.body.removeChild(a);
  console.log('💾 Didownload!');
})();
```

---

## Resume Setelah Browser Restart

```javascript
window._kbliResults = JSON.parse(localStorage.getItem('kbli_results') || '[]');
console.log('Resume:', window._kbliResults.length, 'records dimuat');
```

Lalu jalankan Step 2 dengan `START_INDEX` disesuaikan (atau biarkan 0 — script otomatis skip kode yang sudah selesai).

---

## Estimasi Waktu

| Scope | Kode | Estimasi |
|---|---|---|
| Test | 20 | ~1 menit |
| Sebagian (A-C) | ~200 | ~12 menit |
| Semua | 1.559 | ~60–90 menit |

> Jeda default: ~1.5–2.3 detik per batch × 4 paralel = ~375 ms per kode

---

## Output CSV — Kolom

| Kolom | Contoh |
|---|---|
| code | 01112 |
| version | 2025 |
| ossId | 24e9c9d8-dd45-5405-a820-7ef002c76a84 |
| nameId | Pertanian Serealia Selain Padi dan Jagung |
| nameEn | Other Cereal Farming Except Rice and Corn |
| descriptionId | Kelompok ini mencakup kegiatan pertanian serealia... |
| descriptionEn | Cereal farming activities other than rice and corn... |
| maxForeignOwnership | 100 |
| isDeprecated | false |
| existsInBothVersions | true |
| predecessorCode | (kosong jika tidak ada) |
| successorCode | (kosong jika tidak ada) |
| categoryCode | A |
| categoryNameId | PERTANIAN, KEHUTANAN, DAN PERIKANAN |
| majorGroupCode | 01 |
| groupCode | 011 |
| subgroupCode | 0111 |

---

## Tips Anti-Block

| Teknik | Nilai Default |
|---|---|
| Paralel request | `CONCURRENCY = 4` — turunkan ke 2 jika rate limited |
| Jeda antar batch | `DELAY_BATCH = 1500` ms |
| Jitter | `DELAY_JITTER = 800` ms |
| Retry per kode | 2x retry + backoff 2–4 detik |
| Save progress | Tiap 100 batch ke `localStorage` |
