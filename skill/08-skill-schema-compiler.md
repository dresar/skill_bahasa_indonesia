# 08 — Canonical JSON Schema & Compiler
## AI Skill Factory Indonesia

---

## 1. Prinsip

Canonical JSON adalah **satu-satunya source of truth** untuk artifact yang dihasilkan (skill, PRD, workflow, agent kit). Compiler bersifat **deterministik** — jika template berubah, seluruh file dapat di-*regenerate* dari JSON tersimpan tanpa memanggil AI ulang.

```
Canonical JSON (tervalidasi)  →  Template Engine  →  File Final (.md/.json/.zip)
```

---

## 2. Canonical JSON Schema — Skill (production-ready)

`schemaVersion` di-bump setiap ada perubahan struktur; compiler & validator harus **tolerant terhadap field baru yang belum dikenal** (menyimpannya di `metadata.extra` alih-alih menolak) karena standar Agent Skills eksternal masih berkembang aktif.

```json
{
  "schemaVersion": "1.0.0",
  "artifactType": "skill",
  "skill": {
    "name": "nextjs-error-boundary-setup",
    "slug": "nextjs-error-boundary-setup",
    "version": "1.0.0",
    "description": "Gunakan skill ini saat perlu menyiapkan error boundary dan halaman error kustom di aplikasi Next.js App Router. Cocok dipakai ketika pengguna menyebut 'error.tsx', 'global-error', atau menangani exception tak tertangani di route tertentu.",
    "language": "id-ID",
    "license": "MIT",
    "category": "framework-nextjs",
    "tags": ["nextjs", "error-handling", "app-router"],
    "compatibility": [
      { "agentSlug": "claude-code", "status": "verified" },
      { "agentSlug": "cursor", "status": "verified" },
      { "agentSlug": "codex-cli", "status": "verified" },
      { "agentSlug": "gemini-cli", "status": "likely" },
      { "agentSlug": "antigravity", "status": "verified" }
    ]
  },
  "purpose": {
    "summary": "Menyiapkan error.tsx dan global-error.tsx yang konsisten dengan pola logging proyek.",
    "problemSolved": "Developer sering lupa menangani error boundary per-segment di App Router."
  },
  "triggers": [
    "Pengguna menyebut error.tsx atau global-error.tsx",
    "Pengguna meminta menangani unhandled exception di route Next.js"
  ],
  "nonTriggers": [
    "Penanganan error di luar Next.js App Router",
    "Validasi input form (bukan tanggung jawab skill ini)"
  ],
  "instructions": [
    "Identifikasi apakah proyek memakai App Router sebelum membuat file",
    "Buat error.tsx di segmen yang relevan, bukan hanya di root"
  ],
  "rules": [
    "Selalu tandai error.tsx dengan 'use client'",
    "Jangan menangkap error yang seharusnya ditangani notFound()"
  ],
  "boundaries": {
    "inScope": ["error.tsx", "global-error.tsx", "logging dasar"],
    "outOfScope": ["Sentry/observability setup penuh (skill terpisah)"]
  },
  "workflow": [
    { "step": 1, "action": "Deteksi struktur app/ proyek" },
    { "step": 2, "action": "Buat error.tsx sesuai konvensi proyek" }
  ],
  "examples": [
    { "title": "Error boundary dasar", "input": "...", "output": "..." }
  ],
  "tests": {
    "evalsRef": "evals/evals.json"
  },
  "scripts": [],
  "references": [],
  "resources": [],
  "templates": [],
  "compatibility": {
    "universal": true,
    "agentNotes": {
      "gemini-cli": "Butuh consent prompt saat aktivasi (preview feature)."
    }
  },
  "security": {
    "scanStatus": "pemeriksaan-otomatis",
    "score": 96,
    "findings": []
  },
  "quality": {
    "score": 88,
    "breakdown": { "purposeClarity": 9, "triggerQuality": 8, "instructionQuality": 9 }
  },
  "research": {
    "conducted": true,
    "sources": [
      { "url": "https://nextjs.org/docs/app/building-your-application/routing/error-handling", "checkedAt": "2026-08-08" }
    ]
  },
  "provenance": {
    "generatedBy": { "provider": "anthropic", "model": "..." },
    "pipelineVersion": "1.0",
    "needsVerification": []
  },
  "metadata": { "extra": {} }
}
```

Schema serupa (versi lebih sederhana) berlaku untuk `artifactType: "prd" | "workflow" | "agentKit"`, dengan blok tambahan spesifik (mis. PRD punya `sections[]`, Agent Kit punya `stack{ frontend, backend, database, auth, payment, deployment, testing }`).

---

## 3. Struktur Folder Skill (Output Compiler)

Mengikuti **Agent Skills open standard** (agentskills.io) secara verbatim untuk kompatibilitas lintas-agent, ditambah `evals/` mengikuti konvensi Anthropic untuk behavioral evaluation:

```
nextjs-error-boundary-setup/
├── SKILL.md              # wajib: YAML frontmatter (name, description, ...) + instruksi Markdown
├── scripts/               # opsional — hanya jika ada eksekusi nyata dibutuhkan
├── references/            # opsional — detail panjang dipindah ke sini (progressive disclosure)
├── assets/                # opsional — template/resource
└── evals/
    └── evals.json          # test case Trigger/Compliance/Boundary
```

**Aturan compiler:** folder yang isinya kosong **tidak dibuat sama sekali** — mengikuti prinsip "jika tidak dibutuhkan, jangan dibuat" dari brief.

### Contoh `evals/evals.json`

```json
{
  "evals": [
    {
      "scenario": "User bertanya cara menangani error di route /dashboard",
      "expectedSkillUsage": "digunakan",
      "expectedBehavior": "Membuat error.tsx di app/dashboard/, memakai 'use client'",
      "forbiddenBehavior": "Membuat error handling di luar App Router / menyarankan library eksternal tanpa diminta",
      "verification": "Manual review + automated lint file yang dihasilkan"
    }
  ]
}
```

---

## 4. Compiler → Output Mapping

| Sumber (Canonical JSON) | Output File | Template Engine |
|---|---|---|
| `skill.*`, `purpose`, `triggers`, `instructions`, `rules`, `boundaries`, `workflow` | `SKILL.md` (frontmatter + body) | Handlebars/EJS-style template, deterministik |
| `skill.description`, `examples`, `compatibility` | `README.md` | Template berbeda dari `SKILL.md` (README lebih naratif utk manusia) |
| `compatibility.agentNotes`, tabel adapter | `AGENTS.md` | Menjelaskan cara pakai per agent |
| Seluruh dokumen PRD generator | `PRD.md` | Template PRD terpisah |
| `examples[]` | `examples/*.md` (jika perlu file terpisah) | |
| `scripts[]` | `scripts/*.sh` / `*.py` / `*.js` | Disertai header komentar: tujuan, input, output, permission, dependency |
| `references[]` | `references/*.md` | |
| `resources[]`/`templates[]` | `assets/*` | |
| `tests.evalsRef` | `evals/evals.json` | |
| Seluruh folder skill | `.zip` | Dibuat setelah semua file siap, disimpan via `StorageProvider` |

**Sifat kunci:** jika hanya template README yang berubah (bukan isi JSON), backend cukup memanggil ulang `CompilerService.compile(canonicalJson)` — **tanpa** memanggil AI sama sekali.

---

## 5. Universal Skill vs Agent Adapter

| Lapisan | Isi | Contoh |
|---|---|---|
| **Universal Skill** | `SKILL.md` + folder skill — identik untuk semua agent karena mengikuti standar terbuka yang sama | Nama, description, instructions, boundaries |
| **Agent Adapter** | Metadata instalasi per agent — lokasi folder, cara invoke, catatan kompatibilitas | Tabel di bawah |

### Compatibility Matrix (agent adapter — lihat provenance riset di `00`)

| Agent | Lokasi Global | Lokasi Project | Cara Invoke | Catatan Adapter |
|---|---|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` | Auto-discover + manual `/` | Native, distribusi via plugin juga didukung |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` | Auto-discover + manual `/` | Cross-read folder Claude Code |
| Codex CLI | `~/.codex/skills/` | `.codex/skills/` | Auto-discover | Cross-read folder Claude Code |
| Gemini CLI | `~/.gemini/skills/` | `.gemini/skills/` | Auto-discover + **consent prompt** | Status preview; **legacy adapter** — akses konsumen dihentikan Google 18 Juni 2026 |
| Antigravity | Bervariasi per varian (desktop/IDE/CLI) | `.agents/skills/` | Auto-discover, terintegrasi Manager View (role-based skill assignment) | Native sejak awal 2026 |

Data ini disimpan di tabel `agents` (`06`) dan ditampilkan sebagai badge compatibility di Skill Detail (`02`/`03`).

---

## 6. Packaging & Versioning

- **ZIP**: dibuat deterministik dari folder hasil compile, nama file `{slug}-v{version}.zip`.
- **Semantic Versioning**: `MAJOR.MINOR.PATCH`. PATCH = perbaikan kecil tanpa ubah perilaku; MINOR = penambahan kapabilitas kompatibel-mundur; MAJOR = perubahan boundary/instruksi yang memengaruhi cara agent memakai skill.
- **Snapshot**: setiap versi tersimpan penuh di `skill_versions.canonical_json` — versi lama tidak pernah ditimpa, hanya `current_version_id` yang berpindah.
- **Changelog**: field wajib diisi admin/AI saat membuat versi baru, ditampilkan di tab "Changelog" (`02`).

---

## 7. CLI Masa Depan (arsitektur, bukan implementasi MVP)

```
npx @PROJECT_NAME/cli skill init
npx @PROJECT_NAME/cli skill generate
npx @PROJECT_NAME/cli skill validate
npx @PROJECT_NAME/cli skill build     # = CompilerService, dijalankan lokal
npx @PROJECT_NAME/cli skill install   # salin ke folder agent yang terdeteksi (mirip pola `npx skills i`)
```

CLI **wajib** memakai `packages/schema` dan `packages/compiler` yang sama dengan website (bukan reimplementasi terpisah) — inilah alasan `packages/` dipisah dari `app/` sejak awal di `04-frontend-architecture.md` §2, meski MVP hanya memakainya lewat website.

---

## Catatan Konsistensi
Schema §2 harus sama persis dengan yang dipakai `generateObject` di `07-ai-generation-engine.md` §5. Compatibility matrix §5 harus sinkron dengan seed data tabel `agents` di `06-database-neon.md` §6.

**Lanjutkan ke:** `09-admin-security-operations.md`
