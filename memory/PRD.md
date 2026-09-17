# PRD — Kain Nusantara ERP (repo makisjsbs/KN) · sesi 2026-09-16 · DESIGN STUDIO

## Problem statement (verbatim)
saya ingin anda lanjutkan development dari repo ini https://github.com/makisjsbs/KN
saya ingin anda lanjutkan development fokus pada desaigner, untuk saat ini fitur ini masih sangat basic. beberapa point yang ingin saya kembangkan
1. ketika membuat design baru. kode otomatis dan bisa dikonfigurasi bukan input custom
2. tambahkan kolom category motif/category pattern/ category artwork sesua dengan jenis design yang dipilih
3. design ini sebenarnya adalah pattern yang bisa di implementasikan untuk product, 1 design bisa lebih dari 1 product ... optional (rekomendasi/peruntukan)
4. tambahkan referensi foto yang bisa ditambahkan oleh yang membuat design.
5. tag jangan hanya custom value namun tersimpan juga jadi nextnya bisa diketik sedikit dan langung bisa pilih
6. filter lengkap
7. bisa melihat detail history design beserta dengan versi versinya dan timelinenya juga, catatan & kenapa belum acc, feedback dua arah designer ↔ penilai
8. mekanisme penilaian: skala 0 - 2 kelipatan 0,25
9. setiap artwork setiap versinya diberikan nilai oleh penilai dengan nilai final acc
10. lifecycle design yang baik, tahapan jelas, nanti bisa ditambahkan referensi mockup
11. setiap design punya alternative design (kombinasi warna berbeda)
12. warna harus sesuai master (kode warna benar), tidak bisa asal

## Keputusan pemilik (ask_human)
- Kode: `{DESIGNER}-{TYPE}-{CAT}-{SEQ}` → mis. `BDI-PTR-SLR-001` (inisial desainer · jenis · kategori · urut), pola & prefix bisa diubah di Pusat Pengaturan (grup R&D).
- Foto referensi/mockup: storage LOKAL dulu (storage_service), migrasi object storage nanti.
- Lifecycle disetujui: Draf → Diajukan → Dalam Review → Perlu Revisi (versi baru) → Disetujui (ACC) → Aktif/Produksi → Diarsipkan; halaman detail khusus.
- Nilai: SATU nilai total per versi + catatan; ambang ACC default 1,5 (konfigurasi `rnd.design_acc_min_score`).
- Warna: pakai master `color_library` yang sudah ada.

## Arsitektur yang diimplementasikan (2026-09-16)
Backend (menumpang koleksi `design_gallery`, bukan koleksi baru):
- `services/design_studio_service.py` — kategori (`design_categories`, seed 17 default), kode otomatis (`next_code`, `designer_prefix`), tag tersimpan (`design_tags`), `resolve_colors` (wajib color_library aktif), `resolve_products`, lifecycle `transition()` + `timeline[]`, `score_version()` (0–2 step 0,25, riwayat), `add_feedback()` (side designer/assessor + notifikasi), `new_version()`, colorway CRUD, `enrich()`.
- `routers/design_studio.py` — `/api/design-studio/{meta,next-code,tags,categories}`, `/api/design-gallery/{id}/lifecycle/{submit|start-review|request-revision|approve|activate|archive|reopen}`, `/new-version`, `/versions/{v}/score`, `/feedback`, `/colorways[/{cw}]`, `/files-kind/{artwork|reference|mockup}`.
- `config_catalog_rnd.py` — `rnd.design_code_pattern`, `rnd.design_code_seq_digits`, `rnd.design_prefix_{motif,pattern,artwork}`, `rnd.design_acc_min_score`.
- `design_gallery_service.py` — create (kode otomatis, kategori, warna, produk, timeline), list filter (status/type/category/created_by/product/color), `add_file(kind, version, caption)`; `is_artwork` = kind artwork saja.
- RBAC: lihat rnd.view|hr.view; tulis rnd.manage|hr.manage_attendance|design_request.deliver (designer); menilai rnd.assess (admin/manager).

Frontend (`features/rnd/`):
- `RndDesignsView.jsx` — kartu + KPI + filter lengkap (status, jenis, kategori, desainer, tag, produk, warna, nilai, artwork, lini, cari); klik kartu → halaman detail.
- `DesignFormModal.jsx` — kode preview otomatis (read-only), kategori mengikuti jenis, `ColorPicker` (master), `ProductPicker` (opsional multi), `TagInput` (autocomplete tersimpan), kelola kategori.
- `design/DesignDetailPage.jsx` — stepper lifecycle, `LifecycleActions` (aksi sesuai status & peran, dialog nilai+catatan), tab Ringkasan · Versi & Nilai (`VersionsPanel`, `ScoreInput`) · Timeline · Umpan Balik (`FeedbackThread`) · Warna & Alternatif (`ColorwaysPanel`) · Referensi & Mockup (`FilesPanel`).
- hubTabs: tab "Desain & Pattern" kini terbuka untuk peran `designer`.
- Catatan: frontend disajikan dari bundle statis → `bash scripts/rebuild_frontend.sh` setelah ubah src.

## Uji
- Testing agent iteration_19: 12/12 backend PASS, UI admin & designer PASS (test_reports/iteration_19.json).

## 2026-09-16 (lanjutan) — KPI Desainer × Design Studio
- `rnd_kpi_service.design_studio_stats()` mengagregasi `design_gallery` per desainer (created_by) per periode: `designs`, `design_versions`, `design_scored`, `design_avg_score` (0–2), `design_acc`, `design_acc_rate`, `design_revisions` (event `request_revision`). Digabung ke baris `designer_kpi` (desainer yang hanya punya desain ikut muncul, grade tetap dari round sample), ringkasan (`summary.design_*`), ekspor CSV/Excel/PDF (4 kolom baru), rapor per-desainer (4 metrik baru), dan KPI Saya.
- UI: `DesignerKpiTable` +4 kolom (Versi desain · Nilai desain /2 · ACC desain · Revisi desain); `DesignerKpiView` +2 kartu ringkasan (testid `designer-kpi-summary-design-score`, `designer-kpi-summary-design-revisions`).
- Belum: nilai desain belum masuk bobot grade komposit (masih terpisah, skala berbeda); tren bulanan belum memuat nilai desain. → SELESAI 2026-09-17 (lihat bawah).

## 2026-09-17 — Tren Nilai Desain + Bobot Nilai Desain ke grade komposit
- Setting baru `rnd.kpi_weight_design` (pct, default 20, grup R&D, Pusat Pengaturan) → `rnd_gate.policy` → `rnd_kpi_service.weights()["design"]`.
- `compute_grade`: komponen ke-4 = `design_avg_score × 50` (0–2 → 0–100), dinormalkan bersama on_time/score/acc; keluaran baru `grade_design_pts`. `designer_kpi` menggabungkan statistik Design Studio SEBELUM grade dihitung (desainer yang hanya punya desain kini ikut ter-grade). Rumus di "Cara nilai dihitung" (UI), ekspor PDF/Excel (`formula_of`) dan tooltip tabel menyebut bobot desain.
- Tren: `GET /rnd/reports/designer-kpi/trend?metric=design_score` → `design_score_trend()` (rata nilai versi desain per desainer per bulan; tanggal = `score_at` fallback `at`; hanya desainer yang punya nilai). Tren `metric=grade` juga ikut memasukkan nilai desain bulan itu.
- UI `DesignerKpiView`: baris `designer-kpi-trend-row` (grid 2 kolom xl) = `DesignerKpiTrendChart` (kiri) + `DesignScoreTrendChart` (kanan, testid `design-score-trend`, Y 0–2, garis ambang ACC 1,5). Tautan `designer-kpi-design-weight-link` membuka setting bobot desain.
- Demo data: `python scripts/seed_design_scores_demo.py` (dari /app/backend) menambah versi desain bernilai 3 bulan ke belakang untuk Dewi Lestari, Desainer Demo, Bagas Nugroho.

## 2026-09-17 — Visual Master Produk (Produk & Varian) + showcase Endek Bali Rangrang
- `features/catalog/`: `FamilyHero.jsx` (foto utama + 6 fakta kunci + deskripsi + swatch warna), `VariantCard.jsx` (kartu varian bergambar: thumbnail cover, titik warna, harga, lifecycle, stok, ringkas foto/mockup/artwork), `FamilyInfoTab.jsx` (4 kartu: deskripsi · spesifikasi teknis 2 kolom · kain dasar · atribut varian sebagai chip), `ProductRelations.jsx` (2 kartu: spesifikasi R&D + asal desain & artwork strip), `VariantMedia.jsx` (galeri grid tile dengan filter jenis Semua/Foto/Detail/Mockup/Artwork, badge jenis, ikon foto utama). Tab ber-count, CSS baru di `catalog.css` (hero, variant-card, info-card, media-grid, responsif 390px).
- Backend TIDAK berubah (visual saja, sesuai keputusan pemilik).
- Showcase: `scripts/seed_endek_showcase.py` — induk ENK-BALI-003 diberi axis Warna (4 opsi dari color_library), 3 varian baru (Merah Marun, Kuning Emas, Hitam Pekat), 12 media stok foto lokal (`scripts/demo_media/*.jpg`, Unsplash) jenis photo/detail/mockup/artwork, semua disetujui + cover.

## 2026-09-17 — Bugfix: preview blank setelah rebuild (service worker basi)
- Akar masalah: `public/sw.js` cache-first untuk `/` & `/index.html` → index lama merujuk chunk ber-hash yang sudah hilang; `static_server.js` mengembalikan index.html (SPA fallback) untuk `/static/js/*` yang hilang → JS gagal parse → halaman putih.
- Fix: SW `kn-sw-v2` — navigasi/index network-first (fallback cache hanya saat offline), aset `/static` tetap cache-first; cache `kn-sw-v1-*` dihapus saat activate. `static_server.js`: 404 untuk aset ber-hash yang hilang, `Cache-Control: no-store` untuk html/sw.js. `index.js`: `reg.update()` saat load + auto-reload sekali (bersihkan cache) bila script `/static/js` gagal dimuat.
- Catatan operasional: `static_server.js` berubah → perlu `sudo supervisorctl restart frontend` sekali (sudah dilakukan). Rebuild berikutnya cukup `bash scripts/rebuild_frontend.sh`.

## Backlog / P1–P2
- P1: Migrasi berkas desain ke Emergent Object Storage.
- P1: Tautkan colorway → labdip/proofing (permintaan sample per colorway).
- P2: Mockup AI per colorway (gemini_image_service sudah ada di galeri).
- P2: KPI desainer ikut membaca nilai versi desain (saat ini KPI dari round sample).
- P2: Field `designer_code` per user (override inisial otomatis) di Pengaturan Akun.
- P2: Filter server-side + paginasi bila desain > 2000.
