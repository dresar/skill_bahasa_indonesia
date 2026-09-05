# 10 — Deployment, Testing & Roadmap
## AI Skill Factory Indonesia

---

## 1. Local Development

```bash
git clone <repo>
cp .env.example .env.local
npm install
npm run db:migrate     # drizzle-kit migrate
npm run db:seed
npm run dev
```

`NEXT_PUBLIC_DATA_SOURCE=mock` memungkinkan frontend berjalan **tanpa database sama sekali** (Phase 1–2), memakai `MockRepository` (`04` §3).

## 2. Environment Variables

```
DATABASE_URL=
DIRECT_DATABASE_URL=
AUTH_SECRET=
AUTH_GOOGLE_ID= / AUTH_GOOGLE_SECRET=
AUTH_GITHUB_ID= / AUTH_GITHUB_SECRET=
ANTHROPIC_API_KEY=
GOOGLE_GENERATIVE_AI_API_KEY=
OPENAI_API_KEY=
GROQ_API_KEY=
BLOB_READ_WRITE_TOKEN=
NEXT_PUBLIC_DATA_SOURCE=mock|api
```

Tidak ada secret yang di-hardcode di kode maupun repository — semua lewat `.env` / Vercel Environment Variables, dicontohkan (tanpa nilai asli) di `.env.example`.

## 3. Vercel + Neon

- Setiap PR memicu Preview Deployment Vercel + **Neon branch terpisah** (isolasi data test).
- Production memakai Neon branch `main`, koneksi via driver serverless (`06` §1).
- Fungsi background generation (`05` §6) berjalan sebagai Vercel Function terpisah dari request HTTP awal.

## 4. CI/CD

```
PR dibuka
  → lint + typecheck
  → unit test (frontend + backend)
  → contract test (canonical JSON fixtures vs schema, `07`/`08`)
  → security regression test (fixture skill "jahat" harus terdeteksi, `09` §4)
  → build Next.js
  → migrasi drizzle dijalankan otomatis terhadap Neon preview branch
  → deploy preview
Merge ke main
  → migrasi dijalankan terhadap Neon `main`
  → deploy production
```

## 5. Strategi Testing (ringkasan lintas dokumen)

| Jenis | Cakupan | Detail di |
|---|---|---|
| Unit | Komponen UI murni, Service dengan repository di-mock | `04` §10, `05` §10 |
| Integration | Route Handler vs Neon test branch | `05` §10 |
| Schema/Contract | Canonical JSON fixture vs Zod/JSON Schema | `07`, `08` |
| Compiler test | JSON → file: pastikan tidak ada folder kosong dibuat, format frontmatter benar | `08` §3–4 |
| Generation pipeline test | Mock provider AI, pastikan seluruh 18 tahap tercatat & fallback bekerja | `07` |
| Security test | Fixture "jahat" per kategori risiko (`09` §4.1) wajib terdeteksi | `09` |
| Accessibility test | axe-core di halaman publik utama | `04` §10 |
| Responsive test | Playwright viewport matrix (`03` §2.4) | `04` §10 |
| E2E | Journey inti (`02` §3) end-to-end | `04` §10 |

## 6. Monitoring & Backup

- Error tracking aplikasi (mis. Sentry-kompatibel) untuk Route Handler & Client Component.
- Log generation gagal + security finding severity tinggi dikirim ke channel notifikasi admin (email/webhook — provider diabstraksi, tidak di-hardcode ke satu vendor).
- Backup/recovery mengandalkan **point-in-time recovery** bawaan Neon; kebijakan retensi & prosedur restore didokumentasikan terpisah saat provisioning production (di luar cakupan dokumen ini, dicatat sebagai TODO operasional Phase 10).

## 7. Open-Source Workflow

Berkas wajib di root repo: `README.md`, `LICENSE`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `CHANGELOG.md`, `.env.example`. `CONTRIBUTING.md` menjelaskan: cara setup lokal, konvensi commit, cara menambah skill contoh resmi (untuk MVP, tetap lewat review admin — bukan submission bebas), cara melaporkan kerentanan keamanan lewat `SECURITY.md` (bukan issue publik).

---

## 8. Roadmap Implementasi — 11 Fase

Setiap fase **wajib** memiliki acceptance criteria konkret sebelum lanjut ke fase berikutnya; fase tidak dikerjakan paralel penuh tanpa fondasi fase sebelumnya selesai.

### Phase 0 — Project Setup
- Inisialisasi Next.js + TypeScript + Tailwind + shadcn/ui, struktur folder (`04` §2), lint/format/typecheck, `.env.example`, dokumen open-source (§7).
- **AC:** `npm run dev` jalan, halaman kosong ter-render, CI lint/typecheck hijau.

### Phase 1 — Design System + Public Shell
- Implementasi token (`03` §2), komponen dasar (Button/Card/Badge/Tabs), layout publik (top nav desktop, bottom nav mobile), halaman statis (`/about`, `/roadmap`, `/changelog`).
- **AC:** Semua breakpoint (`03` §2.4) ter-review visual, dark theme konsisten, `prefers-reduced-motion` dihormati.

### Phase 2 — Skill Library + Detail
- `/skills`, `/skills/[slug]`, `/categories`, `/search` memakai `MockSkillRepository`.
- **AC:** Search, filter, pagination, tab detail (README/SKILL.md/Contoh) berfungsi penuh dengan mock data; loading/empty/error state (`02` §5) ada di semua halaman ini.

### Phase 3 — Admin Shell
- Layout `/admin`, sidebar/menu, guard route (belum RBAC penuh — placeholder auth), tabel/card list responsive.
- **AC:** Navigasi admin lengkap dapat diakses, tampilan mobile berubah jadi card list sesuai `03` §3.

### Phase 4 — Database + Auth
- Setup Neon + Drizzle, migrasi awal (`06`), seed data, Auth.js (email/password + Google + GitHub), RBAC (`05` §4).
- **AC:** Login/register berjalan, role tersimpan, `/admin/*` benar-benar terproteksi server-side (dites dengan mencoba akses tanpa role admin).

### Phase 5 — Generation UI
- `/generate/*` form + `ApiXRepository` menggantikan mock secara bertahap, Generation Timeline UI, polling status job.
- **AC:** Submit form membuat baris `generations`, progress tervisualisasi walau backend AI belum tersambung penuh (bisa pakai stub step).

### Phase 6 — AI Orchestrator
- `AiOrchestratorService` + Vercel AI SDK provider abstraction, `ai_providers` CRUD di admin, fallback chain (`05` §5, `07` §6).
- **AC:** Generate skill sungguhan end-to-end dari minimal satu provider, fallback teruji dengan mematikan provider primary secara sengaja.

### Phase 7 — Canonical JSON + Compiler
- Schema penuh (`08` §2), `ValidationService`, `CompilerService` (SKILL.md/README.md/AGENTS.md/PRD.md/ZIP).
- **AC:** Hasil generate dapat diunduh sebagai ZIP yang strukturnya sesuai `08` §3, dan dapat di-*regenerate* ulang tanpa memanggil AI (uji dengan mengubah template lalu compile ulang JSON lama).

### Phase 8 — Security + Quality
- `SecurityScanService` (`09` §4), `QualityEvalService` (`09` §5), integrasi ke pipeline (`07` step 9–10).
- **AC:** Fixture "jahat" per kategori risiko terdeteksi 100% di test suite; skor keamanan **tidak pernah** menampilkan klaim "100% aman" di UI manapun (diverifikasi manual).

### Phase 9 — Comments + Ratings + Collections
- `CommentService`, `RatingService`, `CollectionService`, moderasi admin (`09` §6), anti-abuse rating.
- **AC:** User dapat komentar/rating/bookmark/buat collection; admin dapat moderasi; constraint unique rating per user teruji.

### Phase 10 — Testing + Deployment
- Lengkapi seluruh strategi testing §5, setup CI/CD §4, deploy production Vercel+Neon, monitoring §6.
- **AC:** Seluruh acceptance criteria Phase 0–9 lolos regression test otomatis di CI; deployment production berjalan dengan environment variable lengkap (§2), tanpa secret ter-hardcode.

---

## 9. Roadmap Setelah MVP

| Fase | Fitur |
|---|---|
| Phase 2 (produk) | CLI (`npx @PROJECT_NAME/cli`, `08` §7), Public API terbatas (read-only), import skill dari GitHub repo pihak ketiga (dengan security scan wajib sebelum tampil) |
| Phase 3 (produk) | Marketplace creator + payout, community publishing dengan approval ringan, search engine eksternal (Meilisearch/Typesense) jika volume data membutuhkan |

---

## Catatan Konsistensi (Audit Akhir)

Audit internal terhadap seluruh 10 dokumen (`01`–`10`) mengonfirmasi:
- Setiap route di `02` punya komponen di `04` dan/atau menu admin di `05`/`09`.
- Setiap entitas database di `06` dikonsumsi oleh minimal satu Service di `05`.
- Setiap field canonical JSON di `08` §2 punya mapping compiler yang jelas ke file output.
- Setiap kategori security scan di `09` §4.1 punya representasi di prompt engine (`07` §3, modul 15) agar AI sudah diarahkan menghindarinya sejak generation, bukan hanya ditangkap setelahnya.
- Setiap status skill (`06`) konsisten di Badge (`03`), alur admin (`09`), dan acceptance criteria roadmap (§8).

Dokumen ini menutup rangkaian spesifikasi `00`–`10`. Untuk orientasi awal, kembali ke `00-ringkasan-eksekutif-dan-riset.md`.
