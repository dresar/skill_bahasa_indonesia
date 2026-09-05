# 05 — Backend Architecture
## AI Skill Factory Indonesia

---

## 1. Prinsip

Backend berjalan sebagai **Next.js Route Handlers** (`src/app/api/**/route.ts`) di atas **service layer** yang terpisah dari HTTP concern, agar service dapat dipanggil ulang oleh CLI masa depan tanpa duplikasi logika:

```
Route Handler (HTTP)  →  Service (business logic)  →  Repository (DB access via Drizzle)
                                     ↓
                            AI Orchestrator (khusus generation)
```

---

## 2. Lapisan Service

| Service | Tanggung Jawab |
|---|---|
| `SkillService` | CRUD skill, versi, publish/unpublish/archive |
| `GenerationService` | Membuat & menjalankan generation job, step events |
| `AiOrchestratorService` | Memanggil provider AI via abstraction, retry, fallback |
| `ValidationService` | Validasi canonical JSON terhadap schema + linter aturan skill |
| `SecurityScanService` | Menjalankan security scanner (lihat `09`) |
| `QualityEvalService` | Menghitung Quality Score (rubric `01`/`08`) |
| `CompilerService` | Canonical JSON → file (SKILL.md, README.md, dst.) → ZIP |
| `AuthService` | Login/register/OAuth/session, delegasi ke Auth.js |
| `AuthorizationService` | RBAC check (server-side, dipanggil di setiap Route Handler admin) |
| `CommentService`, `RatingService`, `CollectionService`, `BookmarkService` | Interaksi user |
| `SearchService` | Query full-text search |
| `AuditLogService` | Mencatat semua aksi admin |
| `AnalyticsService` | Agregasi dashboard admin |
| `StorageService` | Upload/ambil file besar via `StorageProvider` |

---

## 3. API Contract

### 3.1 Public

```
GET  /api/skills                 ?q=&category=&agent=&sort=&page=
GET  /api/skills/:slug
GET  /api/categories
GET  /api/agents
GET  /api/prds                   /api/prds/:slug
GET  /api/workflows              /api/workflows/:slug
GET  /api/agent-kits             /api/agent-kits/:slug
GET  /api/templates              /api/templates/:slug
GET  /api/search                 ?q=&type=
GET  /api/collections            /api/collections/:slug
```

### 3.2 User (butuh sesi login)

```
POST   /api/comments             { targetType, targetId, body, parentId? }
DELETE /api/comments/:id
POST   /api/comments/:id/reactions
POST   /api/ratings              { targetType, targetId, score }
POST   /api/bookmarks            { targetType, targetId }
DELETE /api/bookmarks/:id
POST   /api/collections          { name, visibility }
POST   /api/collections/:id/items
POST   /api/reports              { targetType, targetId, reason }
```

### 3.3 Generator (butuh sesi + permission generate)

```
POST /api/generate/skill         → { jobId }
POST /api/generate/prd
POST /api/generate/workflow
POST /api/generate/agent-kit
GET  /api/generate/jobs/:jobId            → status + steps
GET  /api/generate/jobs/:jobId/result     → canonical JSON (setelah selesai)
```

### 3.4 Validation & Security

```
POST /api/validation             { canonicalJson, schemaType }  → hasil validasi
POST /api/security/scan          { canonicalJson }               → hasil scan
POST /api/quality/evaluate       { canonicalJson }                → skor rubric
```

### 3.5 Admin (RBAC, semua di bawah `/api/admin/*`)

```
CRUD  /api/admin/skills, /prds, /workflows, /agent-kits, /categories, /tags, /templates
CRUD  /api/admin/users            (ubah role, suspend)
CRUD  /api/admin/comments         (pin/hide/delete)
CRUD  /api/admin/reports          (resolve/dismiss)
CRUD  /api/admin/ai-providers     (JANGAN pernah kembalikan apiKey mentah ke response)
CRUD  /api/admin/prompt-templates
POST  /api/admin/skills/:id/publish
POST  /api/admin/skills/:id/unpublish
POST  /api/admin/skills/:id/archive
POST  /api/admin/skills/:id/version   (buat versi baru + snapshot)
GET   /api/admin/analytics
GET   /api/admin/audit-logs
GET/PUT /api/admin/settings
```

---

## 4. Autentikasi & Otorisasi

- **AuthN:** Auth.js (NextAuth) — credentials (email/password, hash Argon2/bcrypt) + OAuth Google & GitHub.
- **AuthZ (RBAC):** role disimpan di `user_roles` (`user`, `moderator`, `admin`, `superadmin` — superadmin khusus untuk AI Provider & Settings sensitif). **Setiap** Route Handler admin memanggil `AuthorizationService.assertRole(session, requiredRole)` di baris pertama — tidak boleh mengandalkan UI hiding saja.
- Middleware Next.js melakukan pre-check ringan (redirect ke `/login` jika tidak ada sesi untuk `/admin/*`), tetapi **cek permission final tetap di server**, bukan hanya middleware.

---

## 5. AI Orchestrator & Provider Abstraction

Dibangun di atas **Vercel AI SDK** (bukan memanggil SDK vendor langsung satu-satu), karena SDK ini sudah menyediakan lapisan agnostik-provider dengan `generateText`/`streamText`/`generateObject` yang bekerja seragam lintas Anthropic, OpenAI, Google, dan provider kompatibel-OpenAI seperti Groq.

```ts
interface AiProviderConfig {
  id: string;
  provider: 'anthropic' | 'google' | 'openai' | 'groq' | 'openai-compatible';
  model: string;
  endpoint?: string;       // untuk openai-compatible / self-host
  apiKeySecretRef: string; // referensi ke secret store, BUKAN nilai mentah di DB
  active: boolean;
  priority: number;        // urutan fallback
  timeoutMs: number;
  maxTokens: number;
}

async function generateCanonicalJson(input: GenerationInput): Promise<CanonicalSkillJson> {
  const providers = await getActiveProvidersSortedByPriority();
  for (const cfg of providers) {
    try {
      const model = resolveAiSdkModel(cfg); // -> @ai-sdk/anthropic | @ai-sdk/google | @ai-sdk/openai | openai-compatible
      const { object } = await generateObject({
        model,
        schema: canonicalSkillZodSchema,
        prompt: buildPromptEngine(input),
        maxRetries: 2,
      });
      return object;
    } catch (err) {
      await logGenerationFailure(cfg, err);
      continue; // fallback ke provider berikutnya sesuai priority
    }
  }
  throw new AllProvidersFailedError();
}
```

**Fallback chain:** Primary → retry (backoff pendek) → provider fallback berikutnya sesuai `priority` → jika semua gagal, job ditandai `Failed` dengan pesan jelas, admin dapat retry manual dari Generation Workspace.

`apiKey` **tidak pernah**: disimpan plaintext di kolom biasa (gunakan secret manager/`env` per-deployment atau kolom terenkripsi), dikirim ke frontend dalam response apapun, atau di-log.

---

## 6. Async Generation (Job & Step Events)

Karena generation AI multi-tahap bisa memakan waktu lebih lama dari batas request serverless, request awal **hanya membuat job**, eksekusi berjalan di background:

```
POST /api/generate/skill
  → insert row `generations` (status=queued)
  → trigger eksekusi non-blocking (Vercel serverless function terpisah / Vercel Functions background invocation)
  → return { jobId } segera (respons cepat, tidak menunggu AI selesai)

Worker/eksekutor:
  → jalankan pipeline 18-tahap (lihat `07`)
  → tiap tahap selesai → insert row `generation_steps` (step, status, output ringkas, timestamp)
  → update `generations.status`

Frontend:
  → polling GET /api/generate/jobs/:jobId setiap beberapa detik, atau
  → (opsional peningkatan) Server-Sent Events endpoint untuk step event realtime
```

**Prinsip MVP:** abstraksi eksekusi background dibuat sebagai interface `JobExecutor`, implementasi awal cukup memakai fungsi serverless async sederhana — **jangan memaksakan vendor queue (SQS, BullMQ+Redis, dst.) sebelum volume generation benar-benar membutuhkannya**, sesuai instruksi brief.

---

## 7. Rate Limiting

- Endpoint generate dibatasi per user per jam (konfigurasi di `site_settings`, default masuk akal untuk MVP, admin dapat ubah).
- Endpoint publik (search, list) dibatasi per IP untuk mencegah scraping agresif.
- Implementasi: middleware ringan di Route Handler + tabel/counter di Neon (atau in-memory edge cache jika tersedia) — cukup untuk MVP, dapat diganti Upstash Redis dsb. di roadmap jika diperlukan.

---

## 8. Error Handling

Format error API konsisten:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Canonical JSON tidak lolos validasi skema.",
    "details": [ { "field": "triggers", "issue": "minimal 1 trigger wajib diisi" } ]
  }
}
```

Semua pesan `message` berbahasa Indonesia; `code` tetap bahasa Inggris/konstanta agar mudah ditangani frontend/CLI masa depan.

---

## 9. Audit Log

Setiap aksi admin yang mengubah state (publish, edit, delete, ubah role, ubah provider) menulis baris ke `audit_logs`: `actorId`, `action`, `targetType`, `targetId`, `beforeSnapshot`, `afterSnapshot`, `timestamp`, `ipAddress`. Ditampilkan read-only di `/admin/audit-logs`, tidak bisa dihapus lewat UI.

---

## 10. Testing (Backend)

- **Unit**: tiap Service dengan repository di-mock.
- **Integration**: Route Handler penuh terhadap Neon test branch (Neon mendukung database branching — cocok untuk isolasi test).
- **Contract test**: canonical JSON contoh (fixture) wajib lolos `ValidationService` sebelum PR di-merge (dijalankan di CI).
- **Security test**: fixture skill "jahat" (mengandung `curl | sh`, secret palsu, prompt injection) wajib **terdeteksi** oleh `SecurityScanService` — regression test wajib ada untuk tiap kategori risiko di `09`.

---

## Catatan Konsistensi
Setiap endpoint di §3 dikonsumsi oleh repository yang sama namanya di `04-frontend-architecture.md` §3. Struktur tabel yang direferensikan service di sini didefinisikan penuh di `06-database-neon.md`.

**Lanjutkan ke:** `06-database-neon.md`
