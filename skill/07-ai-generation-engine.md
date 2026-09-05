# 07 — AI Generation Engine
## AI Skill Factory Indonesia

---

## 1. Prinsip Inti

> AI menghasilkan **canonical JSON terstruktur**, tidak pernah file final. AI **tidak boleh** dipercaya sebagai satu-satunya lapisan kualitas — validasi schema, security scan, dan quality evaluation berjalan **setelah** AI selesai, sebelum apa pun dianggap layak tampil ke pengguna.

---

## 2. Pipeline Generation (18 Tahap)

```
1.  USER REQUIREMENT             — input form Generator (Bahasa Indonesia)
2.  REQUIREMENT NORMALIZATION    — parsing ke struktur internal (tujuan, teknologi, agent target)
3.  WEB RESEARCH                 — jika requirement menyentuh teknologi/API yang cepat berubah
4.  DOMAIN ANALYSIS              — AI menentukan domain skill (framework/tooling/workflow/dst.)
5.  SKILL DESIGN                 — single responsibility, scope, non-scope, boundary
6.  RESOURCE DESIGN              — tentukan perlu-tidaknya scripts/references/assets/evals
7.  CANONICAL JSON GENERATION    — output terstruktur sesuai schema (`08`)
8.  SCHEMA VALIDATION            — JSON Schema + Zod, gagal → kembali ke step 7 (auto-retry terbatas)
9.  SECURITY REVIEW              — SecurityScanService (lihat `09`)
10. QUALITY REVIEW               — QualityEvalService, rubric 12 dimensi
11. COMPATIBILITY REVIEW         — cek klaim compatibility terhadap agent yang benar-benar didukung
12. BEHAVIORAL EVALUATION        — hasilkan/lengkapi `evals/evals.json` (Trigger/Compliance/Boundary)
13. NORMALIZATION                — rapikan casing, slug, format token-efisien
14. COMPILATION                  — CompilerService: JSON → SKILL.md/README.md/dst.
15. FILE GENERATION              — tulis file ke StorageProvider
16. PACKAGE                      — susun ZIP
17. VERSION                      — tetapkan semver, simpan snapshot
18. PUBLISH                      — (jika role mengizinkan) atau masuk antrean review admin
```

Setiap tahap dicatat sebagai baris `generation_steps` (lihat `06`) dengan status, ringkasan output, dan token usage — memberi admin visibilitas penuh di Generation Workspace (`09`).

---

## 3. Prompt Engine — 22 Modul Aturan

Prompt engine bersifat **modular**, disusun dari 22 blok aturan yang digabung sesuai jenis artifact (skill/prd/workflow/agent-kit) dan tahap pipeline. Disimpan sebagai `prompt_templates` per `stage`, dapat diedit admin tanpa deploy ulang kode.

| # | Modul | Isi Singkat |
|---|---|---|
| 1 | System Identity | Peran AI (arsitek skill), bahasa output (Bahasa Indonesia untuk narasi, istilah teknis tetap asli) |
| 2 | Research Rules | Kapan wajib web research, prioritas sumber resmi, larangan mengarang versi/API |
| 3 | Requirement Interpretation | Cara menerjemahkan input form jadi tujuan skill yang jelas |
| 4 | Skill Architecture Rules | Single responsibility, progressive disclosure |
| 5 | Metadata Rules | Format `name`/`description`/field frontmatter lain sesuai standar Agent Skills |
| 6 | Instruction Rules | Instruksi harus actionable, tidak generik |
| 7 | Trigger/Discovery Rules | `description` harus berfungsi sebagai discovery mechanism, bukan slogan pemasaran |
| 8 | Boundary Rules | Scope & non-scope eksplisit |
| 9 | Workflow Rules | Urutan langkah yang dapat diikuti agent step-by-step |
| 10 | Example Rules | Contoh nyata, bukan placeholder abstrak |
| 11 | Resource Rules | Kapan `references/`/`assets/` benar-benar diperlukan |
| 12 | Script Rules | Skrip hanya dibuat jika ada kebutuhan eksekusi nyata, harus punya penjelasan I/O |
| 13 | Template Rules | Template resource harus reusable, bukan hardcoded sekali pakai |
| 14 | Compatibility Rules | Pisahkan universal vs klaim khusus agent, jangan klaim tanpa verifikasi |
| 15 | Security Rules | Daftar kategori risiko di `09`, larangan menyimpan secret asli |
| 16 | Quality Rules | Checklist rubric 12 dimensi harus dipenuhi sebelum output final |
| 17 | Behavioral Evaluation Rules | Wajib hasilkan minimal 1 test case per dimensi Trigger/Compliance/Boundary |
| 18 | Token Efficiency Rules | `SKILL.md` ringkas, detail panjang dipindah ke `references/` |
| 19 | Maintainability Rules | Struktur agar mudah di-regenerate ulang tanpa mengubah makna |
| 20 | Output Schema Rules | Output **harus** valid JSON sesuai schema `08`, tanpa teks di luar JSON |
| 21 | Self-Critique Rules | Instruksi "lakukan self-review internal sebelum output" — **tanpa membocorkan chain-of-thought** |
| 22 | Final Validation Rules | Checklist akhir sebelum AI menyatakan hasil selesai |

> **Aturan lintas-modul:** AI diinstruksikan melakukan reasoning internal ("analisis secara internal", "validasi sebelum output") tetapi **dilarang** menuliskan proses berpikirnya di output — output hanya berisi hasil terstruktur + alasan singkat yang relevan per keputusan penting (mis. field `provenance.reasoningNotes`, bukan transkrip berpikir penuh).

---

## 4. Aturan Research

- Wajib dijalankan untuk skill yang menyentuh: framework, library, API, agent, SDK, deployment platform, atau standar keamanan.
- Prioritas sumber: dokumentasi resmi (Anthropic/Claude Code, Cursor, OpenAI/Codex, Google/Gemini/Antigravity, `agentskills.io`) > repository resmi/changelog > sumber teknis primer lain. **Hindari artikel SEO** jika dokumentasi resmi tersedia.
- Setiap fakta versi/API yang dipakai dicatat di `provenance.sources[]`: `{ url, title, checkedAt, note }`.
- Jika web research tidak tersedia saat itu (mis. tool nonaktif), AI **wajib** menandai field terkait sebagai `needsVerification: true` — **dilarang** berpura-pura sudah riset.

---

## 5. Canonical JSON sebagai Kontrak Output

`generateObject` (Vercel AI SDK) dipanggil dengan **schema Zod yang sama persis** dengan `packages/schema` — ini memaksa model mengembalikan struktur valid secara sintaksis dari sisi SDK; validasi semantik (skill-specific rules) tetap dijalankan terpisah oleh `ValidationService` (lihat `05` §5, `08`).

```ts
const { object, usage } = await generateObject({
  model: resolveAiSdkModel(activeProviderConfig),
  schema: canonicalSkillZodSchema,
  system: buildSystemPrompt(promptModules),      // gabungan modul 1–22 relevan
  prompt: buildUserPrompt(requirement, researchContext),
  maxRetries: 2,
});
await recordTokenUsage(generationId, stepName, usage);
```

---

## 6. Retry & Fallback

- **Retry dalam provider yang sama**: 2x untuk kegagalan transient (timeout, output tidak valid JSON) sebelum pindah provider.
- **Fallback antar-provider**: sesuai `priority` di `ai_providers` (lihat `05` §5) — misal Claude sebagai primary, Gemini sebagai fallback pertama, OpenAI-compatible custom sebagai fallback terakhir.
- **Kegagalan total**: job ditandai `Failed` dengan pesan tahap mana yang gagal; admin dapat retry manual dari step yang gagal saja (tidak perlu ulang dari nol) — lihat Generation Workspace di `09`.

---

## 7. Token Usage & Biaya

- `generation_steps.token_usage` menyimpan `{ promptTokens, completionTokens, totalTokens, estimatedCostUsd }` per tahap.
- Dashboard admin (`09`) mengagregasi biaya per provider/per hari/per jenis artifact — dasar untuk keputusan `priority` provider ke depan.

---

## 8. Provenance

Setiap canonical JSON menyimpan blok `provenance`:

```json
{
  "provenance": {
    "generatedBy": { "provider": "anthropic", "model": "..." },
    "sources": [
      { "url": "https://agentskills.io/specification", "title": "Agent Skills Specification", "checkedAt": "2026-08-08" }
    ],
    "needsVerification": ["compatibility.antigravity"],
    "pipelineVersion": "1.0",
    "reasoningNotes": "Skill dibatasi pada satu tanggung jawab: setup testing Playwright, bukan testing secara umum."
  }
}
```

Field ini **tidak dihapus** oleh compiler — sebagian ditampilkan ke admin (untuk audit), sebagian dikompres jadi catatan singkat di `README.md` hasil compile (lihat `08`).

---

## 9. Quality Evaluation (ringkasan; rubric penuh di `01`/`08`)

`QualityEvalService` menghitung skor 0–100 dari checklist 12 dimensi (purpose clarity, trigger quality, instruction quality, dst.) — **bukan skor dari model AI itu sendiri**, melainkan dihitung ulang secara programatik dari struktur canonical JSON (mis. "apakah `boundaries` non-kosong", "apakah `examples` ≥ 1", "apakah panjang `SKILL.md` hasil compile di bawah ambang efisiensi token"). Ini memastikan skor dapat dijelaskan dan tidak "palsu".

---

## Catatan Konsistensi
22 modul di §3 dipetakan 1:1 ke tabel `prompt_templates.stage` di `06`. Schema yang dipakai `generateObject` di §5 **harus** file yang sama persis dengan `08-skill-schema-compiler.md` §2 — tidak boleh ada dua sumber schema yang berbeda.

**Lanjutkan ke:** `08-skill-schema-compiler.md`
