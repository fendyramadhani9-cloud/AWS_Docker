Tutorial Docker AWS — 2 EC2 + PHP + MariaDB
🏗️ Arsitektur
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

Aplikasi yang dipakai:

https://github.com/paknux/app-reservasi.git
1. Buat 2 EC2

Keduanya:

OS  : Ubuntu Server 24.04 LTS

Buat:

EC2-WEB
EC2-DB

Pastikan keduanya berada dalam VPC yang sama supaya bisa komunikasi lewat private IP.

Contoh:

EC2-WEB private IP = 172.31.x.x
EC2-DB  private IP = 172.31.3.163

IP 172.31.3.163 adalah IP DB yang kita pakai saat latihan tadi. Saat ujian, gunakan private IP EC2-DB yang diberikan AWS.

2. Security Group
SG EC2-WEB

Inbound:

SSH   TCP 22   → IP lo
HTTP  TCP 80   → 0.0.0.0/0

Kalau soal meminta port 8080:

TCP 8080 → 0.0.0.0/0
SG EC2-DB

Inbound:

SSH   TCP 22   → IP lo
MySQL TCP 3306 → Security Group EC2-WEB

Jangan:

3306 → 0.0.0.0/0

Karena database tidak boleh dibuka ke internet.

3. EC2-DB — Install Docker

SSH ke EC2-DB.

sudo apt update
sudo apt install -y docker.io

Aktifkan Docker:

sudo systemctl enable --now docker

Cek:

sudo docker --version
4. Download MariaDB
sudo docker pull mariadb:jammy

Cek:

sudo docker images

Harus ada:

mariadb   jammy
5. Buat Container Database

Karena database berada di EC2 berbeda dengan web, kita menggunakan host networking agar MariaDB dapat diakses melalui private IP EC2-DB.

sudo docker run -d \
  --name database-db \
  --network host \
  -e MARIADB_ROOT_PASSWORD=rootpass \
  mariadb:jammy

Cek:

sudo docker ps

Harus ada:

database-db
mariadb:jammy
6. Masuk ke MariaDB
sudo docker exec -it database-db bash

Masuk MariaDB:

mariadb -u root -prootpass

Cek:

SHOW DATABASES;

Keluar:

exit

Kemudian:

exit
7. Download Aplikasi di EC2-DB

Install Git:

sudo apt update
sudo apt install -y git

Clone:

git clone https://github.com/paknux/app-reservasi.git

Masuk:

cd app-reservasi

Cek:

ls -lah

Harus ada:

ajax/
bookings.php
config/
includes/
index.php
reservasi_ruangan.sql
rooms.php
8. Import Database

File database:

reservasi_ruangan.sql

Import langsung ke container:

sudo docker exec -i database-db mariadb -u root -prootpass < reservasi_ruangan.sql

Cek:

sudo docker exec -it database-db \
mariadb -u root -prootpass \
-e "SHOW DATABASES;"

Database aplikasi harus ada:

db_reservasi_ruangan

Jadi DB_NAME yang benar:

db_reservasi_ruangan
9. EC2-WEB — Install Docker & Git

Sekarang pindah ke EC2-WEB.

sudo apt update
sudo apt install -y docker.io git

Aktifkan Docker:

sudo systemctl enable --now docker

Cek:

sudo docker --version
10. Download Image Ubuntu
sudo docker pull ubuntu:24.04

Cek:

sudo docker images

Harus ada:

ubuntu   24.04
11. Buat Directory Bind Mount
sudo mkdir -p /var/mywww

Directory ini nanti menjadi tempat source code aplikasi di host EC2-WEB.

12. Clone Aplikasi
cd /var/mywww

Clone:

sudo git clone https://github.com/paknux/app-reservasi.git .

Perhatikan titik . di belakang.

Cek:

ls -lah

Harus ada:

index.php
config/
includes/
ajax/
...
13. Konfigurasi Database

File konfigurasi aplikasi:

/var/mywww/config/database.php

Edit:

sudo nano /var/mywww/config/database.php

Isi bagian database menjadi:

$DB_HOST = '172.31.3.163';
$DB_NAME = 'db_reservasi_ruangan';
$DB_USER = 'root';
$DB_PASS = 'rootpass';

Catatan:

DB_HOST
↓
PRIVATE IP EC2-DB

Bukan:

localhost

karena MariaDB berada di EC2 lain.

Dan:

DB_NAME = db_reservasi_ruangan
DB_USER = root
DB_PASS = rootpass

Simpan:

CTRL + O
ENTER
CTRL + X
14. Tes Koneksi WEB → DB

Masih di EC2-WEB.

Install netcat:

sudo apt install -y netcat-openbsd

Tes:

nc -zv 172.31.3.163 3306

Kalau berhasil:

Connection to 172.31.3.163 3306 port [tcp/mysql] succeeded!

Artinya:

EC2-WEB
   │
   │ TCP 3306
   ▼
EC2-DB

sudah bisa terhubung.

15. Buat Container Web

Ini bagian penting.

Jangan cuma:

docker run -d ubuntu:24.04

karena container Ubuntu kosong bisa langsung berhenti.

Gunakan:

sudo docker run -d \
  --name web-app \
  -p 80:80 \
  -v /var/mywww:/var/www/html \
  ubuntu:24.04 \
  tail -f /dev/null

Penjelasan:

--name web-app

nama container.

-p 80:80

Port:

EC2 host : Container
80       : 80
-v /var/mywww:/var/www/html

Bind mount:

HOST
/var/mywww
     │
     ▼
CONTAINER
/var/www/html
16. Cek Container
sudo docker ps

Harus ada:

web-app
ubuntu:24.04
0.0.0.0:80->80/tcp
17. Masuk Container
sudo docker exec -it web-app bash

Sekarang prompt berubah kira-kira:

root@xxxx:/#
18. Install Apache + PHP

Di dalam container:

apt update

Install:

apt install -y apache2 php libapache2-mod-php php-mysql

Yang kita butuhkan:

Apache
PHP
PHP MySQL/PDO

Cek PHP:

php -v

Cek Apache:

apache2 -v
19. Cek Source Code

Karena /var/mywww di-bind mount ke /var/www/html, source code otomatis terlihat di container.

ls -lah /var/www/html

Harus ada:

index.php
bookings.php
rooms.php
config/
includes/
ajax/
20. Jalankan Apache
service apache2 start

Cek:

service apache2 status

Harus:

active (running)
21. Tes Website dari Container
curl http://localhost

Kalau HTML aplikasi keluar berarti Apache berhasil melayani aplikasi.

22. Tes Database dari PHP

Ini salah satu tes paling penting.

Masih di container:

php -r '$pdo = new PDO("mysql:host=172.31.3.163;dbname=db_reservasi_ruangan", "root", "rootpass"); echo "DB CONNECTED\n";'

Kalau keluar:

DB CONNECTED

berarti:

PHP
 │
 │ PDO
 ▼
MariaDB
 │
 ▼
EC2-DB

berhasil.

23. Tes dari Browser

Keluar container:

exit

Cari public IP EC2-WEB:

EC2-WEB Public IPv4

Buka:

http://PUBLIC-IP-EC2-WEB

Kalau semuanya benar, aplikasi reservasi muncul.

24. Troubleshooting Dasar
Container mati
sudo docker ps -a

Lihat log:

sudo docker logs web-app
Apache tidak jalan

Masuk:

sudo docker exec -it web-app bash

Cek:

service apache2 status

Start:

service apache2 start
Website tidak bisa dibuka

Cek:

sudo docker ps

Pastikan:

0.0.0.0:80->80/tcp

Kemudian cek Security Group EC2-WEB:

TCP 80 → 0.0.0.0/0
Database error

Dari EC2-WEB:

nc -zv 172.31.3.163 3306

Kalau gagal, cek:

EC2-DB hidup
container MariaDB hidup
private IP benar
SG DB mengizinkan TCP 3306 dari SG WEB
password benar
25. Perintah Docker yang Wajib Hafal
Lihat image
docker images
Download image
docker pull ubuntu:24.04
Buat container
docker run ...
Lihat container aktif
docker ps
Lihat semua container
docker ps -a
Masuk container
docker exec -it web-app bash
Lihat log
docker logs web-app
Inspect
docker inspect web-app
Stop
docker stop web-app
Start
docker start web-app
Restart
docker restart web-app
Hapus container
docker rm -f web-app
26. docker commit

Kalau konfigurasi di dalam container sudah selesai dan guru meminta membuat image baru:

sudo docker commit web-app web-app:final

Cek:

sudo docker images

Harus muncul:

web-app    final
⚠️ Penting

Bind mount:

-v /var/mywww:/var/www/html

tidak ikut masuk ke image hasil docker commit.

Jadi:

Docker image
    │
    ├── Apache
    ├── PHP
    └── konfigurasi di container

sedangkan source aplikasi tetap berada di:

/var/mywww

di host.

Ini penting banget kalau ditanya saat ujian.