# Laporan CI/CD Pipeline & Deployment Process
 
## 1. Identitas Kelompok & Tool CI/CD
 
### Data Kelompok
| Aspek | Keterangan |
|-------|-----------|
| **Kelompok** | 2 |
| **Anggota** | Haidar, Furqon, Benjamin, Hildan, Kenas, Abid, Arsyad |
| **Project** | PBL-Taskflow-Go |
| **Repository** | `https://gitlab.com/haidarra-devops/week-9-devops-taskflow.git` |
| **Bahasa Pemrograman** | Go (Golang) |
| **Versi Go** | 1.23 |

### Tool CI/CD yang Digunakan
| Tool | Fungsi | Versi |
|------|--------|-------|
| **GitLab CI/CD** | pipeline CI/CD | Latest |
| **Docker** | Container image build & registry | v20.x+ |
 
---

## 2. Diagram Alur Pipeline Lengkap
 
### Workflow Pipeline: Push → Vet → Test → Build → Docker → Deploy
 
```
┌─────────────────────────────────────────────────────────────────────┐
│                     GIT PUSH EVENT (Developer)                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Gitlab CI/CD │
                    │   Trigger Job   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
   ┌─────────┐          ┌──────────┐        ┌────────────┐
   │  Vet    │          │  Unit    │        │ Integration│
   │  Code   │          │  Test    │        │   Test     │
   │ (lint)  │          │  (go t)  │        │ (DB Access)      │
   └────┬────┘          └────┬─────┘        └─────┬──────┘
        │                    │                     │
        └────────────────────┼─────────────────────┘
                             │
                    ❌ ADA FAILURE? ❌
                    │         │
              YES  ├─────────┤  NO
                   │         │
                   ▼         ▼
            ┌──────────┐  ┌──────────────┐
            │  NOTIFY  │  │  Build       │
            │  FAILED  │  │  Binary      │
            │  ❌      │  │  (go build)  │
            └──────────┘  └────┬─────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Build Docker Image  │
                    │  (Multi-stage build) │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Push to Registry    │
                    │ (tag: latest, sha)   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Smoke Test (S4)     │
                    │  - Health check      │
                    │  - API validation    │
                    └──────────┬───────────┘
                               │
                    ❌ ADA FAILURE? ❌
                    │         │
              YES  ├─────────┤  NO
                   │         │
                   ▼         ▼
            ┌──────────┐  ┌──────────────┐
            │  NOTIFY  │  │  Deploy to   │
            │  FAILED  │  │  Production  │
            │  ❌      │  │  (S5)        │
            └──────────┘  └────┬─────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Verification        │
                    │  - Port availability │
                    │  - Service ready     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  NOTIFY SUCCESS ✅   │
                    │  Pipeline Complete   │
                    └──────────────────────┘
```

### Rincian Setiap Stage

#### Stage 0: Init
Berfungsi untuk memastikan dependensi siap dan kode bisa dikompilasi dasar.
`.gitlab-ci.yml`:
```bash
image: golang:1.23-alpine

# 1. Definisikan stage yang BENAR dan sesuai dengan yang dipanggil job
stages:
  - test
  - build

variables:
  # Konfigurasi Database (Gunakan satu set saja agar tidak pusing)
  POSTGRES_DB: taskflow_test
  POSTGRES_USER: user
  POSTGRES_PASSWORD: password
  POSTGRES_HOST_AUTH_METHOD: trust
  # URL yang akan dibaca oleh aplikasi/test
  DATABASE_URL: "postgres://user:password@postgres:5432/taskflow_test?sslmode=disable"

services:
  - name: postgres:16-alpine
    alias: postgres

# Perintah yang dijalankan sebelum setiap job
before_script:
  - apk add --no-cache make gcc musl-dev
  - cd pbl-taskflow-go
  - go mod tidy

# Job Utama: Gabungkan Vet, Unit Test, dan Integration Test di sini
# Ini cara paling efektif untuk melihat angka coverage asli
test_job:
  stage: test
  script:
    # Cek static analysis
    - go vet ./...
    # Jalankan semua test (termasuk integration) dan buat laporan coverage
    - go test -v -race -tags=integration -coverprofile=coverage.out -covermode=atomic ./...
    # Munculkan angka coverage di terminal (supaya ditangkap GitLab)
    - go tool cover -func=coverage.out
    # Logika fail jika di bawah 75%
    - |
      TOTAL_COV=$(go tool cover -func=coverage.out | grep total | grep -Eo '[0-9]+\.[0-9]+')
      echo "Total Coverage Akhir: $TOTAL_COV%"
      if [ "$(echo "$TOTAL_COV < 75.0" | bc -l)" -eq 1 ]; then
        echo "❌ Skor Coverage ($TOTAL_COV%) masih di bawah 75%!"
        exit 1
      fi
  coverage: '/total:\s+\(statements\)\s+(\d+.\d+)%/'
  artifacts:
    paths:
      - pbl-taskflow-go/coverage.out
    expire_in: 1 week

# Job opsional untuk build binary jika semua test lulus
compile-binary:
  stage: build
  script:
    - mkdir -p bin
    - go build -o bin/taskflow-api ./cmd/server/main.go
  artifacts:
    paths:
      - pbl-taskflow-go/bin/
```

- Infrastructure as Code: Menggunakan postgres:16-alpine sebagai database asli untuk testing, bukan data palsu.

- Security & Quality: Menjalankan go vet (cek struktur kode) dan go test -race (cek konflik memori) secara simultan.

- Hard-Gate Coverage: Pipeline akan langsung GAGAL jika pengujian mencakup kurang dari 75% kode, memastikan fitur baru tidak di-push tanpa tes.

#### Stage 1: Code Checkout & Setup
- Trigger: Push ke branch `main` atau `develop` serta Merge Request.
- Environment: `golang:1.23-alpine`.
- Action:
     - Sistem checkout otomatis oleh GitLab Runner.

     - Instalasi tooling OS: `apk add make gcc musl-dev` (Wajib untuk mendukung Race Detector).

     - Dependency Management: `go mod tidy` & `go mod download`.`

  <img width="1255" height="509" alt="image" src="https://github.com/user-attachments/assets/8ba0197f-3673-4d8b-8dae-375dd821f3cc" />

#### Stage 2: Vet (Static Analysis)
```bash
go vet ./...
$ go test -v -race -tags=integration -coverprofile=coverage.out -covermode=atomic ./...
```

- `go test -race` (Deteksi Konflik): Menjalankan pengujian logika sekaligus mendeteksi Race Condition, memastikan aplikasi tidak crash atau korup saat diakses banyak pengguna secara bersamaan.
- `-tags=integration` (Validasi Database): Menjalankan pengujian yang terhubung langsung ke PostgreSQL, menjamin seluruh perintah SQL dan interaksi database asli berjalan tanpa error.

<img width="1254" height="951" alt="image" src="https://github.com/user-attachments/assets/b32bb1e1-30ac-407c-a408-19f04abb60cf" />

#### Stage 3: Coverage Validation (Gatekeeper)
```bash
go tool cover -func=coverage.out
```
- `go tool cover` (Audit Kualitas): Menghitung persentase baris kode yang telah diuji (mencapai 75.1%), berfungsi sebagai penjaga gerbang otomatis untuk menolak kode yang kualitas pengujiannya rendah.

<img width="1262" height="958" alt="image" src="https://github.com/user-attachments/assets/10348c6c-845f-4ca8-9ada-51625e221ddc" />


Sebelumnya juga sudah membuat 2 fungsi test baru di file `handler_test.go` yaitu:

<img width="561" height="259" alt="image" src="https://github.com/user-attachments/assets/a119491a-73d2-4dbf-86f3-b19a898df983" />

Dan

<img width="645" height="601" alt="image" src="https://github.com/user-attachments/assets/7f9bf4df-fbaf-4b5c-8a4f-b52aa9368756" />

Sehingga menghasilkan coverage ≥ 75% yaitu 75.1% seperti pada hasil di bawah
Hasil:
```bash
$ TOTAL_COV=$(go tool cover -func=coverage.out | grep total | grep -Eo '[0-9]+\.[0-9]+') # collapsed multi-line command
```


<img width="1075" height="44" alt="image" src="https://github.com/user-attachments/assets/e79e50ca-4d81-4929-b6b0-d8a024f6f9bc" />



Tampilan indikator berhasil pada pipeline:


<img width="1589" height="630" alt="image" src="https://github.com/user-attachments/assets/069b8c85-ff6f-407a-9492-f52e70f1531b" />


Dari hasil beberapa dokumentasi di atas membuktikan bahwa taskflow berhasil memperbaiki semua bug dengan benar, melewati semua test dengan status PASS, telah membuat 2 test baru, dan coverage berada di nilai ≥ 75% yaitu 75.1%
