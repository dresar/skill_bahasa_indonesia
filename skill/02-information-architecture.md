# 02 — Information Architecture
## AI Skill Factory Indonesia

---

## 1. Sitemap Lengkap

### 1.1 Halaman Publik

| Route | Nama Halaman | Deskripsi |
|---|---|---|
| `/` | Beranda | Hero, search, trending, kategori, AI Factory teaser, statistik |
| `/skills` | Skill Library | Daftar skill dengan filter & search |
| `/skills/[slug]` | Detail Skill | Tab Ringkasan/README/SKILL.md/Contoh/Scripts/Resources/Versi/Changelog/Ulasan |
| `/categories` | Daftar Kategori | Grid kategori dengan jumlah skill |
| `/categories/[slug]` | Skill per Kategori | Skill Library terfilter kategori |
| `/prds` | Daftar PRD | PRD hasil generate yang dipublikasikan |
| `/prds/[slug]` | Detail PRD | Isi PRD + metadata + unduh |
| `/workflows` | Daftar Workflow | Workflow otomatisasi/agent |
| `/workflows/[slug]` | Detail Workflow | Isi workflow + langkah + unduh |
| `/agent-kits` | Daftar Agent Kit | Kit lintas-stack (frontend+backend+db+auth) |
| `/agent-kits/[slug]` | Detail Agent Kit | Isi kit + struktur file + unduh |
| `/templates` | Daftar Template | Template skill/PRD siap pakai |
| `/templates/[slug]` | Detail Template | Isi template |
| `/search` | Hasil Pencarian | Hasil lintas tipe konten |
| `/collections` | Daftar Collection | Official + (jika login) collection pribadi |
| `/collections/[slug]` | Detail Collection | Isi collection |
| `/generate` | Generator Hub | Pilih jenis generator |
| `/generate/skill` | Skill Generator | Form + wizard AI Skill Factory |
| `/generate/prd` | PRD Generator | Form PRD |
| `/generate/workflow` | Workflow Generator | Form workflow |
| `/generate/agent-kit` | Agent Kit Generator | Pemilihan stack + generate |
| `/docs` | Docs Hub | Indeks dokumentasi |
| `/docs/[slug]` | Halaman Docs | Konten MDX |
| `/changelog` | Changelog Platform | Rilis fitur platform (bukan skill) |
| `/roadmap` | Roadmap Publik | Fase & rencana ke depan |
| `/about` | Tentang | Visi, tim, kontak |
| `/github` | Redirect/halaman info open-source | Link repo, cara kontribusi |
| `/login`, `/register` | Auth | Form login/registrasi |
| `/profile` | Profil Publik Pengguna | (jika diaktifkan) |
| `/settings` | Pengaturan Akun | Profil, preferensi, koneksi OAuth |

### 1.2 Halaman Admin (`/admin/*`, protected RBAC)

`/admin` (Dashboard) · `/admin/skills` · `/admin/prds` · `/admin/workflows` · `/admin/agent-kits` · `/admin/categories` · `/admin/tags` · `/admin/comments` · `/admin/reports` · `/admin/users` · `/admin/generator` (Generation Workspace) · `/admin/prompts` (Prompt AI) · `/admin/validation` · `/admin/security` · `/admin/analytics` · `/admin/ai-providers` · `/admin/settings` · `/admin/audit-logs`

---

## 2. Navigasi

### Desktop
- **Top nav publik:** Logo — Skills — PRD — Workflows — Agent Kits — Docs — Cmd/Ctrl+K search — Login/Avatar
- **Sidebar admin:** grup menu sesuai tabel §1.2, collapsible, badge jumlah item pending review

### Mobile
- **Bottom navigation publik:** Beranda · Skills · Generate · Search · Akun (5 item maksimal)
- **Admin mobile:** hamburger drawer, bukan sidebar permanen; tabel diganti card list (lihat `03`)

---

## 3. User Journey Utama

### 3.1 Journey: Developer menemukan & memakai skill
```
Beranda → Search "nextjs error handling"
        → Skill Library (filter: kategori=Framework, agent=Claude Code)
        → Detail Skill → tab "SKILL.md" untuk preview isi
        → klik "Salin Perintah Instalasi" ATAU "Unduh ZIP"
        → (opsional, login) Bookmark / tambah ke Collection
```
**Acceptance criteria:** pencarian, filter, preview isi, dan unduh semuanya dapat dilakukan **tanpa login**.

### 3.2 Journey: Developer membuat skill baru (via akses yang diizinkan, atau via admin)
```
/generate/skill → isi form requirement (nama, tujuan, teknologi, agent target)
                → submit → generation job dibuat → progress ditampilkan real-time (step events)
                → hasil canonical JSON dapat dipreview
                → validasi otomatis (schema + security + quality) berjalan
                → jika role mengizinkan publish langsung → publish
                → jika tidak → masuk antrean review admin
```

### 3.3 Journey: Admin generate → publish
```
/admin/generator → "Buat Baru" → isi requirement
                 → timeline: Requirement→Research→PRD→Architecture→Skill→Examples→README→Security→Quality→Validation→Assembly→Package
                 → admin bisa: lihat JSON mentah, edit di Monaco Editor, regenerate step tertentu, retry
                 → Approve → Publish → versi 1.0.0 tersimpan sebagai snapshot
```

### 3.4 Journey: Pengunjung anonim vs pengguna terautentikasi vs admin (permission matrix)

| Aksi | Anonim | User | Admin |
|---|---|---|---|
| Lihat skill/PRD/workflow/agent kit publik | ✅ | ✅ | ✅ |
| Search & filter | ✅ | ✅ | ✅ |
| Unduh ZIP / salin instalasi | ✅ | ✅ | ✅ |
| Rating, komentar, reply, upvote | ❌ | ✅ | ✅ |
| Bookmark, buat collection pribadi | ❌ | ✅ | ✅ |
| Pakai Generator (sesuai kuota/permission) | ❌ (harus login) | ✅ (terbatas) | ✅ (penuh) |
| Publish resmi (masuk Skill Library utama) | ❌ | ❌ | ✅ |
| Moderasi komentar/report | ❌ | ❌ | ✅ |
| Kelola AI provider, prompt, security, analytics | ❌ | ❌ | ✅ |

---

## 4. Halaman Generator — Detail Perilaku

Setiap halaman `/generate/*` mengikuti pola 4 tahap yang sama:

1. **Form Requirement** — Bahasa Indonesia, field wajib divalidasi Zod sebelum submit
2. **Generation Progress** — timeline step (lihat §3.3), disconnect-safe (job tetap berjalan walau tab ditutup)
3. **Hasil & Review** — canonical JSON + preview file yang akan dihasilkan compiler
4. **Aksi lanjutan** — simpan sebagai draft / ajukan review / (jika berwenang) publish langsung

---

## 5. Empty / Error / Loading States (Ringkas — detail penuh di `03`)

Setiap route wajib punya 7 state: **Loading, Empty, Error, Success, Unauthorized, Forbidden, Not Found** — semua berbahasa Indonesia. Contoh copy standar:

| State | Contoh Copy |
|---|---|
| Empty (Skill Library, filter tidak ada hasil) | "Belum ada skill yang cocok dengan filter ini. Coba ubah kata kunci atau kategori." |
| Unauthorized | "Kamu perlu masuk untuk menggunakan fitur ini." + CTA "Masuk" |
| Forbidden (user coba akses `/admin`) | "Kamu tidak memiliki akses ke halaman ini." |
| Not Found (slug skill tidak ada) | "Skill yang kamu cari tidak ditemukan atau sudah dihapus." |
| Error (generation gagal) | "Proses generate gagal di tahap Security Review. Kamu bisa coba ulang atau lihat detail error." |

---

## Catatan Konsistensi
Setiap route pada §1 dijelaskan komponennya di `04-frontend-architecture.md`, setiap aksi pada matrix §3.4 dijelaskan endpoint API-nya di `05-backend-architecture.md`.

**Lanjutkan ke:** `03-ui-ux-design-system.md`
