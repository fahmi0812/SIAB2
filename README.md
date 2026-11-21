# SIAB2 — SchoolHub e-Hadir

SIAB2 (SchoolHub e-Hadir) adalah sistem absensi sekolah berbasis QR. Sistem ini dibuat untuk MAN 1 Rokan Hulu. Kehadiran gerbang dan salat dicatat lewat aplikasi Android resmi (SIAB2 Reader) yang memindai QR siswa atau guru. Absensi kelas diinput manual oleh guru. Siswa tidak pernah mengisi absensi kelasnya sendiri.

## Fitur utama

- Absensi gerbang dan salat dengan pembaca QR Android. Setiap permintaan ditandatangani HMAC, dengan validasi perangkat, nonce, dan kredensial QR.
- Pembaca kartu pintar (smart card) sebagai jalur gerbang alternatif.
- Absensi kelas oleh guru per sesi pelajaran, dengan jadwal dan roster sesi.
- Pembatasan lokasi (geofence) dan kebijakan geolokasi untuk jalur tertentu.
- Cuti dan izin guru, buku piket, dan notifikasi.
- Laporan dan ekspor dokumen (termasuk Excel).
- Jejak audit berantai (audit chain) beserta skrip verifikasinya.
- Pembaruan data real-time lewat SSE dari outbox, plus worker rekonsiliasi.
- Manajemen akun dan role: `ADMIN_TU`, `KEPALA_SEKOLAH`, `GURU_MAPEL`, `GURU_PIKET`, `PEGAWAI`, `SISWA`, `OPERATOR_IT`, `DEVELOPER`.
- Alat bantu operator: generator kartu tanda pengenal dan APK Builder GUI.

## Tech stack

| Bagian | Teknologi |
| --- | --- |
| API | NestJS 11, TypeScript, Prisma 5, PostgreSQL |
| Web | React 18, Vite, React Router, Tailwind CSS |
| Worker | Node.js, BullMQ |
| Antrean dan cache | Redis |
| Android reader | Kotlin, Jetpack Compose (`apps/android-reader`) |
| Reverse proxy | Nginx dan Caddy (`ops/`) |
| Container | Docker Compose (PostgreSQL 16, Redis 7) |
| Pengujian | Jest (API), Vitest (web), Playwright (e2e, a11y, visual) |
| CI | GitHub Actions (`.github/workflows/ci.yml`) |

## Struktur repo

```text
apps/
  api/               NestJS API (modul: attendance-gate, attendance-class, device-reader,
                     qr-credentials, smart-card, reporting, audit, outbox, dan lainnya)
  web/               Frontend React (Vite) beserta tes e2e, a11y, dan visual
  worker/            Worker BullMQ (job terjadwal dan rekonsiliasi)
  android-reader/    Aplikasi Android SIAB2 Reader
packages/shared/     Paket bersama (@schoolhub/shared)
prisma/              Skema, migrasi, dan seed
scripts/             Skrip operasional, smoke test, backup, dan pemeriksaan keamanan
ops/                 Konfigurasi Nginx, Caddy, systemd, dan performa
tools/apk-builder/   APK Builder GUI (Python)
Data Akun/, DataSekolah/
                     Generator tanda pengenal (SPA Vite) untuk kartu akun dan kartu siswa
docs/                ADR, runbook, SOP, dan panduan teknis
docker-compose.*.yml Definisi stack production
```

## Prasyarat

- Node.js 20 (CI memakai 20.20.2) dan npm.
- PostgreSQL 16 dan Redis 7. Cara paling mudah memakai Docker Compose.
- Docker dan Docker Compose untuk menjalankan stack production atau menyiapkan layanan lokal.
- Untuk APK: JDK 17 dan Android SDK (lihat `apps/android-reader/README.md`).
- Untuk APK Builder GUI: Python 3 (lihat `tools/apk-builder/requirements.txt`).

## Menjalankan secara lokal

1. Salin contoh konfigurasi, lalu isi nilainya sendiri. Jangan commit file `.env`.

   Contoh yang tersedia di repo:

   - `.env.production.example`
   - `.env.production.supabase.example` (varian Supabase)
   - `.env.production.test.example` (varian untuk pengujian)

   ```bash
   cp .env.production.example .env
   ```

2. Pasang dependensi dan siapkan database.

   ```bash
   npm install
   npm install --prefix apps/api
   npm install --prefix apps/web
   npm install --prefix apps/worker
   npm run prisma:generate
   npm run prisma:migrate
   npm run prisma:seed
   ```

3. Build API dan web.

   ```bash
   npm run build:all
   ```

4. Jalankan seluruh stack dengan Docker Compose.

   ```bash
   docker compose -f docker-compose.production.yml --env-file .env up -d --build
   ```

5. Cek kesehatan layanan.

   ```bash
   curl -i http://localhost/health/live
   curl -i http://localhost/health/ready
   ```

Untuk pengembangan per aplikasi, `apps/api` menyediakan `npm run start:dev` dan `apps/web` menyediakan `npm run dev` (jalankan dengan `--prefix apps/api` atau `--prefix apps/web` dari root).

## Testing

Perintah dari root repo:

```bash
npm run lint:all             # lint API dan web
npm run typecheck            # typecheck API dan web
npm run test:unit            # unit test API, web, dan worker
npm run test:coverage        # cakupan tes web
npm run test:e2e             # e2e Playwright (web)
npm run test:a11y            # smoke aksesibilitas
npm run test:visual          # regresi visual
npm run test:integration     # integrasi PostgreSQL
npm run test:concurrency     # konkurensi PostgreSQL
npm run test:backend-contract
npm run test:https-smoke
npm run knip                 # deteksi kode tak terpakai
```

Pemeriksaan keamanan:

```bash
npm run security:audit       # npm audit
npm run security:secrets     # pemindaian secret
npm run security:container-scan
npm run security:sbom
npm run security:license
npm run security:risk-acceptance
```

Konvensi tes visual ada di [docs/testing/visual-regression.md](docs/testing/visual-regression.md). Pipeline CI didefinisikan di `.github/workflows/ci.yml`.

## Deployment

Target deployment adalah VPS dengan Docker Compose di belakang reverse proxy. Skrip pendukung ada di `scripts/` (misalnya `deploy_production.sh`, `deployment_status.sh`, `backup_production.sh`).

- [Runbook deployment VPS production](docs/deployment/vps-production-runbook.md)
- [Checklist pasca-deploy](docs/deployment/post-deploy-checklist.md)
- [Runbook produksi](docs/production-runbook.md)
- [Smoke test pasca-deploy](docs/post-deploy-smoke.md)
- [HTTPS production entrypoint](docs/HTTPS_PRODUCTION_ENTRYPOINT.md)
- [Cloudflare Named Tunnel](docs/cloudflare-named-tunnel.md)
- [Migrasi Vercel dan Supabase](docs/MIGRATION_VERCEL_SUPABASE.md)
- [Backup, restore, dan observabilitas](docs/OPERATIONS_BACKUP_RESTORE_OBSERVABILITY.md)

## Dokumentasi

Keputusan arsitektur (ADR):

- [Indeks ADR](docs/adr/README.md)
- [Endpoint Android QR reader](docs/ADR_ANDROID_QR_READER_ENDPOINT.md)
- [Model kepercayaan Android](docs/ADR_ANDROID_TRUST_MODEL.md)
- [Sumber kebenaran produk: QR dan kartu](docs/ADR_PRODUCT_SOURCE_OF_TRUTH_QR_AND_CARD_20260613.md)
- [Terminasi TLS](docs/ADR_TLS_TERMINATION.md)
- [Antrean worker](docs/ADR_WORKER_QUEUE.md)
- [Aliran event SSE](docs/ADR_SSE_EVENT_STREAM.md)

Panduan teknis:

- [Android QR reader](docs/ANDROID_QR_READER.md)
- [Model keamanan QR](docs/QR_SECURITY_MODEL.md)
- [Rencana rollout QR](docs/QR_ROLLOUT_PLAN.md)
- [Kebijakan geolokasi](docs/GEOLOCATION_POLICY.md)
- [Trusted proxy dan rate limiting](docs/TRUSTED_PROXY_AND_RATE_LIMITING.md)
- [APK Update Center](docs/APK_UPDATE_CENTER.md)
- [APK Builder GUI](docs/APK_BUILDER_GUI.md)
- [Performa VPS](docs/SIAB2_VPS_PERFORMANCE.md)

Panduan operator (SOP):

- [Build APK untuk operator](docs/CARA_BUILD_APK_UNTUK_OPERATOR.md)
- [Cetak kartu e-Hadir](docs/CETAK_KARTU_EHADIR.md)
- [Operator: akun siswa dan kartu](docs/SOP_OPERATOR_KARTU_SISWA.md)
- [Operator: lembar akun login](docs/SOP_OPERATOR_LEMBAR_AKUN_LOGIN.md)
- [Import data sekolah](docs/ops/SOP_IMPORT_DATA_SEKOLAH_SIAB2.md)
- [Checklist UAT inti](docs/uat-core-checklist.md)

## Keamanan

Jangan commit file `.env`, kunci, atau data akun dan data siswa. `.gitignore` sudah mengecualikan pola tersebut. Repo ini memakai `gitleaks` (`.gitleaks.toml`) dan pre-commit (`.pre-commit-config.yaml`).
