# Rancangan Pipeline CI/CD

**Nama Proyek:** TokoKu: Aplikasi Toko Online Sederhana
**Teknologi:** Node.js (Express), PostgreSQL, Docker
**Platform CI/CD:** GitHub Actions

Dokumen ini menjelaskan rancangan pipeline otomatis untuk TokoKu, mulai dari kode ditulis hingga aplikasi berjalan di server.

---

## Trigger (Pemicu Pipeline)

| Event | Branch | Stage yang Dijalankan |
|---|---|---|
| `push` | `master` | Semua stage (Source Control sampai Deployment ke production) |
| `push` | `develop` | Source Control sampai Artifact Generation, lalu deploy ke staging |
| `pull_request` | target ke `master` atau `develop` | Source Control, Build, Test, Security Scanning (tanpa deployment) |
| `schedule` (setiap Senin 02.00 UTC) | `master` | Security Scanning (pemindaian kerentanan berkala) |
| `workflow_dispatch` | bebas | Dijalankan manual bila diperlukan |

Branch `feature/*` tidak memicu pipeline secara langsung; pipeline berjalan ketika Pull Request dibuka.

---

## 1. Source Control

**Tujuan:** Mengelola kode sumber dan memastikan setiap perubahan terlacak.

- Repository: GitHub (`hello-world`)
- Strategi branching: `master` (production), `develop` (staging), `feature/*` (pengembangan fitur)
- Perintah/alat:
  - `git checkout -b feature/keranjang-belanja`
  - `git add .` dan `git commit -m "feat: tambah keranjang belanja"`
  - `git push origin feature/keranjang-belanja`
  - Pull Request dengan minimal 1 reviewer
  - Branch protection pada `master` (wajib lulus pipeline sebelum merge)
  - Aksi pipeline: `actions/checkout@v4`

## 2. Build

**Tujuan:** Mengompilasi dan menyiapkan aplikasi agar dapat dijalankan.

- Alat: Node.js 20, npm
- Perintah:
  - `actions/setup-node@v4` (dengan cache npm)
  - `npm ci` (instal dependensi sesuai `package-lock.json`)
  - `npm run lint` (ESLint)
  - `npm run build`

## 3. Test

**Tujuan:** Memastikan fitur berjalan benar dan tidak ada regresi.

- Alat: Jest (unit test), Supertest (pengujian API)
- Perintah:
  - `npm test` (unit test)
  - `npm run test:integration` (uji API dengan database PostgreSQL via service container)
  - `npm run test:coverage` (batas minimal cakupan 80%)
- Contoh yang diuji: perhitungan total keranjang, validasi stok produk, endpoint `/api/produk` dan `/api/pesanan`

## 4. Security Scanning

**Tujuan:** Menemukan kerentanan sebelum aplikasi dirilis.

- Alat dan perintah:
  - `npm audit --audit-level=high` (kerentanan dependensi)
  - CodeQL (`github/codeql-action`) untuk analisis kode statis (SAST)
  - Gitleaks (`gitleaks/gitleaks-action`) untuk mendeteksi secret yang tidak sengaja ter-commit
  - Trivy (`aquasecurity/trivy-action`) untuk memindai image Docker
- Kebijakan: pipeline gagal bila ditemukan kerentanan tingkat **High** atau **Critical**

## 5. Artifact Generation

**Tujuan:** Menghasilkan paket aplikasi siap deploy yang dapat dilacak versinya.

- Alat: Docker, GitHub Container Registry (GHCR)
- Perintah:
  - `docker build -t ghcr.io/<username>/tokoku:${{ github.sha }} .`
  - `docker tag ghcr.io/<username>/tokoku:${{ github.sha }} ghcr.io/<username>/tokoku:latest`
  - `docker push ghcr.io/<username>/tokoku:${{ github.sha }}`
  - `actions/upload-artifact@v4` untuk menyimpan laporan test dan coverage
- Penamaan versi: SHA commit, dan tag semantik (contoh `v1.0.0`) untuk rilis

## 6. Deployment

**Tujuan:** Menjalankan artefak pada lingkungan target.

- **Staging** (otomatis dari `develop`):
  - SSH ke server staging, lalu `docker compose pull && docker compose up -d`
- **Production** (dari `master`, dengan persetujuan manual melalui GitHub Environments):
  - `docker compose pull && docker compose up -d`
  - Health check: `curl -f https://tokoku.example.com/health`
  - Rollback bila gagal: `docker compose up -d` dengan tag image versi sebelumnya
- Alat: `appleboy/ssh-action`, Docker Compose
- Secrets: `SSH_HOST`, `SSH_USER`, `SSH_KEY`, `DB_PASSWORD` (disimpan di GitHub Secrets)

---

## Ringkasan Alur

```
Source Control -> Build -> Test -> Security Scanning -> Artifact Generation -> Deployment
```

Bila salah satu stage gagal, stage berikutnya tidak dijalankan dan tim mendapat notifikasi.