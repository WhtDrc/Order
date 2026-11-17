# Q-Order (template toko)

Pesan lewat QR meja dan pre-order (PO), dengan aplikasi kasir/dapur/admin terpisah.
Next.js 16 (App Router) + Tailwind v4 + Supabase. Satu repo = template. **Tiap toko = satu project Supabase + satu deployment sendiri.**

## Dua aplikasi dalam satu kode

| | Pemesan (mobile app, tanpa login) | Staf (login username + password) |
|---|---|---|
| Alamat | `/t/[token]` (QR meja), `/po`, `/order/[token]` | `/login`, `/kasir`, `/admin/*` |
| Tampilan | Terang, daftar menu, keranjang melayang | Header hijau, papan pesanan, menu admin |
| Fitur | Menu, varian/topping, checkout, bayar tunai/QRIS, status realtime (polling), batal sebelum bayar | Tandai lunas, dapur, laporan, menu, meja + QR, PO, staf, pengaturan |

Role: **owner** (semua), **kasir** (terima bayar, batalkan, dapur), **dapur** (hanya pesanan lunas: siap dan selesai).

## Pasang

```bash
npx create-next-app@latest q-order --ts --tailwind --eslint --app --src-dir --import-alias "@/*"
cd q-order
npm i @supabase/supabase-js @supabase/ssr lucide-react zod qrcode server-only
npm i -D tsx @types/qrcode
```

Salin isi folder ini ke atas project (timpa `src/app/globals.css`, `layout.tsx`, `page.tsx`, dan `next.config.ts`), lalu:

1. `cp .env.example .env.local`, isi nilainya.
2. `supabase link` lalu `supabase db push` (menjalankan `supabase/migrations/0001_schema.sql`). Data contoh opsional: jalankan `supabase/seed.sql` di SQL Editor.
3. Supabase Dashboard, Authentication: matikan **Allow new users to sign up**.
4. Buat owner: `npx tsx --env-file=.env.local scripts/create-owner.ts budi rahasia123 "Budi Santoso"`
5. `npm run dev`, buka `/login`. Di Pengaturan unggah logo dan QRIS, di Meja dan QR cetak QR meja.

## Alur pesanan

`awaiting_payment` (pelanggan pesan) → kasir **Tandai lunas** → `preparing` (dapur) → `ready` → `completed`. Bisa `cancelled` kapan pun sebelum selesai (kasir/owner).

- **Bayar di awal**: pesanan belum lunas tidak muncul di dapur. Tunai: pelanggan bayar ke kasir. QRIS: pelanggan scan QRIS toko, tekan "Saya sudah bayar", kasir cek mutasi lalu tandai lunas.
- **PO**: diatur di Admin, Pre-order (buka, tutup, tanggal ambil, kuota). Jam diperiksa di **server**, termasuk saat kirim pesanan, jadi pesanan setelah tutup ditolak walau halaman masih terbuka. PO tunai berarti bayar saat ambil: kasir menandai lunas saat pelanggan datang.
- **Struk WhatsApp**: terkirim saat lunas dan saat siap lewat `WA_PROVIDER` (`none`, `fonnte`, atau `webhook` untuk menumpang sistem WA yang sudah ada: server mengirim POST JSON `{to, message}`). Hasil tiap pengiriman dicatat di tabel `notification_log`. Di kartu pesanan ada tombol **Struk** untuk kirim manual lewat wa.me.

## Keamanan (ringkas)

- Harga **selalu dihitung ulang di server** dari database (`lib/pricing.ts`, `lib/create-order.ts`). Browser tidak bisa mengirim harga.
- Pesanan dibuat hanya lewat server (service role). Tidak ada policy insert untuk pelanggan.
- RLS di semua tabel. Trigger `guard_order_update` menjaga transisi status dan kolom yang boleh diubah per role, walau API Supabase dipanggil langsung.
- Anti-iseng: maksimal 3 pesanan belum dibayar per meja dan per nomor WA dalam 30 menit.
- `SUPABASE_SERVICE_ROLE_KEY` hanya di server (`lib/supabase/admin.ts` memakai `server-only`).

## Onboarding klien baru

1. Buat project Supabase baru, `supabase db push`, matikan signup.
2. Deploy repo dengan env klien tersebut (Vercel atau VPS), pasang domain.
3. `create-owner`, lalu klien login dan mengisi Pengaturan, Menu, Meja.
4. Perbaikan kode: push ke repo utama, redeploy ke semua klien. Perubahan skema: tambah file migrasi baru dan `db push` per klien.

## Catatan

- Kode ini ditulis tanpa bisa dijalankan di lingkungan penyusunnya (tanpa akses internet untuk `npm install`), jadi belum dikompilasi atau diuji. Perkirakan perbaikan kecil di jalan pertama `npm run dev` / `npm run build`.
- Tipe database ditulis manual di `lib/types.ts`. Untuk tipe otomatis: `supabase gen types typescript`.
- Warna utama toko diatur di Pengaturan (mode terang). Mode gelap memakai warna champagne.
- `proxy.ts` hanya memeriksa sudah login. Pengecekan role ada di `requireStaff` dan RLS.
