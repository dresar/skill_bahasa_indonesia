# 04 — Frontend Architecture
## AI Skill Factory Indonesia

---

## 1. Stack & Prinsip

Next.js (App Router) + TypeScript, Server Components sebagai default, Client Components hanya untuk yang butuh interaktivitas (form, editor, animasi, generation progress). Prinsip: **minimal client JS**, data awal selalu di-fetch di server ketika memungkinkan.

---

## 2. Struktur Proyek

```
src/
  app/
    (public)/
      page.tsx                     → /
      skills/page.tsx              → /skills
      skills/[slug]/page.tsx       → /skills/[slug]
      categories/...
      prds/...
      workflows/...
      agent-kits/...
      templates/...
      search/page.tsx
      collections/...
      generate/
        page.tsx                   → /generate (hub)
        skill/page.tsx
        prd/page.tsx
        workflow/page.tsx
        agent-kit/page.tsx
      docs/[slug]/page.tsx
      changelog/page.tsx
      roadmap/page.tsx
      about/page.tsx
    (auth)/
      login/page.tsx
      register/page.tsx
    (account)/
      profile/page.tsx
      settings/page.tsx
    admin/
      layout.tsx                   → guard RBAC di sini
      page.tsx                     → dashboard
      skills/...
      generator/...
      ai-providers/...
      ...
    api/
      skills/route.ts
      generate/skill/route.ts
      ...
  components/
    ui/                            → shadcn primitives + wrapper
    visual/                        → ColorBends, React Bits wrappers
    generator/                     → GenerationTimeline, JobStatus, CanonicalJsonPreview
    editor/                        → MonacoJSONEditor, MarkdownEditor
    admin/                         → DataTable, AdminCardList, AuditLogRow
  features/
    skills/                        → hooks + components spesifik domain skill
    generation/
    admin/
    auth/
  repositories/
    interfaces/                    → SkillRepository, PrdRepository, dst. (interface saja)
    mock/                          → MockSkillRepository, dst.
    api/                           → ApiSkillRepository, dst. (fetch ke /api/*)
    index.ts                       → factory: pilih mock/api berdasar env
  schemas/                         → Zod schema (form + shared dgn backend via packages/schema)
  hooks/
  lib/                             → utils, fetcher, formatters
  types/
packages/
  schema/                          → canonical JSON schema + Zod, dipakai FE & BE
  compiler/                        → (dipakai backend, tapi tipe di-share ke FE untuk preview)
  ai/
  validators/
  ui/                              → design tokens mentah (jika dipisah dari app)
```

Jika kompleksitas monorepo dirasa tidak perlu di fase awal, `packages/schema` dan `packages/compiler` boleh sementara jadi folder biasa di `src/lib/` — **jangan memaksakan monorepo tooling (Turborepo dll.) sebelum benar-benar diperlukan** (Phase 0, lihat `10`).

---

## 3. Repository Abstraction & Mock Data

Prinsip: UI tidak pernah memanggil `fetch` langsung — selalu lewat repository interface, agar bisa berjalan **tanpa backend** di fase awal.

```ts
// repositories/interfaces/skill-repository.ts
export interface SkillRepository {
  list(params: SkillListParams): Promise<PaginatedResult<SkillSummary>>;
  getBySlug(slug: string): Promise<SkillDetail | null>;
  search(query: string): Promise<SkillSummary[]>;
}

// repositories/mock/mock-skill-repository.ts
export class MockSkillRepository implements SkillRepository { /* baca dari fixtures JSON lokal */ }

// repositories/api/api-skill-repository.ts
export class ApiSkillRepository implements SkillRepository { /* fetch ke /api/skills */ }

// repositories/index.ts
export const skillRepository: SkillRepository =
  process.env.NEXT_PUBLIC_DATA_SOURCE === 'mock'
    ? new MockSkillRepository()
    : new ApiSkillRepository();
```

Repository yang wajib ada di MVP: `SkillRepository`, `CategoryRepository`, `AgentRepository`, `PrdRepository`, `WorkflowRepository`, `AgentKitRepository`, `CommentRepository`, `RatingRepository`, `CollectionRepository`, `UserRepository`, `AnalyticsRepository`, `GenerationRepository`.

Mock data disiapkan sebagai fixture JSON di `src/repositories/mock/fixtures/` (skills, categories, agents, prds, workflows, agent-kits, comments, ratings, users, analytics, generation-runs) — cukup realistis agar frontend dapat dibangun & di-demo penuh sebelum backend siap (Phase 1–2 di roadmap).

---

## 4. Forms & Validasi

- **React Hook Form** untuk semua form (Generator, Comment, Auth, Settings).
- **Zod** sebagai satu-satunya sumber validasi, schema di-share sedapat mungkin dengan backend (`packages/schema`) agar aturan tidak dobel-tulis.
- Error message form: Bahasa Indonesia, spesifik per field (contoh: "Nama skill wajib diisi, minimal 3 karakter" bukan "Invalid input").

---

## 5. State Management

- **Server state** (data dari API/DB): TanStack Query — caching, revalidation, optimistic update untuk aksi ringan (bookmark, upvote).
- **UI state lokal**: `useState`/`useReducer` React biasa — tidak perlu Redux/Zustand untuk MVP kecuali generation workspace admin ternyata butuh state lintas-komponen kompleks (evaluasi ulang di Phase 5).
- **Generation job state**: polling TanStack Query (interval pendek) ke endpoint status job, ditambah opsi upgrade ke Server-Sent Events jika volume generation admin tinggi (lihat `05` §6 Async Generation).

---

## 6. Editors

### Monaco Editor (JSON canonical)
- Syntax highlight JSON, **schema validation** langsung di editor (pakai JSON Schema dari `packages/schema`, via `monaco-jsonschema` pattern).
- Fitur: format, copy, save, diff (versus versi sebelumnya), reset ke hasil AI asli.

### Markdown/MDX Editor (`SKILL.md`, `README.md`, `PRD.md`, `AGENTS.md`)
- Mode Editor / Preview / Split.
- Syntax highlighting via **Shiki**.
- Preview merender Markdown+frontmatter persis seperti akan tampil di file akhir.

---

## 7. API Integration Layer

`ApiXRepository` memanggil Route Handler internal (`/api/*`), **bukan** memanggil provider AI langsung dari client — kunci API tidak pernah menyentuh browser. Semua request admin menyertakan session cookie (Auth.js), divalidasi ulang di server (lihat `05` §4 Authorization).

---

## 8. Performance

- Server Components untuk semua halaman list/detail non-interaktif (Skill Library, Detail, Docs).
- `next/dynamic` untuk: Monaco Editor, ColorBends (WebGL), Command Palette — semua di-lazy-load, tidak masuk initial bundle.
- Image via `next/image`, format modern (AVIF/WebP) otomatis.
- Pagination di semua list (bukan infinite fetch tanpa batas) + **virtualization** untuk daftar admin yang sangat panjang (mis. Audit Logs).
- Caching: Server Components pakai `fetch` cache/revalidate Next.js; data publik (skill list) di-revalidate berkala (ISR), data admin selalu fresh (`no-store`).

---

## 9. SEO

- `generateMetadata` dinamis per skill/prd/workflow (title, description dari canonical JSON).
- Open Graph image otomatis (bisa generate via `next/og` dari template konsisten).
- `sitemap.ts` & `robots.ts` bawaan Next.js, mengecualikan seluruh `/admin/*`.
- JSON-LD `SoftwareSourceCode`/`TechArticle` schema pada halaman skill detail.

---

## 10. Testing (Frontend)

- **Unit**: komponen murni (Badge, ScoreBadge, format util) — Vitest + React Testing Library.
- **Integration**: form Generator (submit → mock repository → render progress).
- **E2E**: Playwright — journey inti dari `02` §3.1–3.3 (cari skill, unduh, generate skill dasar, admin publish).
- **Accessibility test**: axe-core otomatis di CI untuk halaman publik utama.
- **Responsive test**: Playwright viewport matrix mengikuti breakpoint `03` §2.4.

---

## Catatan Konsistensi
Setiap repository interface di §3 punya endpoint pasangan di `05-backend-architecture.md` §3 (API). Setiap komponen editor di §6 memakai schema yang didefinisikan di `08-skill-schema-compiler.md`.

**Lanjutkan ke:** `05-backend-architecture.md`
