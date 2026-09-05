# 01 — Product Overview
## AI Skill Factory Indonesia

---

## 1. Visi Produk

> **AI Skill Factory Indonesia** adalah platform open-source berbahasa Indonesia untuk **menemukan, membuat, memvalidasi, mengevaluasi, mengompilasi, dan menggunakan** AI coding skills, PRD, workflows, templates, dan Agent Kits untuk AI coding agent modern (Claude Code, Cursor, Codex CLI, Gemini CLI, Antigravity, dan agent lain yang mengikuti standar Agent Skills terbuka).

Alur inti produk:

```
DISCOVER → EXPLORE → GENERATE → REVIEW → VALIDATE → EVALUATE → COMPILE → PUBLISH → DOWNLOAD → USE
```

Produk **bukan**:
- Website kumpulan prompt statis
- Kumpulan README hasil scraping GitHub
- Marketplace creator/payout (bukan untuk MVP)
- Clone dari skill directory yang sudah ada

Produk **adalah**:
- AI Development Resource Platform sekaligus AI Skill Factory
- Sistem yang memperlakukan skill sebagai *software artifact* yang bisa diuji, bukan sekadar dokumen Markdown

---

## 2. Masalah yang Diselesaikan

Developer Indonesia yang menggunakan AI coding agent saat ini menghadapi:

1. **Fragmentasi.** Skill, rules, AGENTS.md, prompt, dan workflow tersebar di GitHub, blog, forum, marketplace berbahasa Inggris.
2. **Tidak ada standar kualitas.** Skill hasil copy-paste sering tidak punya boundary, contoh, atau uji perilaku — hanya "kelihatan lengkap" tapi tidak reliable.
3. **Risiko keamanan tersembunyi.** Skill/script pihak ketiga bisa membawa perintah shell destruktif, `curl | sh`, atau prompt injection tanpa disadari pengguna awam.
4. **Barrier bahasa & konteks lokal.** Dokumentasi teknis 95% berbahasa Inggris; developer pemula Indonesia butuh onboarding dalam Bahasa Indonesia yang natural, bukan hasil translate literal.
5. **Tidak ada cara mudah membuat skill baru yang berkualitas** tanpa memahami detail spesifikasi `SKILL.md`, progressive disclosure, dan best practice token efficiency.

---

## 3. Target Pengguna & Persona

| Persona | Kebutuhan Utama | Fitur yang Paling Relevan |
|---|---|---|
| **Dimas — Mahasiswa IT semester akhir** | Belajar memakai Claude Code/Cursor secara efektif, contoh skill nyata | Skill Library, dokumentasi Bahasa Indonesia, Docs Hub |
| **Sarah — Software Engineer startup** | Skill khusus stack timnya (Next.js + Supabase), butuh cepat, tidak mau baca spec panjang | Skill Generator, Agent Kit Builder |
| **Budi — Founder solo/indie hacker** | PRD siap pakai untuk diserahkan ke coding agent, tanpa menulis dari nol | PRD Generator, Agent Kit Generator |
| **Rani — Technical writer/DevRel** | Menulis dan mengelola dokumentasi skill tim secara konsisten | Skill Generator + Markdown Editor + Versioning |
| **Admin platform (internal)** | Menjaga kualitas & keamanan seluruh konten yang terbit | Admin Dashboard, Generation Workspace, Security Scanner, Quality Engine |

---

## 4. Positioning

| Dimensi | Posisi AI Skill Factory Indonesia |
|---|---|
| vs. GitHub awesome-list | Bukan daftar link statis — setiap item tervalidasi, ternilai, dan dapat dikompilasi ulang |
| vs. skill marketplace global (agentskills.me dll.) | Fokus Bahasa Indonesia + kurasi ketat admin, bukan submission bebas komunitas (di MVP) |
| vs. "prompt generator" generik | Output berupa artifact software (folder skill lengkap dengan scripts/references/evals), bukan satu blok teks |
| vs. dokumentasi resmi tiap agent (Claude Code, Cursor dll.) | Melengkapi, bukan menggantikan — menyediakan lapisan *authoring & quality assurance* di atas standar yang sudah ada |

---

## 5. Value Proposition

- **Untuk developer:** "Temukan skill siap pakai atau buat skill baru dalam hitungan menit, tervalidasi otomatis, dan langsung kompatibel dengan agent yang kamu pakai."
- **Untuk tim/startup:** "Standarkan cara tim kamu memberi instruksi ke AI coding agent — sekali disusun, dipakai berulang oleh semua anggota tim, semua agent."
- **Untuk ekosistem:** "Ekosistem Agent Skills berbahasa Indonesia yang aman, terkurasi, dan open-source."

---

## 6. Product Principles

1. **AI menghasilkan DATA, backend melakukan VALIDASI, compiler menghasilkan FILE, frontend menyediakan EXPERIENCE, admin mengontrol QUALITY.** Prinsip ini tidak boleh dilanggar di layer manapun.
2. **Progressive disclosure di produk, bukan hanya di skill.** Halaman skill detail juga menerapkan prinsip ini — ringkasan dulu, detail on-demand.
3. **Tidak ada klaim keamanan absolut.** Semua hasil scan pakai bahasa "risiko terdeteksi", bukan "aman 100%".
4. **Bahasa Indonesia natural, bukan hasil translate.** Istilah teknis resmi (SKILL.md, AGENTS.md, nama agent) tetap bentuk asli.
5. **Setiap dependency harus punya alasan.** Tidak menambah library "karena ada", termasuk React Bits — dipakai selektif.
6. **Mobile bukan warga kelas dua.** Setiap halaman punya perilaku responsive yang didefinisikan eksplisit, bukan "auto-responsive" dari framework.
7. **Semua provider (AI, storage, search, auth) berada di balik abstraction layer.** Tidak ada vendor lock-in yang tidak disengaja.

---

## 7. Scope

### 7.1 Termasuk MVP
- Skill Library (public), Skill Detail, Skill Generator (AI Skill Factory penuh)
- PRD Generator, Workflow Generator, Agent Kit Generator (versi dasar)
- Validasi schema + Security Scanner + Quality Engine
- Compiler → SKILL.md / README.md / AGENTS.md / PRD.md / ZIP
- Auth, komentar, rating, bookmark, collection (personal & official)
- Admin dashboard penuh: generation workspace, moderation, AI provider config, audit log
- Search PostgreSQL full-text
- Deploy Vercel + Neon

### 7.2 Di Luar Scope MVP (Roadmap)
- Marketplace creator dengan payout
- Community publishing tanpa approval admin
- CLI penuh (`npx @PROJECT_NAME/cli`) — hanya disiapkan arsitekturnya
- Public API untuk pihak ketiga
- Import otomatis dari GitHub repo pihak ketiga
- Search engine eksternal (Meilisearch/Typesense/Algolia)

---

## 8. Success Metrics

| Kategori | Metrik | Target Arah (bukan angka final, ditentukan tim produk) |
|---|---|---|
| Aktivasi | % pengunjung yang mencoba Generator dalam kunjungan pertama | naik |
| Kualitas konten | Rata-rata Quality Score skill published | ≥ 80/100 |
| Keamanan | % generation run dengan security warning yang diselesaikan sebelum publish | 100% (tidak ada publish dengan risiko belum direview) |
| Retensi | % pengguna terdaftar yang kembali dalam 30 hari | naik |
| Efisiensi admin | Waktu rata-rata Draft → Published | turun seiring compiler makin matang |
| Adopsi teknis | Jumlah download/instalasi skill per bulan | naik |
| Kesehatan komunitas | Rasio komentar yang di-moderasi vs total komentar | tetap rendah |

---

## Catatan Konsistensi
Dokumen ini adalah rujukan persona, scope, dan prinsip yang dipakai oleh seluruh dokumen `02`–`10`. Setiap fitur baru yang muncul di dokumen lain harus bisa ditelusuri ke salah satu persona atau prinsip di atas.

**Lanjutkan ke:** `02-information-architecture.md`
