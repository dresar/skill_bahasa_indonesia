# 06 — Database Architecture (Neon PostgreSQL)
## AI Skill Factory Indonesia

---

## 1. Mengapa Neon + Drizzle

- **Neon**: PostgreSQL serverless, mendukung **database branching** — setiap Vercel Preview Deployment dapat memakai branch Neon terpisah, cocok untuk isolasi data test/staging tanpa server terpisah.
- **Drizzle ORM**: TypeScript-first, query builder ringan yang cocok untuk runtime edge/serverless, migrasi dikelola sebagai file SQL eksplisit (`drizzle-kit`) — lebih mudah diaudit dibanding migrasi "ajaib".
- Koneksi memakai driver serverless (`@neondatabase/serverless`) agar kompatibel dengan Vercel Functions/Edge tanpa exhaust connection pool.

---

## 2. Entity List & Ringkasan Kolom

> Konvensi umum semua tabel (kecuali dinyatakan lain): `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`, `created_at TIMESTAMPTZ DEFAULT now()`, `updated_at TIMESTAMPTZ DEFAULT now()`, `deleted_at TIMESTAMPTZ NULL` (soft delete).

### 2.1 Identitas & Akses

| Tabel | Kolom Kunci | Catatan |
|---|---|---|
| `users` | `email UNIQUE`, `password_hash NULL`, `name`, `avatar_url`, `oauth_provider`, `oauth_id`, `status` | `password_hash` NULL jika user OAuth-only |
| `user_roles` | `user_id FK→users`, `role ENUM(user,moderator,admin,superadmin)` | Many-to-many via tabel ini (user bisa punya >1 role di masa depan) |
| `audit_logs` | `actor_id FK→users`, `action`, `target_type`, `target_id`, `before_json JSONB`, `after_json JSONB`, `ip_address` | Append-only, tanpa `deleted_at` (tidak boleh dihapus) |

### 2.2 Konten Inti

| Tabel | Kolom Kunci | Catatan |
|---|---|---|
| `skills` | `slug UNIQUE`, `name`, `description`, `current_version_id FK→skill_versions`, `status ENUM(draft,generating,review,approved,published,archived,rejected)`, `owner_id FK→users NULL (NULL = official/admin)`, `quality_score INT`, `security_score INT`, `download_count INT DEFAULT 0` | `status` sama persis dengan enum di `09` |
| `skill_versions` | `skill_id FK→skills`, `version TEXT` (semver), `canonical_json JSONB`, `changelog TEXT`, `published_at` | Snapshot immutable per versi |
| `skill_files` | `skill_version_id FK→skill_versions`, `file_path`, `content_type`, `storage_ref` (via `StorageProvider`) | File hasil compile (SKILL.md, README.md, dst.), bukan disimpan sebagai blob besar di Postgres |
| `categories` | `slug UNIQUE`, `name`, `description`, `icon` | |
| `tags` | `slug UNIQUE`, `name` | |
| `skill_tags` | `skill_id`, `tag_id` | Composite PK |
| `agents` | `slug UNIQUE` (`claude-code`, `cursor`, `codex-cli`, `gemini-cli`, `antigravity`, ...), `name`, `skill_folder_global`, `skill_folder_project`, `notes` | Lihat compatibility matrix `00`/`08` |
| `skill_agents` | `skill_id`, `agent_id`, `compatibility_status ENUM(verified,likely,unverified)` | |
| `prds`, `workflows`, `agent_kits`, `templates` | Pola mirip `skills` (slug, canonical_json versioned, status, owner) | Masing-masing punya tabel `_versions` sendiri mengikuti pola `skill_versions` |

### 2.3 Interaksi Pengguna

| Tabel | Kolom Kunci | Catatan |
|---|---|---|
| `collections` | `owner_id`, `name`, `visibility ENUM(private,public)`, `is_official BOOLEAN` | |
| `collection_items` | `collection_id`, `target_type`, `target_id`, `position` | |
| `comments` | `author_id`, `target_type`, `target_id`, `parent_id NULL`, `body`, `status ENUM(visible,hidden,pinned)` | |
| `comment_reactions` | `comment_id`, `user_id`, `type ENUM(upvote)` | Unique(`comment_id`,`user_id`) |
| `ratings` | `user_id`, `target_type`, `target_id`, `score SMALLINT CHECK 1-5` | Unique(`user_id`,`target_type`,`target_id`) — cegah rating ganda |
| `reports` | `reporter_id`, `target_type`, `target_id`, `reason`, `status ENUM(open,resolved,dismissed)` | |
| `downloads` | `target_type`, `target_id`, `user_id NULL`, `agent_id NULL` | Untuk analytics, `user_id` nullable (anonim boleh unduh) |

### 2.4 AI Factory & Operasional

| Tabel | Kolom Kunci | Catatan |
|---|---|---|
| `ai_providers` | `provider`, `model`, `endpoint NULL`, `api_key_secret_ref`, `active BOOLEAN`, `priority INT`, `timeout_ms`, `max_tokens` | `api_key_secret_ref` **bukan** nilai mentah |
| `prompt_templates` | `name`, `stage` (mengacu 22 kategori aturan di `07`), `content`, `version` | |
| `generations` | `target_type`, `requester_id`, `input_json JSONB`, `status ENUM(queued,running,succeeded,failed)`, `result_skill_version_id NULL` | |
| `generation_steps` | `generation_id FK`, `step_name`, `status ENUM(pending,running,succeeded,failed,skipped)`, `output_summary TEXT`, `token_usage JSONB`, `started_at`, `finished_at` | Urutan step sesuai pipeline `07` |
| `validation_results` | `target_type`, `target_id`, `schema_version`, `is_valid BOOLEAN`, `errors JSONB` | |
| `security_scans` | `target_type`, `target_id`, `score INT`, `findings JSONB`, `scanned_at` | `findings` array kategori risiko + severity |
| `site_settings` | `key UNIQUE`, `value JSONB` | Key-value config (rate limit, kuota generate, dsb.) |

---

## 3. Index & Constraint Penting

```sql
CREATE INDEX idx_skills_status ON skills(status) WHERE deleted_at IS NULL;
CREATE INDEX idx_skills_category ON skills(category_id);
CREATE UNIQUE INDEX idx_skills_slug ON skills(slug) WHERE deleted_at IS NULL;
CREATE INDEX idx_generation_steps_generation_id ON generation_steps(generation_id);
CREATE INDEX idx_comments_target ON comments(target_type, target_id);

-- Full-text search (MVP)
ALTER TABLE skills ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('indonesian', coalesce(name,'')), 'A') ||
    setweight(to_tsvector('indonesian', coalesce(description,'')), 'B')
  ) STORED;
CREATE INDEX idx_skills_search ON skills USING GIN(search_vector);
```

> Konfigurasi `to_tsvector('indonesian', ...)` mengasumsikan text search configuration Bahasa Indonesia tersedia di instance Postgres/Neon; jika tidak tersedia secara bawaan, fallback ke `'simple'` dan lakukan normalisasi kata di layer aplikasi — **diverifikasi saat setup database di Phase 4**.

---

## 4. Relasi & Diagram (ringkas, teks)

```
users ─< user_roles
users ─< skills (owner_id, nullable)
skills ─< skill_versions ─< skill_files
skills ─< skill_agents >─ agents
skills ─< skill_tags >─ tags
skills ─< comments, ratings, downloads, reports (target_type='skill')
collections ─< collection_items >─ (skills | prds | workflows | agent_kits)
generations ─< generation_steps
generations → skill_versions (result, nullable sampai sukses)
ai_providers  (independen, dikonsumsi AiOrchestratorService)
```

---

## 5. Transaksi

Operasi yang **wajib** dibungkus transaksi Drizzle (`db.transaction`):
- Publish skill: update `skills.status` + insert `skill_versions` + insert `skill_files` + tulis `audit_logs` → satu transaksi.
- Generation selesai sukses: insert `skill_versions` + update `generations.status` + `generation_steps` final → satu transaksi.
- Ubah role user: update `user_roles` + `audit_logs` → satu transaksi.

---

## 6. Migrasi & Seed

- Migrasi dikelola `drizzle-kit generate` + `drizzle-kit migrate`, file SQL tersimpan di `drizzle/migrations/`, dijalankan otomatis di CI/CD sebelum deploy (lihat `10`).
- Seed data (`scripts/seed.ts`): kategori dasar, daftar `agents` (Claude Code, Cursor, Codex CLI, Gemini CLI, Antigravity — sesuai `00`/`08`), beberapa skill contoh resmi, `site_settings` default.
- Neon branching dipakai untuk: `main` (production), `preview/*` (otomatis per PR via integrasi Vercel-Neon), `dev` (lokal, opsional).

---

## 7. Keamanan Database

- Least privilege: role aplikasi hanya punya akses ke schema yang diperlukan (bukan superuser).
- `api_key_secret_ref` di `ai_providers` — kunci sebenarnya di environment variable/secret manager, kolom DB hanya referensi/id.
- Soft delete (`deleted_at`) untuk hampir semua tabel konten agar dapat dipulihkan admin; `audit_logs` **append-only**, tidak pernah soft/hard delete lewat aplikasi.
- Backup: mengandalkan point-in-time recovery bawaan Neon; kebijakan retensi dikonfirmasi saat provisioning (lihat `10` §Backup/Recovery).

---

## Catatan Konsistensi
Enum `status` pada `skills`/`prds`/`workflows`/`agent_kits` harus identik dengan Badge status di `03` dan alur admin di `09`. Struktur `canonical_json JSONB` pada `skill_versions` harus valid terhadap schema di `08-skill-schema-compiler.md`.

**Lanjutkan ke:** `07-ai-generation-engine.md`
