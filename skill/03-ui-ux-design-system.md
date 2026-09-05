# 03 — UI/UX & Design System
## AI Skill Factory Indonesia

---

## 1. Visual Language

Tema: **dark developer theme**, modern, premium, futuristik, tapi tetap ringan dan dapat dibaca — bukan gaya "demo WebGL berat". Public UI boleh lebih visual & animated; Admin UI lebih operasional dan dense (mirip dashboard SaaS B2B).

---

## 2. Design Tokens

### 2.1 Warna

```css
--color-background:      #0A0E12;
--color-surface:         #12171D;
--color-surface-raised:  #1A2129;
--color-border:          #232B33;
--color-text-primary:    #F3F6F8;
--color-text-secondary:  #9BA8B2;
--color-text-muted:      #6B7885;
--color-primary:         #06B6D4;  /* signature */
--color-primary-hover:   #22D3EE;
--color-success:         #34D399;
--color-warning:         #FBBF24;
--color-danger:          #F87171;
```

### 2.2 Tipografi
- Font UI: sans-serif geometris modern (mis. `Inter` / `Geist`) untuk teks, `JetBrains Mono` / `Geist Mono` untuk kode, nama file, dan cuplikan `SKILL.md`.
- Skala: `text-xs (12px)` sampai `text-4xl (36px)`, line-height longgar untuk paragraf dokumentasi (1.6–1.7).

### 2.3 Spacing, Radius, Shadow
- Spacing base 4px (`4/8/12/16/24/32/48/64`).
- Radius: `--radius-sm: 6px`, `--radius-md: 10px`, `--radius-lg: 16px` (card besar/hero).
- Shadow: gunakan seminimal mungkin (dark theme) — andalkan `border` + sedikit `glow` warna primary untuk elemen fokus, bukan drop-shadow berat.

### 2.4 Breakpoints
`360, 375, 390, 412, 768, 1024, 1280, 1440, 1920` (px). Desain di-review pada minimal 360 (Android kecil), 390 (iPhone standar), 768 (tablet), 1280 (desktop umum).

---

## 3. Layout per Konteks

| Konteks | Desktop | Mobile |
|---|---|---|
| Navigasi publik | Top nav horizontal | Bottom navigation (5 item) |
| Skill Library | Sidebar filter kiri + grid card kanan | Filter jadi bottom sheet (tombol "Filter" mengambang) |
| Skill Detail Editor (Markdown/JSON) | Split view (editor kiri, preview kanan) | Fullscreen dengan tab switch Editor/Preview |
| Admin — daftar data | Table dengan sort/kolom | Card list vertikal, aksi via swipe/menu titik-tiga |
| Command Palette | Modal tengah, `Cmd/Ctrl+K` | Full-screen sheet, dipicu tombol search di bottom nav |

---

## 4. React Bits — Pemakaian Bermakna

React Bits diinstal per-komponen (disalin ke repo via `jsrepo`/`shadcn` CLI, **bukan** dependency npm yang diimpor dari registry publik — kode menjadi bagian dari repo dan bisa dimodifikasi bebas).

Gunakan **hanya** untuk elemen yang benar-benar menambah pemahaman/kenyamanan pengguna:

| Elemen | Kapan Dipakai | Contoh Kategori React Bits |
|---|---|---|
| Animated heading di Hero | Homepage & Generator hero saja | Text animation (mis. split/reveal text) |
| Card hover feedback | Skill card di Library | Hover effect ringan (scale/border-glow) |
| Scroll reveal | Section "Kenapa AI Skill Factory" di homepage | Scroll-triggered fade/slide |
| Loading/generation progress | Generation Workspace | Custom, bukan React Bits (butuh kontrol step-state) |

**Larangan eksplisit:** jangan memasang animasi React Bits di setiap card admin, setiap baris tabel, atau elemen yang muncul berulang kali dalam jumlah besar (menyebabkan jank & re-render mahal).

---

## 5. ColorBends — Visual Signature

> ⚠️ **Catatan verifikasi (lihat `00`):** belum terverifikasi sebagai nama komponen resmi di katalog publik React Bits. Diperlakukan sebagai komponen kustom bergaya *shader gradient background*, dibangun mengikuti pola React Bits (file lokal, full props-driven, minim dependency). Tim frontend wajib mengecek ulang sebelum implementasi final.

```tsx
import { ColorBends } from '@/components/visual/ColorBends';

<ColorBends
  color="#06B6D4"
  speed={0.1}
  frequency={1.2}
  noise={0.06}
  bandWidth={0.40}
  rotation={45}
  fadeTop={0.95}
  iterations={2}
  intensity={1.1}
/>
```

**Aturan pemakaian:**
- Hanya di: Homepage hero, Generator hero (`/generate`, `/generate/skill` dst.), satu CTA khusus (mis. penutup homepage "Mulai Bangun Skill Pertamamu"), dan empty-state tertentu yang memang dirancang premium (bukan semua empty state).
- **Dilarang** dipasang di seluruh halaman — merusak keterbacaan teks dan performa scroll.
- Wajib overlay gradasi gelap (`background: linear-gradient(...)` dengan opacity) di atas ColorBends sebelum teks diletakkan, untuk menjaga kontras WCAG AA.
- Render sebagai `<canvas>`/WebGL di client component saja (`"use client"`), **lazy-loaded** dan dimatikan otomatis jika `prefers-reduced-motion: reduce`.

---

## 6. Animasi

| Elemen | Jenis Animasi | Catatan |
|---|---|---|
| Page transition | Fade tipis (150–200ms) | Hindari transisi besar yang menunda interaksi |
| Card hover | Scale 1.01–1.02 + border glow | |
| Button feedback | Scale-down saat pressed | |
| Modal/Drawer | Slide + fade | |
| Skeleton loading | Shimmer halus | Dipakai di semua list yang fetch data |
| Generation progress | Step-by-step reveal, bukan progress bar generik | Menunjukkan tahap AI Factory yang sedang berjalan |
| Toast (copy/bookmark/error) | Slide-in dari bawah (mobile) / kanan-atas (desktop) | |

Semua animasi **wajib** menghormati `prefers-reduced-motion` (fallback: instan tanpa transisi). Gunakan CSS transition/`Motion` (library animasi React modern) — hindari WebGL kecuali untuk ColorBends yang memang shader background.

---

## 7. Component System (ringkas)

Dibangun di atas **shadcn/ui + Radix UI** sebagai primitives (accessible by default), di-skin dengan token §2:

`Button` (primary/secondary/ghost/danger) · `Badge` (status: Draft/Review/Approved/Published/Archived/Rejected, tiap status warna berbeda) · `Card` (skill/prd/workflow/agent-kit — layout sama, konten beda) · `Tabs` (dipakai di Skill Detail & Admin) · `DataTable` (admin, dengan versi Card List untuk mobile) · `CommandPalette` · `Dialog/Sheet` (drawer mobile) · `Toast` · `RatingStars` · `MonacoJSONEditor` (wrapper) · `MarkdownEditor` (wrapper Editor/Preview/Split) · `SecurityScoreBadge`, `QualityScoreBadge` (visual skor dengan warna semantik) · `EmptyState`, `ErrorState` (reusable, menerima ilustrasi + copy + CTA opsional)

---

## 8. Accessibility

- Semua interaktif dapat dicapai via keyboard (`Tab`, `Enter`, `Esc` menutup modal).
- Fokus terlihat jelas (`outline` primary color, bukan dihilangkan).
- Kontras teks minimal AA (4.5:1) — khusus di atas ColorBends, wajib diuji manual.
- Semantic HTML (`<nav>`, `<main>`, `<article>` untuk skill detail, heading hierarchy benar).
- ARIA label untuk ikon-only button (mis. tombol bookmark, copy).
- Screen reader: status generation (step aktif) diumumkan via `aria-live="polite"`.

---

## 9. States (per Component, Contoh Konkret)

| Komponen | Loading | Empty | Error | Success |
|---|---|---|---|---|
| Skill Library | Skeleton grid 6 card | "Belum ada skill yang cocok..." + reset filter | "Gagal memuat skill. Coba lagi." + tombol retry | — |
| Generator | Timeline step dengan indikator berjalan | — | "Proses generate gagal di tahap X" + detail + retry | Confetti/toast halus "Skill berhasil dibuat" |
| Comment form | Tombol disabled + spinner kecil | "Jadi yang pertama berkomentar" | "Komentar gagal dikirim, coba lagi" | Toast "Komentar terkirim" |

---

## Catatan Konsistensi
Token pada dokumen ini dipakai literal oleh implementasi Tailwind config di `04-frontend-architecture.md`. Status Draft/Review/Approved/Published/Archived/Rejected pada `Badge` harus sama persis dengan enum status di `06-database-neon.md` dan alur di `09-admin-security-operations.md`.

**Lanjutkan ke:** `04-frontend-architecture.md`
