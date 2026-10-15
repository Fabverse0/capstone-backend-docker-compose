# Dockerisasi Backend Capstone Project

Proyek ini menjalankan backend Capstone Project bersama PostgreSQL menggunakan Docker Compose. Frontend tidak disertakan. Dokumentasi ini berisi langkah untuk menyiapkan environment lokal, menjalankan aplikasi, menguji API dan database, serta membuktikan persistensi data setelah container dibuat ulang.

- **Tugas:** Back-End Intermediate, Session 3
- **Image backend:** `capstone-backend:1.0`
**Repository:** [Fabverse0/capstone-backend-docker-compose](https://github.com/Fabverse0/capstone-backend-docker-compose)

## Isi proyek

- Backend Node.js 22, Express, dan TypeScript.
- PostgreSQL 17 sebagai database lokal.
- Drizzle ORM untuk mengelola skema database.
- `Dockerfile` untuk membangun image backend.
- `compose.yaml` untuk menjalankan backend dan database bersama.
- `api.specs.md` berisi dokumentasi endpoint API.

## Arsitektur dan port

| Koneksi | Alamat | Kegunaan |
| --- | --- | --- |
| Laptop ke API | `http://localhost:8080` | Akses API dan dokumentasi dari browser atau Postman. |
| Backend ke database | `db:5432` | Koneksi internal antarlayanan Compose. Gunakan nama service `db`, bukan `localhost`. |
| Laptop ke database | `127.0.0.1:5433` | Koneksi lokal dari pgAdmin atau klien PostgreSQL lain. Port database hanya dipublikasikan ke komputer ini. |

Compose memetakan port API host `8080` ke port aplikasi di container. Dengan `PORT=5000` pada `.env`, pemetaannya menjadi `8080:5000`. Backend menggunakan `db:5432` untuk database di jaringan Compose. Named volume `db-data` menyimpan berkas database di luar siklus hidup container.

## Prasyarat

- Docker Desktop sudah dijalankan dan menampilkan status engine berjalan.
- Docker Compose v2, yang dapat dipanggil dengan perintah `docker compose`.
- Postman untuk uji endpoint.
- pgAdmin bersifat opsional dan hanya diperlukan jika ingin melihat tabel database melalui antarmuka grafis.

Node.js pada laptop tidak diperlukan untuk menjalankan backend melalui Docker.

## Persiapan environment

1. Clone repository, atau buka folder repository yang sudah tersedia:

   ```powershell
   git clone https://github.com/Fabverse0/capstone-backend-docker-compose.git
   Set-Location capstone-backend-docker-compose
   ```

2. Buat file environment lokal dari template:

   ```powershell
   Copy-Item .env.example .env
   ```

3. Edit `.env` lokal. Isi setidaknya variabel berikut dengan nilai untuk komputer sendiri:

   ```dotenv
   PORT=5000
   POSTGRES_DB=capstone_docker
   POSTGRES_USER=capstone_user
   POSTGRES_PASSWORD=ganti_dengan_password_lokal
   ```

4. Isi `JWT_PRIVATE_KEY` dan `JWT_PUBLIC_KEY` dengan pasangan kunci RSA yang sesuai. Kedua variabel diperlukan untuk registrasi dan login. Jika nilainya kosong atau pasangannya tidak cocok, proses autentikasi dapat gagal. Simpan nilainya hanya di `.env` lokal dan jangan menaruhnya di README, repository publik, tangkapan layar, atau video.

   Nilai key dapat ditulis dalam tanda kutip ganda dan format beberapa baris, seperti format PEM. Gunakan pasangan private/public key yang memang berhubungan satu sama lain.

5. Untuk konfigurasi Compose ini, `DATABASE_URL` di dalam container backend dibentuk dari `POSTGRES_USER`, `POSTGRES_PASSWORD`, dan `POSTGRES_DB` dengan host `db:5432`. Compose mengatur nilai tersebut untuk backend. Jangan ganti koneksi database tugas ini dengan URL database Neon atau `localhost`.

6. Variabel `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, dan `CLOUDINARY_API_SECRET` dipakai fitur unggah media. Variabel ini tidak dibutuhkan untuk uji dasar registrasi dan login. Integrasi Google atau email reset kata sandi dapat memerlukan konfigurasi tambahan seperti `GOOGLE_CLIENT_ID` dan `SMTP_*`.

> `.env` berisi kredensial lokal. Jangan commit atau unggah file tersebut. Unggah `.env.example` saja, dan pastikan seluruh nilainya berupa placeholder kosong, bukan kredensial asli.

## Build dan jalankan

Jalankan perintah dari direktori yang berisi `compose.yaml`:

```powershell
docker compose config -q
docker compose up -d --build
docker compose ps
```

- `docker compose config -q` memeriksa sintaks dan hasil interpolasi Compose.
- `docker compose up -d --build` membangun image backend jika diperlukan, lalu membuat dan menjalankan service di latar belakang.
- `docker compose ps` menampilkan status service. Tunggu sampai backend berstatus `Up` dan database berstatus `healthy`.

Lihat log backend jika perlu mencari penyebab error:

```powershell
docker compose logs --tail=100 backend
```

Setelah kedua service berjalan, terapkan skema database:

```powershell
docker compose exec backend npm run migrate
```

Script `migrate` menjalankan `drizzle-kit push` berdasarkan skema Drizzle di source. Jalankan ini setelah database tersedia dan sebelum menguji endpoint yang membaca atau menulis tabel.

Untuk memeriksa API dari Postman, kirim:

```http
GET http://localhost:8080/api/health
```

Respons yang diharapkan:

```json
{
  "status": "OK"
}
```

Endpoint ini mengonfirmasi proses API dapat diakses. Uji registrasi dan login di bagian berikutnya dipakai untuk memeriksa alur API yang menggunakan database.

Dokumentasi API interaktif juga tersedia di `http://localhost:8080/docs`.

## Uji API dan integrasi database dengan Postman

Pilih **Body → raw → JSON** untuk request POST. Postman akan mengirim `Content-Type: application/json`.

### 1. Buat user relawan

```http
POST http://localhost:8080/api/auth/register/volunteer
```

Contoh body. Gunakan email baru jika request ini sudah pernah dikirim:

```json
{
  "name": "Demo Volunteer",
  "email": "demo-volunteer@example.com",
  "password": "DockerDemo123",
  "confirm_password": "DockerDemo123",
  "role": "volunteer"
}
```

Password harus memiliki setidaknya 8 karakter, satu huruf besar, dan satu angka. Respons sukses yang diharapkan berstatus `201 Created` dan berisi data user. Gunakan email dan password yang sama untuk langkah login berikutnya.

### 2. Login dengan user tersebut

```http
POST http://localhost:8080/api/auth/login
```

```json
{
  "email": "demo-volunteer@example.com",
  "password": "DockerDemo123"
}
```

Respons sukses yang diharapkan berstatus `200 OK`. Login membaca akun dari database dan mengembalikan informasi user. Untuk proyek atau tes lokal, jangan gunakan password akun pribadi.

### 3. Periksa record melalui pgAdmin (opsional)

Buat koneksi server PostgreSQL di pgAdmin dengan pengaturan berikut:

- **Host name/address:** `127.0.0.1`
- **Port:** `5433`
- **Maintenance database:** nilai `POSTGRES_DB` dari `.env`
- **Username:** nilai `POSTGRES_USER` dari `.env`
- **Password:** nilai `POSTGRES_PASSWORD` dari `.env`

Pilih database tersebut, buka Query Tool, lalu jalankan query dengan email yang digunakan pada request registrasi:

```sql
SELECT id, name, email
FROM public.users
WHERE email = 'demo-volunteer@example.com';
```

Satu baris hasil menunjukkan user tersimpan di database. Query ini hanya menampilkan kolom yang diperlukan untuk verifikasi, bukan `password_hash`.

## Uji persistensi named volume

Uji ini membuktikan data tetap tersimpan saat container dihapus dan dibuat kembali.

1. Pastikan request registrasi berhasil. Catat email unik yang dipakai, lalu pastikan record tersebut muncul di pgAdmin dengan query di atas.
2. Hentikan dan hapus container tanpa menghapus volume:

   ```powershell
   docker compose down
   ```

   Perintah ini menghapus container dan network proyek. Named volume `db-data` tetap ada.

3. Buat dan jalankan container lagi tanpa build ulang:

   ```powershell
   docker compose up -d
   docker compose ps
   ```

   Tunggu sampai database kembali berstatus `healthy` dan backend berstatus `Up`.

4. Jika pgAdmin kehilangan koneksi ketika container dihentikan, sambungkan kembali ke server yang sama. Jalankan query `SELECT` dengan email yang sama. Record yang sama seharusnya tetap ditemukan.
5. Kirim lagi request login di Postman menggunakan email dan password tersebut. Respons sukses menunjukkan backend yang berjalan kembali masih dapat membaca akun dari database yang dipulihkan melalui named volume.

**Penting:** jangan menambahkan opsi `--volumes` atau `-v` pada `docker compose down` ketika sedang menguji persistensi. Opsi tersebut menghapus volume database dan datanya.

## Perintah sehari-hari

```powershell
# Lihat status service
docker compose ps

# Lihat log backend
docker compose logs --tail=100 backend

# Jalankan ulang service tanpa build image
docker compose up -d

# Hentikan container, tetapi pertahankan database di named volume
docker compose down
```

Jika source atau `Dockerfile` berubah, bangun image lagi dengan `docker compose up -d --build`. Untuk pemakaian biasa ketika tidak ada perubahan image, `docker compose up -d` cukup.

## Struktur berkas Docker

- `Dockerfile`: menggunakan image dasar Node 22, memasang dependency dari lockfile dengan `npm ci`, membangun TypeScript, lalu menjalankan `npm start`.
- `.dockerignore`: mengecualikan `node_modules`, `.env`, `.git`, dan `dist` dari build context.
- `compose.yaml`: menjalankan service `backend` dan `db`, mengatur environment serta port, menunggu healthcheck database, dan memasang named volume `db-data`.
- `.env.example`: template variabel environment tanpa kredensial asli.

## Sumber dan penggunaan bantuan AI

- Source/starter backend: [Elangprnd/Capstone-Project](https://github.com/Elangprnd/Capstone-Project). Repository ini memisahkan backend dari monorepo sumber; frontend tidak disertakan.
- Dockerfile dan konfigurasi Compose disusun serta dijalankan oleh mahasiswa secara manual. ChatGPT/Codex digunakan sebagai bantuan belajar konsep Docker, diskusi pemecahan masalah, dan penyusunan README ini. README ini tidak memuat kredensial. File `.env` lokal tidak boleh dipublikasikan.

## Berkas pengumpulan tugas

Siapkan satu PDF yang berisi nama, NIM, judul proyek, link repository publik, dan link video YouTube publik atau unlisted. Gunakan pola nama file yang diminta:

```text
Tugas1_BackendIntermediate_Nama_NIM.pdf
```

Pastikan link repository dan video dapat dibuka sebelum mengumpulkan. Video demo maksimal 10 menit sesuai instruksi tugas.
