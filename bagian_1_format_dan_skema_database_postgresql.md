# Bagian 1 — Format dan Skema Database PostgreSQL

Dokumen ini menetapkan rancangan final database PostgreSQL untuk aplikasi ujian berbasis kompetensi dan psikologi (DISC) yang dilengkapi analisis AI serta pembuatan laporan PDF. Skema menggunakan SQL native tanpa ORM, dengan penamaan tabel dan kolom menggunakan `snake_case`.

---

## 1. Prinsip Rancangan dan Aturan Bisnis

1. **Ujian Awal Sekali Seumur Hidup:** Satu peserta hanya dapat mengikuti ujian awal satu kali di seluruh sistem berdasarkan keunikan NIK, nomor telepon, dan email (dikunci melalui `db_exam_claims`).
2. **Remedial Fleksibel via Link 24 Jam:** Ujian remedial hanya dapat dilakukan jika ada otorisasi admin untuk hasil yang belum lulus (`status_result = 'GAGAL'`). Admin dapat menerbitkan otorisasi remidi berulang kali jika diperlukan (kebijakan batas remidi diatur admin di luar sistem). Setiap otorisasi menghasilkan tautan khusus yang berlaku **24 jam**, tidak dapat di-*revoke* manual oleh admin, dan otomatis hangus (`CONSUMED`) begitu peserta menekan **Mulai** ujian remidi.
3. **Riwayat Terpisah per Attempt:** Setiap pelaksanaan ujian direpresentasikan sebagai satu `attempt_id` (UUID) di `db_participant_test` dan satu hasil di `db_results`. Riwayat remedial tidak menimpa atau menyembunyikan hasil sebelumnya.
4. **Pemisahan `is_scored` vs Subkategori & AI:**
   - `final_score` hanya dihitung dari soal dengan `is_scored = TRUE` (soal kompetensi).
   - Soal dengan `is_scored = FALSE` (soal psikologi, Likert, DISC, dan esai) tidak masuk ke `final_score`, tetapi **tetap** memengaruhi nilai/persentase subkategori (`db_analizes`), kategori (`db_analizes`), dan profil DISC (`db_disc`).
   - Seluruh jawaban peserta—baik `is_scored = TRUE` maupun `FALSE`—digunakan oleh AI sebagai konteks analisis kualitatif.
5. **AI Hanya Menghasilkan Narasi:** AI menghasilkan narasi kekuatan, kelemahan, profil DISC, dan rekomendasi pengembangan. AI tidak mengubah nilai numerik (`final_score`) maupun status kelulusan (`status_result`).
6. **Immutability via Snapshot:** Soal, opsi, bobot, konfigurasi timer, ambang lulus, dan versi rumus di-*snapshot* ke `db_participant_test` ketika ujian dimulai agar hasil historis tidak berubah saat admin mengedit bank soal atau pengaturan.
7. **Pendaftaran & Instruksi:** Data peserta baru disimpan ke `db_participants` ketika peserta menyetujui instruksi dan menekan **Mulai**. Sebelum itu, data pendaftaran hanya berada di sisi frontend/session dan wajib divalidasi ulang di server saat mulai.
8. **Otoritas Waktu Server:** Timestamp server (`TIMESTAMPTZ`) menjadi acuan mutlak untuk timer global, timer per soal, masa berlaku link remidi, dan penalti pelanggaran. Timer di browser hanya tampilan visual.
9. **Validasi Wilayah NIK:** Enam digit pertama NIK divalidasi terhadap `db_area` saat pendaftaran; tidak disimpan sebagai Foreign Key di `db_participants`.

---

## 2. Diagram Relasi Utama

```text
db_category 1---N db_subcat
db_question_type 1---N db_question
db_question 1---N db_answer
db_answer N---N db_subcat (melalui db_answer_value)

db_participants 1---1 db_exam_claims
db_participants 1---N db_participant_test
db_participants 1---N db_remedial_authorizations

db_participant_test 1---N db_participants_answer
db_participant_test 1---N db_exam_violations
db_participant_test 1---1 db_results

db_results 1---N db_analizes
db_results 1---1 db_disc
db_pdf_templates 1---N db_results
```

---

## 3. Daftar Tabel (18 Tabel)

| No | Tabel | Fungsi |
|---|---|---|
| 1 | `db_admin` | Akun tunggal admin dan status penguncian login bertahap |
| 2 | `db_area` | Referensi kode wilayah 6 digit untuk validasi awal NIK |
| 3 | `db_category` | Kategori penilaian (Kompetensi & Psikologi/DISC) |
| 4 | `db_subcat` | Subkategori/indikator penilaian |
| 5 | `db_question_type` | Jenis soal (`MC`, `ESSAY`, `LIKERT`, `DISC`, dll.) |
| 6 | `db_question` | Bank soal, penanda `is_scored`, dan status publikasi (`DRAFT`/`PUBLISHED`/`ARCHIVED`) |
| 7 | `db_answer` | Opsi jawaban dan penanda kunci benar (`is_true`) |
| 8 | `db_answer_value` | Bobot nilai opsi terhadap satu atau lebih subkategori |
| 9 | `db_participants` | Identitas lengkap peserta |
| 10 | `db_exam_claims` | Pengunci hak ujian awal satu kali per peserta |
| 11 | `db_remedial_authorizations` | Otorisasi remedial oleh admin dengan masa berlaku 24 jam |
| 12 | `db_participant_test` | Setiap pelaksanaan ujian (`attempt`), konfigurasi timer, dan *snapshot* |
| 13 | `db_participants_answer` | Jawaban peserta per soal dalam sebuah *attempt* (sekali jawab, tanpa revisi) |
| 14 | `db_exam_violations` | Catatan pelanggaran *best-effort* dan penalti waktu |
| 15 | `db_pdf_templates` | Versi template PDF HTML dan CSS |
| 16 | `db_results` | Hasil akhir numerik, status kelulusan, status AI, dan metadata file PDF |
| 17 | `db_analizes` | Persentase capaian per kategori dan subkategori per hasil |
| 18 | `db_disc` | Empat dimensi persentase DISC (`D`, `I`, `S`, `C`) per hasil |

---

## 4. DDL PostgreSQL Final

> Jalankan pada database PostgreSQL (Neon PostgreSQL) yang kosong. Untuk pembaruan pada database yang sudah berisi data produksi, gunakan skrip migrasi bertahap.

```sql
-- ============================================================================
-- 1. EKSTENSI, TIPE DATA ENUM & FUNGSI TRIGGER
-- ============================================================================
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TYPE content_status AS ENUM ('DRAFT', 'PUBLISHED', 'ARCHIVED');
CREATE TYPE attempt_status AS ENUM ('IN_PROGRESS', 'COMPLETED', 'EXPIRED', 'CANCELLED');
CREATE TYPE process_status AS ENUM ('PENDING', 'PROCESSING', 'COMPLETED', 'FAILED');
CREATE TYPE remedial_status AS ENUM ('AUTHORIZED', 'CONSUMED', 'REVOKED', 'EXPIRED');
CREATE TYPE pdf_template_status AS ENUM ('DRAFT', 'PUBLISHED', 'ARCHIVED');

CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ============================================================================
-- 2. TABEL MASTER & REFERENSI
-- ============================================================================
CREATE TABLE db_admin (
    admin_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    admin_slot SMALLINT NOT NULL DEFAULT 1 UNIQUE CHECK (admin_slot = 1),
    username VARCHAR(80) NOT NULL UNIQUE CHECK (BTRIM(username) <> ''),
    password_hash TEXT NOT NULL CHECK (BTRIM(password_hash) <> ''),
    failed_attempts SMALLINT NOT NULL DEFAULT 0 CHECK (failed_attempts BETWEEN 0 AND 3),
    lock_stage SMALLINT NOT NULL DEFAULT 0 CHECK (lock_stage BETWEEN 0 AND 3),
    locked_until TIMESTAMPTZ,
    is_permanently_locked BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (NOT is_permanently_locked OR locked_until IS NULL)
);

CREATE TABLE db_area (
    area_id VARCHAR(6) PRIMARY KEY CHECK (area_id ~ '^[0-9]{6}$'),
    area_name VARCHAR(120) NOT NULL CHECK (BTRIM(area_name) <> ''),
    regency_name VARCHAR(120),
    province_name VARCHAR(120) NOT NULL CHECK (BTRIM(province_name) <> ''),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE db_category (
    category_id VARCHAR(6) PRIMARY KEY CHECK (BTRIM(category_id) <> ''),
    category_name VARCHAR(120) NOT NULL UNIQUE CHECK (BTRIM(category_name) <> ''),
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE db_subcat (
    subcat_id VARCHAR(8) PRIMARY KEY CHECK (BTRIM(subcat_id) <> ''),
    category_id VARCHAR(6) NOT NULL REFERENCES db_category(category_id) ON DELETE RESTRICT,
    subcat_name VARCHAR(120) NOT NULL CHECK (BTRIM(subcat_name) <> ''),
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (category_id, subcat_name),
    UNIQUE (subcat_id, category_id)
);

CREATE TABLE db_question_type (
    type_id VARCHAR(20) PRIMARY KEY CHECK (BTRIM(type_id) <> ''),
    type_name VARCHAR(80) NOT NULL UNIQUE CHECK (BTRIM(type_name) <> ''),
    description TEXT
);

CREATE TABLE db_question (
    question_id VARCHAR(10) PRIMARY KEY CHECK (BTRIM(question_id) <> ''),
    type_id VARCHAR(20) NOT NULL REFERENCES db_question_type(type_id) ON DELETE RESTRICT,
    question_text TEXT NOT NULL CHECK (BTRIM(question_text) <> ''),
    question_url TEXT,
    is_scored BOOLEAN NOT NULL DEFAULT TRUE,
    content_status content_status NOT NULL DEFAULT 'DRAFT',
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (NOT (LOWER(type_id) = 'essay' AND is_scored = TRUE)),
    CHECK (content_status <> 'PUBLISHED' OR published_at IS NOT NULL)
);

CREATE TABLE db_answer (
    answer_id VARCHAR(15) PRIMARY KEY CHECK (BTRIM(answer_id) <> ''),
    question_id VARCHAR(10) NOT NULL REFERENCES db_question(question_id) ON DELETE RESTRICT,
    answer_text TEXT NOT NULL CHECK (BTRIM(answer_text) <> ''),
    answer_url TEXT,
    is_true BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (question_id, answer_id)
);

CREATE TABLE db_answer_value (
    answer_id VARCHAR(15) NOT NULL REFERENCES db_answer(answer_id) ON DELETE CASCADE,
    subcat_id VARCHAR(8) NOT NULL REFERENCES db_subcat(subcat_id) ON DELETE RESTRICT,
    subcat_value NUMERIC(8,2) NOT NULL CHECK (subcat_value >= 0),
    PRIMARY KEY (answer_id, subcat_id)
);

CREATE TABLE db_pdf_templates (
    template_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    version_number INTEGER NOT NULL UNIQUE CHECK (version_number > 0),
    template_name VARCHAR(120) NOT NULL CHECK (BTRIM(template_name) <> ''),
    html_template TEXT NOT NULL CHECK (BTRIM(html_template) <> ''),
    css_template TEXT NOT NULL,
    template_status pdf_template_status NOT NULL DEFAULT 'DRAFT',
    created_by UUID REFERENCES db_admin(admin_id) ON DELETE RESTRICT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ,
    CHECK (template_status <> 'PUBLISHED' OR published_at IS NOT NULL)
);

-- ============================================================================
-- 3. TABEL PESERTA, KLAIM & OTORISASI REMEDIAL
-- ============================================================================
CREATE SEQUENCE participant_number_seq START 1;

CREATE TABLE db_participants (
    participant_id VARCHAR(10) PRIMARY KEY
        DEFAULT ('P' || LPAD(nextval('participant_number_seq')::TEXT, 9, '0')),
    full_name VARCHAR(160) NOT NULL CHECK (BTRIM(full_name) <> ''),
    nik VARCHAR(16) NOT NULL UNIQUE CHECK (nik ~ '^[0-9]{16}$'),
    birth_place VARCHAR(120) NOT NULL CHECK (BTRIM(birth_place) <> ''),
    full_address TEXT NOT NULL CHECK (BTRIM(full_address) <> ''),
    phone VARCHAR(14) NOT NULL UNIQUE CHECK (phone ~ '^62[0-9]{9,12}$'),
    email VARCHAR(254) NOT NULL CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    date_of_birth DATE NOT NULL CHECK (date_of_birth <= CURRENT_DATE),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE db_exam_claims (
    participant_id VARCHAR(10) PRIMARY KEY
        REFERENCES db_participants(participant_id) ON DELETE RESTRICT,
    initial_attempt_id UUID UNIQUE,
    claimed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE db_remedial_authorizations (
    authorization_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    participant_id VARCHAR(10) NOT NULL
        REFERENCES db_participants(participant_id) ON DELETE RESTRICT,
    previous_attempt_id UUID NOT NULL,
    authorized_by UUID NOT NULL REFERENCES db_admin(admin_id) ON DELETE RESTRICT,
    reason TEXT NOT NULL CHECK (BTRIM(reason) <> ''),
    authorization_status remedial_status NOT NULL DEFAULT 'AUTHORIZED',
    expires_at TIMESTAMPTZ NOT NULL,
    consumed_attempt_id UUID UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    consumed_at TIMESTAMPTZ,
    CHECK (expires_at > created_at),
    CHECK (
        (authorization_status = 'CONSUMED' AND consumed_attempt_id IS NOT NULL AND consumed_at IS NOT NULL)
        OR
        (authorization_status <> 'CONSUMED' AND consumed_attempt_id IS NULL AND consumed_at IS NULL)
    )
);

-- ============================================================================
-- 4. TABEL PELAKSANAAN UJIAN & JAWABAN
-- ============================================================================
CREATE TABLE db_participant_test (
    attempt_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    participant_id VARCHAR(10) NOT NULL
        REFERENCES db_participants(participant_id) ON DELETE RESTRICT,
    previous_attempt_id UUID REFERENCES db_participant_test(attempt_id) ON DELETE RESTRICT,
    remedial_authorization_id UUID UNIQUE
        REFERENCES db_remedial_authorizations(authorization_id) ON DELETE RESTRICT,
    attempt_number SMALLINT NOT NULL CHECK (attempt_number >= 1),
    attempt_status attempt_status NOT NULL DEFAULT 'IN_PROGRESS',
    started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deadline_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    global_duration_seconds INTEGER NOT NULL CHECK (global_duration_seconds > 0),
    question_duration_seconds INTEGER NOT NULL CHECK (question_duration_seconds > 0),
    violation_penalty_seconds INTEGER NOT NULL DEFAULT 300 CHECK (violation_penalty_seconds >= 0),
    formula_version VARCHAR(40) NOT NULL CHECK (BTRIM(formula_version) <> ''),
    scoring_config_snapshot JSONB NOT NULL CHECK (jsonb_typeof(scoring_config_snapshot) = 'object'),
    question_snapshot JSONB NOT NULL CHECK (jsonb_typeof(question_snapshot) = 'object'),
    snapshot_hash CHAR(64) NOT NULL CHECK (snapshot_hash ~ '^[a-f0-9]{64}$'),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (participant_id, attempt_number),
    CHECK (deadline_at > started_at),
    CHECK (completed_at IS NULL OR completed_at >= started_at),
    CHECK (
        (attempt_status = 'COMPLETED' AND completed_at IS NOT NULL)
        OR
        (attempt_status <> 'COMPLETED')
    ),
    CHECK (
        (attempt_number = 1 AND previous_attempt_id IS NULL AND remedial_authorization_id IS NULL)
        OR
        (attempt_number > 1 AND previous_attempt_id IS NOT NULL AND remedial_authorization_id IS NOT NULL)
    )
);

ALTER TABLE db_remedial_authorizations
    ADD CONSTRAINT fk_remedial_previous_attempt
    FOREIGN KEY (previous_attempt_id) REFERENCES db_participant_test(attempt_id) ON DELETE RESTRICT,
    ADD CONSTRAINT fk_remedial_consumed_attempt
    FOREIGN KEY (consumed_attempt_id) REFERENCES db_participant_test(attempt_id) ON DELETE RESTRICT;

ALTER TABLE db_exam_claims
    ADD CONSTRAINT fk_exam_claim_initial_attempt
    FOREIGN KEY (initial_attempt_id) REFERENCES db_participant_test(attempt_id) ON DELETE RESTRICT;

CREATE TABLE db_participants_answer (
    attempt_id UUID NOT NULL
        REFERENCES db_participant_test(attempt_id) ON DELETE CASCADE,
    question_id VARCHAR(10) NOT NULL
        REFERENCES db_question(question_id) ON DELETE RESTRICT,
    answer_id VARCHAR(15),
    essay_answer TEXT,
    is_true BOOLEAN,
    answered_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (attempt_id, question_id),
    CHECK (
        (answer_id IS NOT NULL AND essay_answer IS NULL)
        OR
        (answer_id IS NULL AND essay_answer IS NOT NULL AND BTRIM(essay_answer) <> '')
    ),
    CHECK (is_true IS NULL OR answer_id IS NOT NULL),
    FOREIGN KEY (question_id, answer_id)
        REFERENCES db_answer(question_id, answer_id) ON DELETE RESTRICT
);

CREATE TABLE db_exam_violations (
    violation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attempt_id UUID NOT NULL
        REFERENCES db_participant_test(attempt_id) ON DELETE CASCADE,
    client_event_id UUID NOT NULL,
    violation_type VARCHAR(40) NOT NULL CHECK (BTRIM(violation_type) <> ''),
    penalty_seconds INTEGER NOT NULL DEFAULT 300 CHECK (penalty_seconds >= 0),
    occurred_at_client TIMESTAMPTZ,
    received_at_server TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata JSONB NOT NULL DEFAULT '{}'::JSONB CHECK (jsonb_typeof(metadata) = 'object'),
    UNIQUE (attempt_id, client_event_id)
);

-- ============================================================================
-- 5. TABEL HASIL, ANALISIS & LAPORAN PDF
-- ============================================================================
CREATE TABLE db_results (
    result_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attempt_id UUID NOT NULL UNIQUE
        REFERENCES db_participant_test(attempt_id) ON DELETE RESTRICT,
    final_score NUMERIC(5,2) CHECK (final_score IS NULL OR final_score BETWEEN 0 AND 100),
    status_result VARCHAR(10) CHECK (status_result IS NULL OR status_result IN ('LULUS', 'GAGAL')),
    scoring_status process_status NOT NULL DEFAULT 'PENDING',
    ai_analysis JSONB CHECK (ai_analysis IS NULL OR jsonb_typeof(ai_analysis) = 'object'),
    ai_analysis_status process_status NOT NULL DEFAULT 'PENDING',
    ai_model VARCHAR(120),
    ai_prompt_version VARCHAR(40),
    ai_generated_at TIMESTAMPTZ,
    pdf_template_id BIGINT REFERENCES db_pdf_templates(template_id) ON DELETE RESTRICT,
    pdf_storage_key TEXT,
    pdf_hash CHAR(64) CHECK (pdf_hash IS NULL OR pdf_hash ~ '^[a-f0-9]{64}$'),
    pdf_generated_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (
        (final_score IS NULL AND status_result IS NULL)
        OR
        (final_score IS NOT NULL AND status_result IS NOT NULL)
    ),
    CHECK (
        (pdf_storage_key IS NULL AND pdf_hash IS NULL AND pdf_generated_at IS NULL)
        OR
        (pdf_storage_key IS NOT NULL AND pdf_hash IS NOT NULL AND pdf_generated_at IS NOT NULL)
    )
);

CREATE TABLE db_analizes (
    analize_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    result_id UUID NOT NULL
        REFERENCES db_results(result_id) ON DELETE CASCADE,
    category_id VARCHAR(6) REFERENCES db_category(category_id) ON DELETE RESTRICT,
    subcat_id VARCHAR(8) REFERENCES db_subcat(subcat_id) ON DELETE RESTRICT,
    score_point NUMERIC(5,2) NOT NULL CHECK (score_point BETWEEN 0 AND 100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (
        (category_id IS NOT NULL AND subcat_id IS NULL)
        OR
        (category_id IS NULL AND subcat_id IS NOT NULL)
    )
);

CREATE TABLE db_disc (
    disc_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    result_id UUID NOT NULL UNIQUE
        REFERENCES db_results(result_id) ON DELETE CASCADE,
    d_percent NUMERIC(5,2) NOT NULL CHECK (d_percent BETWEEN 0 AND 100),
    i_percent NUMERIC(5,2) NOT NULL CHECK (i_percent BETWEEN 0 AND 100),
    s_percent NUMERIC(5,2) NOT NULL CHECK (s_percent BETWEEN 0 AND 100),
    c_percent NUMERIC(5,2) NOT NULL CHECK (c_percent BETWEEN 0 AND 100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (d_percent + i_percent + s_percent + c_percent = 100.00)
);

-- ============================================================================
-- 6. INDEKS UNIK & PERFORMA
-- ============================================================================
CREATE UNIQUE INDEX uq_db_participants_email_lower
    ON db_participants (LOWER(BTRIM(email)));

CREATE UNIQUE INDEX uq_one_published_pdf_template
    ON db_pdf_templates (template_status)
    WHERE template_status = 'PUBLISHED';

CREATE UNIQUE INDEX uq_analize_result_category
    ON db_analizes (result_id, category_id)
    WHERE category_id IS NOT NULL;

CREATE UNIQUE INDEX uq_analize_result_subcat
    ON db_analizes (result_id, subcat_id)
    WHERE subcat_id IS NOT NULL;

-- Aktifkan baris di bawah HANYA jika 1 soal pilihan ganda hanya memiliki 1 kunci benar:
-- CREATE UNIQUE INDEX uq_db_answer_one_true_per_question
--     ON db_answer (question_id) WHERE is_true = TRUE;

CREATE INDEX idx_db_subcat_category ON db_subcat (category_id);
CREATE INDEX idx_db_question_status_type ON db_question (content_status, type_id);
CREATE INDEX idx_db_answer_question ON db_answer (question_id);
CREATE INDEX idx_db_answer_value_subcat ON db_answer_value (subcat_id);
CREATE INDEX idx_remedial_participant_status ON db_remedial_authorizations (participant_id, authorization_status);
CREATE INDEX idx_attempt_participant ON db_participant_test (participant_id, started_at DESC);
CREATE INDEX idx_attempt_status_deadline ON db_participant_test (attempt_status, deadline_at);
CREATE INDEX idx_answers_attempt ON db_participants_answer (attempt_id);
CREATE INDEX idx_exam_violations_attempt ON db_exam_violations (attempt_id, received_at_server);
CREATE INDEX idx_results_created ON db_results (created_at DESC);
CREATE INDEX idx_results_pending_jobs ON db_results (scoring_status, ai_analysis_status)
    WHERE scoring_status <> 'COMPLETED' OR ai_analysis_status <> 'COMPLETED';
CREATE INDEX idx_analizes_result ON db_analizes (result_id);

-- ============================================================================
-- 7. PEMASANGAN TRIGGER UPDATED_AT
-- ============================================================================
CREATE TRIGGER trg_db_admin_updated_at BEFORE UPDATE ON db_admin FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER trg_db_area_updated_at BEFORE UPDATE ON db_area FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER trg_db_category_updated_at BEFORE UPDATE ON db_category FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER trg_db_subcat_updated_at BEFORE UPDATE ON db_subcat FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER trg_db_question_updated_at BEFORE UPDATE ON db_question FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER trg_db_participants_updated_at BEFORE UPDATE ON db_participants FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER trg_db_participants_answer_updated_at BEFORE UPDATE ON db_participants_answer FOR EACH ROW EXECUTE FUNCTION set_updated_at();
CREATE TRIGGER trg_db_results_updated_at BEFORE UPDATE ON db_results FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## 5. Rincian Struktur Kolom & Aturan Kunci per Tabel

### 5.1 `db_admin`
- `admin_slot` dikunci `= 1` untuk membatasi sistem hanya memiliki satu akun admin aktif.
- `password_hash` menyimpan hash **Argon2id**.
- Penguncian login: gagal 3x $\rightarrow$ `lock_stage = 1` (lock 5 menit); gagal 3x lagi $\rightarrow$ `lock_stage = 2` (lock 10 menit); gagal 3x lagi $\rightarrow$ `lock_stage = 3` (`is_permanently_locked = TRUE`). Pembukaan kunci permanen dilakukan melalui halaman IT terpisah menggunakan kode `UNLOCK` yang diverifikasi hash-nya.

### 5.2 `db_participants`
- Menyimpan `full_name`, `nik` (16 digit unik), `birth_place`, `full_address`, `phone` (format `62...` unik), `email` (unik *case-insensitive*), dan `date_of_birth` (dipakai sebagai password PDF `ddmmyyyy`).
- Jika peserta yang sudah terdaftar mencoba mendaftar ulang tanpa link remidi yang sah, sistem menolak dan menampilkan tautan WhatsApp admin untuk meminta salinan PDF hasil ujian.

### 5.3 `db_remedial_authorizations`
- Diterbitkan admin untuk `previous_attempt_id` yang berstatus `GAGAL`.
- `expires_at` wajib diisi `created_at + INTERVAL '24 hours'`.
- Token URL remidi (minimal 32 byte acak) di-hash (SHA-256) sebelum disimpan/diverifikasi di server. Admin tidak dapat me-*revoke* link setelah diterbitkan; link otomatis hangus jika melewati 24 jam atau begitu peserta menekan **Mulai** (`authorization_status = 'CONSUMED'`).

### 5.4 `db_participant_test` & `db_participants_answer`
- `question_snapshot` membekukan daftar soal, urutan tampilan, opsi, kunci benar, dan bobot `subcat_value` saat ujian dimulai.
- `scoring_config_snapshot` membekukan ambang lulus (`passing_grade`), pemetaan ID subkategori DISC, dan aturan frasa esai.
- `db_participants_answer` menerapkan konsep sekali jawab per soal: setelah jawaban disimpan dan peserta berpindah ke soal berikutnya (baik manual maupun *auto-advance* saat timer soal habis), endpoint backend menolak perubahan jawaban untuk soal yang sudah dilewati.

### 5.5 `db_results`, `db_analizes`, dan `db_disc`
- `db_results.status_result` diisi oleh backend (`'LULUS'` atau `'GAGAL'`) berdasarkan perbandingan `final_score` terhadap `passing_grade` di `scoring_config_snapshot`.
- `db_analizes` menyimpan dua jenis baris per `result_id` (diatur constraint XOR): skor per `subcat_id` dan skor per `category_id` (berlaku untuk kategori kompetensi maupun psikologi).
- `db_disc` menyimpan empat persentase dimensi (`d_percent`, `i_percent`, `s_percent`, `c_percent`) dengan constraint jumlah tepat `100.00`.

---

## 6. Transaksi Kritis & Keamanan Operasional

1. **Mulai Ujian Awal (Transaksi ACID):** Validasi ulang input & 6 digit NIK terhadap `db_area` $\rightarrow$ `INSERT db_participants` $\rightarrow$ `INSERT db_exam_claims` $\rightarrow$ bentuk `question_snapshot` & `scoring_config_snapshot` $\rightarrow$ `INSERT db_participant_test` (`attempt_number = 1`) $\rightarrow$ `UPDATE db_exam_claims.initial_attempt_id` $\rightarrow$ `COMMIT`.
2. **Mulai Ujian Remidi (Transaksi ACID):** Verifikasi token remidi $\rightarrow$ `SELECT ... FOR UPDATE` pada `db_remedial_authorizations` (pastikan `authorization_status = 'AUTHORIZED'` dan `expires_at > NOW()`) $\rightarrow$ `INSERT db_participant_test` (`attempt_number` berikutnya, `previous_attempt_id`, `remedial_authorization_id`) $\rightarrow$ `UPDATE db_remedial_authorizations` menjadi `'CONSUMED'` $\rightarrow$ `COMMIT`.
3. **Koneksi Driver Neon:** Gunakan WebSocket `Pool` / `Client` dari `pg (node-postgres)` untuk seluruh operasi transaksi multi-langkah (`BEGIN ... COMMIT` dan `FOR UPDATE`).
4. **Penyimpanan Rahasia & PDF:** Kunci enkripsi, *secret* HMAC, token Google Drive, dan API key AI wajib disimpan di *environment variables* server. PDF hasil ujian disimpan di folder privat Google Drive API v3 dan dienkripsi AES-256 (`qpdf`) menggunakan password tanggal lahir `ddmmyyyy`.