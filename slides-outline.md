# Garis Besar Slide — Pertemuan 9: Case CI/CD

---

## SLIDE 1 — Cover

**Case: Continuous Integration & Continuous Deployment (CI/CD)**
Operasional Pengembang (DevOps) | Pertemuan 9

---

## SLIDE 2 — Recap & Bridge

**Apa yang sudah kita kuasai:**

| Pertemuan | Topik                         | Relevansi Hari Ini               |
| --------- | ----------------------------- | -------------------------------- |
| P4        | Pemeliharaan Kode & Pengujian | Unit/integration test dalam CI   |
| P5        | Deployment & Config as Code   | Pipeline-as-Code (YAML)          |
| P6        | Kontainerisasi                | Docker image sebagai artifact CD |

> **Pertanyaan pemantik**: *Bayangkan ada 5 developer dalam satu tim — bagaimana cara memastikan semua perubahan kode yang masuk selalu dalam kondisi bisa berjalan, kapan saja, tanpa manual cek satu per satu?*

---

# ══════════════════════════════

# BAGIAN 1: KONSEP DASAR CI/CD

# ══════════════════════════════

## SLIDE 3 — Masalah Tanpa CI/CD: "Integration Hell"

**Skenario nyata** di tim yang belum pakai CI/CD:

```
Senin   → Dev A & B mulai coding di branch masing-masing
Jumat   → Dev A selesai, merge ke main ✅
Jumat   → Dev B coba merge → 47 konflik! 😱
Sabtu   → Debug konflik + bug baru muncul
Senin   → Deployment manual gagal karena env berbeda
Selasa  → Baru bisa rilis (1,5 minggu untuk 1 fitur kecil)
```

**Akar masalah:**

- Integrasi dilakukan jarang dan terlambat
- Tidak ada otomasi pengujian
- Deploy manual → rentan human error
- Feedback loop sangat lambat

### SLIDE 4 — Apa Itu CI/CD?

```
CI — Continuous Integration
─────────────────────────────
Developer push kode → Build otomatis → Test otomatis → Feedback cepat
Tujuan: Pastikan kode selalu bisa di-build dan semua test lulus

CD — Continuous Delivery
─────────────────────────────
Setelah CI hijau → Kode selalu siap di-deploy ke production
Deploy dilakukan dengan PERSETUJUAN MANUAL
Tujuan: Rilis bisa dilakukan kapan saja tanpa persiapan panjang

CD — Continuous Deployment
─────────────────────────────
Setelah CI hijau → Deploy ke production OTOMATIS
Tanpa intervensi manusia sama sekali
Tujuan: Zero-touch deployment, feedback dari user secepatnya
```

### SLIDE 5 — CI vs CD vs CD: Apa Bedanya?

```
Commit ──► CI Pipeline ──► Artifact siap
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
   Continuous Delivery                Continuous Deployment
   Kode siap di-deploy                Deploy langsung otomatis
   ──────────────────                 ──────────────────────
   ✅ CI lulus                        ✅ CI lulus
   ✅ Artifact dibuat                 ✅ Artifact dibuat
   👆 Manusia klik "Deploy"           🤖 Deploy otomatis
              │                                 │
              ▼                                 ▼
          Staging/Prod                     Production
```

**Mana yang lebih baik?** → Tergantung konteks!

- Fintech/healthcare: Continuous Delivery (audit trail, approval)
- SaaS/startup: Continuous Deployment (kecepatan iterasi)

### SLIDE 6 — Posisi CI/CD dalam DevOps Lifecycle

```
  Plan ──► Code ──► Build ──► Test ──► Release ──► Deploy ──► Operate ──► Monitor
                     │          │          │           │
                     └──────────┴──────────┘           │
                            CI Pipeline                 │
                                                        └──────────────────┐
                                                              CD Pipeline   │
                                                                            ▼
                                                                       Feedback Loop
```

CI/CD adalah **jantung** dari DevOps — tanpanya, "Operate" dan "Monitor" tidak bisa berjalan cepat.

---

## SLIDE 7 — Anatomi CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                       CI/CD PIPELINE                        │
├──────────────┬──────────────────────────────────────────────┤
│  TRIGGER     │ push, PR, tag, schedule, manual dispatch     │
├──────────────┼──────────────────────────────────────────────┤
│  SOURCE      │ Checkout kode dari Git repository            │
├──────────────┼──────────────────────────────────────────────┤
│  BUILD       │ Install deps, compile, package               │
│              │ (pip install, npm install, mvn package, dll) │
├──────────────┼──────────────────────────────────────────────┤
│  TEST        │ Unit test, integration test, code coverage   │
├──────────────┼──────────────────────────────────────────────┤
│  ARTIFACT    │ Simpan hasil build (Docker image, JAR, zip)  │
│              │ Push ke registry / artifact store            │
├──────────────┼──────────────────────────────────────────────┤
│  DEPLOY      │ Deploy ke staging atau production            │
│              │ (SSH, Kubernetes, cloud run, dll)            │
├──────────────┼──────────────────────────────────────────────┤
│  NOTIFY      │ Slack, email, dashboard — sukses atau gagal  │
└──────────────┴──────────────────────────────────────────────┘
```

### SLIDE 8 — Pipeline Triggers

```yaml
on:
  push:
    branches: [main, develop]     # ← Setiap push ke branch ini
  pull_request:
    branches: [main]              # ← Setiap PR ke main
  schedule:
    - cron: '0 2 * * *'          # ← Setiap hari jam 02:00
  workflow_dispatch:              # ← Manual trigger dari UI
  release:
    types: [published]           # ← Saat release baru dibuat
```

**Strategi trigger yang umum:**

- `push` ke `develop` → jalankan CI (build + test)
- `PR` ke `main` → jalankan CI + review gate
- `push` ke `main` → jalankan CI + CD (deploy staging)
- `tag` release → deploy ke production

### SLIDE 9 — Jenis Test dalam Pipeline

| Level                      | Contoh                  | Kecepatan  | Biaya        |
| -------------------------- | ----------------------- | ---------- | ------------ |
| **Unit Test**        | pytest, JUnit, Jest     | Detik      | Sangat murah |
| **Integration Test** | DB connection, API call | Menit      | Murah        |
| **End-to-End Test**  | Selenium, Playwright    | Menit–Jam | Mahal        |
| **Performance Test** | k6, Locust, JMeter      | Menit–Jam | Mahal        |

**Prinsip Fail-Fast:**

```
Jalankan yang CEPAT dan MURAH dulu
            ↓
     Unit Test (detik)
            ↓ lulus?
  Integration Test (menit)
            ↓ lulus?
   E2E Test (menit-jam)
            ↓ lulus?
        DEPLOY ✅
```

Jika unit test gagal, tidak perlu buang waktu menunggu E2E test.

### SLIDE 10 — Artifact & Immutable Build

**Artifact** = output terpaketkan dari proses build:

- Python/Flask → Docker image
- Java → JAR / WAR file
- JavaScript → bundled JS / Docker image
- Go → binary executable

**Prinsip Immutable Artifact:**

```
SALAH ❌:
  Build di dev → build ulang di staging → build ulang di prod
  (Masing-masing bisa menghasilkan hasil berbeda!)

BENAR ✅:
  Build SEKALI → simpan di registry dengan tag versi
  Deploy artifact yang SAMA ke semua environment
  dev ──► staging ──► prod  (image yang persis sama)
```

**Tag strategy:**

```
myapp:sha-a3f2c1d    ← Tag berdasarkan commit SHA (unik)
myapp:v1.2.3         ← Tag berdasarkan semantic version
myapp:latest         ← Tag convenience (selalu yang terbaru)
```

---

# ══════════════════════════════

# BAGIAN 2: GITHUB ACTIONS

# ══════════════════════════════

### SLIDE 11 — GitHub Actions: Tool CI/CD Pilihan Lab

**Mengapa GitHub Actions?**

- Gratis untuk repo publik (2.000 menit/bulan private)
- Terintegrasi langsung dengan GitHub (code + pipeline dalam satu tempat)
- Marketplace 20.000+ action yang bisa dipakai ulang
- Sintaks YAML yang konsisten dan mudah dipelajari
- Runner: ubuntu, windows, macos tersedia built-in

**Konsep Kunci:**

```
Workflow  → File YAML di .github/workflows/ (unit terbesar)
  │
  ├── Event → Trigger yang menjalankan workflow
  │
  └── Job → Kumpulan steps di satu runner
        │
        └── Step → Satu perintah (run:) atau action (uses:)
```

### SLIDE 12 — Anatomi File Workflow

```yaml
name: CI Pipeline                  # ① Nama Workflow

on:                                # ② TRIGGER
  push:
    branches: [main]

jobs:                              # ③ JOBS
  build-and-test:                  # Nama job (bebas)
    runs-on: ubuntu-latest         # ④ RUNNER

    steps:                         # ⑤ STEPS
      - name: Checkout kode
        uses: actions/checkout@v4  # Pakai action dari marketplace

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install deps
        run: pip install -r requirements.txt  # Jalankan shell command

      - name: Unit Test
        run: pytest --cov=./ --cov-report=xml
```

### SLIDE 13 — Multi-Job Pipeline dengan Dependencies

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps: [...]          # Unit test + lint

  build:
    runs-on: ubuntu-latest
    needs: test            # ← Hanya jalan jika 'test' sukses
    steps: [...]          # Docker build + push

  deploy-staging:
    runs-on: ubuntu-latest
    needs: build           # ← Hanya jalan jika 'build' sukses
    steps: [...]          # Deploy ke staging

  deploy-prod:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production  # ← Memerlukan manual approval
    steps: [...]
```

**Dependency graph:**

```
test ──► build ──► deploy-staging ──► deploy-prod
                                       (approval)
```

---

# ══════════════════════════════

# BAGIAN 3: BEST PRACTICES CI/CD

# ══════════════════════════════

### SLIDE 14 — 8 Best Practices CI/CD

1. **Commit kecil dan sering**Commit besar = merge conflict besar. Target: commit tiap beberapa jam.
2. **Pipeline harus cepat**Target < 10 menit untuk feedback CI. Pipeline lambat = developer tidak menunggu.
3. **Fail Fast**Jalankan test cepat dulu. Jangan buang waktu jika unit test sudah gagal.
4. **Immutable Artifact**Build sekali, deploy ke semua environment. Tidak ada rebuild di production.
5. **Environment Parity**Staging harus semirip mungkin dengan production (data, config, infrastruktur).
6. **Pipeline as Code**Semua konfigurasi pipeline ada di Git — bisa di-review, di-audit, di-revert.
7. **Visibility**Semua orang di tim bisa melihat status pipeline. Notifikasi ke Slack/email.
8. **Rollback Strategy**
   Selalu ada cara kembali ke versi sebelumnya jika deployment bermasalah.

### SLIDE 15 — Branching Strategy & CI/CD

**Git Flow** (cocok untuk rilis terjadwal):

```
main         ●────────────────────●────────── (production)
              \                  /
develop        ●──●──●──●──●──●              (staging)
                \     \
feature          ●──●   ●──●──●              (dev)
```

**Trunk-Based Development** (cocok untuk continuous deployment):

```
main/trunk   ●──●──●──●──●──●──●──●──●      (langsung ke prod)
              \   \       \
feature        ●   ●──●    ●                 (short-lived, maks 1-2 hari)
```

| Aspek        | Git Flow                   | Trunk-Based            |
| ------------ | -------------------------- | ---------------------- |
| Cocok untuk  | Rilis terjadwal, tim besar | Startup, deploy sering |
| Kompleksitas | Lebih kompleks             | Lebih sederhana        |
| CI/CD speed  | Lebih lambat               | Lebih cepat            |

### SLIDE 16 — Anti-Pattern: Jangan Lakukan Ini!

| Anti-Pattern                   | Risiko                             | Solusi                             |
| ------------------------------ | ---------------------------------- | ---------------------------------- |
| "Works on my machine"          | Env dev ≠ prod → bug tersembunyi | Docker + environment parity        |
| Pipeline di-skip saat deadline | Bug lolos ke production            | Budaya: pipeline = non-negotiable  |
| Artifact tidak di-tag versi    | Tidak bisa rollback tepat          | Commit SHA atau semver sebagai tag |
| Test coverage rendah           | Pipeline lulus tapi bug ada        | Enforce minimum coverage threshold |
| Manual deploy tanpa pipeline   | Human error, tidak reproducible    | Semua deploy via pipeline          |
| Tidak ada notifikasi gagal     | Tim tidak tahu ada masalah         | Slack/email integration            |

### SLIDE 17 — Environment Strategy & Deployment

```
┌────────────┐    CI lulus    ┌────────────┐   Approved   ┌────────────┐
│  Development│ ─────────────► │  Staging   │ ───────────► │ Production │
│  (dev/feat) │               │  (mirror)  │              │  (live)    │
└────────────┘               └────────────┘              └────────────┘
     │                             │                           │
  Setiap push                Otomatis dari                Manual approval
  ke feature branch          merge ke main               atau auto (CD)
```

**Smoke Test setelah Deploy:**

```bash
# Pastikan service merespons setelah deploy
curl -f https://staging.myapp.com/health || exit 1
echo "✅ Smoke test passed"
```

---

# BAGIAN 4: DEVSECOPS DALAM CI/CD

# SLIDE 18 — Mengapa Security Masuk ke Pipeline?

**Fakta:**

- Bug keamanan yang ditemukan di production: **100× lebih mahal** diperbaiki vs saat coding
- Rata-rata: bug ditemukan **80 hari** setelah diintroduksi ke kode
- CI/CD yang cepat tanpa security = deploy bug keamanan lebih cepat

**Prinsip "Shift Left"** = geser pemeriksaan keamanan ke lebih awal:

```
Kode ditulis → [🔒 scan] → Build → [🔒 scan] → Deploy
                  ↑                      ↑
            Lebih murah             Lebih mahal
            & lebih cepat            diperbaiki
```

### SLIDE 19 — Security Checks yang Bisa Ditambahkan ke Pipeline

| Check                     | Tool            | Apa yang Dideteksi                  | Posisi dalam Pipeline  |
| ------------------------- | --------------- | ----------------------------------- | ---------------------- |
| **Secrets Scan**    | Gitleaks        | Credential/API key di kode          | Awal (sebelum build)   |
| **SAST**            | Semgrep, Bandit | Code vulnerability (injection, dll) | Setelah checkout       |
| **Dependency Scan** | Trivy, Snyk     | CVE di library yang digunakan       | Setelah install deps   |
| **Container Scan**  | Trivy           | CVE di base image Docker            | Setelah build image    |
| **DAST**            | OWASP ZAP       | Serangan runtime (saat app jalan)   | Setelah deploy staging |

**Untuk pemula**: Mulai dari **Dependency Scan** dulu — mudah diintegrasikan, dampak langsung.

### SLIDE 20 — Studi Kasus: Log4Shell & Dependency Scan

**CVE-2021-44228 (Log4Shell) — CVSS Score: 10.0 (tertinggi)**

- Library Log4j digunakan oleh jutaan aplikasi Java
- Celah: eksekusi kode remote hanya dengan mengirim string `${jndi:ldap://...}`
- Jutaan server terekspos dalam hitungan jam setelah pengumuman

**Bagaimana Dependency Scan membantu:**

```
Developer push kode dengan Log4j 2.14.1
              ↓
    Pipeline: Dependency Scan (Trivy)
              ↓
    ❌ CRITICAL: CVE-2021-44228 ditemukan di log4j:2.14.1
       Fix: upgrade ke log4j:2.17.1
              ↓
    Pipeline BLOKIR — tidak bisa merge ke main
    Developer dapat notifikasi langsung
```

**Tanpa pipeline security scan**: Vulnerability ini lolos ke production.

### SLIDE 21 — Secrets Management dalam Pipeline

**JANGAN hardcode credential:**

```yaml
# ❌ BERBAHAYA — tersimpan di Git, semua orang bisa lihat
env:
  DB_PASSWORD: "superpassword123"
  AWS_SECRET: "wJalrXUtnFEMI/K7MDENG..."
```

**Gunakan GitHub Secrets:**

```yaml
# ✅ BENAR — nilai dienkripsi, tidak pernah muncul di log
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  AWS_SECRET: ${{ secrets.AWS_SECRET_KEY }}
```

**Cara tambah secret di GitHub:**
Settings → Secrets and variables → Actions → New repository secret

---

## PENUTUP

### SLIDE 22 — Ringkasan Pertemuan

```
CI/CD = Otomasi seluruh proses dari push kode hingga deploy

Pipeline: Trigger → Source → Build → Test → Artifact → Deploy → Notify

Best Practices:
  ✓ Commit kecil & sering
  ✓ Pipeline < 10 menit
  ✓ Fail fast
  ✓ Immutable artifact
  ✓ Pipeline as Code

DevSecOps tambahan:
  ✓ Dependency scan
  ✓ Secrets management
  ✓ Shift Left
```

### SLIDE 23 — Preview Pertemuan 10 & Selanjutnya

- **P10** — Kuliah Tamu Praktisi: Masa Depan & Pola Lanjutan DevSecOps
- **P11** — Kubernetes & Microservices
  → Image Docker dari pipeline ini akan di-deploy ke Kubernetes

### SLIDE 24 — Tugas Individu

**Buat CI/CD Pipeline sederhana** menggunakan GitHub Actions:

| Komponen                                       | Poin          |
| ---------------------------------------------- | ------------- |
| Workflow berjalan saat push ke main            | 20            |
| Install dependencies + unit test (pytest/jest) | 25            |
| Code coverage minimal 70%                      | 15            |
| Build Docker image                             | 25            |
| Push ke GHCR + README dengan badge status      | 15            |
| **Total**                                | **100** |

**Bonus (+20)**: Tambahkan dependency scan (Trivy) yang memblokir jika ada CRITICAL CVE.

Submit: Link repository GitHub (Actions log harus bisa dilihat) | Deadline: Pertemuan 11
