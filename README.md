# Stock Ops — PWA Gudang untuk ERPNext v16

Aplikasi **Progressive Web App (mobile-first)** untuk operasi gudang di atas ERPNext/Frappe v16:
**Material Request** (Permintaan Transfer & Pembelian), **Stock Entry** (Barang Masuk/Keluar/Transfer),
**Stok Gudang**, **Pergerakan Stok**, **Stok Menipis + Bulk Request**, **Stock Opname**, **Scan barcode**,
**foto barang**, **geolokasi**, **offline + auto-sync**, dan **push notification** — disajikan langsung
oleh bench di `/stock_ops`. Pola seperti Frappe HR (HRMS).

> Status: aktif dikembangkan. Lihat `PRD.md` untuk spesifikasi lengkap.

---

## ✨ Fitur

- **Dokumen**: MR Transfer, MR Purchase, Stock In/Out/Transfer (draft → submit), foto → attachment, geolokasi.
- **Stok**: saldo per gudang, pergerakan (Stock Ledger) dengan filter arah/gudang/tanggal, detail item (scan/cari → stok lintas gudang + mutasi).
- **Stok menipis** berbasis **reorder level** item per gudang + **bulk purchase request** (centang banyak item → 1 MR).
- **Stock Opname** → Stock Reconciliation (hanya item berselisih).
- **Offline-first**: master data di-cache, transaksi masuk outbox, **auto-sync** saat online; idempoten via `external_localid`.
- **Per-employee warehouse**: tiap karyawan hanya melihat gudangnya (lewat field di Employee, fallback User Permission).
- **PWA**: installable + service worker; **push notification** (Web Push/VAPID) saat dokumen disubmit.
- **Multi-bahasa** ID/EN, **dark mode**, dan **4 varian tampilan** (v2 Modern / Klasik / A Dashboard / B Kompak).

---

## 🧱 Arsitektur & Struktur Repo

```
stockinoutrmi/
├── backend/stock_ops/        # Custom Frappe app (Python): DocType, API, hooks, www page, service worker
├── frontend/                 # PWA (Vue 3 + Vite + Pinia) — source
├── design-v2/                # Mockup HTML desain v2
├── PRD.md                    # Product Requirements
├── ss_*.png, desainv*.png    # Screenshot & referensi desain
└── README.md
```

- **Backend** = app `stock_ops` di dalam `frappe-bench/apps/`. Menyediakan API whitelisted (`stock_ops.api.*`,
  `stock_ops.push.*`), custom field, DocType, dan halaman `www/stock_ops` (SPA shell + `sw.js` + `manifest.json`).
- **Frontend** = SPA Vue yang di-build, hasilnya disalin ke `apps/stock_ops/stock_ops/public/stock_ops/`
  dan disajikan Frappe di route **`/stock_ops`**. Saat dev, Vite proxy ke site lokal.

---

## ✅ Prasyarat

- **frappe-bench** dengan **Frappe v16 + ERPNext v16** dan sebuah site (mis. `erp.localhost`).
- **Node 18+** (frontend; Node 20+ untuk service worker PWA penuh), **Python 3.11+**, MariaDB, Redis.
- Backend ini diuji pada Frappe 16.17 / ERPNext 16.16 (WSL Ubuntu).

---

## 🚀 Instalasi Backend (custom app)

Dari direktori `frappe-bench`:

```bash
# 1. Ambil app ini ke bench
bench get-app stock_ops https://github.com/denzizzy966/stockinoutrmi --branch new-develop
#    (repo ini berisi app di subfolder backend/stock_ops — lihat catatan di bawah)

# 2. Install ke site
bench --site <site> install-app stock_ops

# 3. Jalankan migrate (membuat DocType: Stock Ops Push Subscription, Stock Ops Employee Warehouse)
bench --site <site> migrate
```

`after_install` otomatis membuat **custom field** (`external_localid`, `stock_ops_geolocation` di Material
Request & Stock Entry; `external_localid` di Stock Reconciliation; `stock_ops_warehouses` di Employee) dan
**VAPID keys** untuk push. Jika perlu jalankan ulang manual:

```bash
bench --site <site> execute stock_ops.setup.install.setup_custom_fields
bench --site <site> execute stock_ops.push.ensure_vapid_keys
```

> **Catatan struktur**: app Frappe biasanya berada di root repo. Di repo ini app ada di `backend/stock_ops/`.
> Jika `bench get-app` tidak menemukannya, salin manual: `cp -r backend/stock_ops ~/frappe-bench/apps/` lalu
> `bench --site <site> install-app stock_ops`. (Untuk produksi disarankan memisah app ke repo sendiri di root.)

---

## 💻 Menjalankan Frontend (mode dev, hot reload)

```bash
cd frontend
npm install

# buat file rahasia (TIDAK di-commit) untuk proxy dev:
#   .env.local
#   FRAPPE_URL=http://erp.localhost:8000
#   FRAPPE_API_KEY=xxxxx
#   FRAPPE_API_SECRET=xxxxx
npm run dev          # http://localhost:5173
```

- Backend harus jalan: di bench → `bench start`.
- Vite proxy meneruskan `/api`,`/files`,`/private`,`/assets` ke site (Host `erp.localhost`) dan menyuntik
  **token** dari `.env.local` di sisi Node — **token tidak masuk bundle browser**.
- Membuat API key/secret untuk user dev: di Frappe Desk → User → "API Access" → Generate Keys.

---

## 📦 Build & Sajikan via bench (mode produksi: cookie + CSRF)

Build di mesin yang punya `node_modules` (mis. Windows), lalu salin ke app di bench.

**Build (PowerShell):**
```powershell
cd frontend
$env:STOCKOPS_OUTDIR="dist-frappe"; $env:STOCKOPS_WWW="dist-frappe\index_www.html"
npm run build
node scripts/frappe-postbuild.mjs
Remove-Item Env:\STOCKOPS_OUTDIR; Remove-Item Env:\STOCKOPS_WWW
```

**Salin ke app + clear cache (bash di bench):**
```bash
APP=~/frappe-bench/apps/stock_ops/stock_ops
SRC=/path/ke/frontend/dist-frappe
rm -rf "$APP/public/stock_ops"; mkdir -p "$APP/public/stock_ops"
cp -r "$SRC"/. "$APP/public/stock_ops/"
rm -f "$APP/public/stock_ops/index_www.html"
cp "$SRC/index_www.html" "$APP/www/stock_ops/index.html"
bench --site <site> clear-cache
```

Akses: **`http://<site>/stock_ops`** (login Frappe dulu; guest diarahkan ke `/login`). App auto-login dari
sesi & memakai cookie + CSRF — **tanpa token di klien**.

---

## 📖 Cara Penggunaan

1. **Login** ke ERPNext, buka **`/stock_ops`**.
2. **Beranda**: jam/tanggal, lonceng 🔔 + banner **stok menipis**, menu peluncur, daftar dokumen terbaru.
3. **Buat dokumen** (➕): pilih jenis → tambah item (cari / **scan barcode 📷**) → qty → **foto** → **📍 Tag Lokasi** → Simpan.
   - Online → tersimpan draft di ERPNext; offline → masuk **Outbox**, auto-sync saat online.
4. **Stok**: saldo per gudang (progress bar). **Mutasi**: pergerakan dengan filter arah/gudang/tanggal.
5. **Detail item** (dari Stok / Cari-Scan): stok lintas gudang + mutasi + aksi cepat In/Out/Transfer.
6. **Stok Menipis**: centang item → **Request Beli** (1 MR Pembelian; qty saran = reorder qty item).
7. **Transfer Cepat** & **Stock Opname** (→ Stock Reconciliation) dari menu.
8. **Setelan**: bahasa (ID/EN), tema (terang/gelap), **Tampilan** (v2/Klasik/A/B), notifikasi push, sync/outbox, default company & gudang, laporan.

**Pengaitan gudang per karyawan**: buka **Employee** di Desk → isi field **"Stock Ops Warehouses"** (bisa
beberapa gudang). App otomatis membatasi stok/mutasi/opname user tsb ke gudang itu. (Alternatif: User Permission Warehouse.)

**Stok menipis** memakai **reorder level** per item-per-gudang (child *Item Reorder*); jika item tak punya
reorder level, dipakai **Ambang** (default 10, bisa diubah di layar).

---

## 🔒 Keamanan

- **Jangan commit** `frappe_api_keys_erp_mg.csv` atau `frontend/.env.local` (sudah di `.gitignore`).
- Token/secret hanya untuk skrip server-side & proxy dev — **tidak pernah** ditanam di bundle frontend.
- Produksi memakai **sesi Frappe (cookie) + CSRF**.

---

## 📄 Lisensi

MIT — lihat `backend/stock_ops/license.txt`.
