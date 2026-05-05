# Pertemuan 9 — Case: CI/CD (Continuous Integration & Continuous Deployment)

**Mata Kuliah**: Operasional Pengembang (DevOps)
**Pertemuan ke**: 9 dari 16
**Durasi**: 3 × 50 menit (150 menit)
**Tanggal**: 29 April 2026

---

## Posisi dalam Silabus

```
P1  Pengantar DevSecOps & SDLC/Agile
P2  Modern DevSecOps & Dukungan Teknis Dasar
P3  Prinsip Dasar Keamanan & CIA
P4  Pemeliharaan Kode & Pengujian       ← test yang akan dipakai di CI
P5  Deployment & Configuration as Code  ← Pipeline-as-Code
P6  Kontainerisasi                      ← Docker image sebagai artifact
P7  Mini Project (Hari 1)
P8  Mini Project (Hari 2)
▶ P9  Case CI/CD                        ← HARI INI
P10 Kuliah Tamu: Masa Depan DevSecOps
P11 Kubernetes & Microservices
P12 Kubernetes #2
P13 Manajemen Rilis
P14 Monitoring
P15 Final Project
P16 Final Project
```

**Jembatan konsep ke depan**: Image Docker dari pipeline ini → akan di-deploy ke **Kubernetes (P11)**.

---

## Capaian Pembelajaran

Setelah mengikuti pertemuan ini, mahasiswa mampu:
1. Menjelaskan konsep, tujuan, dan manfaat CI/CD dalam siklus pengembangan perangkat lunak.
2. Membedakan Continuous Integration, Continuous Delivery, dan Continuous Deployment.
3. Mengidentifikasi dan menjelaskan setiap tahapan dalam CI/CD pipeline.
4. Mengimplementasikan CI/CD pipeline menggunakan GitHub Actions.
5. Menerapkan best practices CI/CD (immutable artifact, fail-fast, pipeline as code).
6. **(Tambahan DevSecOps)** Mengintegrasikan dependency security scan ke dalam pipeline.

---

## Jadwal Sesi (150 menit)

| Segmen | Waktu | Aktivitas |
|--------|-------|-----------|
| Pembukaan | 10 mnt | Review mini project + bridge ke CI/CD |
| **Materi 1** | 20 mnt | Konsep CI/CD, masalah tanpa CI/CD |
| **Materi 2** | 25 mnt | Anatomi pipeline, triggers, jenis test |
| **Materi 3** | 20 mnt | GitHub Actions: workflow, job, step |
| **Materi 4** | 15 mnt | Best practices + branching strategy |
| **Materi 5** | 10 mnt | DevSecOps sebagai tambahan (30%) |
| **Lab** | 30 mnt | Implementasi CI/CD pipeline |
| Kuis | 10 mnt | 10 soal pilihan ganda |
| Penutup | 10 mnt | Refleksi, tugas, preview Kubernetes |

---

## Komposisi Materi

```
70% CI/CD FUNDAMENTALS & BEST PRACTICES
├── Konsep & definisi CI, CD (Delivery), CD (Deployment)
├── Masalah tanpa CI/CD (Integration Hell)
├── Anatomi pipeline (trigger → source → build → test → artifact → deploy → notify)
├── GitHub Actions (workflow, job, step, runner, action)
├── Artifact & immutable build strategy
├── Best practices (fail-fast, pipeline as code, tagging, dll)
└── Branching strategy (Git Flow vs Trunk-Based)

30% DEVSECOPS TAMBAHAN
├── Prinsip Shift Left (security lebih awal = lebih murah)
├── Dependency vulnerability scan (Trivy/Snyk)
└── Secrets management (GitHub Secrets)
```

---

## Struktur File Modul

```
pertemuan-09-cicd/
├── README.md                  ← File ini
├── slides-outline.md          ← Garis besar 24 slide kuliah
├── lab-secure-pipeline.md     ← Lab CI/CD + bonus DevSecOps
└── quiz-pertemuan-09.gift.txt ← 10 soal GIFT untuk Moodle
```
