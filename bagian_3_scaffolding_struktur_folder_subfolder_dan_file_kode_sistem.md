# Bagian 3 — Scaffolding Struktur Folder, Subfolder, dan File Kode Sistem

Dokumen ini menetapkan arsitektur folder, subfolder, dan tanggung jawab setiap file kode untuk aplikasi ujian berbasis **Next.js (App Router) + TypeScript** yang berjalan di runtime **Node.js (v24)** dengan driver database tunggal **`pg` (`node-postgres`)** dan **SQL Native** tanpa ORM.

## 1. Pohon Direktori Proyek Lengkap

```text
exam-system/
├── .env.example
├── .gitignore
├── next.config.ts
├── package.json
├── tsconfig.json
├── Dockerfile
├── docker-compose.yml
│
├── scripts/
│   ├── migrate.ts
│   ├── seed.ts
│   └── create-admin.ts
│
└── src/
    ├── app/
    │   ├── layout.tsx
    │   ├── globals.css
    │   ├── (public)/
    │   │   ├── page.tsx
    │   │   ├── register/
    │   │   │   └── page.tsx
    │   │   ├── remedial/
    │   │   │   └── [token]/
    │   │   │       └── page.tsx
    │   │   ├── instructions/
    │   │   │   └── page.tsx
    │   │   └── verify/
    │   │       └── [code]/
    │   │           └── page.tsx
    │   │
    │   ├── (auth)/
    │   │   ├── admin/
    │   │   │   └── page.tsx
    │   │   └── it-unlock/
    │   │       └── page.tsx
    │   │
    │   ├── (admin)/
    │   │   └── dashboard/
    │   │       ├── layout.tsx
    │   │       ├── page.tsx
    │   │       ├── questions/
    │   │       │   ├── page.tsx
    │   │       │   ├── new/
    │   │       │   │   └── page.tsx
    │   │       │   └── [questionId]/
    │   │       │       └── page.tsx
    │   │       ├── participants/
    │   │       │   ├── page.tsx
    │   │       │   └── [attemptId]/
    │   │       │       └── page.tsx
    │   │       └── pdf-templates/
    │   │           └── page.tsx
    │   │
    │   ├── (participant)/
    │   │   └── result/
    │   │       └── [attemptId]/
    │   │           └── page.tsx
    │   │
    │   ├── exam/
    │   │   └── [attemptId]/
    │   │       ├── layout.tsx
    │   │       └── page.tsx
    │   │
    │   └── api/
    │       ├── health/
    │       │   └── route.ts
    │       ├── auth/
    │       │   ├── admin-login/
    │       │   │   └── route.ts
    │       │   ├── admin-logout/
    │       │   │   └── route.ts
    │       │   └── it-unlock/
    │       │       └── route.ts
    │       ├── public/
    │       │   ├── validate-registration/
    │       │   │   └── route.ts
    │       │   └── remedial/
    │       │       └── [token]/
    │       │           └── route.ts
    │       ├── exam/
    │       │   ├── start/
    │       │   │   └── route.ts
    │       │   ├── start-remedial/
    │       │   │   └── route.ts
    │       │   └── [attemptId]/
    │       │       ├── state/
    │       │       │   └── route.ts
    │       │       ├── answer/
    │       │       │   └── route.ts
    │       │       ├── violation/
    │       │       │   └── route.ts
    │       │       └── finish/
    │       │           └── route.ts
    │       ├── results/
    │       │   └── [attemptId]/
    │       │       ├── route.ts
    │       │       └── download-pdf/
    │       │           └── route.ts
    │       └── admin/
    │           ├── categories/
    │           │   └── route.ts
    │           ├── questions/
    │           │   ├── route.ts
    │           │   └── [questionId]/
    │           │       ├── route.ts
    │           │       └── status/
    │           │           └── route.ts
    │           ├── upload-media/
    │           │   └── route.ts
    │           ├── participants/
    │           │   └── route.ts
    │           ├── remedial/
    │           │   └── authorize/
    │           │       └── route.ts
    │           └── pdf-templates/
    │               ├── route.ts
    │               ├── [templateId]/
    │               │   └── publish/
    │               │       └── route.ts
    │               └── preview/
    │                   └── route.ts
    │
    ├── components/
    │   ├── ui/
    │   │   ├── button.tsx
    │   │   ├── input.tsx
    │   │   ├── modal.tsx
    │   │   ├── badge.tsx
    │   │   └── toast.tsx
    │   ├── dashboard/
    │   │   ├── admin-sidebar.tsx
    │   │   ├── stats-overview.tsx
    │   │   ├── participant-table.tsx
    │   │   └── remedial-link-modal.tsx
    │   ├── questions/
    │   │   ├── question-form.tsx
    │   │   ├── answer-weight-editor.tsx
    │   │   ├── media-dropzone.tsx
    │   │   └── question-preview-modal.tsx
    │   ├── exams/
    │   │   ├── registration-form.tsx
    │   │   ├── duplicate-warning-card.tsx
    │   │   ├── instruction-checklist.tsx
    │   │   ├── exam-cardboard.tsx
    │   │   ├── dual-timer-bar.tsx
    │   │   └── violation-guard.tsx
    │   ├── reports/
    │   │   ├── result-summary-card.tsx
    │   │   ├── ai-narrative-panel.tsx
    │   │   └── template-editor-pane.tsx
    │   └── charts/
    │       ├── subcategory-bar-chart.tsx
    │       ├── category-radar-chart.tsx
    │       └── disc-quadrant-chart.tsx
    │
    ├── modules/
    │   ├── authentication/
    │   │   ├── admin-auth.service.ts
    │   │   ├── lockout.policy.ts
    │   │   └── session.guard.ts
    │   ├── participants/
    │   │   ├── registration.service.ts
    │   │   └── duplicate-check.service.ts
    │   ├── question-bank/
    │   │   ├── question.service.ts
    │   │   └── media.service.ts
    │   ├── categories/
    │   │   └── taxonomy.service.ts
    │   ├── exams/
    │   │   ├── snapshot.builder.ts
    │   │   ├── exam-start.service.ts
    │   │   └── remedial.service.ts
    │   ├── exam-engine/
    │   │   ├── timer.service.ts
    │   │   ├── answer-submission.service.ts
    │   │   ├── violation.service.ts
    │   │   └── exam-completion.service.ts
    │   ├── scoring-engine/
    │   │   ├── competency.scorer.ts
    │   │   ├── psychology.scorer.ts
    │   │   ├── disc.normalizer.ts
    │   │   └── scoring.orchestrator.ts
    │   ├── results/
    │   │   └── result.service.ts
    │   ├── reports/
    │   │   ├── chart-fragment.builder.ts
    │   │   ├── pdf-generator.service.ts
    │   │   └── report-pipeline.worker.ts
    │   ├── pdf-designer/
    │   │   ├── template.service.ts
    │   │   └── placeholder.renderer.ts
    │   └── ai-analysis/
    │       ├── prompt.builder.ts
    │       └── ai-analyzer.service.ts
    │
    ├── lib/
    │   ├── db/
    │   │   ├── pool.ts
    │   │   └── transaction.ts
    │   ├── cloudinary/
    │   │   └── client.ts
    │   ├── google-drive/
    │   │   └── client.ts
    │   ├── gemini/
    │   │   └── client.ts
    │   ├── openrouter/
    │   │   └── client.ts
    │   ├── pdf/
    │   │   ├── playwright-renderer.ts
    │   │   └── qpdf-encryptor.ts
    │   ├── security/
    │   │   ├── password.ts
    │   │   ├── crypto-token.ts
    │   │   └── html-escape.ts
    │   └── validators/
    │       ├── participant.schema.ts
    │       ├── question.schema.ts
    │       └── ai-output.schema.ts
    │
    ├── database/
    │   ├── migrations/
    │   │   └── 001_initial_schema.sql
    │   ├── seeds/
    │   │   ├── 001_areas.sql
    │   │   ├── 002_categories_subcats.sql
    │   │   ├── 003_question_types.sql
    │   │   └── 004_default_pdf_template.sql
    │   └── queries/
    │       ├── admin.queries.ts
    │       ├── participant.queries.ts
    │       ├── question.queries.ts
    │       ├── exam.queries.ts
    │       ├── remedial.queries.ts
    │       ├── result.queries.ts
    │       └── template.queries.ts
    │
    └── types/
        ├── db.types.ts
        ├── snapshot.types.ts
        ├── scoring.types.ts
        ├── ai.types.ts
        └── api.types.ts
```

---

## 2. Rincian Tanggung Jawab per Lapisan

### 2.1 `src/lib/` — Infrastruktur & Integrasi Eksternal

| Direktori / File | Tanggung Jawab Utama |
|---|---|
| `lib/db/pool.ts` | Menginisialisasi satu instance `pg.Pool` terpusat ke Neon PostgreSQL (SSL aktif, konfigurasi `max`, `idleTimeoutMillis`, `connectionTimeoutMillis`). |
| `lib/db/transaction.ts` | Menyediakan helper `withTransaction(async (client: PoolClient) => ...)` yang otomatis menjalankan `BEGIN`, `COMMIT`, dan `ROLLBACK` serta melepaskan koneksi kembali ke pool. |
| `lib/cloudinary/client.ts` | Mengelola unggahan gambar soal/opsi ke Cloudinary dan validasi URL media. |
| `lib/google-drive/client.ts` | Mengunggah file PDF terenkripsi ke folder privat Google Drive API v3 dan membaca stream file saat diunduh. |
| `lib/gemini/client.ts` | Klien utama Google Gemini SDK untuk generasi analisis kualitatif terstruktur. |
| `lib/openrouter/client.ts` | Klien cadangan (*fallback*) OpenRouter API apabila layanan Gemini mengalami gangguan atau *rate limit*. |
| `lib/pdf/playwright-renderer.ts` | Menjalankan Chromium headless via Playwright untuk merender gabungan HTML + CSS menjadi buffer PDF A4. |
| `lib/pdf/qpdf-encryptor.ts` | Mengeksekusi binary `qpdf` untuk mengenkripsi file PDF dengan standar **AES-256** menggunakan password tanggal lahir peserta (`ddmmyyyy`). |
| `lib/security/password.ts` | Hashing dan verifikasi password admin menggunakan **Argon2id**. |
| `lib/security/crypto-token.ts` | Pembuatan token acak 32-byte untuk link remidi 24 jam, hashing SHA-256 token, dan pembuatan tanda tangan HMAC kode verifikasi PDF. |
| `lib/security/html-escape.ts` | Fungsi sanitasi string agar tidak terjadi injeksi HTML saat placeholder diisi ke template PDF. |
| `lib/validators/*.ts` | Validasi skema input menggunakan **Zod** (format 16 digit NIK, telepon `62...`, struktur soal, dan validasi JSON output AI). |

---

### 2.2 `src/database/` — SQL Native & Kueri Terparameterisasi

Seluruh komunikasi ke PostgreSQL wajib menggunakan **parameterized queries (`$1`, `$2`, ...)** dan tidak pernah menggabungkan input pengguna secara langsung ke string SQL.

* `database/migrations/001_initial_schema.sql`: Berisi seluruh DDL final dari **Bagian 1**.
* `database/seeds/`: Data awal master wilayah (`db_area`), 4 kategori beserta 13 subkategorinya (`db_category`, `db_subcat`), jenis soal (`MC`, `ESSAY`, `LIKERT`, `DISC`), serta template PDF bawaan (`v1`).
* `database/queries/*.queries.ts`: Kumpulan fungsi SQL murni yang menerima `Pool` atau `PoolClient` (sehingga dapat dijalankan baik di luar maupun di dalam transaksi `withTransaction`).

---

### 2.3 `src/modules/` — Logika Bisnis Inti

1. **`authentication/`**:
   * `lockout.policy.ts`: Menerapkan aturan penguncian bertahap admin (gagal 3x $\rightarrow$ 5 menit, gagal 3x lagi $\rightarrow$ 10 menit, gagal 3x lagi $\rightarrow$ permanen) dan verifikasi kode `UNLOCK` pada halaman IT.
   * `session.guard.ts`: Middleware/helper verifikasi cookie sesi admin (HTTP-only, Secure, SameSite) dan token sesi ujian peserta.

2. **`participants/`**:
   * `duplicate-check.service.ts`: Mengecek apakah NIK, telepon, atau email sudah pernah terdaftar di `db_participants`. Jika sudah ada, mengembalikan payload kontak WhatsApp admin untuk permintaan ulang PDF.

3. **`exams/`**:
   * `snapshot.builder.ts`: Membekukan soal berstatus `PUBLISHED`, opsi jawaban, bobot `subcat_value`, durasi global & per soal, serta `scoring_config_snapshot` beserta hash SHA-256.
   * `exam-start.service.ts`: Transaksi ACID pembuatan `db_participants` + `db_exam_claims` + `db_participant_test` (`attempt_number = 1`).
   * `remedial.service.ts`: Pembuatan otorisasi remidi berdurasi **24 jam** oleh admin, validasi token remidi, dan transaksi `SELECT ... FOR UPDATE` saat peserta memulai tes remidi (`CONSUMED`).

4. **`exam-engine/`**:
   * `timer.service.ts`: Menghitung sisa waktu sah berdasarkan `NOW()` server, `deadline_at`, dan akumulasi penalti di `db_exam_violations`.
   * `answer-submission.service.ts`: Menyimpan jawaban ke `db_participants_answer` dengan aturan **sekali jawab per soal** (menolak revisi untuk soal yang sudah dijawab atau dilewati).
   * `violation.service.ts`: Mencatat event pelanggaran *best-effort* secara idempoten (`UNIQUE (attempt_id, client_event_id)`).
   * `exam-completion.service.ts`: Menutup `attempt_status` menjadi `'COMPLETED'`/`'EXPIRED'`, memicu `scoring.orchestrator.ts`, lalu memicu `report-pipeline.worker.ts` secara asinkron.

5. **`scoring-engine/`**:
   * Mengimplementasikan seluruh rumus pada **Bagian 2** (`final_score`, `status_result`, `subcat_score`, `category_score`, normalisasi DISC `100.00%`, serta skala diagram kuadran `45%`).

6. **`pdf-designer/`, `ai-analysis/`, dan `reports/`**:
   * `placeholder.renderer.ts`: Mengganti seluruh placeholder datar `{{...}}` dengan data ter-escape dan memvalidasi tidak ada placeholder tersisa sebelum render.
   * `ai-analyzer.service.ts`: Mengirim seluruh riwayat jawaban (`is_scored = TRUE` maupun `FALSE`) ke Gemini SDK (fallback OpenRouter API) dan memvalidasi hasil JSON.
   * `report-pipeline.worker.ts`: Orkestrasi asinkron di proses Node.js: Jalankan AI $\rightarrow$ Render HTML/CSS via Playwright $\rightarrow$ Enkripsi AES-256 via `qpdf` (`ddmmyyyy`) $\rightarrow$ Unggah ke Google Drive API v3 $\rightarrow$ Update `db_results`.

---

### 2.4 `src/app/` — Routing & Antarmuka Pengguna

* **`(public)/`**:
  * `/` & `/register`: Halaman utama & form registrasi peserta (validasi 6 digit awal NIK ke `db_area` & cek duplikasi).
  * `/remedial/[token]`: Halaman pendaftaran otomatis terisi untuk peserta remidi dengan link 24 jam yang masih aktif.
  * `/instructions`: Halaman tata tertib ujian dengan kotak *checklist* wajib sebelum tombol **Mulai** aktif.
  * `/verify/[code]`: Halaman publik verifikasi keaslian sertifikat/laporan PDF berdasarkan kode verifikasi.
* **`(auth)/`**:
  * `/admin`: Login admin dengan perlindungan *lockout* bertahap.
  * `/it-unlock`: Halaman terpisah untuk membuka kunci permanen akun admin menggunakan kode `UNLOCK`.
* **`(admin)/dashboard/`**:
  * Manajemen bank soal (CRUD pertanyaan/opsi, drag-and-drop gambar Cloudinary, bobot subkategori, preview soal, dan toggle soft-delete `PUBLISHED`/`ARCHIVED`).
  * Daftar seluruh *attempt* peserta (menampilkan riwayat ujian awal maupun remidi berdampingan, skor akhir, tautan buka PDF, serta tombol **Remidi** untuk status `GAGAL`).
  * Editor template PDF HTML & CSS dengan live preview.
* **`exam/[attemptId]/`**:
  * Antarmuka ujian *cardboard* tanpa tombol kembali (*no-back*), *timer* ganda (global + per soal dengan *auto-advance*), dan pemantau pelanggaran *best-effort*.
* **`(participant)/result/[attemptId]/`**:
  * Halaman hasil akhir dengan grafik **Apache ECharts**, ringkasan analisis AI, dan tombol unduh PDF terenkripsi.