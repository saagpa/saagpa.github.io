🛡️ Golden Rules: WhatsApp Data Crawling & Safety
1. Kematangan & Reputasi Akun
- Umur Akun: Wajib berumur minimal 3 bulan. Akun baru sangat rentan terkena banned otomatis.
- Aktivitas Organik: Akun harus aktif digunakan untuk chat normal. Jangan biarkan akun "kosong" hanya untuk crawling.

2. Pola Kerja & Jeda (Cooling Down)
- Siklus Kerja: Terapkan pola 1 Hari Crawling, 1 Hari Libur. Jangan melakukan pengambilan data dua hari berturut-turut.
- Jeda Sesi: Wajib memberikan jeda minimal 24 jam setelah satu sesi selesai sebelum memulai sesi berikutnya.
- Maksimal Data: Ambil maksimal 200–300 member per sesi. Untuk grup besar (700+), bagi menjadi 3 sesi kerja dalam seminggu.

3. Persiapan Teknis (Wajib)
- Scroll Utuh: Sebelum menjalankan script, list member harus di-scroll secara manual dari atas sampai paling bawah hingga semua foto profil/nama muncul. Jika tidak di-scroll sampai mentok, script akan error karena data belum dimuat (load) oleh WhatsApp.
- Update Parameter: Pastikan mengubah variabel targetHariKe di dalam script sebelum mulai:
Hari ke-1: targetHariKe = 1
Hari ke-2 (setelah libur): targetHariKe = 2
...dan seterusnya agar data tidak duplikat.

4. Simulasi Perilaku Manusia
- Tab Tetap Aktif: Jangan minimize browser saat script berjalan. Pastikan tab WhatsApp Web tetap terbuka (boleh tertutup jendela lain seperti Excel, tapi jangan di-minimize ke taskbar).
- Kecepatan (Delay): Jangan mempercepat delay script. Biarkan script berjalan dengan ritme alami (10-15 detik per profil).
- Jam Operasional: Jalankan script di jam kerja (08.00 - 20.00).

5. Pemeliharaan & Manajemen Data
- Interaksi Normal: Setelah selesai, kirim 1–2 pesan manual ke kontak yang dikenal.
- Refresh Session: Tutup atau refresh WhatsApp Web setelah data diamankan (copy-paste ke Excel).
- Verifikasi: Selalu cek hasil console.log(results) di akhir sesi sebelum menutup browser.

Code Script:
// Deklarasikan di luar agar bisa diakses kapan saja dari Console
var results = "Nama\tNomor\n"; 

async function startProfessionalExtraction() {
    // =================================================================
    // CONFIGURATION
    // =================================================================
    let targetHariKe = 1;     
    let jumlahPerSesi = 250;  
    let startIndexHeader = 5; 
    // =================================================================

    let allRows = document.querySelectorAll('div[role="listitem"]');
    let startIndex = startIndexHeader + ((targetHariKe - 1) * jumlahPerSesi);
    let endIndex = startIndex + jumlahPerSesi;

    if (startIndex >= allRows.length) {
        console.error("Error: Indeks mulai melebihi jumlah member.");
        return;
    }
    if (endIndex > allRows.length) endIndex = allRows.length;

    let totalToProcess = endIndex - startIndex;
    let groupName = document.querySelector('header span[dir="auto"]')?.innerText || "Info Grup";

    console.log("------------------------------------------");
    console.log("STEALTH EXTRACTION ENGINE - ACTIVE");
    console.log("Target Sesi Ini : " + totalToProcess + " Member");
    console.log("------------------------------------------");

    let countSuccess = 0;

    for (let i = startIndex; i < endIndex; i++) {
        let currentRows = document.querySelectorAll('div[role="listitem"]');
        let currentRow = currentRows[i];
        if (!currentRow) continue;

        currentRow.scrollIntoView({ behavior: 'smooth', block: 'center' });
        await new Promise(r => setTimeout(r, 2000 + Math.random() * 1000));

        let clickTarget = currentRow.querySelector('span[title]') || currentRow;
        clickTarget.click();
        
        let dynamicLoad = 4500 + (Math.random() * 2500);
        await new Promise(r => setTimeout(r, dynamicLoad));

        let panel = document.querySelector('section');
        if (panel) {
            let nameInPanel = panel.querySelector('span[data-testid="selectable-text"]')?.innerText || "Unknown";
            
            if (nameInPanel !== groupName && !nameInPanel.includes("Group info")) {
                let phone = "Tidak Ditemukan";
                let spans = panel.querySelectorAll('span');
                
                for (let s of spans) {
                    let text = s.innerText.trim();
                    if (text && (text.includes('+62') || (text.replace(/[\s\-\+]/g, '').length >= 10 && /^\+?[\d\s\-]+$/.test(text)))) {
                        if (!text.includes(',')) {
                            phone = text;
                            break;
                        }
                    }
                }
                // DISINI DATA DISIMPAN KE VARIABEL GLOBAL
                results += nameInPanel + "\t" + phone + "\n";
                countSuccess++;
                console.log("[" + countSuccess + "/" + totalToProcess + "] " + nameInPanel + " -> " + phone);
            }
        }

        let backBtn = document.querySelector('span[data-icon="back-refreshed"]') || 
                      document.querySelector('button[aria-label="Back"]');
        if (backBtn) {
            backBtn.click();
            await new Promise(r => setTimeout(r, 1500 + Math.random() * 1000));
        }

        // Istirahat rutin
        if (countSuccess % 7 === 0 && countSuccess > 0) {
            await new Promise(r => setTimeout(r, 10000));
        }
    }

    console.log("------------------------------------------");
    console.log("PROSES SELESAI: " + countSuccess + " DATA DIAMBIL");
    console.log("Ketik 'console.log(results)' untuk melihat data.");
    console.log("------------------------------------------");
    
    copy(results); // Otomatis copy ke clipboard
}

// Jalankan
startProfessionalExtraction();