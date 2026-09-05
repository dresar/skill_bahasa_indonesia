# 09 — Admin, Security & Operations
## AI Skill Factory Indonesia

---

## 1. Admin Dashboard (`/admin`)

Kartu ringkasan: total skill, published, draft, review, total download, total user, total komentar, jumlah generation (hari ini/total), success rate generation, penggunaan AI (token/biaya), skill terpopuler, kategori terpopuler, aktivitas terbaru (feed audit log ringkas).

---

## 2. Alur Status Skill (berlaku juga PRD/Workflow/Agent Kit)

```
Draft → Generating → Review → Approved → Published → Archived
                                   ↓
                                Rejected
```

| Status | Siapa Bisa Masuk | Aksi Tersedia |
|---|---|---|
| Draft | Admin/AI membuat manual | Edit, Generate, Delete |
| Generating | Otomatis saat job berjalan | Lihat progress, Cancel |
| Review | Otomatis setelah generation selesai, atau submit manual | Validate, Security Scan, Approve, Reject, Edit JSON |
| Approved | Admin approve | Publish |
| Published | Admin publish | Unpublish, Archive, Version (buat versi baru) |
| Archived | Admin archive | Restore (kembali ke Draft) |
| Rejected | Admin reject dari Review | Edit ulang → kembali ke Draft |

Aksi lain yang tersedia di semua status: **Duplicate**, **Preview** (tampilan seperti akan muncul publik), **Compare versions** (diff dua versi canonical JSON).

---

## 3. Generation Workspace (`/admin/generator`)

Menampilkan **timeline 18-tahap** dari `07` secara visual per job:

```
Requirement → Research → PRD → Architecture → Skill → Examples → README
→ Security → Quality → Validation → Assembly → Package
```

(Penamaan tahap di UI disederhanakan dari 18 tahap teknis menjadi kelompok yang mudah dipahami admin; detail teknis penuh tetap tercatat di `generation_steps`.)

Admin dapat: melihat **raw JSON** tiap tahap, **edit JSON** langsung (Monaco Editor), **regenerate** satu tahap tertentu tanpa mengulang semuanya, **retry** tahap yang gagal, **approve/reject**, dan **compare versions**.

---

## 4. Security Scanner

### 4.1 Kategori yang Diperiksa

API keys/tokens/passwords/cookies/credentials/private keys/secrets · URL mencurigakan · eksekusi skrip jarak jauh · `curl`/`wget` pipe-to-shell · perintah shell arbitrer · perintah destruktif · penghapusan filesystem · credential harvesting · eksfiltrasi data · privilege escalation · payload terobfuskasi · instruksi terenkode · **prompt injection** · dependency confusion · dependency berbahaya · instalasi paket mencurigakan · akses jaringan tak dijelaskan · panggilan eksternal tersembunyi · instruksi untuk melewati kontrol keamanan · instruksi untuk mengabaikan kebijakan sistem.

### 4.2 Prinsip Least Privilege

Jika skill membutuhkan skrip shell: **wajib** dijelaskan tujuan, input, output, permission yang dibutuhkan, dependency, dan perilaku saat gagal — sebelum diizinkan lolos review. Jika butuh akses jaringan: wajib dijelaskan alasannya di `scripts[].description`.

### 4.3 Bahasa Hasil Scan (WAJIB)

Scanner **tidak pernah** menyatakan "100% aman". Gunakan istilah:

| Kondisi | Istilah yang Dipakai |
|---|---|
| Tidak ditemukan masalah pada kategori yang diperiksa | "Pemeriksaan otomatis: tidak ada risiko terdeteksi pada kategori yang diperiksa" |
| Ditemukan pola berisiko | "Risiko terdeteksi: {kategori}" + detail lokasi |
| Pola ambigu | "Peringatan: pola berikut memerlukan review manual" |
| Selalu ditampilkan di footer hasil scan | "Hasil ini adalah pemeriksaan otomatis dan tidak menggantikan review manual." |

Contoh tampilan skor: **"Keamanan 96/100 — 2 peringatan memerlukan review manual"** (bukan "100% Aman").

---

## 5. Quality Engine

Rubric 12 dimensi (dihitung programatik dari struktur canonical JSON, lihat `07` §9): Purpose · Trigger · Instructions · Workflow · Boundary · Examples · Documentation · Compatibility · Security · Token efficiency · Maintainability · Testability.

Skor 0–100 = rata-rata tertimbang breakdown per dimensi, **selalu** ditampilkan dengan breakdown-nya (bukan angka tunggal tanpa penjelasan) agar admin/user dapat memverifikasi kenapa skor tersebut diberikan.

---

## 6. Moderasi

| Objek | Aksi Admin |
|---|---|
| Komentar | Pin, Hide, Delete, Reply, Ban penulis |
| Report | Resolve, Dismiss, lihat konteks target |
| User | Suspend, ubah role, lihat riwayat aktivitas |
| Rating | Hapus jika terindikasi manipulasi (mis. banyak rating dari IP/akun mencurigakan dalam waktu singkat) |

**Anti-abuse rating:** satu user hanya boleh 1 rating per target (constraint DB, lihat `06`); rate-limit submit rating per user per jam; flag otomatis jika distribusi rating berubah drastis dalam waktu singkat untuk direview admin.

---

## 7. AI Provider Management (`/admin/ai-providers`)

Form per provider: `provider`, `model`, `endpoint` (opsional, untuk custom OpenAI-compatible), `apiKey` (input write-only — **tidak pernah ditampilkan kembali setelah disimpan**, hanya indikator "Terpasang"), `active`, `priority`, `timeout`, `maxTokens`. Urutan `priority` menentukan fallback chain (`05`/`07`).

## 8. Prompt Management (`/admin/prompts`)

Daftar 22 modul aturan (`07` §3) sebagai entri `prompt_templates` yang dapat diedit admin per `stage`, dengan versioning sederhana (riwayat perubahan tercatat di `audit_logs`).

## 9. Validation & Security Panel

Riwayat seluruh `validation_results` dan `security_scans`, dapat difilter per skill/tanggal/severity — dipakai admin untuk melihat tren kualitas dari waktu ke waktu, bukan hanya hasil terakhir.

## 10. Analytics

Grafik: pertumbuhan skill published per minggu, distribusi quality score, distribusi security score, top 10 skill terunduh, top kategori, funnel generation (queued → succeeded/failed), biaya AI per provider.

## 11. Audit Logs (`/admin/audit-logs`)

Tabel read-only (mobile: card list), filter per aktor/aksi/tanggal, tiap baris dapat di-expand untuk melihat `beforeSnapshot`/`afterSnapshot`.

## 12. Site Settings

Kuota generate per role, rate limit publik, teks legal (Terms/Privacy jika ada), toggle fitur (mis. matikan sementara registrasi publik).

---

## Catatan Konsistensi
Enum status §2 identik dengan `06-database-neon.md` §2.2 dan Badge di `03-ui-ux-design-system.md` §7. Kategori scanner §4.1 dipakai sebagai daftar regression test wajib di `05-backend-architecture.md` §10.

**Lanjutkan ke:** `10-deployment-testing-roadmap.md`
