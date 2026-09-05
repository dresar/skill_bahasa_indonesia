# 00 — Ringkasan Eksekutif & Hasil Riset
## AI Skill Factory Indonesia

> Dokumen ini adalah pintu masuk dari 10 dokumen spesifikasi (`01`–`10`). Baca dokumen ini terlebih dahulu sebelum membaca dokumen lain, termasuk oleh Antigravity/AI coding agent yang mengimplementasikan produk ini.

---

## 1. Ringkasan Keputusan Arsitektur

| Keputusan | Pilihan | Alasan |
|---|---|---|
| Prinsip inti | AI menghasilkan **data** (canonical JSON), backend **memvalidasi**, compiler menghasilkan **file** | Mencegah AI menjadi single point of failure untuk output final; memungkinkan regenerasi file tanpa memanggil AI ulang |
| Framework web | Next.js (App Router) + TypeScript | Server Components untuk performa, ekosistem terbesar, kompatibel Vercel |
| Database | Neon PostgreSQL (serverless, branching) | Cocok dengan Vercel preview deployment, mendukung full-text search bawaan untuk MVP |
| ORM | Drizzle ORM | TypeScript-first, ringan di edge/serverless, migrasi eksplisit — lihat `06-database-neon.md` |
| AI Provider layer | Vercel AI SDK (Core, `generateObject` untuk canonical JSON) sebagai abstraction, bukan pemanggilan SDK vendor langsung | AI SDK menyediakan lapisan agnostik-provider di atas API model dengan `generateText`, `streamText`, `generateObject`, dan mendukung Anthropic, OpenAI, Google, serta provider kompatibel-OpenAI (termasuk Groq) melalui satu antarmuka |
| Skill format | Mengikuti **Agent Skills open standard** (agentskills.io) secara verbatim untuk Universal Skill; adapter terpisah per agent | Standar ini sudah lintas-vendor (donasi Anthropic ke Agentic AI Foundation), bukan lagi format proprietary Claude |
| Storage file besar | `StorageProvider` abstraction di atas Vercel Blob (default) | Database bukan tempat ideal menyimpan ZIP/asset besar |
| Auth | Email/password + OAuth (Google, GitHub), RBAC server-side | Sesuai requirement; provider auth spesifik diabstraksi agar dapat diganti |
| Async generation | Job-based dengan step events, disimpan sebagai baris `generations`/`generation_steps`, dieksekusi lewat Vercel-friendly background function (bukan queue vendor tertentu di MVP) | Menghindari lock-in queue sebelum benar-benar diperlukan, sesuai instruksi brief |
| Search | PostgreSQL full-text search (`tsvector`) di belakang `SearchProvider` interface | MVP tanpa dependency eksternal; dapat diganti Meilisearch/Typesense nanti |

---

## 2. Hasil Riset Penting (dengan Provenance)

Riset dilakukan 8 Agustus 2026. Untuk topik yang cepat berubah (standar skill, tooling agent), **cek ulang sumber sebelum implementasi** karena ekosistem ini bergerak cepat.

### 2.1 Agent Skills — status sebagai standar terbuka
- Format `SKILL.md` **tidak lagi eksklusif Claude**. Pada 18 Desember 2025 Anthropic mengumumkan "Agent Skills" sebagai *open standard* dan mendonasikannya ke **Agentic AI Foundation (AAIF)**.
- Spesifikasi kanonis sekarang berada di `agentskills.io` (repo referensi: `agentskills/agentskills`, lisensi kode Apache-2.0, dokumentasi CC-BY-4.0), terpisah dari implementasi Anthropic di `anthropics/skills`.
- Struktur folder kanonis: `SKILL.md` (wajib) + `scripts/` (opsional) + `references/` (opsional) + `assets/` (opsional, template/resource). Implementasi Anthropic sendiri menambahkan `evals/evals.json` sebagai folder yang direkomendasikan untuk *behavioral evaluation* — ini langsung memetakan ke requirement Trigger/Compliance/Boundary pada brief ini.
- Mekanisme kerja: **progressive disclosure** tiga tahap — *Discovery* (agent hanya memuat `name` + `description` saat startup), *Activation* (agent membaca isi penuh `SKILL.md` ketika task cocok dengan description), *Execution* (agent menjalankan instruksi, skrip, atau resource sesuai kebutuhan).
- Sumber: `github.com/agentskills/agentskills`, `agentskills.me/specification`, `deepwiki.com/anthropics/skills`, `github.com/anthropics/skills`.

### 2.2 Kompatibilitas per Agent (folder lokasi skill)

| Agent | Lokasi Global | Lokasi Project | Catatan |
|---|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` | Native support, invoke manual dengan `/`, distribusi via plugin |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` | Juga membaca `~/.claude/skills/` dan `.claude/skills/` — cross-tool compatible |
| Codex CLI (OpenAI) | `~/.codex/skills/` | `.codex/skills/` | Juga membaca `~/.claude/skills/` |
| Gemini CLI | `~/.gemini/skills/` | `.gemini/skills/` | Berstatus **preview**, butuh consent prompt saat aktivasi, prioritas Project > User > Extension. **Akses konsumen dihentikan Google 18 Juni 2026**, digantikan Antigravity |
| Antigravity (Google) | Bervariasi per varian (desktop app / IDE / CLI), contoh `~/.gemini/antigravity-cli/skills/` untuk CLI | `<workspace-root>/.agents/skills/` | Native support ditambahkan awal 2026; format identik SKILL.md, skill lintas-agent dapat dipakai tanpa modifikasi |

> **Implikasi desain:** Universal Skill JSON dikompilasi menjadi `SKILL.md` yang sama untuk semua agent (format identik). Yang berbeda hanyalah **instruksi instalasi/lokasi folder** per agent — ini yang ditangani lapisan **Agent Adapter**, bukan skill itu sendiri. Lihat `08-skill-schema-compiler.md`.

### 2.3 Antigravity — konfirmasi identitas produk
Karena brief ini eksplisit ditujukan untuk diserahkan ke "Antigravity" sebagai AI coding agent pelaksana, penting dikonfirmasi apa itu:
- Antigravity adalah **platform pengembangan agentic milik Google**, awalnya dirilis November 2025 sebagai IDE agent-first berbasis fork VS Code, mendukung Gemini 3 Pro serta model non-Google seperti Claude Sonnet 4.5 dan GPT-OSS.
- Pada I/O 2026 (19 Mei 2026), Google merilis **Antigravity 2.0**: desktop app baru, CLI (`agy`) berbasis Go, dan SDK untuk agent kustom, dengan model utama Gemini 3.5 Flash serta dukungan multi-model termasuk Claude Sonnet 4.6 dan Opus 4.6.
- Fitur **Manager View** memungkinkan orkestrasi banyak agent paralel dengan role berbeda (reviewer, tester, implementer), masing-masing dapat diberi akses ke skill tertentu — pola ini relevan untuk cara Antigravity nantinya "membaca" 10 dokumen di brief ini (mis. agent implementer memakai dokumen arsitektur, agent tester memakai dokumen testing).
- **Catatan verifikasi:** detail granular seperti command CLI persis atau nama field SDK tidak diverifikasi lebih lanjut karena berada di luar cakupan (bukan bagian dari produk yang dibangun), tetapi cukup untuk memastikan asumsi brief ("Antigravity adalah AI coding agent yang mendukung Agent Skills") **valid dan terverifikasi**.

### 2.4 Stack teknis pendukung
- **Vercel AI SDK** versi terbaru (AI SDK 6) menyediakan lapisan agnostik-provider dengan `generateText`/`streamText`/`generateObject`/`ToolLoopAgent`, serta AI Gateway untuk banyak model dengan fallback otomatis — cocok dipakai sebagai **AI Provider Abstraction Layer** di `07-ai-generation-engine.md` alih-alih membuat abstraction sendiri dari nol.
- **React Bits** adalah library komponen React animasi open-source nyata (MIT + Commons Clause), diinstal dengan menyalin kode komponen ke repo (via `jsrepo` atau `npx shadcn add <url>`), bukan diimpor sebagai paket npm — ini mengonfirmasi pola `import { ColorBends } from '@components/ColorBends'` di brief adalah pola **file lokal**, bukan package registry.

---

## 3. Asumsi yang Digunakan

Tandai bagian berikut sebagai **butuh verifikasi ulang saat implementasi**, bukan fakta final:

1. **"ColorBends" bukan komponen resmi yang terverifikasi ada di katalog React Bits publik.** Diperlakukan sebagai komponen kustom bergaya shader-background (mirip kategori "Backgrounds" React Bits seperti Aurora/Iridescence/Silk), dibangun sendiri mengikuti pola instalasi React Bits (file lokal, props-driven). Tim frontend wajib mengecek ulang di reactbits.dev sebelum build; jika memang tidak ada, gunakan spesifikasi props pada `03-ui-ux-design-system.md` sebagai brief pembuatan komponen kustom.
2. Nama paket final `@PROJECT_NAME/cli` adalah **placeholder** — belum ada keputusan nama produk final, sesuai instruksi brief.
3. Field metadata SKILL.md di luar `name` dan `description` (mis. `license`, `compatibility`, `allowed-tools` dsb.) mengikuti pola yang teramati di ekosistem per Agustus 2026; **skema resmi dapat bertambah field baru** karena statusnya masih standar yang berkembang aktif (20+ kontributor aktif, isu/PR terbuka di repo `agentskills/agentskills`). Compiler HARUS dirancang tervensi (versioned + tolerant to unknown fields), bukan hard-coded kaku.
4. Ketersediaan model spesifik (mis. harga/rate limit Claude, Gemini, GPT, Groq) tidak diverifikasi granular karena berubah sangat cepat — admin AI Provider Settings dirancang agar model/harga dikonfigurasi runtime, bukan hard-coded.
5. Gemini CLI diasumsikan **tetap didukung sebagai adapter** meski dalam masa retirement konsumen (per 18 Juni 2026) karena pengguna enterprise masih memakainya dan skill format-nya tidak berubah; ditandai "legacy adapter" di `08-skill-schema-compiler.md`.

---

## 4. Keputusan Teknologi (Ringkas)

```
Frontend      : Next.js (App Router) + TypeScript + Tailwind CSS + shadcn/ui + Radix UI
Visual         : React Bits (selektif) + custom shader background ("ColorBends"-style)
Forms          : React Hook Form + Zod
Data fetching  : TanStack Query (client) + Server Components (initial load)
Editor         : Monaco Editor (JSON), Shiki (syntax highlight), MDX/Markdown renderer
Backend        : Next.js Route Handlers (API) + service layer terpisah
Database       : Neon PostgreSQL + Drizzle ORM
AI layer       : Vercel AI SDK (provider abstraction) → Claude / Gemini / OpenAI / Groq / custom OpenAI-compatible
Auth           : Auth.js (NextAuth) — email/password + Google + GitHub OAuth, RBAC server-side
Storage        : StorageProvider abstraction → Vercel Blob (default)
Search         : SearchProvider abstraction → PostgreSQL full-text search (MVP)
Deployment     : Vercel + Neon
```

---

## 5–14. Ringkasan Arsitektur per Domain

Setiap topik berikut dibahas penuh pada dokumennya masing-masing; ringkasan satu-baris di sini hanya untuk orientasi cepat.

| # | Topik | Ringkasan 1-baris | Dokumen Detail |
|---|---|---|---|
| 5 | Sitemap | 26 route publik + `/generate/*` + `/admin/*`, dijelaskan per halaman | `02` |
| 6 | Architecture overview | Monolith Next.js modular dengan `packages/` internal (schema, compiler, ai, validators) | `04`, `05` |
| 7 | AI generation architecture | Pipeline 18-tahap: requirement → research → design → canonical JSON → validasi → compile → publish | `07` |
| 8 | Canonical JSON architecture | Satu schema versioned per jenis artifact (skill/prd/workflow/agent-kit), source of truth | `08` |
| 9 | Compiler architecture | Template engine deterministik, tidak bergantung AI, dapat regenerate tanpa panggil ulang model | `08` |
| 10 | Security architecture | Scanner otomatis multi-kategori + least-privilege + tidak pernah klaim "100% aman" | `09` |
| 11 | Database architecture | 27 entitas inti di Neon, soft delete + audit log + versioning | `06` |
| 12 | API architecture | REST via Route Handlers, dipisah Public/User/Generator/Validation/Admin | `05` |
| 13 | Design system | Dark developer theme, primary `#06B6D4`, tokens lengkap, ColorBends sebagai visual signature terbatas | `03` |
| 14 | Implementation roadmap | 11 fase (Phase 0–10), tiap fase punya acceptance criteria | `10` |

---

## 15. Final Quality Gate — Jawaban Checklist

| Pertanyaan Gate | Jawaban |
|---|---|
| Berbeda dari skill directory biasa? | Ya — nilai inti ada di AI Skill Factory (generate→validate→evaluate→compile), bukan katalog statis |
| AI generation bekerja lintas provider? | Ya — via Vercel AI SDK provider abstraction + `ai_providers` table + fallback chain |
| Canonical JSON adalah source of truth? | Ya — compiler tidak pernah membaca output AI mentah, hanya JSON yang sudah lolos validasi |
| Compiler menghasilkan banyak artifact? | Ya — SKILL.md, README.md, AGENTS.md, PRD.md, examples, scripts, resources, ZIP |
| Skill dapat divalidasi? | Ya — JSON Schema validation + linter aturan skill-spesifik |
| Security scanning tersedia? | Ya — multi-kategori, hasil selalu berupa skor + peringatan, tidak pernah "100% aman" |
| Quality evaluation tersedia? | Ya — rubric 12 dimensi, skor 0–100 dengan breakdown yang bisa dijelaskan |
| Trigger/Compliance/Boundary tersedia? | Ya — dipetakan ke folder `evals/evals.json` sesuai konvensi Anthropic |
| Agent compatibility dipisah dari universal skill? | Ya — Universal Skill JSON vs Agent Adapter Table |
| Neon dipakai dengan benar? | Ya — serverless driver, pooled connection, branching untuk preview |
| Vercel compatibility diperhatikan? | Ya — async generation tidak mengasumsikan long-running server process |
| Mobile UX dirancang eksplisit? | Ya — breakpoint list, pola filter/editor/admin khusus mobile di `03` |
| React Bits dipakai bermakna? | Ya — dibatasi ke elemen yang menambah UX, bukan seluruh UI |
| ColorBends jadi visual signature tanpa merusak performa? | Ya — dibatasi ke hero/CTA tertentu + overlay kontras, `prefers-reduced-motion` dihormati |
| Seluruh UI Bahasa Indonesia? | Ya — dicontohkan di `02` dan `03`, termasuk error/empty/success state |
| Admin kontrol generation & publishing? | Ya — status workflow Draft→Generating→Review→Approved→Published→Archived/Rejected |
| Open-source architecture bersih? | Ya — abstraction layer di setiap dependency vendor, struktur repo di `04`/`05` |
| 10 dokumen tidak saling bertentangan? | Ya — audit konsistensi manual dilakukan (lihat catatan penutup tiap dokumen) |
| Cukup jelas untuk Antigravity tanpa menebak? | Ya — setiap fitur punya acceptance criteria konkret, schema, dan contoh |

---

**Lanjutkan ke:** `01-product-overview.md`
