# Bagian 2 — Formula Perhitungan dan Spesifikasi Output Laporan

Dokumen ini menetapkan aturan matematis dan logika backend untuk perhitungan skor akhir (`final_score`), skor subkategori (`subcat_score`), skor kategori (`category_score`), normalisasi profil DISC, skala diagram kuadran DISC, kontrak struktur JSON analisis AI, serta aturan injeksi variabel ke template PDF HTML.

Seluruh perhitungan dieksekusi di backend menggunakan driver PostgreSQL native (`pg`) dan bersumber dari *snapshot* yang dibekukan saat ujian dimulai (`db_participant_test.question_snapshot` dan `db_participant_test.scoring_config_snapshot`), bukan dari bank soal aktif.

---

## 1. Sumber Data dan Struktur Snapshot

Saat peserta memulai ujian (`attempt_status = 'IN_PROGRESS'`), sistem membekukan konfigurasi dan butir soal ke dalam `db_participant_test`.

### 1.1 Struktur `scoring_config_snapshot` (JSONB)

```json
{
  "formula_version": "v1.0.0",
  "passing_grade": 75.00,
  "disc_subcat_mapping": {
    "ekstrovert": "SUB00011",
    "introvert": "SUB00012",
    "task_oriented": "SUB00013",
    "people_oriented": "SUB00014"
  },
  "essay_phrase_rules": [
    {
      "question_id": "Q000000020",
      "subcat_id": "SUB00009",
      "max_score": 10.00,
      "phrases": [
        { "keyword": "tanggung jawab", "weight": 5.00 },
        { "keyword": "prosedur keselamatan", "weight": 5.00 }
      ]
    }
  ]
}
```

### 1.2 Struktur `question_snapshot` (JSONB)

```json
{
  "questions": [
    {
      "question_id": "Q000000001",
      "type_id": "MC",
      "is_scored": true,
      "question_text": "Contoh pertanyaan kompetensi",
      "answers": [
        {
          "answer_id": "A000000000001",
          "answer_text": "Pilihan A",
          "is_true": true,
          "values": [
            { "subcat_id": "SUB00001", "subcat_value": 2.00 }
          ]
        }
      ]
    }
  ]
}
```

---

## 2. Perhitungan `final_score` dan `status_result`

**Disimpan ke:** `db_results.final_score` dan `db_results.status_result`

`final_score` mengukur persentase ketepatan jawaban peserta khusus pada butir soal yang berstatus `is_scored = TRUE` (soal kompetensi) dalam `question_snapshot`.

### 2.1 Formula `final_score`

$$
\text{final\_score} = \text{ROUND}\left(
\frac{N_{\text{benar, scored}}}{N_{\text{total, scored}}} \times 100,\ 2
\right)
$$

Keterangan komponen:
* $N_{\text{benar, scored}}$: Jumlah baris pada `db_participants_answer` untuk `attempt_id` terkait di mana `is_true = TRUE` **dan** soal tersebut memiliki `is_scored = TRUE` di dalam `question_snapshot`.
* $N_{\text{total, scored}}$: Jumlah total soal di dalam `question_snapshot.questions` yang memiliki `is_scored = TRUE`.
* **Penanganan Pembagian Nol:** Jika $N_{\text{total, scored}} = 0$, maka `final_score = 0.00`.

### 2.2 Penentuan `status_result`

`status_result` diisi langsung oleh backend (bukan *generated column*) dengan membandingkan `final_score` terhadap `passing_grade` yang tersimpan pada `scoring_config_snapshot`:

| Kondisi | Nilai `db_results.status_result` |
|---|---|
| `final_score >= scoring_config_snapshot.passing_grade` | `'LULUS'` |
| `final_score < scoring_config_snapshot.passing_grade` | `'GAGAL'` |

---

## 3. Perhitungan `subcat_score`

**Disimpan ke:** `db_analizes.score_point` dengan `subcat_id` terisi (`category_id = NULL`).

Setiap subkategori (`subcat_id`) yang muncul pada `question_snapshot` (atau pada `essay_phrase_rules`) dihitung persentase capaiannya berdasarkan jenis soal yang memengaruhinya.

### 3.1 Subkategori dari Soal Kompetensi (`is_scored = TRUE`)

Nilai dihitung dari rasio total bobot `subcat_value` pada kunci jawaban benar yang berhasil dijawab benar oleh peserta terhadap total bobot `subcat_value` dari seluruh kunci jawaban benar pada subkategori tersebut di dalam `question_snapshot`.

$$
\text{subcat\_score}_{\text{scored}} = \text{ROUND}\left(
\frac{
\sum \text{subcat\_value pada kunci benar dari soal } \text{is\_scored} = \text{TRUE} \text{ yang dijawab benar peserta}
}{
\sum \text{subcat\_value pada seluruh kunci benar dari soal } \text{is\_scored} = \text{TRUE} \text{ di snapshot}
} \times 100,\ 2
\right)
$$

Ketentuan:
* Hanya opsi dengan `is_true = TRUE` di `question_snapshot` yang memiliki bobot pembilang dan penyebut.
* Soal yang tidak dijawab atau dijawab salah (`is_true = FALSE`) menyumbang `0` pada pembilang, tetapi tetap dihitung pada penyebut.
* Jika penyebut `= 0`, maka `subcat_score = 0.00`.

### 3.2 Subkategori dari Soal Psikologi, Likert, DISC & Esai (`is_scored = FALSE`)

Pada soal dengan `is_scored = FALSE`, tidak ada konsep benar/salah (`is_true = FALSE` pada opsi pilihan, atau `NULL` pada esai). Nilai dihitung dengan membandingkan poin yang diperoleh peserta terhadap poin maksimum yang mungkin diraih pada setiap soal di `question_snapshot`.

#### A. Soal Pilihan (Likert / DISC / Psikologi Pilihan)

$$
\text{subcat\_score}_{\text{unscored}} = \text{ROUND}\left(
\frac{
\sum_{q \in Q_{\text{sub}}} \text{subcat\_value}(q, \text{opsi\_dipilih\_peserta})
}{
\sum_{q \in Q_{\text{sub}}} \max_{a \in q.\text{answers}} \text{subcat\_value}(q, a)
} \times 100,\ 2
\right)
$$

Ketentuan:
* $Q_{\text{sub}}$ adalah himpunan soal `is_scored = FALSE` dalam `question_snapshot` yang memiliki bobot `subcat_value > 0` untuk `subcat_id` tersebut.
* Jika peserta melewati soal karena waktu habis (*timeout*), poin pembilang untuk soal tersebut adalah `0`, sedangkan penyebut tetap menggunakan nilai maksimum opsi pada soal tersebut.
* Jika penyebut `= 0`, maka `subcat_score = 0.00`.

#### B. Soal Esai (`type_id = 'ESSAY'`, `is_scored = FALSE`)

Jika subkategori memiliki aturan frasa pada `scoring_config_snapshot.essay_phrase_rules`:
* Jawaban teks peserta (`db_participants_answer.essay_answer`) dinormalisasi (*lowercase*, spasi dirapikan).
* Setiap `keyword` yang ditemukan pada teks jawaban menyumbang `weight` (dibatasi maksimal sebesar `max_score` per soal).
* Poin yang didapat dan `max_score` ditambahkan ke pembilang dan penyebut `subcat_score` untuk `subcat_id` terkait.
* Jika soal esai tidak memiliki aturan frasa numerik, jawaban esai tetap disimpan dan diteruskan sepenuhnya ke modul analisis AI.

---

## 4. Perhitungan `category_score`

**Disimpan ke:** `db_analizes.score_point` dengan `category_id` terisi (`subcat_id = NULL`).

`category_score` adalah nilai rata-rata aritmetika dari seluruh `subcat_score` yang berada di bawah `category_id` yang sama (`db_subcat.category_id`) dan aktif diuji pada *attempt* tersebut.

Berlaku untuk **seluruh kategori** yang teruji (baik kategori kompetensi seperti *Pengetahuan Dasar* dan *Driving Skill*, maupun kategori psikologi seperti *Personality* dan *DISC Parameter*):

$$
\text{category\_score} = \text{ROUND}\left(
\frac{
\sum_{s \in S_{\text{cat}}} \text{subcat\_score}(s)
}{
|S_{\text{cat}}|
},\ 2
\right)
$$

Keterangan:
* $S_{\text{cat}}$ adalah daftar subkategori milik `category_id` tersebut yang memiliki record hasil pada `db_analizes` untuk `result_id` yang sama.
* Jika $|S_{\text{cat}}| = 0$, kategori tersebut tidak di-*insert* ke `db_analizes`.

---

## 5. Perhitungan Skor dan Normalisasi DISC

**Disimpan ke:** `db_disc` (`d_percent`, `i_percent`, `s_percent`, `c_percent`).

Empat dimensi DISC dibentuk dari kombinasi dua `subcat_score` sumbu pembentuknya. Pemetaan sumbu **wajib menggunakan ID subkategori** dari `scoring_config_snapshot.disc_subcat_mapping`, bukan nama subkategori.

### 5.1 Penggabungan Skor Mentah (Raw Score)

$$
D_{\text{raw}} = \text{subcat\_score}(\text{ID}_{\text{Ekstrovert}}) + \text{subcat\_score}(\text{ID}_{\text{Task Oriented}})
$$

$$
I_{\text{raw}} = \text{subcat\_score}(\text{ID}_{\text{Ekstrovert}}) + \text{subcat\_score}(\text{ID}_{\text{People Oriented}})
$$

$$
S_{\text{raw}} = \text{subcat\_score}(\text{ID}_{\text{Introvert}}) + \text{subcat\_score}(\text{ID}_{\text{People Oriented}})
$$

$$
C_{\text{raw}} = \text{subcat\_score}(\text{ID}_{\text{Introvert}}) + \text{subcat\_score}(\text{ID}_{\text{Task Oriented}})
$$

### 5.2 Normalisasi ke Persentase 100,00%

Hitung total skor mentah:

$$
T_{\text{DISC}} = D_{\text{raw}} + I_{\text{raw}} + S_{\text{raw}} + C_{\text{raw}}
$$

* **Kasus Nol ($T_{\text{DISC}} = 0$):** Jika seluruh sumbu bernilai nol (misalnya tidak ada soal DISC yang dijawab), backend **tidak boleh** membuat profil DISC palsu dan tidak melakukan `INSERT` ke `db_disc`.
* **Kasus Normal ($T_{\text{DISC}} > 0$):** Hitung persentase awal masing-masing dimensi dengan pembulatan 2 angka di belakang koma:

$$
X_{\text{pct}} = \text{ROUND}\left(\frac{X_{\text{raw}}}{T_{\text{DISC}}} \times 100,\ 2\right), \quad X \in \{D, I, S, C\}
$$

### 5.3 Koreksi Selisih Pembulatan (Constraint `SUM = 100.00`)

Tabel `db_disc` memberlakukan constraint:

```sql
CHECK (d_percent + i_percent + s_percent + c_percent = 100.00)
```

Langkah penyesuaian di backend sebelum `INSERT`:
1. Hitung selisih pembulatan:
   $$\Delta = \text{ROUND}\left(100.00 - (D_{\text{pct}} + I_{\text{pct}} + S_{\text{pct}} + C_{\text{pct}}),\ 2\right)$$
2. Jika $\Delta \neq 0.00$, tambahkan $\Delta$ ke dimensi dengan nilai persentase terbesar.
3. Jika terdapat nilai maksimum yang seri (*tie*), gunakan urutan prioritas deterministik: `d_percent` $\rightarrow$ `i_percent` $\rightarrow$ `s_percent` $\rightarrow$ `c_percent`.

---

## 6. Skala Diagram Kuadran DISC untuk Template PDF

Diagram kuadran Cartesian DISC pada template HTML menggunakan 4 kuadran di mana lebar dan tinggi maksimum tiap blok kuadran dibatasi hingga **45%** dari wadah utama (agar menyisakan margin visual 5% di tiap sisi).

Jika profil `db_disc` tersedia, tentukan nilai dominan tertinggi:

$$
M = \max(\text{d\_percent},\ \text{i\_percent},\ \text{s\_percent},\ \text{c\_percent})
$$

Hitung keempat dimensi CSS kuadran (dibulatkan 2 desimal):

$$
\text{disc\_left\_pct} = \text{ROUND}\left(\frac{\max(\text{d\_percent},\ \text{c\_percent})}{M} \times 45,\ 2\right)
$$

$$
\text{disc\_right\_pct} = \text{ROUND}\left(\frac{\max(\text{i\_percent},\ \text{s\_percent})}{M} \times 45,\ 2\right)
$$

$$
\text{disc\_top\_pct} = \text{ROUND}\left(\frac{\max(\text{d\_percent},\ \text{i\_percent})}{M} \times 45,\ 2\right)
$$

$$
\text{disc\_bottom\_pct} = \text{ROUND}\left(\frac{\max(\text{c\_percent},\ \text{s\_percent})}{M} \times 45,\ 2\right)
$$

*(Catatan: Jika data `db_disc` tidak tersedia karena $T_{\text{DISC}} = 0$, keempat variabel skala diisi `0.00` dan nilai dimensi diisi `0.00`).*

---

## 7. Kontrak Output JSON Analisis AI (`db_results.ai_analysis`)

Modul AI (Gemini SDK / OpenRouter API) menerima ringkasan skor kategori/subkategori, persentase DISC, serta seluruh teks pertanyaan dan jawaban peserta (baik `is_scored = TRUE` maupun `is_scored = FALSE`, termasuk jawaban esai).

AI **wajib** mengembalikan JSON terstruktur (divalidasi dengan schema validator di backend) agar langsung terpetakan ke placeholder template PDF:

```json
{
  "disc_summary": "Ringkasan narasi profil kepribadian DISC peserta secara objektif.",
  "disc_pros": [
    "Poin kekuatan gaya perilaku berdasarkan dominasi DISC."
  ],
  "disc_cons": [
    "Poin area yang perlu diwaspadai dari gaya perilaku DISC."
  ],
  "strength_tagline": "Frasa singkat keunggulan utama",
  "strength_text": "Uraian lengkap kekuatan kompetensi dan kepribadian peserta.",
  "weakness_tagline": "Frasa singkat area pengembangan",
  "weakness_text": "Uraian lengkap aspek yang masih perlu ditingkatkan.",
  "development_tagline": "Frasa singkat fokus tindak lanjut",
  "development_text": "Rekomendasi pengembangan konkret dan dapat ditindaklanjuti."
}
```

Jika proses AI gagal (`ai_analysis_status = 'FAILED'`), nilai `final_score` dan `status_result` tetap sah. Saat PDF dirender dalam kondisi AI belum tersedia, backend mengisi narasi fallback standar yang aman agar PDF tetap dapat diterbitkan.

---

## 8. Spesifikasi Placeholder Template PDF (`db_pdf_templates`)

Template HTML (`db_pdf_templates.html_template`) menggunakan placeholder datar `{{nama_variabel}}`. CSS (`db_pdf_templates.css_template`) disuntikkan ke dalam tag `<style>` pada dokumen HTML sebelum dirender oleh **Playwright + Chromium** dan dienkripsi AES-256 dengan **qpdf** memakai password tanggal lahir peserta (`ddmmyyyy`).

### 8.1 Daftar Lengkap Placeholder Resmi

#### A. Identitas & Hasil Ujian

| Placeholder | Sumber Data | Format / Keterangan |
|---|---|---|
| `{{participant_id}}` | `db_participants.participant_id` | Contoh: `P000000001` |
| `{{attempt_number}}` | `db_participant_test.attempt_number` | Nomor percobaan (`1` untuk awal, `2` dst. untuk remidi) |
| `{{nama_lengkap}}` | `db_participants.full_name` | Teks di-escape HTML |
| `{{nik_ktp}}` | `db_participants.nik` | 16 digit NIK |
| `{{tempat_lahir}}` | `db_participants.birth_place` | Teks di-escape HTML |
| `{{tanggal_lahir}}` | `db_participants.date_of_birth` | Format `DD-MM-YYYY` |
| `{{completed_at}}` | `db_participant_test.completed_at` | Diformat ke zona waktu `Asia/Jakarta` (WIB) |
| `{{final_score}}` | `db_results.final_score` | Format 2 desimal (contoh: `82.50`) |
| `{{status_class}}` | Turunan `db_results.status_result` | `'pass'` jika `LULUS`, `'fail'` jika `GAGAL` |
| `{{status_result}}` | `db_results.status_result` | `'LULUS'` atau `'GAGAL'` |

#### B. Fragmen Grafik Kompetensi & Psikologi

| Placeholder | Sumber Data | Keterangan |
|---|---|---|
| `{{SUBCATEGORY_CHART_ROWS}}` | `db_analizes` (baris `subcat_id`) + `db_subcat` | Fragmen HTML bar chart subkategori yang dirakit backend |
| `{{CATEGORY_CHART_COLUMNS}}` | `db_analizes` (baris `category_id`) + `db_category` | Fragmen HTML grafik kategori yang dirakit backend |

#### C. Profil & Diagram Kuadran DISC

| Placeholder | Sumber Data | Format / Keterangan |
|---|---|---|
| `{{dominance}}` | `db_disc.d_percent` | Format 2 desimal (contoh: `32.50`) |
| `{{influence}}` | `db_disc.i_percent` | Format 2 desimal |
| `{{steadiness}}` | `db_disc.s_percent` | Format 2 desimal |
| `{{conscientiousness}}` | `db_disc.c_percent` | Format 2 desimal |
| `{{disc_left_pct}}` | Rumus Bagian 6 | Skala persen sisi kiri (`0.00` – `45.00`) |
| `{{disc_right_pct}}` | Rumus Bagian 6 | Skala persen sisi kanan (`0.00` – `45.00`) |
| `{{disc_top_pct}}` | Rumus Bagian 6 | Skala persen sisi atas (`0.00` – `45.00`) |
| `{{disc_bottom_pct}}` | Rumus Bagian 6 | Skala persen sisi bawah (`0.00` – `45.00`) |

#### D. Narasi Analisis AI

| Placeholder | Sumber Data (`db_results.ai_analysis`) | Keterangan |
|---|---|---|
| `{{ai_disc_summary}}` | `ai_analysis.disc_summary` | Teks di-escape HTML |
| `{{ai_disc_pros_html}}` | `ai_analysis.disc_pros[]` | Elemen `<li>...</li>` yang dirakit backend (tiap item di-escape) |
| `{{ai_disc_cons_html}}` | `ai_analysis.disc_cons[]` | Elemen `<li>...</li>` yang dirakit backend (tiap item di-escape) |
| `{{ai_strength_tagline}}` | `ai_analysis.strength_tagline` | Teks di-escape HTML |
| `{{ai_strength_text}}` | `ai_analysis.strength_text` | Teks di-escape HTML |
| `{{ai_weakness_tagline}}` | `ai_analysis.weakness_tagline` | Teks di-escape HTML |
| `{{ai_weakness_text}}` | `ai_analysis.weakness_text` | Teks di-escape HTML |
| `{{ai_development_tagline}}` | `ai_analysis.development_tagline` | Teks di-escape HTML |
| `{{ai_development_text}}` | `ai_analysis.development_text` | Teks di-escape HTML |

#### E. Aset & Metadata Dokumen

| Placeholder | Sumber Data | Keterangan |
|---|---|---|
| `{{logo_perusahaan}}` | Konfigurasi environment / aset aplikasi | URL logo resmi |
| `{{verification_url}}` | Konfigurasi aplikasi | URL dasar halaman verifikasi keaslian dokumen |
| `{{verification_link}}` | Turunan `result_id` + HMAC signature | URL lengkap verifikasi dokumen |
| `{{verification_code}}` | Turunan `result_id` + HMAC signature | Kode pendek verifikasi dokumen |
| `{{created_at}}` | `db_results.pdf_generated_at` / waktu render | Tanggal cetak PDF dalam zona waktu `Asia/Jakarta` |

### 8.2 Aturan Keamanan Rendering Template

1. **HTML Escaping Wajib:** Seluruh data input pengguna dan teks dari AI wajib di-escape (`&`, `<`, `>`, `"`, `'`) sebelum dimasukkan ke template untuk mencegah injeksi HTML/XSS di dalam Chromium headless.
2. **Strict Placeholder Validation:** Setelah semua placeholder resmi diganti, backend memindai sisa pola `{{...}}`. Jika masih ditemukan placeholder yang tidak dikenal atau belum terisi, proses render **wajib digagalkan** (*throw error*) agar PDF cacat tidak pernah diunggah ke Google Drive.

---

## 9. Urutan Eksekusi Pipeline Saat Ujian Selesai

1. **Penutupan Attempt (Transaksi `pg`):**
   * Validasi status `attempt_status = 'IN_PROGRESS'` dan hitung apakah melewati `deadline_at` (termasuk penalti dari `db_exam_violations`).
   * Ubah `attempt_status` menjadi `'COMPLETED'` (atau `'EXPIRED'` jika waktu habis) dan isi `completed_at = NOW()`.
   * Hitung `final_score`, `status_result`, `subcat_score`, `category_score`, dan `db_disc` langsung dari `question_snapshot` + `scoring_config_snapshot`.
   * Simpan ke `db_results` (`scoring_status = 'COMPLETED'`), `db_analizes`, dan `db_disc` dalam satu transaksi atomik.
2. **Analisis AI (Proses Latar Belakang):**
   * Ubah `ai_analysis_status = 'PROCESSING'`.
   * Panggil Gemini SDK (dengan fallback OpenRouter API jika gagal), validasi output JSON, lalu simpan ke `db_results.ai_analysis` (`ai_analysis_status = 'COMPLETED'`).
3. **Generasi, Enkripsi & Unggah PDF:**
   * Ambil template berstatus `'PUBLISHED'` dari `db_pdf_templates`.
   * Render HTML/CSS menjadi PDF melalui **Playwright + Chromium**.
   * Enkripsi file PDF dengan **qpdf** menggunakan password `ddmmyyyy` dari `db_participants.date_of_birth`.
   * Hitung `SHA-256` file terenkripsi, unggah ke folder privat **Google Drive API v3**, lalu simpan `pdf_template_id`, `pdf_storage_key`, `pdf_hash`, dan `pdf_generated_at` ke `db_results`.