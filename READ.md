# MFPS — Material Flow Protection System

**PT. Pertamina Patra Niaga — Refinery Unit V Balikpapan**
Security HSSE | Ref. TKO B07-009/KPI48540/2025-S9

> Aplikasi mobile web (PWA) untuk digitalisasi pencatatan arus material di lingkungan kilang — barang masuk, barang keluar, dan scan dokumen berbasis AI.

---

## 🚀 Live App

**[https://andriylegality.github.io/mfps/](https://andriylegality.github.io/mfps/)**

---

## 📦 Modul

| Modul | Versi | Deskripsi |
|---|---|---|
| **Barang Masuk** | IBC v2.0 | Penerimaan material dari vendor — checklist dokumen, stamp DO, routing ke WH/Area Kerja |
| **Barang Keluar** | OBC v4.1 | Pengeluaran material — 4-step MRA workflow, 20 fraud detection rules |
| **Scan to Data** | AI (Claude) | OCR Surat Jalan & Bon Bintang menggunakan Claude AI |

---

## 🗂️ Struktur File

```
mfps/
├── index.html          # Launcher — halaman utama
├── inbound.html        # IBC v2.0 — Barang Masuk
├── outbound.html       # OBC v4.1 — Barang Keluar + Scan AI
└── docs/
    └── specs/
        └── outbond-chemical-workflow.md   # Spesifikasi OBC v5.x (DRAFT)
```

---

## 🔄 Alur Kerja

### Barang Masuk (IBC v2.0)
```
Email Vendor → Input Ticket → Vendor Tiba → Checklist Dokumen → Stamp DO → Routing
```
- Supply → **Warehouse 1** (New WH ex-Persiba)
- Kontrak/Jasa → **Area Kerja** (sesuai KAK)

### Barang Keluar (OBC v4.1)
```
Creator → Function Head → Legality Officer → Gate Security → RELEASED
```

---

## 👥 Peran Pengguna

### Inbound
| Peran | Akses |
|---|---|
| Legality Officer | Akses penuh — input, checklist, stamp, database, kontainer |

### Outbound
| Peran | Fungsi |
|---|---|
| Creator | Buat MRA (Material Release Authorization) |
| Function Head | Approve/reject MRA |
| Legality Officer | Verifikasi dokumen + digital stamp |
| Gate Security Inspector | Inspeksi fisik di gate |
| Security Section Head | Policy authorization — kasus eskalasi |
| Shift Superintendent (CCR) | Emergency override + offline fallback |

---

## 🤖 AI Integration

- **Model:** `claude-haiku-4-5-20251001`
- **Endpoint:** `https://api.anthropic.com/v1/messages`
- **Fungsi:** Ekstraksi otomatis data dari foto Surat Jalan atau Bon Bintang

---

## 💾 Teknologi

- **Frontend:** HTML5 + CSS3 + Vanilla JavaScript (ES6+)
- **Storage:** localStorage (client-side, tanpa server)
- **Platform:** Progressive Web App (PWA) — dapat diinstal di iOS/Android
- **Font:** Outfit + JetBrains Mono (Google Fonts)
- **Icon:** Font Awesome 6.5.1

---

## 📋 Dokumentasi

| Dokumen | Keterangan |
|---|---|
| [BRD v1.2](docs/BRD_v1.2.docx) | Business Requirements Document — spesifikasi lengkap |
| [OBC v5.x Chemical Spec](docs/specs/outbond-chemical-workflow.md) | Spesifikasi workflow chemical-aware (DRAFT) |

---

## 🗺️ Roadmap

- [x] IBC v2.0 — Inbound receiving management
- [x] OBC v4.1 — Outbound 4-step MRA workflow
- [x] Scan to Data — Claude AI OCR
- [ ] OBC v5.x — Chemical-aware workflow (HSSE/OH authority)
- [ ] Server-side storage + append-only audit log
- [ ] Real-time gate monitor dashboard

---

*Aplikasi ini dikembangkan untuk keperluan internal PT. Pertamina Patra Niaga — Refinery Unit Balikpapan. Legality - Security HSSE & bersifat Confidential.*
