# BooksLib - DevSecOps Pipeline (Track A)

Ini dokumentasi pengerjaan technical test DevSecOps Internship 2026. Repo ini adalah fork dari [bookslib](https://github.com/sncyber-ops/bookslib.git) yang sudah saya tambahkan pipeline Jenkins dengan beberapa security gates.

---

## 1. Arsitektur & Alur Kerja

Konsep yang saya terapkan adalah *Shift-Left Security* — keamanan dicek sedini mungkin sebelum kode masuk ke production. Setiap ada push ke branch manapun, Jenkins langsung menjalankan serangkaian security check secara berurutan:

```
Push ke GitHub
      │
      ▼
┌─────────────────────────────────────────────┐
│  Gate 0  │ IaC Scanning       │ Checkov      │  ← Cek Dockerfile & docker-compose
│  Gate 1  │ Secret Scanning    │ Trufflehog   │  ← Cek hardcoded credentials
│  Gate 2  │ SAST               │ Semgrep      │  ← Cek celah logika source code
├──────────┤────────────────────┴──────────────┤
│          │ Build Docker Images               │  ← Build image semua service
├──────────┤───────────────────────────────────┤
│  Gate 3  │ Container Scanning │ Trivy        │  ← Scan CVE di image (CRITICAL = fail)
│          │ Generate SBOM      │ Trivy        │  ← Buat manifest komposisi software
├──────────┤───────────────────────────────────┤
│          │ Deploy (branch main saja)         │  ← docker compose up -d
├──────────┤───────────────────────────────────┤
│  Gate 4  │ DAST               │ OWASP ZAP    │  ← Simulasi serangan dari luar
└──────────┴───────────────────────────────────┘
```

**Catatan:** Kalau Gate 1, 2, atau 3 gagal, Jenkins otomatis membuat GitHub Issue supaya ada jejak temuan keamanannya.

Branch `develop` hanya menjalankan tahap scan (Gate 0–3). Deploy dan DAST hanya berjalan di branch `main`.

---

## 2. Cara Menjalankan

**Prasyarat:**
- Docker & Docker Compose sudah terinstall
- Port 8080 (Jenkins) dan 3000 (aplikasi) tidak dipakai proses lain

**Langkah-langkah:**

1. Clone repo ini:
   ```bash
   git clone https://github.com/earthwrld/bookslib.git
   cd bookslib
   ```

2. Jalankan Jenkins:
   ```bash
   docker compose -f jenkins-docker-compose.yaml up -d
   ```
   Buka `http://localhost:8080`, selesaikan setup awal Jenkins, dan install plugin yang direkomendasikan.

3. Tambahkan kredensial GitHub:
   - Masuk ke **Manage Jenkins → Credentials**
   - Buat credential baru, tipe **Secret text**
   - ID: `github-token`, isi dengan GitHub Personal Access Token Anda
   - Ini wajib ada agar Jenkins bisa membuat Issue otomatis saat ada temuan

4. Buat Pipeline Job:
   - Buat job baru → tipe **Pipeline**
   - Pipeline script from SCM → Git
   - Repository URL: URL repo ini, branch: `main`
   - Script path: `Jenkinsfile`

5. Klik **Build Now**. Pipeline akan berjalan otomatis.

Setelah selesai, aplikasi bisa diakses di `http://localhost:3000`. Laporan ZAP dan SBOM tersedia di tab **Artifacts** di halaman build Jenkins.

---

## 3. Alasan Pemilihan Tools

| Tools | Alasan |
|---|---|
| **Jenkins** | Sesuai requirement. Self-hosted, kontrol penuh terhadap environment. |
| **Docker-in-Docker (DooD)** | Cara paling praktis menjalankan scanner (Trivy, Semgrep, ZAP) di dalam Jenkins tanpa setup Kubernetes. Trade-off: Jenkins dapat akses ke Docker socket host, kurang ideal di production enterprise. |
| **Checkov** | Bisa scan Dockerfile & docker-compose sekaligus dalam satu perintah, gratis, dan cukup akurat. |
| **Trufflehog** | Spesialis deteksi credentials & secret, false positive-nya lebih rendah dibanding tools sejenis. |
| **Semgrep** | Cepat, support banyak bahasa (Python, JS, Go), dan konfigurasi rulesnya fleksibel. |
| **Trivy** | All-in-one: bisa scan CVE sekaligus generate SBOM format CycloneDX. Tidak perlu dua tools berbeda. |
| **OWASP ZAP** | Standar industri untuk DAST, open source, cocok untuk baseline scan di environment lokal. |
| **Git Flow (main/develop)** | Branch `develop` untuk development & testing, `main` untuk production. Deploy hanya dari `main` supaya kode yang masuk ke production sudah melewati review. |

---

## 4. Kendala & Rencana Improvement

**Kendala yang ditemui:**

- **Path volume Docker-in-Docker tidak match** — Saat Jenkins (yang berjalan di dalam container) mencoba mount volume ke tools seperti ZAP atau Semgrep, path yang digunakan adalah path di dalam container Jenkins, bukan path di host OS. Solusinya dengan meresolve path host secara manual via `env.HOST_WORKSPACE` di Groovy script, lalu path itu yang dipakai untuk volume mount.

- **Semgrep false positive dari artifact sisa** — Semgrep sempat gagal karena ikut membaca file `zap-report.html` sisa build sebelumnya, dan menganggap link `http://localhost:3000` di dalamnya sebagai temuan keamanan. Solusinya dengan menambahkan `.semgrepignore` untuk mengecualikan file hasil report.

**Kalau ada waktu lebih:**

1. **Kubernetes (K3s) + Rolling Update** — Deployment pakai docker-compose masih ada downtime sebentar saat container restart. Migrasi ke K3s memungkinkan zero-downtime deployment yang diminta di nilai plus.
2. **SonarQube** — Tambahkan code quality analysis di luar SAST murni, supaya pipeline juga bisa deteksi code smell dan technical debt.
3. **Jenkins Agent terpisah** — Saat ini Jenkins controller dan agent jalan di node yang sama. Lebih baik dipisah supaya akses Docker socket tidak langsung di controller.

---

*Terima kasih sudah meluangkan waktu untuk mereview hasil pengerjaan ini.*