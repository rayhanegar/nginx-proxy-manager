# Konfigurasi Nginx Proxy Manager

Repositori ini berisi konfigurasi Docker Compose untuk Nginx Proxy Manager, sebuah aplikasi berbasis web untuk manajemen reverse proxy nginx.

## ⚠️ Pemberitahuan Keamanan

Repositori ini telah dikonfigurasi untuk berbagi publik. Seluruh data sensitif telah dihapus, meliputi:

- Berkas environment (`.env`)
- Data basis data (`mysql/`)
- Data aplikasi (`data/`)
- Sertifikat dan kunci SSL
- Berkas log
- Berkas cadangan

## Arsitektur Sistem

### Gambaran Umum Arsitektur

Sistem ini dibangun menggunakan arsitektur kontainer Docker dengan komponen-komponen yang terisolasi namun saling berkomunikasi melalui jaringan virtual yang terdefinisi. Arsitektur ini mengimplementasikan prinsip separation of concerns dengan memisahkan layanan reverse proxy dari layanan basis data.

### Topologi Jaringan

Sistem menggunakan dua jaringan Docker yang terisolasi:

#### 1. Jaringan `proxy-network` (Eksternal)
- **Tipe Driver:** Bridge
- **Subnet:** 172.20.0.0/16
- **Gateway:** 172.20.0.1
- **Fungsi:** Jaringan bersama untuk komunikasi antar layanan yang memerlukan akses reverse proxy
- **Label:** 
  - `com.devsecops.description`: "Shared network for reverse proxy"
  - `com.devsecops.managed-by`: "nginx-proxy-manager"

#### 2. Jaringan `npm-internal` (Internal)
- **Tipe Driver:** Bridge
- **Fungsi:** Jaringan privat untuk komunikasi internal antara Nginx Proxy Manager dan basis data
- **Keamanan:** Terisolasi dari jaringan eksternal untuk meningkatkan keamanan basis data

### Diagram Arsitektur

```
┌─────────────────────────────────────────────────────────────┐
│                        Host System                           │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         proxy-network (172.20.0.0/16)                │   │
│  │                                                       │   │
│  │  ┌─────────────────────────────────────────┐        │   │
│  │  │  nginx-proxy-manager (172.20.0.10)      │        │   │
│  │  │  - Port 80:80   (HTTP)                  │        │   │
│  │  │  - Port 81:81   (Admin UI)              │        │   │
│  │  │  - Port 443:443 (HTTPS)                 │        │   │
│  │  └──────────────┬──────────────────────────┘        │   │
│  │                 │                                     │   │
│  └─────────────────┼─────────────────────────────────────┘   │
│                    │                                         │
│  ┌─────────────────┼─────────────────────────────────────┐   │
│  │                 │    npm-internal                      │   │
│  │                 │                                      │   │
│  │                 ▼                                      │   │
│  │  ┌─────────────────────────────────────────┐         │   │
│  │  │  npm-db (MariaDB)                       │         │   │
│  │  │  - Port 3306 (Internal only)            │         │   │
│  │  └─────────────────────────────────────────┘         │   │
│  │                                                        │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

## Spesifikasi Layanan

### 1. Layanan nginx-proxy-manager

**Image Docker:** `jc21/nginx-proxy-manager:latest`

**Fungsi:** Menyediakan antarmuka web untuk manajemen reverse proxy nginx, termasuk konfigurasi SSL/TLS otomatis menggunakan Let's Encrypt.

**Konfigurasi Jaringan:**
- **IP Address Statis:** 172.20.0.10 pada jaringan `proxy-network`
- **Port Mapping:**
  - `80:80` - HTTP traffic untuk proxy
  - `81:81` - Antarmuka administrasi web
  - `443:443` - HTTPS traffic untuk proxy

**Variabel Lingkungan:**
- `DB_SQLITE_FILE=""` - Menonaktifkan SQLite, memaksa penggunaan MySQL
- `DB_MYSQL_HOST="npm-db"` - Hostname layanan basis data
- `DB_MYSQL_PORT=3306` - Port koneksi MySQL
- `DB_MYSQL_USER="npm"` - Username basis data
- `DB_MYSQL_PASSWORD` - Password basis data (dari variabel environment)
- `DB_MYSQL_NAME="npm"` - Nama basis data
- `DISABLE_IPV6='true'` - Menonaktifkan dukungan IPv6

**Volume Mounts:**
- `./data:/data` - Penyimpanan konfigurasi aplikasi dan log
- `./letsencrypt:/etc/letsencrypt` - Penyimpanan sertifikat SSL

**Health Check:**
- Perintah: `/bin/check-health`
- Interval: 10 detik
- Timeout: 3 detik
- Retries: 3 kali

### 2. Layanan npm-db

**Image Docker:** `jc21/mariadb-aria:latest`

**Fungsi:** Menyediakan basis data MariaDB dengan storage engine Aria untuk menyimpan konfigurasi Nginx Proxy Manager.

**Konfigurasi Jaringan:**
- **Jaringan:** npm-internal (tanpa IP statis)
- **Port:** 3306 (hanya dapat diakses dari jaringan internal)

**Variabel Lingkungan:**
- `MYSQL_ROOT_PASSWORD` - Password root MySQL (dari variabel environment)
- `MYSQL_DATABASE='npm'` - Nama basis data yang akan dibuat
- `MYSQL_USER='npm'` - Username aplikasi
- `MYSQL_PASSWORD` - Password aplikasi (dari variabel environment)
- `MARIADB_AUTO_UPGRADE='1'` - Mengaktifkan upgrade otomatis skema basis data

**Volume Mounts:**
- `./mysql:/var/lib/mysql` - Penyimpanan persisten data basis data

**Health Check:**
- Perintah: `mysqladmin ping`
- Interval: 10 detik
- Timeout: 5 detik
- Retries: 5 kali

## Alokasi Alamat IP pada proxy-network

| Layanan | Alamat IP | Fungsi |
|---------|-----------|--------|
| Gateway | 172.20.0.1 | Gateway jaringan Docker |
| nginx-proxy-manager | 172.20.0.10 | Reverse proxy server (IP statis) |
| Range Tersedia | 172.20.0.2-172.20.255.254 | Tersedia untuk layanan lain yang memerlukan akses proxy |

**Catatan:** Penggunaan alamat IP statis untuk nginx-proxy-manager memastikan konsistensi konfigurasi dan memudahkan referensi dari layanan lain dalam jaringan yang sama.

## Instruksi Instalasi

### 1. Kloning Repositori

```bash
git clone <url-repositori-anda>
cd nginx-proxy-manager
```

### 2. Pembuatan Berkas Environment

Buat berkas `.env` di direktori root dengan kredensial yang aman:

```bash
# .env
NPM_DB_PASSWORD=password-basis-data-yang-aman
NPM_ROOT_PASSWORD=password-root-mysql-yang-aman
```

**Penting:** Gunakan password yang kuat dan unik. Jangan pernah melakukan commit berkas `.env` ke version control.

### 3. Struktur Direktori

Direktori-direktori berikut akan dibuat secara otomatis saat kontainer dijalankan:

- `data/` - Berisi data aplikasi, konfigurasi, dan log Nginx Proxy Manager
- `mysql/` - Berisi data basis data MySQL/MariaDB
- `letsencrypt/` - Berisi sertifikat SSL (jika menggunakan Let's Encrypt)

### 4. Menjalankan Layanan

```bash
docker-compose up -d
```

Perintah ini akan:
1. Membuat jaringan `proxy-network` dan `npm-internal`
2. Menjalankan kontainer `npm-db` terlebih dahulu
3. Menunggu hingga basis data sehat (healthy)
4. Menjalankan kontainer `nginx-proxy-manager`

### 5. Mengakses Antarmuka Web

Buka browser dan navigasi ke:
- **HTTP:** `http://ip-server-anda:81`
- **HTTPS:** `https://ip-server-anda:443` (setelah konfigurasi SSL)

### Kredensial Login Awal

Pada penggunaan pertama, gunakan kredensial default berikut (segera ubah setelah login):
- **Email:** `admin@example.com`
- **Password:** `changeme`

## Konfigurasi Lanjutan

### Modifikasi Docker Compose

Berkas `docker-compose.yml` dapat disesuaikan untuk kebutuhan spesifik dengan mempertimbangkan:
1. Penyesuaian mapping port jika terjadi konflik
2. Penambahan variabel lingkungan sesuai kebutuhan
3. Modifikasi konfigurasi jaringan untuk integrasi dengan layanan lain

### Integrasi dengan Layanan Lain

Untuk mengintegrasikan layanan lain dengan reverse proxy:

```yaml
services:
  layanan-anda:
    image: image-anda:latest
    networks:
      - proxy-network
    # Konfigurasi lainnya...

networks:
  proxy-network:
    external: true
    name: proxy-network
```

## Prosedur Backup dan Restore

### Prosedur Backup

Untuk melakukan backup konfigurasi sistem:

```bash
# Backup data aplikasi
sudo tar -czf npm-data-backup-$(date +%Y%m%d).tar.gz data/

# Backup basis data
docker-compose exec npm-db mysqldump -u npm -p"${NPM_DB_PASSWORD}" npm > npm-database-backup-$(date +%Y%m%d).sql
```

### Prosedur Restore

Untuk melakukan restore dari backup:

```bash
# Restore data aplikasi
sudo tar -xzf npm-data-backup-YYYYMMDD.tar.gz

# Restore basis data
docker-compose exec -T npm-db mysql -u npm -p"${NPM_DB_PASSWORD}" npm < npm-database-backup-YYYYMMDD.sql
```

## Praktik Keamanan Terbaik

1. **Perubahan Kredensial Default:** Segera ubah kredensial admin default pada login pertama
2. **Penggunaan Password Kuat:** Tetapkan password yang kuat dan kompleks dalam berkas `.env`
3. **Pembaruan Berkala:** Lakukan pembaruan image Docker secara berkala untuk mendapatkan patch keamanan terbaru
4. **Konfigurasi Firewall:** Konfigurasikan aturan firewall untuk membatasi akses sesuai kebutuhan
5. **Implementasi SSL/TLS:** Gunakan sertifikat SSL untuk semua proxy host yang dikelola
6. **Backup Berkala:** Lakukan backup konfigurasi dan data secara terjadwal dan teratur
7. **Isolasi Jaringan:** Manfaatkan isolasi jaringan untuk memisahkan layanan internal dari eksternal
8. **Monitoring Log:** Pantau log secara berkala untuk mendeteksi aktivitas mencurigakan

## Pemecahan Masalah

### Masalah Umum

1. **Konflik Port:** Pastikan port 80, 81, dan 443 tidak digunakan oleh layanan lain di host system
2. **Masalah Permission:** Direktori `data/` dan `mysql/` mungkin memerlukan permission yang sesuai
3. **Koneksi Basis Data:** Verifikasi bahwa kontainer MySQL berjalan dan kredensial sudah benar
4. **Masalah Jaringan:** Periksa apakah jaringan Docker sudah terbuat dengan benar

### Pemeriksaan Log

Periksa log kontainer untuk diagnosis masalah:

```bash
# Semua layanan
docker-compose logs

# Layanan spesifik
docker-compose logs nginx-proxy-manager
docker-compose logs npm-db

# Follow log secara real-time
docker-compose logs -f nginx-proxy-manager
```

### Pemeriksaan Status Layanan

```bash
# Status kontainer
docker-compose ps

# Pemeriksaan health check
docker inspect nginx-proxy-manager | grep -A 10 Health

# Pemeriksaan koneksi jaringan
docker network inspect proxy-network
docker network inspect npm-internal
```

## Monitoring dan Maintenance

### Monitoring Performa

Log akses dan error dapat ditemukan di direktori `data/logs/`:
- `fallback_access.log` - Log akses untuk default host
- `fallback_error.log` - Log error untuk default host
- `proxy-host-*_access.log` - Log akses per proxy host
- `proxy-host-*_error.log` - Log error per proxy host

### Maintenance Berkala

Lakukan maintenance berkala meliputi:
1. Pemeriksaan penggunaan disk untuk direktori `data/` dan `mysql/`
2. Rotasi log secara otomatis (sudah dikonfigurasi)
3. Pembaruan image Docker
4. Verifikasi backup

```bash
# Pembaruan image
docker-compose pull
docker-compose up -d

# Pembersihan resource yang tidak digunakan
docker system prune -a
```

## Kontribusi

Kontribusi dalam bentuk issue report atau pull request sangat diterima. Pastikan untuk mengikuti praktik terbaik dalam pengembangan dan dokumentasi yang jelas untuk setiap perubahan yang diusulkan.

## Lisensi

Konfigurasi ini disediakan sebagaimana adanya untuk keperluan edukasi dan praktis. Pastikan kepatuhan terhadap semua lisensi perangkat lunak yang relevan yang digunakan dalam sistem ini.
