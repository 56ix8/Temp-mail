# SVARA NETWORK: Ephemeral Mail Infrastructure (Rx-Only)

Svara Network adalah sistem manajemen email temporer berbasis arsitektur serverless. Proyek ini memanfaatkan ekosistem Cloudflare untuk menangani lalu lintas data masuk secara real-time dengan persistensi data yang dikontrol secara otonom. Edisi ini difokuskan pada modul *Receive-Only* (Rx).

## Persyaratan Sistem
- Domain aktif dengan akses penuh ke DNS Management.
- Akun Cloudflare (Free Tier memadai).

---

## PHASE 1: Data Persistence & Object Storage

Tahap awal melibatkan penyediaan infrastruktur penyimpanan untuk metadata email masuk dan aset lampiran. Kita akan menggunakan Cloudflare D1 sebagai database relasional dan R2 untuk penyimpanan objek biner.

### 1.1 Inisialisasi Database (Cloudflare D1)
Database ini berfungsi untuk menyimpan log transmisi masuk.

1. Buka Dashboard Cloudflare > Storage & Databases > D1.
2. Buat database baru dengan nama `svara-db`.
3. Masuk ke tab Console dan eksekusi skrip SQL berikut untuk membuat skema tabel:

```sql
CREATE TABLE IF NOT EXISTS emails (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    recipient TEXT,
    sender TEXT,
    subject TEXT,
    body_text TEXT,
    body_html TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### 1.2 Konfigurasi Vault (Cloudflare R2)
R2 digunakan untuk menampung file biner (attachment) yang diekstrak dari payload email.

1. Buka Dashboard Cloudflare > Storage & Databases > R2.
2. Buat bucket baru dengan nama `svara-vault`.
3. Buka tab Settings pada bucket tersebut, cari bagian Object Lifecycle Rules.
4. Tambahkan aturan baru: Set agar objek otomatis dihapus (Delete objects) setelah usia 1 hari. Langkah ini krusial untuk menjaga efisiensi ruang penyimpanan dan privasi data.

---

## PHASE 2: Data Ingestion & Worker Initialization

Tahap ini berfokus pada penangkapan arus data masuk (email mentah) dan meneruskannya ke komputasi edge (Cloudflare Worker) sebelum diekstrak ke database.

### 2.1 Konfigurasi Email Routing (Catch-all)
Fitur ini digunakan untuk menangkap seluruh variasi alamat email di bawah domain utama dan mem-bypass proses ke Worker.

1. Buka Dashboard Cloudflare > Pilih domain yang akan digunakan.
2. Navigasi ke menu Email > Email Routing.
3. Klik Get Started dan ikuti proses injeksi DNS Records (TXT dan MX) secara otomatis.
4. Setelah status domain aktif, masuk ke tab Routing Rules.
5. Aktifkan fitur Catch-all address.
6. Pada kolom Action, atur ke Send to a Worker (Pilih Worker yang akan dibuat pada langkah 2.2).

### 2.2 Deployment Serverless Worker & Bindings
Worker berfungsi sebagai otak parser utama untuk membedah payload email mentah menjadi objek terstruktur.

1. Buka Dashboard Cloudflare > Workers & Pages.
2. Klik Create application > Pilih tab Workers > Klik Create Worker.
3. Beri nama `svara-worker`, lalu klik Deploy.
4. Buka tab Settings > Bindings (atau Variables).
5. Tambahkan koneksi D1 Database:
   - Variable name: `DB` (wajib uppercase)
   - D1 database: Pilih `svara-db`
6. Tambahkan koneksi R2 Bucket:
   - Variable name: `BUCKET` (wajib uppercase)
   - R2 bucket: Pilih `svara-vault`

---

## PHASE 3: Core Logic Engine

Tahap ini menginjeksi logika utama ke dalam Cloudflare Worker untuk membaca dan memproses email masuk.

### 3.1 Injeksi Kode Utama (Worker Script)

1. Buka halaman Worker `svara-worker` > klik Edit Code.
2. Timpa seluruh isi dengan kode JavaScript berikut, lalu klik Deploy:

```javascript
// --- SVARA NETWORK CORE ENGINE (RX-ONLY) ---

function extractPart(raw, type) {
    let idx = raw.indexOf(`Content-Type: ${type}`);
    if (idx === -1) return null;
    let sliced = raw.substring(idx);
    let headerEnd = sliced.indexOf("\r\n\r\n");
    if (headerEnd === -1) headerEnd = sliced.indexOf("\n\n");
    if (headerEnd === -1) return null;
    let body = sliced.substring(headerEnd).trim();
    let nextBound = body.indexOf("\r\n--");
    if (nextBound === -1) nextBound = body.indexOf("\n--");
    if (nextBound !== -1) body = body.substring(0, nextBound);
    
    body = body.replace(/=\r\n/g, "").replace(/=\n/g, "");
    body = body.replace(/=([0-9A-F]{2})/gi, (match, hex) => {
        try { return decodeURIComponent('%' + hex); } catch(e) { 
            try { return String.fromCharCode(parseInt(hex, 16)); } catch(e) { return match; }
        }
    });
    return body;
}

export default {
  async email(message, env, ctx) {
    const recipient = message.to;
    const sender = message.headers.get("from") || message.from;
    const subject = message.headers.get("subject") || "(Tanpa Subjek)";
    const rawEmail = await new Response(message.raw).text();
    
    let cleanText = "Pesan teks tidak tersedia."; let cleanHtml = ""; let attachmentsHtml = ""; 

    try {
        if (rawEmail.includes("multipart/")) {
            cleanText = extractPart(rawEmail, "text/plain") || cleanText; cleanHtml = extractPart(rawEmail, "text/html") || "";
            const boundaryMatch = rawEmail.match(/boundary="?([^"\r\n]+)"?/i);
            if (boundaryMatch) {
                const parts = rawEmail.split("--" + boundaryMatch[1]);
                for (let part of parts) {
                    if (part.includes("Content-Disposition: attachment") || part.includes("Content-Disposition: inline; filename")) {
                        let fnameMatch = part.match(/filename="?([^"\r\n]+)"?/i); let fname = fnameMatch ? fnameMatch[1] : "file.bin";
                        let headerEnd = part.indexOf("\r\n\r\n"); if (headerEnd === -1) headerEnd = part.indexOf("\n\n");
                        if (headerEnd !== -1) {
                            let b64 = part.substring(headerEnd).replace(/\s+/g, "");
                            try {
                                let binString = atob(b64); let bytes = new Uint8Array(binString.length);
                                for (let i = 0; i < binString.length; i++) bytes[i] = binString.charCodeAt(i);
                                let fileKey = Date.now() + "_" + fname; await env.BUCKET.put(fileKey, bytes.buffer);
                                attachmentsHtml += `<div style="margin-top:20px; padding:15px; border:1px solid #334155; border-radius:12px; background:#0f172a; color:#f8fafc; font-family:monospace;"><p>📎 <b>${fname}</b></p><a href="https://${env.WORKER_HOST}/api/download/${fileKey}?key=${env.ADMIN_KEY}" target="_blank" style="display:inline-block; padding:8px 16px; background:#06b6d4; color:#030712; text-decoration:none; border-radius:6px; font-weight:bold;">⬇ Download File</a></div>`;
                            } catch(err) {}
                        }
                    }
                }
            }
        } else { let headerEndIdx = rawEmail.indexOf("\r\n\r\n"); cleanText = headerEndIdx !== -1 ? rawEmail.substring(headerEndIdx).trim() : rawEmail; }
        cleanHtml += attachmentsHtml;
    } catch (e) { console.error(e); }

    await env.DB.prepare("INSERT INTO emails (recipient, sender, subject, body_text, body_html) VALUES (?, ?, ?, ?, ?)").bind(recipient, sender, subject, cleanText, cleanHtml).run();
    await env.DB.prepare("DELETE FROM emails WHERE created_at <= datetime('now', '-1 day')").run();
  },

  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    if (request.method === "OPTIONS") return new Response(null, { headers: { "Access-Control-Allow-Origin": "*", "Access-Control-Allow-Methods": "GET, OPTIONS", "Access-Control-Allow-Headers": "Content-Type, X-Secret-Key" } });

    const userKey = request.headers.get("X-Secret-Key") || url.searchParams.get("key");
    const validCodes = env.VALID_CLASS_CODES ? env.VALID_CLASS_CODES.split(',') : ["TESTING123"];
    const isMember = validCodes.includes(userKey); 
    const isAdmin = userKey === env.ADMIN_KEY;
    
    if (!isMember && !isAdmin) return new Response(JSON.stringify({ error: "Unauthorized" }), { status: 401, headers: { "Access-Control-Allow-Origin": "*" } });

    if (url.pathname === "/api/admin" && request.method === "GET") {
        if (!isAdmin) return new Response("Akses Ditolak", { status: 403 });
        const { results: inbox } = await env.DB.prepare("SELECT recipient, sender, subject, created_at FROM emails ORDER BY created_at DESC LIMIT 50").all();

        let inboxRows = inbox.map(row => `<tr><td style="padding:10px; border-bottom:1px solid #334155;">${new Date(row.created_at).toLocaleString('id-ID')}</td><td style="padding:10px; border-bottom:1px solid #334155; color:#a78bfa;">${row.sender.replace(/[<>]/g, '')}</td><td style="padding:10px; border-bottom:1px solid #334155; color:#34d399;">${row.recipient}</td><td style="padding:10px; border-bottom:1px solid #334155;">${row.subject}</td></tr>`).join('');

        const adminHtml = `<!DOCTYPE html><html><head><title>Svara Telemetry</title><meta name="viewport" content="width=device-width, initial-scale=1.0"><style>body{background-color:#030712; color:#cbd5e1; font-family:monospace; padding:20px;} h1, h2{color:#38bdf8;} table{width:100%; border-collapse:collapse; margin-bottom:40px; background:#0f172a; border-radius:8px; overflow:hidden;} th{background:#1e293b; padding:12px; text-align:left; color:#f8fafc; font-size:14px;} td{font-size:12px; word-break:break-all;}</style></head><body><h1 style="text-align:center; font-size:2em; text-transform:uppercase; letter-spacing:2px; margin-bottom:5px;">OMNISCIENT TELEMETRY</h1><p style="text-align:center; color:#64748b; margin-bottom:40px;">Real-time Ingress Monitoring</p><h2>⬇️ INGRESS (Incoming)</h2><table><thead><tr><th>Waktu (UTC)</th><th>Pengirim</th><th>Penerima</th><th>Subjek</th></tr></thead><tbody>${inboxRows || '<tr><td colspan="4" style="text-align:center;">N/A</td></tr>'}</tbody></table></body></html>`;
        return new Response(adminHtml, { headers: { "Content-Type": "text/html" } });
    }

    if (url.pathname.startsWith("/api/download/") && request.method === "GET") {
        const fileKey = url.pathname.replace("/api/download/", ""); const object = await env.BUCKET.get(fileKey); 
        if (!object) return new Response("File expired.", { status: 404 });
        const headers = new Headers(); object.writeHttpMetadata(headers); headers.set("etag", object.httpEtag); headers.set("Access-Control-Allow-Origin", "*"); headers.set("Content-Disposition", `attachment; filename="${fileKey.substring(14)}"`);
        return new Response(object.body, { headers });
    }

    if (url.pathname === "/api/emails" && request.method === "GET") {
      const address = url.searchParams.get("address"); if (!address) return new Response("Missing address", { status: 400 });
      const { results } = await env.DB.prepare("SELECT * FROM emails WHERE recipient = ? ORDER BY created_at DESC").bind(address).all();
      return new Response(JSON.stringify(results), { headers: { "Content-Type": "application/json", "Access-Control-Allow-Origin": "*" } });
    }
    
    return new Response("System Online (Rx Engine)", { status: 200 });
  }
};
```

### 3.2 Konfigurasi Variables & Secrets
Untuk keamanan tingkat tinggi, seluruh kredensial dan parameter penting dikelola melalui sistem Secrets.

1. Buka halaman Worker `svara-worker` > Settings > Variables and Secrets.
2. Tambahkan variabel dengan mengklik Add pada bagian Secrets:
   - Variable name: `ADMIN_KEY` | Value: Buat sandi kuat (contoh: `SVARABOS-99`) untuk akses dasbor.
   - Variable name: `VALID_CLASS_CODES` | Value: `KELAS-A1,KELAS-B2` (String dipisahkan koma untuk otorisasi login antarmuka).
   - Variable name: `WORKER_HOST` | Value: Isi dengan URL worker tanpa awalan HTTPS (contoh: `svara-worker.username.workers.dev`).
3. Deploy ulang Worker untuk menerapkan konfigurasi terbaru.

---

## PHASE 4: Antarmuka Pengguna (Frontend UI) & Deployment

Tahap akhir ini berfokus pada penyajian antarmuka statis yang akan di-hosting menggunakan infrastruktur Cloudflare Pages.

1. Siapkan file `index.html` yang berisi antarmuka Svara Network Anda.
2. Modifikasi konstanta Worker URL di dalam skrip `index.html`:
   `const WORKER_URL = 'https://[URL_WORKER_ANDA].workers.dev';`
3. Buka Dashboard Cloudflare > navigasi ke menu Workers & Pages.
4. Klik Create application > pilih tab Pages > pilih Upload assets.
5. Buat nama proyek (contoh: `svara-mail-ui`), lalu unggah berkas `index.html`.
6. Klik Deploy site. Tautan publik Anda telah siap digunakan.

---

## Arsitektur Keamanan & Pemeliharaan
- **Zero-Trust Gatekeeper**: Antarmuka dilindungi oleh `VALID_CLASS_CODES`. Klien tanpa kode tidak dapat menarik data.
- **Auto-Purge 24 Jam**: Worker secara otonom akan menghapus email yang usianya melebihi 24 jam setiap kali ada data baru yang masuk untuk mencegah *overload* database.
- **Omniscient Telemetry**: Administrator dapat memantau log masuk melalui endpoint tersembunyi `https://[URL_WORKER]/api/admin?key=[ADMIN_KEY]`.
