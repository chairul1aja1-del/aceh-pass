# ACEH PASS

Sistem tiket QR untuk acara: pengunjung mendapat tiket digital otomatis, petugas memindai QR di gerbang,
panitia memantau dashboard dan membaca kritik, saran, kesan, serta pesan pengunjung.

Teknologi: **Node.js** (server.js, hanya dependensi `pg`) + **PostgreSQL** (db/schema.sql) + halaman web tunggal (public/index.html).

## Struktur

```
server.js          server HTTP + API (POST /api) + penyaji halaman web
package.json
db/schema.sql      tabel dan fungsi PostgreSQL (semua logika data ada di sini)
public/index.html  aplikasi web (pengunjung, petugas, panitia)
test/api.test.js   tes API dengan database tiruan (npm test)
.env.example       contoh pengaturan
```

## Menjalankan di komputer sendiri

1. Pasang Node.js 18+ dan PostgreSQL, lalu buat database kosong, misalnya `aceh_pass`.
2. Salin `.env.example` menjadi `.env`, lalu isi `DATABASE_URL`, `ADMIN_PASSWORD`, dan `PETUGAS_PASSWORD`.
3. Jalankan (Node.js 20.6 ke atas membaca `.env` otomatis lewat skrip `dev`):
   ```bash
   npm install
   npm run dev
   ```
   Di Node 18, atur variabel environment secara manual lalu jalankan `npm start`.
4. Buka http://localhost:3000. Tabel dibuat otomatis saat server pertama kali jalan.

> Kamera pemindai hanya bisa dipakai di `https://` atau `localhost`. Untuk uji di HP, gunakan hosting HTTPS (lihat bawah).

## Mengunggah ke GitHub

GitHub menyimpan kode, tetapi **tidak menjalankan** server Node. Alurnya: unggah ke GitHub, lalu sambungkan repositori ke layanan hosting.

1. Buat repositori baru di github.com, lalu unggah seluruh isi folder ini (tombol *Add file > Upload files*, termasuk folder `db`, `public`, `test`).
   Jangan unggah `.env` atau `node_modules` (sudah ada di `.gitignore`).
2. Sambungkan ke hosting Node, contoh **Render** (render.com) atau **Railway** (railway.app):
   - Buat **PostgreSQL** baru di layanan itu, lalu salin *connection string*-nya.
   - Buat **Web Service** dari repositori GitHub tadi. Build command: `npm install`, start command: `npm start`.
   - Isi Environment Variables:

     | Nama | Isi |
     |---|---|
     | `DATABASE_URL` | connection string PostgreSQL |
     | `DATABASE_SSL` | `true` (umumnya diperlukan di hosting cloud) |
     | `ADMIN_PASSWORD` | sandi panitia, minimal 8 karakter |
     | `PETUGAS_PASSWORD` | sandi petugas, minimal 8 karakter |

   - Deploy. Alamat yang diberikan hosting (https://...) adalah alamat web ACEH PASS.
3. Cek `https://alamat-anda/healthz`: harus menampilkan `{"ok":true,"db":true}`.

## Alamat penting

| Fungsi | Alamat |
|---|---|
| Tiket pengunjung | `/` |
| QR akses web (tanpa login, bisa diunduh/dicetak) | `/#qr` |
| Login staf | `/#operator` |
| Cek kesehatan server | `/healthz` |

**Login:** petugas memakai nama pengguna `petugas` + `PETUGAS_PASSWORD`. Panitia memakai nama bebas + `ADMIN_PASSWORD`.
Mengubah sandi: ubah environment variable lalu restart server.

## Pengaturan lain (opsional)

| Variabel | Fungsi |
|---|---|
| `PORT` | port server (hosting biasanya mengisi sendiri) |
| `ALLOWED_ORIGINS` | alamat web lain yang boleh memanggil API, dipisah koma. Perlu bila halaman di-hosting di tempat lain (mis. GitHub Pages) |
| `TRUST_PROXY` | `1` (bawaan) membaca IP asli dari header proxy untuk pembatas laju |
| `AUTO_MIGRATE` | `0` untuk mematikan pembuatan tabel otomatis |

Masa berlaku tiket (30 menit) dan lama sesi login (6 jam) ada di tabel `app_settings`:

```sql
update app_settings set value = '45' where key = 'ticket_ttl_min';
```

(Ubah juga konstanta masa berlaku di `public/index.html` agar hitung mundur di layar pengunjung cocok.)

## Menjalankan halaman di hosting statis terpisah

Bila `public/index.html` ingin ditaruh di GitHub Pages/Netlify dan server Node di tempat lain:
1. Di `index.html` isi `const SERVER_URL_MANUAL = 'https://alamat-server-anda/api';`
2. Di server, isi `ALLOWED_ORIGINS` dengan alamat halaman statis (mis. `https://nama.github.io`).

## Tes

```bash
npm test
```

Tes memakai database tiruan, jadi tidak memerlukan PostgreSQL. Skema SQL (`db/schema.sql`) belum diuji otomatis: jalankan `npm run migrate` dan coba alur tiket pada database percobaan sebelum dipakai di acara.

## Keamanan

- Semua query berparameter; tabel dikunci (RLS) dan hanya bisa diakses lewat fungsi di `db/schema.sql`.
- Sandi disimpan sebagai hash bcrypt; token sesi disimpan sebagai hash SHA-256.
- Kunci tanda tangan QR (`QR_SECRET`) berada di kode halaman web, jadi bukan rahasia sejati. Server memeriksa format dan waktu tiket, serta menolak pemakaian ganda secara atomik. Untuk acara besar, pertimbangkan tanda tangan yang diperiksa di server.
