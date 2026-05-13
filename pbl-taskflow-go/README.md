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
##### 1
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

##### 2
Memperbaiki BUG, terdapat 3 BUG untuk diperbaiki

**BUG 1: `BUG: != seharusnya =`**

Sebelum
```
FROM tasks WHERE status != $1 ORDER BY created_at DESC`
```
Setelah
```
FROM tasks WHERE status = $1 ORDER BY created_at DESC`
```
Dokumentasi:

<img width="754" height="229" alt="image" src="https://github.com/user-attachments/assets/0238ed02-5cd1-4c34-9bb2-5eb8aaa93170" />

**BUG 2: `BUG: != seharusnya ==`**

Sebelum
```
if t.Status != status { // BUG: seharusnya == bukan !=
```
Setelah
```
if t.Status == status { // BUG: seharusnya == bukan !=
```
Dokumentasi:

<img width="681" height="232" alt="image" src="https://github.com/user-attachments/assets/bfff5b97-d0c1-4a6a-9104-2db698559f71" />

**BUG 3: `BUG: integer division — hasil selalu 0 kecuali semua task selesai.`**

Sebelum
```
return float64(completed/len(tasks)) * 100
```
Setelah
```
return (float64(completed) / float64(len(tasks))) * 100
```
Dokumentasi:

<img width="519" height="255" alt="image" src="https://github.com/user-attachments/assets/5137e323-9581-470f-bb64-97c78e05c088" />

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

#### Stage 4: Smoke Test


Setelah selesai Covarage validation, berikut adalah aristektur untuk smoke testing otomatis berjalan:

```
┌──────────────────────────────────────────────────────────────┐
│                BUILD DOCKER IMAGE SELESAI                    │
│             Image sudah ada di GitLab Registry               │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   smoke_test_job     │
                    │   Stage: verify      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Docker-in-Docker    │
                    │  Service Aktif       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Login ke Registry   │
                    │  docker login        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Pull Image App      │
                    │  $CONTAINER_IMAGE    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Buat Network Docker │
                    │  smoke-net           │
                    └──────────┬───────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ▼                                 ▼
   ┌──────────────────────┐          ┌──────────────────────┐
   │  Jalankan PostgreSQL │          │   Jalankan App       │
   │  postgres-smoke      │◄─────────│   app-smoke          │
   │  port 5432           │ DATABASE │   port 8080          │
   └──────────┬───────────┘          └──────────┬───────────┘
              │                                 │
              ▼                                 ▼
   ┌──────────────────────┐          ┌──────────────────────┐
   │  Tunggu Database     │          │  Tunggu App Start    │
   │  pg_isready          │          │  sleep 15            │
   └──────────┬───────────┘          └──────────┬───────────┘
              │                                 │
              └────────────────┬────────────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Buat Socat Tunnel   │
                    │ localhost:8080       │
                    │        ke app:8080   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Smoke Test /health  │
                    │  curl localhost:8080 │
                    └──────────┬───────────┘
                               │
                    ❌ ADA FAILURE? ❌
                    │         │
              YES  ├─────────┤  NO
                   │         │
                   ▼         ▼
            ┌──────────┐  ┌──────────────────────┐
            │  LOG APP │  │ Smoke Test /stats    │
            │  FAILED  │  │ /api/v1/stats        │
            │  exit 1  │  └──────────┬───────────┘
            └────┬─────┘             │
                 │        ❌ ADA FAILURE? ❌
                 │        │         │
                 │  YES  ├─────────┤  NO
                 │       │         │
                 │       ▼         ▼
                 │ ┌──────────┐  ┌──────────────────────┐
                 │ │  LOG APP │  │ SMOKE TEST BERHASIL  │
                 │ │  FAILED  │  │ ✅                   │
                 │ │  exit 1  │  └──────────┬───────────┘
                 │ └────┬─────┘             │
                 │      │                   │
                 └──────┴─────────┬─────────┘
                                  ▼
                    ┌──────────────────────┐
                    │  Cleanup Container   │
                    │  stop & remove       │
                    │  app + postgres      │
                    └──────────────────────┘

```

**Isolated Network:** Membuat smoke-net agar kontainer Aplikasi dan Database PostgreSQL dapat saling berkomunikasi via hostname.

**Ephemeral Environment:** Database dan Aplikasi dijalankan secara sementara dalam satu tahapan (verify) untuk mensimulasikan kondisi produksi.

**Port Forwarding (Socat):** Menggunakan socat sebagai jembatan (tunnel) untuk menghubungkan ``localhost:8080`` pada GitLab Runner ke kontainer aplikasi di dalam Docker service. Hal ini memungkinkan perintah curl dijalankan langsung ke target localhost.




# Docker Build & Container Registry

Dokumen ini mencakup implementasi Docker build dan integrasi registry untuk layanan Taskflow API. Implementasi menggunakan Dockerfile multi-stage untuk menghasilkan image produksi yang minimal, yang kemudian secara otomatis di-build, di-tag, dan di-push ke GitLab Container Registry melalui pipeline CI/CD.

---

## Struktur Pipeline

Pipeline terdiri dari 6 stage yang berjalan secara berurutan:

```
init → test → build → package → verify → release
```

| Stage | Job | Keterangan |
|---|---|---|
| `init` | `prepare_module` | `go mod tidy`, menghasilkan artifact `go.mod` |
| `test` | `test_job` | `go test -v ./...` |
| `build` | `compile-binary` | Build binary, menghasilkan artifact `bin/taskflow-api` |
| `package` | `build-docker-image` | **Build & push Docker image** ← bagian ini |
| `verify` | `smoke_test_job` | Smoke test endpoint `/health` dan `/api/v1/stats` |
| `release` | `update-stable-image` | Tag image sebagai `stable` jika smoke test lulus |

Stage `package` hanya dieksekusi pada branch `main`.

---

### Penjelasan Desain

**Stage 1 — Builder** menggunakan `golang:1.23-alpine` sebagai environment kompilasi. Urutan instruksi dirancang untuk memaksimalkan cache layer Docker: `go.mod` dan `go.sum` disalin dan dependency diunduh terlebih dahulu sebelum source code di-copy, sehingga layer download dependency tidak perlu diulang selama `go.mod` tidak berubah.

Flag kompilasi yang digunakan:

| Flag | Keterangan |
|---|---|
| `CGO_ENABLED=0` | Menonaktifkan CGO sehingga binary bersifat static dan tidak bergantung pada shared library sistem |
| `GOOS=linux GOARCH=amd64` | Cross-compile untuk target Linux x86-64 |
| `-ldflags="-w -s"` | Menghapus debug info dan symbol table untuk memperkecil ukuran binary |

**Stage 2 — Runtime** menggunakan `scratch` sebagai base image — image kosong tanpa OS, shell, atau library apapun. Hanya dua file yang disalin dari stage builder: sertifikat CA untuk koneksi TLS dan binary `taskflow-api`. Hasilnya adalah image yang sangat kecil dengan attack surface minimal.

---

## Konfigurasi Pipeline CI/CD

Job `build-docker-image` pada stage `package`:

```yaml
variables:
  CONTAINER_IMAGE: $CI_REGISTRY_IMAGE:sha-$CI_COMMIT_SHORT_SHA
  STABLE_IMAGE:    $CI_REGISTRY_IMAGE:stable
  LATEST_IMAGE:    $CI_REGISTRY_IMAGE:latest

build-docker-image:
  stage: package
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  variables:
    DOCKER_TLS_CERTDIR: ""
  needs:
    - compile-binary
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY
        -u $CI_REGISTRY_USER --password-stdin
    - docker build -t $CONTAINER_IMAGE .
    - docker push $CONTAINER_IMAGE
  only:
    - main
```

Catatan konfigurasi:

- `DOCKER_TLS_CERTDIR: ""` — diset kosong agar Docker daemon dan client berkomunikasi tanpa TLS di dalam environment DinD pada GitLab shared runner.
- `needs: [compile-binary]` — job ini hanya berjalan setelah stage `build` sukses, sekaligus mengambil artifact binary yang dihasilkan.
- `only: [main]` — membatasi eksekusi hanya pada branch `main`.

---

## Log Eksekusi Pipeline

Hasil pipeline untuk commit `96bca13b`:
<img width="1280" height="831" alt="image" src="https://github.com/user-attachments/assets/ea5e4537-4f88-46e0-95ca-36a58ac5f70c" />
<img width="1280" height="831" alt="image" src="https://github.com/user-attachments/assets/b0b54e6d-6523-4284-90fe-4f123f39af63" />
<img width="1280" height="709" alt="image" src="https://github.com/user-attachments/assets/c74694c2-70ea-493f-bee5-61a47b0e0c9a" />
<img width="1280" height="379" alt="image" src="https://github.com/user-attachments/assets/13b5f843-c4e3-46e2-82d6-828ad6d7908d" />

| Tahap | Waktu | Durasi | Status |
|---|---|---|---|
| Preparing docker+machine executor | 02:45:39 | 00:28 | Berhasil |
| Preparing environment | 02:46:07 | 00:01 | Berhasil |
| Getting source from Git repository | 02:46:08 | 00:01 | Berhasil |
| Downloading artifacts (compile-binary) | 02:46:09 | 00:01 | Berhasil |
| Executing step_script | 02:46:10 | 00:44 | Berhasil |
| Cleaning up project directory | 02:46:54 | 00:01 | Berhasil |
| **Job succeeded** | 02:46:56 | — | **Berhasil** |

**Runner:** `green-4.saas-linux-small-amd64` | **Job ID:** `14231429555`

**Detail build per layer:**

| Layer | Instruksi | Durasi |
|---|---|---|
| #5 | `FROM golang:1.23-alpine` | 5.8s |
| #6 | `RUN apk add --no-cache git ca-certificates` (29 packages, 20 MiB) | 3.4s |
| #7 | `WORKDIR /build` | 0.0s |
| #8 | `COPY go.mod go.sum ./` | 0.0s |
| #9 | `RUN go mod download` | 1.4s |
| #10 | `COPY . .` | 0.0s |
| #11 | `RUN go vet ./...` | 24.8s |
| #12 | `RUN go build ... -o taskflow-api` | 0.7s |
| #13 | `COPY ca-certificates.crt` | 0.0s |
| #14 | `COPY taskflow-api` | 0.1s |
| #15 | Exporting image | 0.1s |

---

## Autentikasi Registry

Login menggunakan CI/CD variables bawaan GitLab tanpa kredensial yang di-hardcode:

```bash
$ echo "$CI_REGISTRY_PASSWORD" | docker login $CI_REGISTRY \
    -u $CI_REGISTRY_USER --password-stdin
Login Succeeded
```

---

## Tagging dan Push Image

Image di-tag menggunakan format `sha-<7-karakter-commit>` yang didefinisikan di variabel global pipeline:

```
CONTAINER_IMAGE = $CI_REGISTRY_IMAGE:sha-$CI_COMMIT_SHORT_SHA
                → registry.gitlab.com/haidarra-devops/week-9-devops-taskflow:sha-96bca13b
```

```bash
$ docker build -t $CONTAINER_IMAGE .
$ docker push $CONTAINER_IMAGE
```

Hasil push:

```
337a2c1016c5: Pushed
de0c226909cb: Pushed
sha-96bca13b: digest: sha256:b64d6f0d18c906f12e469c911678ecb30c21ef1ce60e238311f2bcb6a45ad349
size: 738
```

---

## Registry — Image yang Dipublikasikan

<img width="1600" height="174" alt="image" src="https://github.com/user-attachments/assets/bffb8900-3770-4e7d-b196-4fa8d39c11f5" />

| Tag | Ukuran | Digest | Dipublikasikan |
|---|---|---|---|
| `latest` | 3.36 MiB | `b64d6f0` | 3 hari lalu |
| `sha-96bca13b` | 3.36 MiB | `b64d6f0` | 3 hari lalu |

Kedua tag memiliki digest yang sama, mengonfirmasi bahwa keduanya merujuk pada image yang identik.

---

## Verifikasi Ukuran Image

```bash
$ docker images --format "Tag {{.Tag}} | Size {{.Size}}" | grep $CI_COMMIT_SHORT_SHA
Tag sha-96bca13b | Size 9.65MB
```

---

## Perbandingan Ukuran Image

| Metode Build | Base Image | Ukuran Final |
|---|---|---|
| Multi-stage (implementasi ini) | `golang:1.23-alpine` + `scratch` | **9.65 MB** |
| Single-stage (pembanding) | `golang:1.22` | ~300 MB (estimasi) |

Multi-stage build menghasilkan image sekitar **97% lebih kecil** dibandingkan single-stage build. Pada pendekatan single-stage, seluruh Go toolchain (~300 MB), dependensi build, dan source code ikut terbawa ke dalam image final. Dengan multi-stage, stage runtime hanya menerima dua file dari stage builder — binary `taskflow-api` dan sertifikat CA.


#### Stage 5: Rollback & Stable Tag

Di directory `pbl-taskflow-go`, jalankan command berikut:

```text
make rollback ROLLBACK_TAG=<image-tag> IMAGE=registry.gitlab.com/haidarra-devops/week-9-devops-taskflow
```

Jika ingin rollback ke image tertentu:

```text
make rollback ROLLBACK_TAG=sha-<commit-short-sha> IMAGE=registry.gitlab.com/haidarra-devops/week-9-devops-taskflow
```

##### Contoh

Rollback ke image tertentu:

```text
make rollback ROLLBACK_TAG=sha-1a2b3c4d IMAGE=registry.gitlab.com/haidarra-devops/week-9-devops-taskflow
````

Rollback ke image stable:

```text
make rollback ROLLBACK_TAG=stable IMAGE=registry.gitlab.com/haidarra-devops/week-9-devops-taskflow
````

##### Bukti Screenshot

Rollback:

<img width="1164" height="388" alt="image" src="https://github.com/user-attachments/assets/23f8cd17-8c5d-4a7a-8de1-adcb792cd6d5" />

<img width="1169" height="385" alt="image" src="https://github.com/user-attachments/assets/107c880c-3e02-4dee-ba93-f0651267b570" />

Stable tag:

<img width="1140" height="751" alt="image" src="https://github.com/user-attachments/assets/48ea12a6-84a2-41f1-b161-9bd6bc957132" />

<img width="1541" height="389" alt="image" src="https://github.com/user-attachments/assets/d040f2e3-9c60-432e-a579-f55b5b016a01" />


#### Stage 6 — Security Audit Pipeline

##### Tujuan
Menambahkan audit keamanan ke GitLab CI agar pipeline gagal otomatis jika ditemukan vulnerability dengan severity `HIGH` atau `CRITICAL`, sambil tetap menghasilkan artifact laporan JSON.

##### Kategori Security Scan yang Dikerjakan
1. **SCA (Dependency CVE)** menggunakan `trivy fs`.
2. **Container Image Scan** menggunakan `trivy image`.

##### Implementasi di Pipeline
1. Menambah stage `security-scan`.
2. Job `security_sca_fs`:
   - Menjalankan `trivy fs` dua kali:
     - Output JSON artifact (`--exit-code 0`)
     - Gate blocker (`--exit-code 1`)
3. Job `security_container_image`:
   - Menjalankan `trivy image` dua kali:
     - Output JSON artifact (`--exit-code 0`)
     - Gate blocker (`--exit-code 1`)
4. Artifact yang dihasilkan:
   - `security-reports/trivy-fs-report.json`
   - `security-reports/trivy-image-report.json`
5. Job release/stable tag dibuat bergantung ke hasil stage security scan agar tidak jalan saat security job gagal.

##### Hasil Uji (Branch `main`)
1. `security_sca_fs` menemukan `3` vulnerability (`HIGH: 1`, `CRITICAL: 2`) dan job gagal dengan `exit code 1`.
2. `security_container_image` menemukan `12` vulnerability (`HIGH: 9`, `CRITICAL: 3`) dan job gagal dengan `exit code 1`.
3. Kedua job tetap meng-upload artifact JSON (`201 Created`).
4. Pipeline `main` berstatus **Failed** pada stage `security-scan`, menandakan gate keamanan berjalan sesuai requirement.

##### Analisis Temuan (Ringkas)
1. **True positive** utama: `github.com/jackc/pgx/v5` (dipakai langsung oleh repository PostgreSQL).
2. `golang.org/x/crypto` dan beberapa CVE stdlib perlu analisis reachability lebih lanjut; sebagian belum dapat dipastikan exploitable pada runtime aplikasi saat ini.

##### False Positive vs True Positive per Kategori
Analisis ini membedakan tiga status:
1. **True Positive**: CVE terdeteksi benar pada package/version yang ada.
2. **Potentially Non-Reachable**: CVE valid, namun jalur kode rentan kemungkinan tidak dieksekusi oleh aplikasi saat ini.
3. **False Positive (scanner error)**: scanner salah deteksi package/version/CVE mapping.

Validasi reachability tambahan menggunakan `govulncheck`:
```bash
govulncheck ./...
```
Hasil:
- `Your code is affected by 0 vulnerabilities.`
- Ada vulnerability pada module/dependency, tetapi code path rentan **tidak dipanggil** oleh kode aplikasi saat ini.

1. **Kategori A — SCA (`trivy fs`)**
   - **True Positive**
     - `CVE-2026-33816` pada `github.com/jackc/pgx/v5` karena dependency ini dipakai langsung di `internal/repository/postgres.go`.
   - **Potentially Non-Reachable (requires reachability validation)**
     - `CVE-2024-45337` dan `CVE-2025-22869` pada `golang.org/x/crypto` (komponen `ssh`), perlu validasi reachability karena aplikasi tidak memakai fitur SSH secara eksplisit.
2. **Kategori D — Container Image (`trivy image`)**
   - **True Positive**
     - `CVE-2026-33816` (`pgx/v5`) tetap muncul pada image runtime.
     - Beberapa CVE `stdlib` terkait TLS/X509 berpotensi relevan untuk API service berbasis HTTP.
   - **Potentially Non-Reachable (requires reachability validation)**
     - CVE `stdlib` pada `archive/tar`/`archive/zip` dapat menjadi tidak relevan jika jalur parsing arsip tidak dipakai di runtime.
3. **Kesimpulan False Positive**
   - Sampai laporan ini ditulis, **tidak ada false positive murni yang terkonfirmasi**.
   - Temuan yang belum pasti diklasifikasikan sebagai **potentially non-reachable**, bukan false positive.

### Rekomendasi Perbaikan
1. Upgrade `github.com/jackc/pgx/v5` ke minimal `v5.9.0`.
2. Upgrade `golang.org/x/crypto` ke minimal `v0.35.0`.
3. Upgrade Go toolchain/runtime ke patch terbaru dan rebuild image.
4. Pertahankan gate `HIGH/CRITICAL` agar kerentanan tidak lolos ke rilis `stable`.

### Bukti Screenshot Skenario 6
1. Job `security_sca_fs` gagal karena gate HIGH/CRITICAL + artifact ter-upload.

<img width="1440" height="811" alt="Image" src="https://github.com/user-attachments/assets/57b56fc9-d21a-4b43-997e-74b324d64fd1" />


2. Job `security_container_image` gagal karena gate HIGH/CRITICAL + artifact ter-upload.

<img width="1440" height="811" alt="Image" src="https://github.com/user-attachments/assets/6c6f084b-ce4e-4f9c-bdc2-e10efb11bba1" />


3. Daftar pipeline menunjukkan pipeline `main` status **Failed** setelah stage security scan.

<img width="1440" height="811" alt="Image" src="https://github.com/user-attachments/assets/130f725a-4115-4eda-b2fe-e26d650b7d9a" />


4. Filter branch `main` menampilkan histori pipeline dan status gagal pada pipeline security scan.

<img width="1440" height="811" alt="Image" src="https://github.com/user-attachments/assets/0f339a99-a8fc-4ec2-bfee-0fe39e526b70" />
