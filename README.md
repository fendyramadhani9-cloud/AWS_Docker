# Tutorial Docker AWS — 2 EC2 + PHP + MariaDB

## 🏗️ Arsitektur

```text
                    INTERNET
                        │
                        ▼
              ┌──────────────────┐
              │    EC2-WEB/APP   │
              │ Ubuntu 24.04 LTS │
              │                  │
              │ Docker           │
              │ ubuntu:24.04     │
              │ ┌──────────────┐ │
              │ │   web-app    │ │
              │ │ Apache + PHP │ │
              │ └──────┬───────┘ │
              │        │         │
              │ /var/mywww       │
              └────────┼─────────┘
                       │
             Private IP :3306
                       │
                       ▼
              ┌──────────────────┐
              │     EC2-DB       │
              │ Ubuntu 24.04 LTS │
              │                  │
              │ Docker           │
              │ mariadb:jammy    │
              │ ┌──────────────┐ │
              │ │ database-db  │ │
              │ │   MariaDB    │ │
              │ └──────────────┘ │
              └──────────────────┘
```

**Aplikasi yang dipakai:**  
https://github.com/paknux/app-reservasi.git

---

## 1. Buat 2 EC2

**Keduanya:**
* **OS:** Ubuntu Server 24.04 LTS

**Buat:**
* `EC2-WEB`
* `EC2-DB`

Pastikan keduanya berada dalam VPC yang sama supaya bisa komunikasi lewat private IP.

**Contoh:**
* `EC2-WEB` private IP = `172.31.x.x`
* `EC2-DB` private IP = `172.31.3.163`

> **Catatan:** IP `172.31.3.163` adalah IP DB yang kita pakai saat latihan tadi. Saat ujian, gunakan private IP EC2-DB yang diberikan AWS.

---

## 2. Security Group

### SG EC2-WEB

**Inbound:**
* `SSH` TCP 22 → IP lo
* `HTTP` TCP 80 → `0.0.0.0/0`

Kalau soal meminta port 8080:
* `TCP 8080` → `0.0.0.0/0`

---

### SG EC2-DB

**Inbound:**
* `SSH` TCP 22 → IP lo
* `MySQL` TCP 3306 → Security Group EC2-WEB

Jangan:
```text
3306 → 0.0.0.0/0
```
Karena database tidak boleh dibuka ke internet.

---

## 3. EC2-DB — Install Docker

SSH ke EC2-DB.

```bash
sudo apt update
sudo apt install -y docker.io
```

Aktifkan Docker:
```bash
sudo systemctl enable --now docker
```

Cek:
```bash
sudo docker --version
```

---

## 4. Download MariaDB

```bash
sudo docker pull mariadb:jammy
```

Cek:
```bash
sudo docker images
```

Harus ada:
```text
mariadb   jammy
```

---

## 5. Buat Container Database

Karena database berada di EC2 berbeda dengan web, kita menggunakan host networking agar MariaDB dapat diakses melalui private IP EC2-DB.

```bash
sudo docker run -d \
  --name database-db \
  --network host \
  -e MARIADB_ROOT_PASSWORD=rootpass \
  mariadb:jammy
```

Cek:
```bash
sudo docker ps
```

Harus ada:
```text
database-db
mariadb:jammy
```

---

## 6. Masuk ke MariaDB

```bash
sudo docker exec -it database-db bash
```

Masuk MariaDB:
```bash
mariadb -u root -prootpass
```

Cek:
```sql
SHOW DATABASES;
```

Keluar:
```bash
exit
```

Kemudian:
```bash
exit
```

---

## 7. Download Aplikasi di EC2-DB

Install Git:
```bash
sudo apt update
sudo apt install -y git
```

Clone:
```bash
git clone https://github.com/paknux/app-reservasi.git
```

Masuk:
```bash
cd app-reservasi
```

Cek:
```bash
ls -lah
```

Harus ada:
```text
ajax/
bookings.php
config/
includes/
index.php
reservasi_ruangan.sql
rooms.php
```

---

## 8. Import Database

File database:
```text
reservasi_ruangan.sql
```

Import langsung ke container:
```bash
sudo docker exec -i database-db mariadb -u root -prootpass < reservasi_ruangan.sql
```

Cek:
```bash
sudo docker exec -it database-db \
mariadb -u root -prootpass \
-e "SHOW DATABASES;"
```

Database aplikasi harus ada:
```text
db_reservasi_ruangan
```

Jadi DB_NAME yang benar:
```text
db_reservasi_ruangan
```

---

## 9. EC2-WEB — Install Docker & Git

Sekarang pindah ke EC2-WEB.

```bash
sudo apt update
sudo apt install -y docker.io git
```

Aktifkan Docker:
```bash
sudo systemctl enable --now docker
```

Cek:
```bash
sudo docker --version
```

---

## 10. Download Image Ubuntu

```bash
sudo docker pull ubuntu:24.04
```

Cek:
```bash
sudo docker images
```

Harus ada:
```text
ubuntu   24.04
```

---

## 11. Buat Directory Bind Mount

```bash
sudo mkdir -p /var/mywww
```

Directory ini nanti menjadi tempat source code aplikasi di host EC2-WEB.

---

## 12. Clone Aplikasi

```bash
cd /var/mywww
```

Clone:
```bash
sudo git clone https://github.com/paknux/app-reservasi.git .
```

*(Perhatikan titik `.` di belakang)*

Cek:
```bash
ls -lah
```

Harus ada:
```text
index.php
config/
includes/
ajax/
...
```

---

## 13. Konfigurasi Database

File konfigurasi aplikasi:
```text
/var/mywww/config/database.php
```

Edit:
```bash
sudo nano /var/mywww/config/database.php
```

Isi bagian database menjadi:
```php
$DB_HOST = '172.31.3.163';
$DB_NAME = 'db_reservasi_ruangan';
$DB_USER = 'root';
$DB_PASS = 'rootpass';
```

Catatan:
```text
DB_HOST
  ↓
PRIVATE IP EC2-DB
```

Bukan:
```text
localhost
```
karena MariaDB berada di EC2 lain.

Dan:
* `DB_NAME = db_reservasi_ruangan`
* `DB_USER = root`
* `DB_PASS = rootpass`

Simpan:
* `CTRL + O`
* `ENTER`
* `CTRL + X`

---

## 14. Tes Koneksi WEB → DB

Masih di EC2-WEB.

Install netcat:
```bash
sudo apt install -y netcat-openbsd
```

Tes:
```bash
nc -zv 172.31.3.163 3306
```

Kalau berhasil:
```text
Connection to 172.31.3.163 3306 port [tcp/mysql] succeeded!
```

Artinya:
```text
EC2-WEB
   │
   │ TCP 3306
   ▼
EC2-DB
```
sudah bisa terhubung.

---

## 15. Buat Container Web

Ini bagian penting.

Jangan cuma:
```bash
docker run -d ubuntu:24.04
```
karena container Ubuntu kosong bisa langsung berhenti.

Gunakan:
```bash
sudo docker run -d \
  --name web-app \
  -p 80:80 \
  -v /var/mywww:/var/www/html \
  ubuntu:24.04 \
  tail -f /dev/null
```

**Penjelasan:**
* `--name web-app`: nama container.
* `-p 80:80`: Port (EC2 host : Container = 80 : 80).
* `-v /var/mywww:/var/www/html`: Bind mount:
  ```text
  HOST
  /var/mywww
       │
       ▼
  CONTAINER
  /var/www/html
  ```

---

## 16. Cek Container

```bash
sudo docker ps
```

Harus ada:
```text
web-app
ubuntu:24.04
0.0.0.0:80->80/tcp
```

---

## 17. Masuk Container

```bash
sudo docker exec -it web-app bash
```

Sekarang prompt berubah kira-kira:
```text
root@xxxx:/#
```

---

## 18. Install Apache + PHP

Di dalam container:
```bash
apt update
```

Install:
```bash
apt install -y apache2 php libapache2-mod-php php-mysql
```

Yang kita butuhkan:
* Apache
* PHP
* PHP MySQL/PDO

Cek PHP:
```bash
php -v
```

Cek Apache:
```bash
apache2 -v
```

---

## 19. Cek Source Code

Karena `/var/mywww` di-bind mount ke `/var/www/html`, source code otomatis terlihat di container.

```bash
ls -lah /var/www/html
```

Harus ada:
```text
index.php
bookings.php
rooms.php
config/
includes/
ajax/
```

---

## 20. Jalankan Apache

```bash
service apache2 start
```

Cek:
```bash
service apache2 status
```

Harus:
```text
active (running)
```

---

## 21. Tes Website dari Container

```bash
curl http://localhost
```

Kalau HTML aplikasi keluar berarti Apache berhasil melayani aplikasi.

---

## 22. Tes Database dari PHP

Ini salah satu tes paling penting.

Masih di container:
```bash
php -r '$pdo = new PDO("mysql:host=172.31.3.163;dbname=db_reservasi_ruangan", "root", "rootpass"); echo "DB CONNECTED\n";'
```

Kalau keluar:
```text
DB CONNECTED
```

berarti:
```text
PHP
 │
 │ PDO
 ▼
MariaDB
 │
 ▼
EC2-DB
```
berhasil.

---

## 23. Tes dari Browser

Keluar container:
```bash
exit
```

Cari public IP EC2-WEB:
```text
EC2-WEB Public IPv4
```

Buka:
```text
http://PUBLIC-IP-EC2-WEB
```

Kalau semuanya benar, aplikasi reservasi muncul.

---

## 24. Troubleshooting Dasar

### Container mati
```bash
sudo docker ps -a
```
Lihat log:
```bash
sudo docker logs web-app
```

### Apache tidak jalan
Masuk:
```bash
sudo docker exec -it web-app bash
```
Cek:
```bash
service apache2 status
```
Start:
```bash
service apache2 start
```

### Website tidak bisa dibuka
Cek:
```bash
sudo docker ps
```
Pastikan:
```text
0.0.0.0:80->80/tcp
```
Kemudian cek Security Group EC2-WEB:
```text
TCP 80 → 0.0.0.0/0
```

### Database error
Dari EC2-WEB:
```bash
nc -zv 172.31.3.163 3306
```
Kalau gagal, cek:
* EC2-DB hidup
* container MariaDB hidup
* private IP benar
* SG DB mengizinkan TCP 3306 dari SG WEB
* password benar

---

## 25. Perintah Docker yang Wajib Hafal

| Perintah | Fungsi |
| :--- | :--- |
| `docker images` | Lihat image |
| `docker pull ubuntu:24.04` | Download image |
| `docker run ...` | Buat container |
| `docker ps` | Lihat container aktif |
| `docker ps -a` | Lihat semua container |
| `docker exec -it web-app bash` | Masuk container |
| `docker logs web-app` | Lihat log |
| `docker inspect web-app` | Inspect |
| `docker stop web-app` | Stop |
| `docker start web-app` | Start |
| `docker restart web-app` | Restart |
| `docker rm -f web-app` | Hapus container |

---

## 26. docker commit

Kalau konfigurasi di dalam container sudah selesai dan guru meminta membuat image baru:

```bash
sudo docker commit web-app web-app:final
```

Cek:
```bash
sudo docker images
```

Harus muncul:
```text
web-app    final
```

---

## ⚠️ Penting

Bind mount:
```text
-v /var/mywww:/var/www/html
```
tidak ikut masuk ke image hasil `docker commit`.

Jadi:
```text
Docker image
    │
    ├── Apache
    ├── PHP
    └── konfigurasi di container
```

sedangkan source aplikasi tetap berada di:
```text
/var/mywww
```
di host.

Ini penting banget kalau ditanya saat ujian.