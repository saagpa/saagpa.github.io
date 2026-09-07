# Scraper Kemendikdasmen — Data Sekolah

**Target:** `https://sekolah.data.kemendikdasmen.go.id/sekolah`
**Strategi:** Baca DOM hasil render Angular (tidak perlu dekripsi API) — klik pagination list → masuk detail → back → ulangi
**Data diambil:** Nama, Alamat, Email, Telepon

---

## Cara Pakai

1. Buka `https://sekolah.data.kemendikdasmen.go.id/sekolah` di Chrome
2. Ketik keyword di kolom pencarian (misal nama provinsi, atau nama jenjang seperti `SD`, `SMP`, `SMA`) lalu klik **Cari**
3. Tunggu hasil muncul
4. Buka Console (`F12`) — paste **Step 1** untuk cek selector DOM (sekali saja pertama kali)
5. Kalau selector cocok, langsung paste **Script Utama**

---

## Step 0b — Probe Angular (jalankan jika Step 0 return 0 link)

```javascript
(function probe2() {
  var allText = document.body.innerText;
  console.log('Cuplikan teks halaman (500 char pertama):', allText.slice(0, 500));

  var clickables = document.querySelectorAll('[routerlink], [ng-reflect-router-link], [class*="card"], [class*="item"], [class*="list"], [class*="school"], [class*="sekolah"]');
  console.log('Elemen clickable/card:', clickables.length);
  if (clickables.length) {
    console.log('Contoh 0 - tag:', clickables[0].tagName, '| class:', clickables[0].className);
    console.log('Contoh 0 - teks:', clickables[0].innerText.slice(0, 100));
    console.log('Contoh 0 - routerlink:', clickables[0].getAttribute('routerlink') || clickables[0].getAttribute('ng-reflect-router-link'));
  }

  var allLinks = document.querySelectorAll('a');
  console.log('Total <a> di halaman:', allLinks.length);
  [...allLinks].slice(0, 10).forEach(function(a, i) {
    console.log('a[' + i + ']', a.href, '|', a.className, '|', a.innerText.trim().slice(0, 40));
  });
})();
```

---

## Step 0 — Cek Selector (jalankan sekali untuk verifikasi)

```javascript
(function probe() {
  // Cari kartu sekolah di halaman list
  var cards = document.querySelectorAll('a[href*="/sekolah/"]');
  console.log('Link sekolah ditemukan:', cards.length);
  if (cards.length) {
    console.log('Contoh href:', cards[0].href);
    console.log('Contoh teks:', cards[0].innerText.slice(0, 100));
  }

  // Cari tombol next pagination
  var btns = [...document.querySelectorAll('button, a, li')].filter(el =>
    /next|selanjutnya|›|»|chevron.right/i.test(el.getAttribute('aria-label') || el.className || el.innerText)
  );
  console.log('Tombol next kandidat:', btns.length);
  btns.forEach((b, i) => console.log(i, b.tagName, b.className, b.innerText.trim().slice(0,30)));
})();
```

---

## Script Utama — Scrape List + Detail

```javascript
(async function scrapeSekolah() {

  // ══════════════════════════════════════════
  //   KONFIGURASI — sesuaikan jika perlu
  var PAGE_START    = 0;     // halaman mulai (0-based, sesuai param ?page=)
  var PAGE_END      = 9999;  // halaman akhir inklusif (9999 = semua)
  var PAGE_SIZE     = 48;    // kartu per halaman (12 / 24 / 48)
  var RENDER_WAIT   = 3000;  // ms tunggu Angular render setelah navigasi
  var DETAIL_WAIT   = 3000;  // ms tunggu halaman detail
  var BACK_WAIT     = 2500;  // ms tunggu setelah back()
  var DELAY_MIN     = 800;
  var DELAY_MAX     = 1800;
  // ══════════════════════════════════════════

  var sleep = function(ms) { return new Promise(function(r) { setTimeout(r, ms); }); };
  var rand  = function(a, b) { return a + Math.floor(Math.random() * (b - a + 1)); };

  // Load progress — keyed by NPSN
  window._sekolahResults = window._sekolahResults && window._sekolahResults.length
    ? window._sekolahResults
    : JSON.parse(localStorage.getItem('sekolah_dom_results') || '[]');

  var doneNPSN = new Set(window._sekolahResults.map(function(r) { return r.npsn; }).filter(Boolean));
  console.log('Progress sebelumnya:', window._sekolahResults.length, 'sekolah');

  // ── Navigasi ke halaman & ukuran awal yang dikonfigurasi ──
  var initUrl = new URL(location.href);
  initUrl.searchParams.set('page', PAGE_START);
  initUrl.searchParams.set('size', PAGE_SIZE);
  history.pushState(null, '', initUrl.toString());
  window.dispatchEvent(new PopStateEvent('popstate'));
  console.log('Navigasi ke page=' + PAGE_START + ' size=' + PAGE_SIZE + ' ...');
  await sleep(RENDER_WAIT);
  await waitForCards(8000);

  // ── Ambil semua kartu dari halaman list (via tombol Lihat) ──
  function getCards() {
    return [...document.querySelectorAll('button')].filter(function(el) {
      return el.innerText.trim() === 'Lihat';
    }).map(function(btn) {
      var card = btn.parentElement && btn.parentElement.parentElement;
      var text = card ? card.innerText : '';
      var npsn = (text.match(/NPSN\s*[:\-]?\s*([A-Z0-9]+)/) || [])[1] || '';
      var lines = text.split('\n').map(function(l) { return l.trim(); }).filter(function(l) {
        return l.length > 5 && !/^(NPSN|Swasta|Negeri|Sandingkan|Lihat)$/i.test(l);
      });
      return { btn: btn, npsn: npsn, namaKartu: lines[0] || '' };
    });
  }

  // ── Cari tombol Next PrimeNG ──
  function getNextBtn() {
    var btn = document.querySelector('button.p-paginator-next');
    if (btn && !btn.disabled && !btn.classList.contains('p-disabled')) return btn;
    return null;
  }

  // ── Tunggu URL berubah dari currentUrl ──
  function waitForNavigation(currentUrl, timeout) {
    timeout = timeout || 8000;
    return new Promise(function(resolve) {
      var start = Date.now();
      var iv = setInterval(function() {
        if (location.href !== currentUrl || Date.now() - start > timeout) {
          clearInterval(iv);
          resolve(location.href);
        }
      }, 200);
    });
  }

  // ── Tunggu hingga elemen muncul di DOM ──
  function waitFor(selector, timeout) {
    timeout = timeout || 8000;
    return new Promise(function(resolve) {
      var start = Date.now();
      var iv = setInterval(function() {
        var el = document.querySelector(selector);
        if (el || Date.now() - start > timeout) { clearInterval(iv); resolve(el); }
      }, 300);
    });
  }

  // ── Scrape halaman detail sekolah ──
  function scrapeDetail(url) {
    // Hanya cocokkan leaf node (div tanpa children) dengan exact match label
    var getText = function(label) {
      var allEls = document.querySelectorAll('div, span, td, th, dt, p, li, label, strong, b');
      for (var el of allEls) {
        if (el.children.length > 0) continue;
        var text = (el.innerText || '').trim();
        if (text.toLowerCase() === label.toLowerCase()) {
          var next = el.nextElementSibling;
          if (next) return next.innerText.trim();
        }
      }
      return '';
    };

    var nama = '';
    var h1 = document.querySelector('h1');
    if (h1) nama = h1.innerText.trim();
    if (!nama) { var h2 = document.querySelector('h2'); if (h2) nama = h2.innerText.trim(); }

    var bodyText = document.body.innerText || '';
    var npsn    = getText('NPSN')    || (bodyText.match(/\bNPSN\b[^\w]*([A-Z0-9]{5,})/) || [])[1] || '';
    var telepon = getText('Telepon');
    var email   = getText('Email');

    // Alamat: coba getText dulu, fallback cari baris yang mengandung Kec./Kota/Prov.
    var alamat = getText('Alamat');
    if (!alamat) {
      var lines = bodyText.split('\n').map(function(l) { return l.trim(); }).filter(Boolean);
      for (var k = 0; k < lines.length; k++) {
        if (/\b(Kec\.|Kota|Prov\.|Jl\.|JL\.|Desa|Kel\.)\b/i.test(lines[k]) && lines[k].length > 20) {
          alamat = lines[k];
          break;
        }
      }
    }

    if (!email)   email   = (bodyText.match(/[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}/) || [])[0] || '';
    if (!telepon) telepon = (bodyText.match(/(?:\+62|0)[0-9\s\-]{8,14}/) || [])[0] || '';

    return { url: url, npsn: npsn, nama: nama, alamat: alamat, email: email, telepon: telepon };
  }

  // ── Tunggu Lihat buttons muncul kembali di list ──
  function waitForCards(timeout) {
    timeout = timeout || 8000;
    return new Promise(function(resolve) {
      var start = Date.now();
      var iv = setInterval(function() {
        var btns = [...document.querySelectorAll('button')].filter(function(el) { return el.innerText.trim() === 'Lihat'; });
        if (btns.length > 0 || Date.now() - start > timeout) { clearInterval(iv); resolve(btns.length); }
      }, 300);
    });
  }

  // ── MAIN LOOP ────────────────────────────────────────
  // Re-fetch cards setiap iterasi karena DOM Angular di-render ulang setelah history.back()
  var pageNo      = PAGE_START;
  var hasNext     = true;
  var doneOnPage  = new Set();  // key kartu yang sudah diproses di halaman ini

  while (hasNext) {
    if (pageNo > PAGE_END) {
      console.log('Halaman akhir (' + PAGE_END + ') tercapai — selesai!');
      hasNext = false;
      break;
    }
    console.log('Halaman ' + pageNo + ' (0-based) — ambil kartu...');
    await sleep(RENDER_WAIT);
    doneOnPage.clear();

    var pageLoop = true;
    while (pageLoop) {
      var cards = getCards();
      var card = null;
      for (var j = 0; j < cards.length; j++) {
        var key = cards[j].npsn || cards[j].namaKartu;
        if (!doneNPSN.has(cards[j].npsn) && !doneOnPage.has(key)) {
          card = cards[j];
          break;
        }
      }

      if (!card) { pageLoop = false; break; }

      var key = card.npsn || card.namaKartu;
      doneOnPage.add(key);
      console.log('  [' + doneOnPage.size + '/' + cards.length + '] ' + card.namaKartu + ' (' + card.npsn + ')');

      var urlBefore = location.href;
      card.btn.click();

      var newUrl = await waitForNavigation(urlBefore, 5000);
      await sleep(DETAIL_WAIT);
      await waitFor('h1, h2', 6000);

      var data = scrapeDetail(newUrl);
      if (data.nama) {
        window._sekolahResults.push(data);
        doneNPSN.add(data.npsn || card.npsn);
        console.log('    OK ' + data.nama + ' | ' + data.telepon);
      } else {
        console.warn('    SKIP — nama tidak ditemukan di ' + newUrl);
      }

      history.back();
      await sleep(BACK_WAIT);
      await waitForCards(8000);
      await sleep(rand(DELAY_MIN, DELAY_MAX));

      if (window._sekolahResults.length % 20 === 0 && window._sekolahResults.length > 0) {
        localStorage.setItem('sekolah_dom_results', JSON.stringify(window._sekolahResults));
        console.log('Disimpan: ' + window._sekolahResults.length + ' sekolah');
      }
    }

    var nextBtn = getNextBtn();
    if (nextBtn) {
      console.log('Klik halaman berikutnya...');
      nextBtn.click();
      await sleep(RENDER_WAIT);
      pageNo++;
    } else {
      console.log('Tidak ada halaman berikutnya — selesai!');
      hasNext = false;
    }
  }

  // Final save & download
  localStorage.setItem('sekolah_dom_results', JSON.stringify(window._sekolahResults));
  console.log('\n✅ SELESAI — ' + window._sekolahResults.length + ' sekolah');
  console.table(window._sekolahResults.slice(0, 5));
  downloadCSV(window._sekolahResults);

  function downloadCSV(data) {
    if (!data.length) { console.warn('Tidak ada data.'); return; }
    function esc(v) { return '"' + String(v == null ? '' : v).replace(/"/g, '""') + '"'; }
    var rows = ['No,NPSN,Nama,Alamat,Email,Telepon,URL'];
    data.forEach(function(r, i) {
      rows.push([i + 1, esc(r.npsn), esc(r.nama), esc(r.alamat), esc(r.email), esc(r.telepon), esc(r.url)].join(','));
    });
    var blob = new Blob([rows.join('\r\n')], { type: 'text/csv;charset=utf-8' });
    var a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = 'sekolah_' + new Date().toISOString().slice(0, 10) + '.csv';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    console.log('CSV didownload! (' + data.length + ' rows)');
  }
})();
```

---

## Download Ulang CSV

```javascript
(function() {
  var data = JSON.parse(localStorage.getItem('sekolah_dom_results') || '[]');
  if (!data.length) { console.warn('Belum ada hasil.'); return; }
  function esc(v) { return '"' + String(v == null ? '' : v).replace(/"/g, '""') + '"'; }
  var rows = ['No,Nama,Alamat,Email,Telepon,URL'];
  data.forEach(function(r, i) {
    rows.push([i + 1, esc(r.nama), esc(r.alamat), esc(r.email), esc(r.telepon), esc(r.url)].join(','));
  });
  var blob = new Blob([rows.join('\r\n')], { type: 'text/csv;charset=utf-8' });
  var a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'sekolah_' + new Date().toISOString().slice(0, 10) + '.csv';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  console.log('Downloaded', data.length, 'records');
})();
```

---

## Resume (jika browser ditutup)

```javascript
window._sekolahResults = JSON.parse(localStorage.getItem('sekolah_dom_results') || '[]');
console.log('Resume:', window._sekolahResults.length, 'sekolah dimuat');
// Lakukan pencarian ulang di situs, lalu jalankan Script Utama — URL yang sudah ada akan di-skip
```

---

## Tips

- Jalankan **Step 0** dulu untuk pastikan selector ketemu — kalau link count = 0, berarti struktur DOM berbeda, share screenshot ke developer script
- Kalau `history.pushState` tidak memicu render Angular, coba ganti navigasi dengan `window.location.href = url` (lebih lambat tapi lebih pasti)
- Kurangi `CONCURRENCY` atau tambah `DETAIL_WAIT` jika halaman sering kosong saat di-scrape
- Data tersimpan di `localStorage` tiap 20 sekolah — aman jika tab reload
