# Bagian 4 — Desain Laporan Hasil Test dan Template PDF

Dokumen ini menetapkan rancangan visual modern bergaya *editorial executive report*, struktur HTML (`db_pdf_templates.html_template`), stylesheet CSS (`db_pdf_templates.css_template`), panduan gaya bahasa analisis AI (*witty, santai, dewasa 18+ yang tetap sopan & relevan*), serta pratinjau hasil jadi untuk dirender menjadi dokumen PDF berukuran **A4** melalui **Playwright + Chromium** dan dienkripsi menggunakan **qpdf** (password tanggal lahir peserta `ddmmyyyy`).

## 1. Arah Estetika & Sistem Visual

* **Palet Warna Editorial & Berkelas:**
  * *Primary Ink & Header:* Deep Obsidian / Midnight Navy (`#0f172a` ke `#1e293b`) dipadukan dengan aksen *Warm Champagne Gold* (`#d97706` / `#f59e0b`) agar terkesan eksklusif, dewasa, dan tidak norak.
  * *Surface & Kartu:* Latar belakang bersih (`#ffffff`) dengan panel lembut (*soft alabaster* `#f8fafc`) dan garis tepi halus (`#e2e8f0`).
  * *Palet Harmonis DISC:* Menggunakan gradasi *muted jewel tones* yang elegan dan tidak mencolok mata—**D** (*Crimson Terracotta* `#e11d48`), **I** (*Amber Gold* `#d97706`), **S** (*Emerald Sage* `#059669`), dan **C** (*Royal Sapphire* `#2563eb`).
  * *Alert & Disclaimer Tegas:* Merah menyala berkarakter (*Vivid Crimson* `#dc2626` dengan latar `#fef2f2` dan aksen border tebal) khusus untuk menegaskan klausul hukum *Disclaimer*.
* **Visualisasi Grafik Modern:**
  * **Pilar Kategori:** Kartu ringkas dengan *circular progress ring* (SVG `stroke-dasharray`) sehingga lebih dinamis dibanding kotak angka polos.
  * **Indikator Subkategori:** *Pill progress bar* berujung bulat dengan gradasi *Slate-to-Indigo* dan *Teal* yang bersih.
  * **Pemetaan Kepribadian DISC:** Menggunakan **Radar / Diamond Compass Chart** + 4 *Axis Quadrant Target* modern yang tetap digerakkan secara presisi oleh variabel skala `45%` (`{{disc_left_pct}}`, `{{disc_right_pct}}`, `{{disc_top_pct}}`, `{{disc_bottom_pct}}`).

---

## 2. Panduan Gaya Bahasa Analisis AI (`prompt.builder.ts`)

Agar hasil analisis AI terasa hidup, dewasa (18+), *relatable*, dan tidak kaku seperti teks buku pelajaran, *system prompt* untuk Gemini SDK / OpenRouter API diatur dengan ketentuan berikut:

1. **Nada & Gaya Penuturan:** Menggunakan bahasa Indonesia percakapan profesional yang santai, hangat, jenaka (*witty*), dan sedikit humor dewasa (18+) yang cerdas—seperti obrolan jujur dengan mentor senior sambil ngopi, tanpa merendahkan peserta dan tetap menjaga batasan profesionalisme kerja.
2. **Struktur Output JSON Tetap Ketat:** Meskipun bahasanya santai dan menghibur, struktur *key* JSON yang dikembalikan oleh AI tetap wajib mematuhi kontrak pada **Bagian 2** (`disc_summary`, `disc_pros`, `disc_cons`, `strength_tagline`, `strength_text`, `weakness_tagline`, `weakness_text`, `development_tagline`, `development_text`).
3. **Contoh Gaya Narasi AI:**
   * *Tagline Keunggulan:* `"Otak Encer, Rem Tetap Pakem — Bukan Tipe yang Gas Pol Tanpa Mikir"`
   * *Narasi Kekuatan:* `"Skor Safety Knowledge (90%) dan Responsibility (92%) kamu membuktikan satu hal: kamu bukan tipe yang modal nekat lalu pasrah pada nasib. Di saat orang lain masih sibuk panik atau cari kambing hitam, kamu sudah ceklis prosedur dari A sampai Z..."`
   * *Narasi Area Pengembangan:* `"Di indikator People Oriented (69%), kelihatan banget kamu lebih nyaman berurusan dengan mesin atau aturan daripada drama perasaan manusia. Jujur saja, kalau rekan tim mulai lambat, godaan buat bilang 'sudah sini biar gue aja yang kerjain' pasti besar, kan?"`

---

## 3. Kode HTML Template (`db_pdf_templates.html_template`)

Seluruh placeholder datar `{{nama_variabel}}` tetap mempertahankan kontrak yang sama dengan mesin rendering backend.

```html
<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Executive Assessment Report - {{nama_lengkap}}</title>
  <style>
    {{CSS_TEMPLATE}}
  </style>
</head>
<body>

  <!-- ===================================================================== -->
  <!-- HALAMAN 1: EXECUTIVE SUMMARY, CAPAIAN KATEGORI & PETA KEPRIBADIAN DISC -->
  <!-- ===================================================================== -->
  <div class="page">
    <!-- Top Editorial Banner -->
    <header class="exec-header">
      <div class="brand-left">
        <div class="logo-frame">
          <img src="{{logo_perusahaan}}" alt="Logo Perusahaan" class="brand-logo" />
        </div>
        <div class="title-stack">
          <div class="eyebrow">CONFIDENTIAL ASSESSMENT DOSSIER &bull; ATTEMPT #{{attempt_number}}</div>
          <h1 class="main-title">Laporan Evaluasi Kompetensi &amp; Profil Kepribadian</h1>
        </div>
      </div>
      <div class="header-right">
        <span class="id-chip">ID: <strong>{{participant_id}}</strong></span>
        <span class="verif-chip">REF: {{verification_code}}</span>
      </div>
    </header>

    <!-- Hero Section: Profile Card & Score Showcase -->
    <section class="hero-panel">
      <div class="profile-card">
        <div class="card-kicker">PROFIL KANDIDAT / PESERTA</div>
        <div class="candidate-name">{{nama_lengkap}}</div>
        <div class="profile-grid">
          <div class="p-item">
            <span class="p-label">Nomor Induk Kependudukan</span>
            <span class="p-value mono">{{nik_ktp}}</span>
          </div>
          <div class="p-item">
            <span class="p-label">Tempat &amp; Tanggal Lahir</span>
            <span class="p-value">{{tempat_lahir}}, {{tanggal_lahir}}</span>
          </div>
          <div class="p-item full">
            <span class="p-label">Waktu Penyelesaian Asesmen</span>
            <span class="p-value">{{completed_at}}</span>
          </div>
        </div>
      </div>

      <div class="score-showcase {{status_class}}">
        <span class="sc-eyebrow">SKOR KOMPETENSI</span>
        <div class="sc-value">{{final_score}}</div>
        <div class="sc-pill {{status_class}}">{{status_result}}</div>
        <span class="sc-caption">Evaluasi butir terukur (<code>is_scored = true</code>)</span>
      </div>
    </section>

    <!-- Ringkasan Pilar Kategori -->
    <section class="panel-box">
      <div class="panel-head">
        <h2 class="panel-title">Pilar Capaian Kategori</h2>
        <span class="panel-hint">Agregat rata-rata indikator aktif (0% &ndash; 100%)</span>
      </div>
      <div class="category-columns-grid">
        {{CATEGORY_CHART_COLUMNS}}
      </div>
    </section>

    <!-- Rincian Indikator Subkategori -->
    <section class="panel-box">
      <div class="panel-head">
        <h2 class="panel-title">Spektrum Indikator Subkategori</h2>
        <span class="panel-hint">Pemetaan terperinci seluruh aspek kompetensi &amp; karakter</span>
      </div>
      <div class="subcat-list">
        {{SUBCATEGORY_CHART_ROWS}}
      </div>
    </section>

    <!-- Pemetaan Kepribadian DISC -->
    <section class="panel-box no-mb">
      <div class="panel-head">
        <h2 class="panel-title">Peta Dinamika Kepribadian DISC</h2>
        <span class="panel-hint">Proporsi 4 dimensi perilaku (Total = 100,00%)</span>
      </div>

      <div class="disc-modern-grid">
        <!-- 4 Kartu Dimensi Elegan -->
        <div class="disc-stack">
          <div class="disc-row-card tone-d">
            <div class="disc-emblem">D</div>
            <div class="disc-meta">
              <div class="disc-title-row">
                <span class="disc-label">Dominance</span>
                <span class="disc-score">{{dominance}}%</span>
              </div>
              <div class="disc-bar-bg"><div class="disc-bar-fill" style="width: {{dominance}}%;"></div></div>
              <span class="disc-sub">Tegas &bull; Eksekutor Cepat &bull; Berorientasi Target</span>
            </div>
          </div>

          <div class="disc-row-card tone-i">
            <div class="disc-emblem">I</div>
            <div class="disc-meta">
              <div class="disc-title-row">
                <span class="disc-label">Influence</span>
                <span class="disc-score">{{influence}}%</span>
              </div>
              <div class="disc-bar-bg"><div class="disc-bar-fill" style="width: {{influence}}%;"></div></div>
              <span class="disc-sub">Ekspresif &bull; Persuasif &bull; Membangun Energi Tim</span>
            </div>
          </div>

          <div class="disc-row-card tone-s">
            <div class="disc-emblem">S</div>
            <div class="disc-meta">
              <div class="disc-title-row">
                <span class="disc-label">Steadiness</span>
                <span class="disc-score">{{steadiness}}%</span>
              </div>
              <div class="disc-bar-bg"><div class="disc-bar-fill" style="width: {{steadiness}}%;"></div></div>
              <span class="disc-sub">Konsisten &bull; Sabar &bull; Menjaga Harmoni</span>
            </div>
          </div>

          <div class="disc-row-card tone-c">
            <div class="disc-emblem">C</div>
            <div class="disc-meta">
              <div class="disc-title-row">
                <span class="disc-label">Conscientiousness</span>
                <span class="disc-score">{{conscientiousness}}%</span>
              </div>
              <div class="disc-bar-bg"><div class="disc-bar-fill" style="width: {{conscientiousness}}%;"></div></div>
              <span class="disc-sub">Presisi &bull; Analitis &bull; Patuh Standar Mutu</span>
            </div>
          </div>
        </div>

        <!-- Radar / Compass Quadrant Chart (Skala Maksimal 45% per sumbu) -->
        <div class="compass-stage">
          <span class="c-axis c-top">EKSTROVERT</span>
          <span class="c-axis c-bottom">INTROVERT</span>
          <span class="c-axis c-left">TASK</span>
          <span class="c-axis c-right">PEOPLE</span>

          <div class="compass-board">
            <!-- Lingkaran konsentris target -->
            <div class="ring ring-outer"></div>
            <div class="ring ring-mid"></div>
            <div class="ring ring-inner"></div>

            <!-- Sumbu silang -->
            <div class="crosshair-v"></div>
            <div class="crosshair-h"></div>

            <!-- Kelopak Kuadran Modern (Menggunakan variabel skala 45%) -->
            <div class="petal petal-d" style="width: {{disc_left_pct}}%; height: {{disc_top_pct}}%;">
              <span>D</span>
            </div>
            <div class="petal petal-i" style="width: {{disc_right_pct}}%; height: {{disc_top_pct}}%;">
              <span>I</span>
            </div>
            <div class="petal petal-c" style="width: {{disc_left_pct}}%; height: {{disc_bottom_pct}}%;">
              <span>C</span>
            </div>
            <div class="petal petal-s" style="width: {{disc_right_pct}}%; height: {{disc_bottom_pct}}%;">
              <span>S</span>
            </div>

            <div class="center-pivot"></div>
          </div>
        </div>
      </div>
    </section>

    <footer class="doc-footer">
      <span>Halaman 1 dari 2 &bull; {{nama_lengkap}} ({{participant_id}})</span>
      <span>Dokumen Terenkripsi &bull; Kode: <strong>{{verification_code}}</strong></span>
    </footer>
  </div>

  <!-- ===================================================================== -->
  <!-- HALAMAN 2: ANALISIS KUALITATIF AI (SANTAI & WITTY) + DISCLAIMER TEGAS -->
  <!-- ===================================================================== -->
  <div class="page page-break">
    <header class="exec-header compact">
      <div class="brand-left">
        <div class="logo-frame small">
          <img src="{{logo_perusahaan}}" alt="Logo Perusahaan" class="brand-logo small" />
        </div>
        <div class="title-stack">
          <div class="eyebrow">PERSONALITY &amp; BEHAVIORAL INSIGHTS</div>
          <h1 class="main-title small">Bedah Karakter &amp; Catatan Pengembangan</h1>
        </div>
      </div>
      <div class="header-right">
        <span class="id-chip">{{nama_lengkap}}</span>
        <span class="verif-chip">Skor: {{final_score}} ({{status_result}})</span>
      </div>
    </header>

    <!-- Sintesis Perilaku DISC -->
    <section class="panel-box compact-mb">
      <div class="panel-head">
        <h2 class="panel-title">Membaca Gaya Kerja &amp; Karaktermu (DISC)</h2>
        <span class="panel-hint">Analisis kecenderungan alami di lapangan</span>
      </div>
      <p class="editorial-lead">{{ai_disc_summary}}</p>

      <div class="dual-insight-grid">
        <div class="insight-col plus">
          <div class="insight-col-head">
            <span class="dot plus"></span>
            <h3>Nilai Jual &amp; Sisi Kerenmu</h3>
          </div>
          <ul class="insight-list">
            {{ai_disc_pros_html}}
          </ul>
        </div>

        <div class="insight-col minus">
          <div class="insight-col-head">
            <span class="dot minus"></span>
            <h3>Rem Tangan yang Perlu Dijaga</h3>
          </div>
          <ul class="insight-list">
            {{ai_disc_cons_html}}
          </ul>
        </div>
      </div>
    </section>

    <!-- 3 Kartu Narasi AI -->
    <section class="narrative-stack">
      <article class="narrative-card card-strength">
        <div class="nc-top">
          <span class="nc-tag tag-strength">SENJATA ANDALAN</span>
          <h3 class="nc-headline">{{ai_strength_tagline}}</h3>
        </div>
        <p class="nc-copy">{{ai_strength_text}}</p>
      </article>

      <article class="narrative-card card-weakness">
        <div class="nc-top">
          <span class="nc-tag tag-weakness">TITIK BLIND-SPOT</span>
          <h3 class="nc-headline">{{ai_weakness_tagline}}</h3>
        </div>
        <p class="nc-copy">{{ai_weakness_text}}</p>
      </article>

      <article class="narrative-card card-dev">
        <div class="nc-top">
          <span class="nc-tag tag-dev">UPGRADE LEVEL BERIKUTNYA</span>
          <h3 class="nc-headline">{{ai_development_tagline}}</h3>
        </div>
        <p class="nc-copy">{{ai_development_text}}</p>
      </article>
    </section>

    <!-- Verifikasi Keaslian Dokumen -->
    <section class="verif-strip">
      <div class="verif-info">
        <div class="verif-heading">OTENTIKASI DOKUMEN DIGITAL</div>
        <p class="verif-sub">
          Validasi keaslian laporan ini secara daring melalui tautan resmi penyelenggara:
        </p>
        <div class="verif-link-text">{{verification_link}}</div>
        <div class="verif-portal-text">Portal: {{verification_url}}</div>
      </div>
      <div class="verif-stamp">
        <span class="stamp-caption">KODE OTENTIKASI</span>
        <div class="stamp-code">{{verification_code}}</div>
        <span class="stamp-time">{{created_at}}</span>
      </div>
    </section>

    <!-- DISCLAIMER TEGAS BERWARNA MERAH MENYALA -->
    <section class="disclaimer-alert" role="alert">
      <div class="disclaimer-header">
        <span class="disclaimer-icon">!</span>
        <span class="disclaimer-title">DISCLAIMER &amp; BATASAN TANGGUNG JAWAB HUKUM</span>
      </div>
      <p class="disclaimer-body">
        Hasil tes ini <strong>HANYA</strong> merepresentasikan kompetensi, karakter, dan kondisi peserta tes pada saat ujian diadakan. Selain itu, hasil tes ini <strong>HANYA DAPAT DIGUNAKAN</strong> oleh penyelenggara untuk keperluan penilaian tes yang sedang diadakan; segala bentuk penggunaan, penyebarluasan, atau pemanfaatan hasil tes di luar keperluan tersebut <strong>BUKAN MERUPAKAN TANGGUNG JAWAB PENYELENGGARA</strong>, serta wajib berdasarkan persetujuan tertulis dan ketentuan hukum yang berlaku.
      </p>
    </section>

    <footer class="doc-footer">
      <span>Halaman 2 dari 2 &bull; Confidential Assessment Report</span>
      <span>Diterbitkan: {{created_at}}</span>
    </footer>
  </div>

</body>
</html>
```

---

## 4. Kode CSS Template (`db_pdf_templates.css_template`)

```css
@page {
  size: A4 portrait;
  margin: 11mm;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
  -webkit-print-color-adjust: exact !important;
  print-color-adjust: exact !important;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Arial, sans-serif;
  color: #1e293b;
  background-color: #ffffff;
  font-size: 9.2pt;
  line-height: 1.42;
}

.page {
  width: 100%;
  min-height: 273mm;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.page-break {
  page-break-before: always;
}

/* Executive Dark Header */
.exec-header {
  background: linear-gradient(135deg, #0f172a 0%, #1e293b 100%);
  color: #ffffff;
  border-radius: 10px;
  padding: 12px 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
  border-bottom: 3px solid #d97706;
}

.exec-header.compact {
  padding: 9px 14px;
  margin-bottom: 8px;
}

.brand-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.logo-frame {
  background: #ffffff;
  padding: 5px 8px;
  border-radius: 7px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.logo-frame.small {
  padding: 4px 6px;
}

.brand-logo {
  height: 32px;
  width: auto;
  object-fit: contain;
}

.brand-logo.small {
  height: 24px;
}

.eyebrow {
  font-size: 7pt;
  font-weight: 700;
  letter-spacing: 0.1em;
  color: #fbbf24;
  text-transform: uppercase;
  margin-bottom: 2px;
}

.main-title {
  font-size: 13pt;
  font-weight: 800;
  letter-spacing: -0.01em;
  color: #ffffff;
}

.main-title.small {
  font-size: 11.2pt;
}

.header-right {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 4px;
}

.id-chip {
  background: rgba(255, 255, 255, 0.12);
  border: 1px solid rgba(255, 255, 255, 0.22);
  color: #f8fafc;
  padding: 2.5px 9px;
  border-radius: 999px;
  font-size: 7.6pt;
}

.verif-chip {
  font-family: 'Courier New', monospace;
  font-size: 7.4pt;
  color: #fde68a;
  letter-spacing: 0.05em;
}

/* Hero Profile + Score Showcase */
.hero-panel {
  display: grid;
  grid-template-columns: 1.7fr 1fr;
  gap: 10px;
  margin-bottom: 10px;
}

.profile-card {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 11px 14px;
}

.card-kicker {
  font-size: 7pt;
  font-weight: 700;
  letter-spacing: 0.08em;
  color: #64748b;
  margin-bottom: 2px;
}

.candidate-name {
  font-size: 13pt;
  font-weight: 800;
  color: #0f172a;
  margin-bottom: 8px;
  padding-bottom: 6px;
  border-bottom: 1px solid #e2e8f0;
}

.profile-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px 12px;
}

.p-item {
  display: flex;
  flex-direction: column;
}

.p-item.full {
  grid-column: span 2;
}

.p-label {
  font-size: 7pt;
  color: #64748b;
}

.p-value {
  font-size: 8.6pt;
  font-weight: 600;
  color: #0f172a;
}

.mono {
  font-family: 'Courier New', monospace;
  letter-spacing: 0.03em;
}

.score-showcase {
  border-radius: 10px;
  padding: 11px 12px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  text-align: center;
  border: 1px solid #cbd5e1;
  background: #f8fafc;
}

.score-showcase.pass {
  background: linear-gradient(180deg, #ecfdf5 0%, #d1fae5 100%);
  border-color: #6ee7b7;
}

.score-showcase.fail {
  background: linear-gradient(180deg, #fef2f2 0%, #fee2e2 100%);
  border-color: #fca5a5;
}

.sc-eyebrow {
  font-size: 7.2pt;
  font-weight: 800;
  letter-spacing: 0.08em;
  color: #475569;
}

.sc-value {
  font-size: 27pt;
  font-weight: 900;
  line-height: 1.08;
  color: #0f172a;
  margin: 3px 0;
}

.sc-pill {
  padding: 3px 16px;
  border-radius: 999px;
  font-size: 8.2pt;
  font-weight: 800;
  letter-spacing: 0.08em;
  color: #ffffff;
}

.sc-pill.pass { background: #059669; }
.sc-pill.fail { background: #dc2626; }

.sc-caption {
  font-size: 6.8pt;
  color: #64748b;
  margin-top: 5px;
}

/* Content Panels */
.panel-box {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 10px 13px;
  margin-bottom: 9px;
}

.compact-mb {
  margin-bottom: 7px;
}

.no-mb {
  margin-bottom: 0;
}

.panel-head {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 7px;
  padding-bottom: 4px;
  border-bottom: 1px solid #f1f5f9;
}

.panel-title {
  font-size: 9.8pt;
  font-weight: 800;
  color: #0f172a;
}

.panel-hint {
  font-size: 7.4pt;
  color: #64748b;
}

/* Category Columns (Modern Ring / Gauge Cards) */
.category-columns-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 8px;
}

.cat-col-item {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 7px 6px;
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

.cat-ring-wrap {
  position: relative;
  width: 46px;
  height: 46px;
  margin-bottom: 4px;
}

.cat-ring-val {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 8pt;
  font-weight: 800;
  color: #0f172a;
}

.cat-col-name {
  font-size: 7.6pt;
  font-weight: 700;
  color: #334155;
  line-height: 1.2;
}

/* Subcategory Modern Pill Bars */
.subcat-list {
  display: grid;
  grid-template-columns: 1fr 1fr;
  column-gap: 18px;
  row-gap: 5.5px;
}

.subcat-row {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.subcat-meta {
  display: flex;
  justify-content: space-between;
  font-size: 8pt;
}

.subcat-label {
  color: #334155;
  font-weight: 600;
}

.subcat-val {
  font-weight: 800;
  color: #0f172a;
}

.subcat-track {
  height: 6px;
  background: #f1f5f9;
  border-radius: 999px;
  overflow: hidden;
}

.subcat-fill {
  height: 100%;
  background: linear-gradient(90deg, #1e3a8a 0%, #3b82f6 100%);
  border-radius: 999px;
}

/* DISC Modern Grid & Compass Chart */
.disc-modern-grid {
  display: grid;
  grid-template-columns: 1.3fr 1fr;
  gap: 14px;
  align-items: center;
}

.disc-stack {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.disc-row-card {
  display: flex;
  align-items: center;
  gap: 9px;
  padding: 6px 9px;
  border-radius: 8px;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
}

.disc-emblem {
  width: 28px;
  height: 28px;
  border-radius: 7px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 900;
  font-size: 11pt;
  color: #ffffff;
  flex-shrink: 0;
}

.tone-d .disc-emblem, .tone-d .disc-bar-fill { background: #e11d48; }
.tone-i .disc-emblem, .tone-i .disc-bar-fill { background: #d97706; }
.tone-s .disc-emblem, .tone-s .disc-bar-fill { background: #059669; }
.tone-c .disc-emblem, .tone-c .disc-bar-fill { background: #2563eb; }

.disc-meta {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2.5px;
}

.disc-title-row {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
}

.disc-label {
  font-size: 8.5pt;
  font-weight: 800;
  color: #0f172a;
}

.disc-score {
  font-size: 9.5pt;
  font-weight: 800;
  color: #0f172a;
}

.disc-bar-bg {
  height: 4.5px;
  background: #e2e8f0;
  border-radius: 999px;
  overflow: hidden;
}

.disc-bar-fill {
  height: 100%;
  border-radius: 999px;
}

.disc-sub {
  font-size: 7pt;
  color: #64748b;
}

/* Compass Radar Quadrant (Scale max 45%) */
.compass-stage {
  position: relative;
  padding: 16px 28px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.c-axis {
  position: absolute;
  font-size: 6.6pt;
  font-weight: 800;
  letter-spacing: 0.06em;
  color: #475569;
  background: #f1f5f9;
  padding: 1.5px 5px;
  border-radius: 3px;
}

.c-top { top: 1px; left: 50%; transform: translateX(-50%); }
.c-bottom { bottom: 1px; left: 50%; transform: translateX(-50%); }
.c-left { left: 0; top: 50%; transform: translateY(-50%); }
.c-right { right: 0; top: 50%; transform: translateY(-50%); }

.compass-board {
  position: relative;
  width: 164px;
  height: 164px;
  border-radius: 50%;
  background: radial-gradient(circle, #ffffff 0%, #f1f5f9 100%);
  border: 1.5px solid #cbd5e1;
  overflow: hidden;
}

.ring {
  position: absolute;
  border-radius: 50%;
  border: 1px dashed #cbd5e1;
}

.ring-outer { inset: 12px; }
.ring-mid { inset: 34px; }
.ring-inner { inset: 56px; }

.crosshair-v {
  position: absolute;
  left: 50%;
  top: 0;
  bottom: 0;
  width: 1px;
  background: #94a3b8;
}

.crosshair-h {
  position: absolute;
  top: 50%;
  left: 0;
  right: 0;
  height: 1px;
  background: #94a3b8;
}

.petal {
  position: absolute;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 8.5pt;
  font-weight: 900;
  color: #ffffff;
}

.petal-d {
  right: 50%;
  bottom: 50%;
  background: rgba(225, 29, 72, 0.85);
  border-top-left-radius: 999px;
}

.petal-i {
  left: 50%;
  bottom: 50%;
  background: rgba(217, 119, 6, 0.85);
  border-top-right-radius: 999px;
}

.petal-c {
  right: 50%;
  top: 50%;
  background: rgba(37, 99, 235, 0.85);
  border-bottom-left-radius: 999px;
}

.petal-s {
  left: 50%;
  top: 50%;
  background: rgba(5, 150, 105, 0.85);
  border-bottom-right-radius: 999px;
}

.center-pivot {
  position: absolute;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #0f172a;
  border: 2px solid #ffffff;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* Halaman 2: AI Narratives */
.editorial-lead {
  font-size: 8.6pt;
  color: #1e293b;
  margin-bottom: 8px;
  text-align: justify;
}

.dual-insight-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 9px;
}

.insight-col {
  border-radius: 8px;
  padding: 7px 10px;
}

.insight-col.plus {
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
}

.insight-col.minus {
  background: #fffbeb;
  border: 1px solid #fde68a;
}

.insight-col-head {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 4px;
}

.insight-col-head h3 {
  font-size: 8pt;
  font-weight: 800;
  color: #0f172a;
}

.dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
}

.dot.plus { background: #059669; }
.dot.minus { background: #d97706; }

.insight-list {
  padding-left: 14px;
  font-size: 7.8pt;
  color: #334155;
}

.insight-list li {
  margin-bottom: 2px;
}

.narrative-stack {
  display: flex;
  flex-direction: column;
  gap: 7px;
  margin-bottom: 8px;
}

.narrative-card {
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-left-width: 4px;
  border-radius: 8px;
  padding: 8px 11px;
}

.card-strength { border-left-color: #2563eb; }
.card-weakness { border-left-color: #d97706; }
.card-dev { border-left-color: #059669; }

.nc-top {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 3px;
}

.nc-tag {
  font-size: 6.8pt;
  font-weight: 800;
  padding: 2px 7px;
  border-radius: 4px;
  letter-spacing: 0.05em;
  flex-shrink: 0;
}

.tag-strength { background: #dbeafe; color: #1e40af; }
.tag-weakness { background: #fef3c7; color: #92400e; }
.tag-dev { background: #d1fae5; color: #065f46; }

.nc-headline {
  font-size: 8.8pt;
  font-weight: 800;
  color: #0f172a;
}

.nc-copy {
  font-size: 8.3pt;
  color: #334155;
  text-align: justify;
  line-height: 1.36;
}

/* Verification Strip */
.verif-strip {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  background: #f8fafc;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  padding: 7px 11px;
  margin-bottom: 7px;
}

.verif-heading {
  font-size: 7.8pt;
  font-weight: 800;
  color: #0f172a;
}

.verif-sub {
  font-size: 7pt;
  color: #475569;
}

.verif-link-text {
  font-family: 'Courier New', monospace;
  font-size: 7.2pt;
  color: #1d4ed8;
  margin-top: 2px;
}

.verif-portal-text {
  font-size: 6.7pt;
  color: #64748b;
}

.verif-stamp {
  text-align: center;
  flex-shrink: 0;
}

.stamp-caption {
  display: block;
  font-size: 6.3pt;
  font-weight: 700;
  color: #64748b;
  margin-bottom: 2px;
}

.stamp-code {
  font-family: 'Courier New', monospace;
  font-size: 9.2pt;
  font-weight: 800;
  background: #0f172a;
  color: #fbbf24;
  padding: 3px 10px;
  border-radius: 5px;
  letter-spacing: 0.08em;
}

.stamp-time {
  display: block;
  font-size: 6.6pt;
  color: #64748b;
  margin-top: 2px;
}

/* DISCLAIMER TEGAS MERAH MENYALA */
.disclaimer-alert {
  background: #fef2f2;
  border: 1.5px solid #ef4444;
  border-left: 6px solid #dc2626;
  border-radius: 7px;
  padding: 7px 11px;
}

.disclaimer-header {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 3px;
}

.disclaimer-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #dc2626;
  color: #ffffff;
  font-size: 8pt;
  font-weight: 900;
}

.disclaimer-title {
  font-size: 7.5pt;
  font-weight: 900;
  letter-spacing: 0.05em;
  color: #b91c1c;
  text-transform: uppercase;
}

.disclaimer-body {
  font-size: 7pt;
  color: #7f1d1d;
  line-height: 1.34;
  text-align: justify;
}

.disclaimer-body strong {
  color: #991b1b;
  font-weight: 800;
}

/* Footer */
.doc-footer {
  display: flex;
  justify-content: space-between;
  font-size: 7.2pt;
  color: #64748b;
  padding-top: 5px;
  border-top: 1px solid #e2e8f0;
  margin-top: 5px;
}
```

---

## 5. Kontrak Fragmen Grafik Backend (`chart-fragment.builder.ts`)

* **Contoh Output `{{CATEGORY_CHART_COLUMNS}}` (Circular Ring SVG):**
  *(Nilai `stroke-dashoffset` dihitung di backend dari keliling lingkaran `113.1` untuk radius `r = 18`: `offset = 113.1 * (1 - score / 100)`)*
```html
<div class="cat-col-item">
  <div class="cat-ring-wrap">
    <svg width="46" height="46" viewBox="0 0 46 46">
      <circle cx="23" cy="23" r="18" fill="none" stroke="#e2e8f0" stroke-width="4.5" />
      <circle cx="23" cy="23" r="18" fill="none" stroke="#1e3a8a" stroke-width="4.5"
              stroke-dasharray="113.1" stroke-dashoffset="18.85"
              stroke-linecap="round" transform="rotate(-90 23 23)" />
    </svg>
    <div class="cat-ring-val">83%</div>
  </div>
  <div class="cat-col-name">Pengetahuan Dasar</div>
</div>
```

* **Contoh Output `{{SUBCATEGORY_CHART_ROWS}}`:**
```html
<div class="subcat-row">
  <div class="subcat-meta">
    <span class="subcat-label">Safety Knowledge</span>
    <span class="subcat-val">90.00%</span>
  </div>
  <div class="subcat-track"><div class="subcat-fill" style="width: 90.00%;"></div></div>
</div>
```