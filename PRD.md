# PRD — Stock Ops PWA (ERPNext v16)

> Custom Frappe App berupa Progressive Web App (PWA) mobile-first untuk operasi gudang:
> Material Request (Transfer & Purchase) dan Stock Entry (In/Out/Transfer), dengan dukungan
> **offline-capable** (read cache + outbox queue) dan upload **foto barang** ke attachment.
>
> Status: **DRAFT untuk review**
> Tanggal: 2026-05-30
> Target backend: **Frappe 16.17.2 / ERPNext 16.16.0** (version-16)

---

## 1. Ringkasan & Tujuan

Membuat aplikasi internal (PWA) yang dipasang sebagai **custom app di dalam frappe-bench**, mirip cara
**Frappe HR (hrms)** menyajikan PWA-nya. App fokus pada alur stok lapangan agar petugas gudang bisa
bekerja cepat dari HP — termasuk saat sinyal buruk/offline — lalu tersinkron otomatis saat online,
mirip perilaku **POS ERPNext offline**.

**Tujuan utama:**
1. Input transaksi stok dari HP secara cepat (mobile-first, kamera, scan).
2. Tetap bisa membuat transaksi saat offline; sinkron otomatis saat online.
3. Lampirkan foto barang (bukti fisik) ke dokumen ERPNext.
4. Tidak mengubah core ERPNext — semua lewat DocType & API standar.

**Non-tujuan (versi awal):**
- Bukan pengganti penuh ERPNext desktop.
- Tidak menangani edit konkuren kompleks / merge konflik tingkat lanjut (lihat §6).
- Tidak menangani Manufacture/Repack/Subcontract (bisa menyusul).

---

## 2. Pengguna & Use Case

| Peran | Kebutuhan |
|---|---|
| Petugas Gudang | Buat Stock Entry (terima/keluar/transfer), foto barang, scan item |
| Staff Pengadaan / Requestor | Buat Material Request (Purchase) — minta beli |
| Staff Antar-Gudang | Buat Material Request (Material Transfer) — minta pindah stok |
| Supervisor / Approver | Lihat daftar, submit/approve, pantau status |

**User journey inti (contoh — terima barang offline):**
1. Buka app di HP (sudah login, app shell ter-cache).
2. Pilih "Stock Entry → Material Receipt".
3. Scan/cari item (dari cache master), isi qty + gudang tujuan.
4. Ambil foto barang dengan kamera.
5. Simpan → karena offline, transaksi masuk **Outbox** (status: *pending*).
6. Saat sinyal kembali → app auto-sync: buat dokumen di ERPNext + upload foto → status *synced*.

---

## 3. Scope & Fitur

### 3.1 Pemetaan istilah → DocType ERPNext

| Istilah Anda | DocType ERPNext | Field penanda |
|---|---|---|
| **MR** | Material Request | `material_request_type = "Material Transfer"` |
| **PR** | Material Request | `material_request_type = "Purchase"` |
| **Stock Out** | Stock Entry | `stock_entry_type = "Material Issue"` |
| **Stock In** | Stock Entry | `stock_entry_type = "Material Receipt"` |
| **Stock Transfer** | Stock Entry | `stock_entry_type = "Material Transfer"` |

### 3.2 CORE (Phase fungsi inti)

- **C1. Auth & sesi** — login ke Frappe, simpan sesi, deteksi online/offline.
- **C2. Master data cache** — Item, Warehouse, UOM, Company, (Supplier untuk PR) di-cache ke IndexedDB.
- **C3. Material Request — Transfer (MR)** — buat, daftar, detail, submit.
- **C4. Material Request — Purchase (PR)** — buat, daftar, detail, submit.
- **C5. Stock Entry — In/Out/Transfer** — buat, daftar, detail, submit.
- **C6. Foto barang** — ambil/pilih foto, lampirkan ke dokumen (File / attachment).
- **C7. Offline outbox + sync engine** — antrian transaksi & foto, auto-flush saat online.
- **C8. Scan barcode/QR item** (pakai kamera) untuk isi item cepat.

### 3.3 NON-CORE (menyusul)

- N1. Approval workflow di app (submit/cancel/approve role-based). ✅ submit & cancel
- N2. Notifikasi (push) saat status berubah. ✅ (Web Push/VAPID, trigger on_submit)
- N3. Laporan ringkas / dashboard stok. ✅ (Laporan: jumlah per subtipe, bulan ini)
- N8. Daftar dokumen dari server (tab Server). ✅
- N9. Service worker / PWA installable. ✅ (di-serve bench)
- N10. Stok gudang per user/employee. ✅
- N11. Pergerakan stok (Stock Ledger) + filter arah/gudang/item. ✅
- N12. Pengaitan gudang per Employee (Table MultiSelect di Employee). ✅
- N13. UI refresh: SearchInput pill, chips scroll, sticky toolbar, skeleton, animasi. ✅
- N4. Multi-bahasa (ID/EN).
- N5. Cetak/PDF dokumen.
- N6. Stock reconciliation / opname.
- N7. Konversi MR → Stock Entry / Purchase Order langsung dari app.

### 3.4 Urutan pengerjaan (sesuai permintaan: UI dulu untuk dicoba)

- **Phase 0 — UI Mock (clickable, data dummy/lokal)** ← *yang pertama dibuat agar Anda coba*
- **Phase 1 — Core online** (C1–C6, konek ke `erp.localhost`)
- **Phase 2 — Offline & sync** (C7, C2 cache, C8 scan)
- **Phase 3 — Non-core** (N1–N7 sesuai prioritas)
- **Phase 4 — Deploy ke produksi** (`erp.globalmagicko.com`)

---

## 4. Arsitektur Teknis

### 4.1 Bentuk app (mengikuti pola HRMS)

```
frappe-bench/
└── apps/
    └── stock_ops/                  # custom app baru (python wrapper minimal)
        ├── stock_ops/
        │   ├── hooks.py            # daftar PWA route, website_route_rules
        │   ├── api.py              # whitelisted endpoint khusus (mis. bundling)
        │   └── public/
        │       └── stock_ops/      # hasil build frontend di-copy ke sini
        └── frontend/               # source PWA (Vite + Vue 3)
            ├── src/
            ├── vite.config.js      # plugin PWA + proxy ke site lokal
            └── package.json
```

- Frontend di-build oleh **Vite**, output ditaruh di `public/`, lalu Frappe serve di route
  **`/stock_ops`** (lewat `website_route_rules` + `www` index, sama seperti `/hrms`).
- Login memakai **sesi Frappe** (cookie). Untuk dev frontend, Vite proxy ke
  `http://erp.localhost:8000` agar API & cookie jalan.

### 4.2 Tech stack frontend

| Lapisan | Pilihan | Alasan |
|---|---|---|
| Framework | **Vue 3** (script setup) | sama dengan HRMS, ekosistem Frappe |
| UI kit | **frappe-ui** + **Ionic Vue** | komponen mobile-native (HRMS pakai ini) |
| Build/PWA | **Vite** + `vite-plugin-pwa` (Workbox) | service worker + manifest otomatis |
| State | **Pinia** | store outbox, sesi, cache status |
| DB offline | **IndexedDB via Dexie.js** | cache master + outbox + blob foto |
| HTTP/data | **frappe-ui resources** + axios | konsisten dgn Frappe |
| Router | **Vue Router** | navigasi SPA |
| Scan | `@zxing/browser` atau `html5-qrcode` | barcode/QR via kamera |

### 4.3 Diagram alur data (ringkas)

```
[Vue UI] → tulis → [Dexie: outbox + foto blob]
                         │  (online?)
                         ▼
                   [Sync Engine] ── POST ──▶ [Frappe REST API] ──▶ [DocType]
                         │                         ▲
                         └── upload_file ──────────┘ (lampirkan foto ke doc)

[Frappe REST] ──get──▶ [Dexie: master cache] ──baca──▶ [Vue UI] (saat offline)
```

---

## 5. Model Data & Pemetaan Field

### 5.1 Material Request (MR & PR)

`Material Request` (parent):
- `material_request_type`: `"Material Transfer"` (MR) | `"Purchase"` (PR)
- `transaction_date`, `schedule_date`, `company`
- child `items` → **Material Request Item**:
  - `item_code`, `qty`, `uom`, `schedule_date`
  - `warehouse` (target / "for warehouse")
  - `from_warehouse` (sumber, untuk tipe Material Transfer)

### 5.2 Stock Entry (In/Out/Transfer)

`Stock Entry` (parent):
- `stock_entry_type`: `"Material Receipt"` | `"Material Issue"` | `"Material Transfer"`
- `company`, `posting_date`, `posting_time`
- `from_warehouse` / `to_warehouse` (default header, opsional)
- child `items` → **Stock Entry Detail**:
  - `item_code`, `qty`, `uom`, `basic_rate` (opsional)
  - `s_warehouse` (source — untuk Issue/Transfer)
  - `t_warehouse` (target — untuk Receipt/Transfer)

### 5.3 Foto / Attachment

- Upload via endpoint standar Frappe **`/api/method/upload_file`** (multipart):
  - `file`, `is_private`, `doctype`, `docname`, `fieldname` (opsional), `folder`.
- File ter-link ke dokumen lewat `attached_to_doctype` + `attached_to_name`.
- Strategi: setelah dokumen terbuat (online/saat sync), upload tiap foto dan attach ke doc tsb.
- (Opsional) jika ingin foto melekat pada baris item tertentu → simpan referensi item_code di
  deskripsi/keterangan file, atau pakai custom field "Photos" (Attach) — diputuskan di Phase 1.

### 5.4 Master data yang di-cache

`Item` (code, name, image, stock_uom, has_batch/serial, barcode), `Warehouse`, `UOM`,
`Company`, `Supplier` (untuk PR). Cache via list API + filter perusahaan & disabled=0.

---

## 6. Strategi Offline & Sync (mirip POS offline)

### 6.1 Prinsip

- **Read**: master data di-prefetch & disimpan di IndexedDB → UI bisa cari item/gudang tanpa internet.
- **Write (create-only di v1)**: transaksi yang dibuat offline → masuk **Outbox** dengan
  `localId` (UUID) + status. Tidak mengedit dokumen ERPNext yang sudah ada saat offline (hindari konflik).

### 6.2 Skema Outbox (Dexie)

```
outbox: {
  localId: uuid,
  type: "MR" | "PR" | "STOCK_ENTRY",
  payload: {...},                 // body siap kirim ke API
  photos: [blobRef...],           // referensi ke tabel photos
  status: "pending"|"syncing"|"synced"|"error",
  remoteName: null|string,        // nama doc ERPNext setelah sync
  error: null|string,
  createdAt, updatedAt
}
photos: { id, localId, blob, filename, itemCode }
meta:   { masterLastSync, session }
```

### 6.3 Sync engine

1. Trigger: event `online`, app foreground, atau tombol "Sync sekarang".
2. Ambil item outbox `status=pending` berurutan (FIFO).
3. Set `syncing` → POST buat dokumen (`/api/resource/<DocType>` atau
   `frappe.client.insert`). Idempotensi: kirim header/marker `localId` (custom field opsional)
   agar retry tidak menduplikasi; bila tanpa custom field, cek duplikat via field unik.
4. Bila sukses → simpan `remoteName`, upload semua foto via `upload_file`, set `synced`.
5. Bila gagal jaringan → kembalikan `pending` (retry backoff). Bila gagal validasi server →
   `error` + tampilkan pesan agar user perbaiki.
6. Submit/draft: v1 default **draft** (docstatus 0) agar aman; submit jadi aksi terpisah
   (online) — bisa dikonfigurasi.

### 6.4 Konflik & batas v1

- Hanya **create**; tidak ada offline-edit dokumen submitted.
- Nomor dokumen (`name`) dibuat **server** → di app pakai `localId` sampai tersync.
- Stok aktual tidak divalidasi offline (qty tersedia bisa berubah) → validasi final di server,
  error ditampilkan untuk koreksi.

---

## 7. Auth & Keamanan

- **Dev/lokal**: login via halaman Frappe (`/login`) → cookie sesi dipakai PWA (sama origin).
- **Produksi**: app di-serve dari domain yang sama (`erp.globalmagicko.com/stock_ops`) → cookie sesi.
- **API key/secret** (file `frappe_api_keys_erp_mg.csv`) **hanya** untuk skrip/integrasi server-side
  atau seeding/test — **tidak** ditanam di bundle frontend (rawan bocor). PWA pakai sesi user.
- Permission mengikuti Role ERPNext user (Stock User, Stock Manager, dll) — app tidak bypass.
- File foto default **private** (`is_private=1`) kecuali ditentukan lain.

> Catatan keamanan: CSV berisi secret. Jangan commit ke git publik; tambahkan ke `.gitignore`.

---

## 8. UI/UX (Mobile-First)

### 8.1 Navigasi (bottom tab, ala app native)

```
┌───────────────────────────────┐
│  Stock Ops            ●online  │  ← status bar + indikator sync
├───────────────────────────────┤
│                               │
│   (konten halaman)            │
│                               │
├───────────────────────────────┤
│  🏠       📋        ➕      🔄 │
│ Beranda  Daftar   Buat   Sync │  ← bottom tab
└───────────────────────────────┘
```

### 8.2 Daftar layar

1. **Login** (jika belum ada sesi).
2. **Beranda** — ringkasan: jumlah outbox pending, terakhir sync, shortcut buat MR/PR/Stock Entry.
3. **Buat (➕)** — pilih jenis: MR (Transfer) / PR (Purchase) / Stock In / Stock Out / Transfer.
4. **Form transaksi** — header (gudang, tanggal) + daftar item (tambah baris, scan, qty) + foto + Simpan.
5. **Tambah item** — search item dari cache, atau scan barcode, set qty/uom/gudang.
6. **Daftar dokumen** — filter per jenis & status (draft/submitted/pending-sync), pull-to-refresh.
7. **Detail dokumen** — lihat item, foto, status sync, tombol submit (online).
8. **Outbox/Sync** — daftar transaksi pending/error, retry manual, lihat error.
9. **Pengaturan** — pilih company/gudang default, sync master, logout.

### 8.3 Prinsip UX

- Tombol besar, target sentuh ≥ 44px, input numerik untuk qty.
- Indikator **online/offline** & **jumlah pending** selalu terlihat.
- Kamera 1-tap untuk foto; thumbnail bisa dihapus sebelum simpan.
- Optimistic UI: setelah Simpan langsung muncul di daftar dengan badge "pending".

---

## 9. API ERPNext yang Dipakai

| Keperluan | Endpoint |
|---|---|
| Login | `POST /api/method/login` |
| User info | `GET /api/method/frappe.auth.get_logged_user` |
| List master/dokumen | `GET /api/resource/<DocType>?fields=..&filters=..&limit_page_length=..` |
| Detail | `GET /api/resource/<DocType>/<name>` |
| Buat dokumen | `POST /api/resource/<DocType>` (body JSON) |
| Submit | `PUT /api/resource/<DocType>/<name>` (`docstatus`) atau method submit |
| Upload foto | `POST /api/method/upload_file` (multipart) |
| Cek versi/ping | `GET /api/method/frappe.ping` |

(Phase lanjut bisa buat 1 whitelisted method `stock_ops.api.create_with_attachments`
untuk membungkus buat-dokumen + attach foto dalam 1 transaksi server.)

---

## 10. Setup Lingkungan Dev (WSL)

Terdeteksi: WSL **Ubuntu-24.04**, `~/frappe-bench`, Frappe 16.17.2 / ERPNext 16.16.0,
site **`erp.localhost`**. Langkah:

```bash
# di WSL
cd ~/frappe-bench
bench new-app stock_ops            # buat app
bench --site erp.localhost install-app stock_ops
cd apps/stock_ops && mkdir frontend && cd frontend
# scaffold Vite + Vue + PWA, frappe-ui, ionic, dexie ...
yarn dev                           # dev server (proxy ke erp.localhost:8000)
# build → output ke ../stock_ops/public/stock_ops, lalu akses /stock_ops
```

Akses dev: `http://erp.localhost:8000` (backend) + Vite dev server (frontend).

---

## 11. Roadmap / Milestone

| Phase | Isi | Hasil yang bisa dicoba |
|---|---|---|
| **P0** | UI mock clickable (dummy data, no backend) | Klik-klik semua layar di HP/browser |
| **P1** ✅ | App `stock_ops` + custom field `external_localid` + API (get_bootstrap/create_transaction/submit) + frontend tersambung (token proxy dev) | Buat dokumen riil di `erp.localhost` |
| **P2** ✅ | Foto→IndexedDB (Blob), auto-sync saat online, scan barcode kamera (@zxing). SW PWA aktif saat build Node20+/WSL | Buat saat offline, auto-sync, scan |
| **P3** | Non-core (approval, notif, laporan, dll) | Sesuai prioritas |
| **P3.5** ✅ | SPA tersaji bench di `/stock_ops` (cookie + CSRF, auto-login sesi, www page + redirect guest) | Akses `erp.localhost:8000/stock_ops` |
| **P4** | Deploy ke `erp.globalmagicko.com` (install app + build + HTTPS → scan & PWA install jalan di HP) | Produksi |

---

## 12. Keputusan Teknis (FINAL — hasil review)

1. **Idempotensi sync** → **Ya**, tambah **custom field `external_localid`** (Data, hidden, unique)
   di **Material Request** & **Stock Entry**. Diisi `localId` (UUID) dari app; sebelum insert saat
   sync, cek apakah sudah ada doc dengan `external_localid` sama → cegah duplikat saat retry.
2. **Submit vs draft** → simpan sebagai **draft** (`docstatus=0`). Submit = aksi terpisah (online).
3. **Foto** → di-attach ke **dokumen** (`attached_to_doctype`+`attached_to_name`), bukan per item line.
4. **Batch/Serial** → **belum dipakai** → form tidak menampilkan field batch/serial (siapkan hook untuk nanti).
5. **Default Company & Warehouse** → disimpan di **halaman Settings** app (per device/user),
   dipakai sebagai prefill form. Tidak ada pembatasan keras per user di v1.
6. **Barcode** → **belum dipakai**, tapi **tombol scan disiapkan** (UI + handler), tinggal aktifkan
   saat field `barcode` Item terisi.

---

## 13. Keputusan yang sudah disepakati

- MR = Material Request (Material Transfer); PR = Material Request (Purchase).
- Stock Entry: In=Material Receipt, Out=Material Issue, Transfer=Material Transfer.
- Arsitektur: custom Frappe app (Vue3 + frappe-ui + Ionic PWA, di-serve bench) — pola HRMS.
- Dev di `erp.localhost` dulu, lalu deploy ke produksi.
- Offline = read cache + outbox queue (create-only) dengan auto-sync.
- Mobile-first; foto barang di-upload ke attachment.
- Bangun **UI mock dulu (P0)** agar bisa langsung dicoba.

### Catatan P0 (UI mock)
- Lokasi: `D:\Workspace\StockInOutRMI\frontend` (standalone Vite + Vue 3, data dummy, tanpa backend).
- Dijalankan dengan `npm install` lalu `npm run dev`, dibuka di browser/HP (mobile-first).
- Komponen P0 pakai CSS custom ringan agar cepat dicoba; di P1 diintegrasikan ke app bench
  `stock_ops` (frappe-ui + Ionic) dan disambungkan ke API ERPNext.
```
