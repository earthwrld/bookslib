# BukuLib - DevSecOps Internship Test Submission

Halo! Ini adalah dokumentasi hasil pengerjaan technical test aku untuk posisi DevSecOps Internship (Track A). Di repo ini, aku udah nambahin pipeline CI/CD pakai Jenkins buat mastiin aplikasinya aman sebelum bener-bener di-deploy ke production.

## 1. Arsitektur & Alur Kerja (Workflow)
Aku nerapin konsep *Shift-Left Security* di pipeline Jenkins ini. Intinya, setiap ada kode baru yang di-push, Jenkins bakal jalanin serangkaian satpam (Security Gates) secara otomatis:

1. **Setup & Checkout:** Ambil kode terbaru dari repository.
2. **Gate 0 - IaC Scanning (Checkov):** Ngecek konfigurasi `Dockerfile` dan `docker-compose` biar nggak ada *misconfig* infrastruktur (misal jalanin container pake akses root).
3. **Gate 1 - Secret Scanning (Trufflehog):** Sweeping kode barangkali ada API key atau password database yang nggak sengaja ke-commit (hardcoded).
4. **Gate 2 - SAST (Semgrep):** Ngecek *source code* mentah buat nyari celah logika kaya SQL Injection atau XSS tanpa harus jalanin aplikasinya.
5. **Build Image:** Nge-build Docker image buat persiapan tahap selanjutnya.
6. **Gate 3 - Container Scanning (Trivy):** Nge-scan image Docker yang barusan dibikin buat ngecek *vulnerabilities* (CVE) di level OS atau library-nya. Kalau nemu celah level CRITICAL, pipeline langsung gagal (blocking).
7. **Generate SBOM:** Bikin *Software Bill of Materials* (kayak label komposisi bahan aplikasi) pake format CycloneDX buat kepatuhan supply chain.
8. **Deploy to Production:** Kalau semua lolos dan ini adalah branch `main`, aplikasi bakal di-deploy pakai `docker compose`. Kalau branch `develop`, pipeline berhenti sebelum deploy (jadi cuma buat ngetest keamanannya aja).
9. **Gate 4 - DAST (OWASP ZAP):** Nyerang aplikasi yang udah jalan dari luar (simulasi hacker) buat ngecek celah misal header HTTP yang kurang aman. Laporannya disimpen di Jenkins.

*Catatan:* Kalau Gate 1, 2, atau 3 gagal, Jenkins udah disetting buat otomatis manggil GitHub API buat bikin **Issue** baru di repo ini, ngasih tau tim ada celah yang harus diperbaiki.

## 2. Langkah-langkah Mereproduksi Sistem
Kalau mau nyoba jalanin di lokal, ikuti langkah ini ya:

**Prasyarat:**
- Udah install Docker & Docker Compose.
- Udah sedia Jenkins yang jalan di lokal. Di sini aku pakai Jenkins versi docker dengan binding `docker.sock` supaya bisa manfaatin Docker-in-Docker (DooD).

**Cara Deploy:**
1. Clone repository ini:
   ```bash
   git clone https://github.com/[username-github-kamu]/bookslib.git
   cd bookslib
   ```
2. Setup Jenkins:
   - Jalankan Jenkins container yang udah disediain di file `jenkins-docker-compose.yaml`:
     ```bash
     docker compose -f jenkins-docker-compose.yaml up -d
     ```
   - Masuk ke Jenkins UI (localhost:8080), setup awal dan install plugin standar.
   - Bikin kredensial baru (tipe Secret text) dengan ID `github-token` dan isi dengan GitHub Personal Access Token kamu. Ini wajib biar Jenkins bisa otomatis bikin Issue di GitHub kalau nemu bug.
3. Bikin Job Pipeline di Jenkins:
   - Bikin job baru, pilih tipe "Pipeline".
   - Di bagian Pipeline script, pilih "Pipeline script from SCM".
   - Masukin URL repo ini dan set branch ke `main` atau `develop`.
   - Script path isi dengan `Jenkinsfile`.
4. Save, lalu klik **Build Now**. Jenkins bakal otomatis ngejalanin semua tahap security test dan langsung nge-deploy aplikasinya (kalau branch `main`).
5. Selesai! Cek aplikasi jalan di `http://localhost:3000`. Laporan ZAP (HTML) dan SBOM (JSON) bisa didownload langsung dari halaman artifact Jenkins.

## 3. Trade-off & Alasan Pemilihan Tools
- **Jenkins + Docker-in-Docker (DooD):** Buat jalanin *security scanning* (Trivy, Semgrep, dll), aku pakai docker container yang di-spin-up oleh Jenkins melalui `/var/run/docker.sock`. Trade-off-nya ini sebenernya kurang *secure* buat production karena Jenkins dapet akses root ke host Docker, tapi ini cara paling masuk akal, fleksibel, dan ringan buat ngerjain test ini di lokal tanpa harus nge-setup Kubernetes atau agent server terpisah.
- **Trufflehog & Semgrep:** Aku milih dua kombinasi ini buat Secret Scan & SAST karena performanya cepet banget, gampang dipanggil lewat CLI docker, dan lumayan akurat buat nemuin celah umum bahasa pemrograman modern.
- **Trivy:** Aku pakai buat Container Scanning sekaligus bikin SBOM karena praktis *all-in-one*, ringan, cepat, dan format SBOM-nya udah mengikuti standar industri (CycloneDX).
- **Git Flow:** Pipeline sengaja diset biar branch `develop` cuma ngetest keamanan tanpa lanjut deploy. Trade-off-nya developer nggak dapet *preview link*, tapi ngebantu banget mencegah celah masuk ke `main` sejak awal.

## 4. Kendala & Improvement Kedepannya
Jujur, proses setupnya lumayan menantang dan ada beberapa kendala teknis yang sempet bikin stuck:
- **Konflik Volume Docker-in-Docker:** Pas mau nge-generate report ZAP dan Semgrep di dalam Jenkins, file laporannya sempet gagal kesimpen karena path direktori di Host OS dan di dalam container Jenkins itu beda. Aku ngakalin ini dengan nyari path asli di host (`env.HOST_WORKSPACE`) pakai script Groovy, terus path itu yang dipassing ke *volume mount* Dockernya. 
- **Semgrep nge-scan artifact sisa (False Positive):** Semgrep sempet ngasih status FAILED karena dia ngebaca file `zap-report.html` dari sisa *build* hari sebelumnya. Solusinya simpel, aku buatin `.semgrepignore` biar artifact sisa nggak ikut di-scan lagi.

**Kalau ada waktu lebih banyak, ini yang pengen aku perbaiki:**
1. Mindahin sistem deployment pakai **Kubernetes (K3s)** biar bener-bener bisa nerapin *Zero-Downtime Deployment* (misal pakai *Rolling Update*). Pakai docker-compose kaya sekarang emang gampang banget, tapi selalu ada jeda *downtime* sedetik/dua detik pas container direstart ulang.
2. Integrasi **Sonarqube** buat inspeksi *Code Quality* secara menyeluruh, jadi bukan sekadar ngecek celah bahaya, tapi juga ngingetin kalau ada kode yang berantakan atau *code smell*.
3. Misahin Jenkins agent dari Jenkins controller pakai node yang beda, biar arsitekturnya lebih aman dari potensi *privilege escalation* Docker socketnya.

---
Terima kasih banyak udah ngeluangin waktu buat review hasil kerjaku!