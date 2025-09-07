# Langkah Instalasi DOMjudge dengan Docker

## 0. Catatan
1. Domjudge tidak bisa diinstall di coolify karena pada coolify menggunakna LXC, harus dedicated VM sendiri (KVM)
2. Create VM
3. Domjudge berhasil diinstall di ceits pada 07-09-2025 dengan os Ubuntu 24.04 server. Pada OS ubuntu ini menggunakan cgroup v2, jadi perlu diaktifkan versi 1
```bash
#Edit File Konfigurasi GRUB:
#Buka file /etc/default/grub dengan editor nano sebagai root.
sudo nano /etc/default/grub

#Ubah Baris GRUB_CMDLINE_LINUX_DEFAULT:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash cgroup_enable=memory swapaccount=1 systemd.unified_cgroup_hierarchy=0"

#update dan reboot
sudo update-grub
sudo reboot

#seelah reboot, cek
cat /proc/cmdline

#Perintah diatas akan menunjukkan parameter kernel yang sebenarnya sedang digunakan saat boot. Saya sangat yakin Anda tidak akan melihat systemd.unified_cgroup_hierarchy=0 di dalam outputnya.
```

## 1. Install Docker & Docker Compose

Pastikan sistem sudah up to date, lalu install Docker dan plugin Docker Compose:

```bash
sudo apt update

# Install Docker
curl -fsSL https://get.docker.com | bash

# Add user to docker group (optional)
sudo usermod -aG docker $USER

# Verify Docker
docker --version

# Install docker compose plugin
sudo apt install docker-compose-plugin -y

# Verify
docker compose version

```

## 2. Siapkan direktori proyek DOMjudge

```bash
mkdir ~/domjudge
cd ~/domjudge
```

## 3. Buat file docker-compose.yml

Buat file `docker-compose.yml` dan isi dengan konfigurasi berikut:

```yaml
version: '3.9'

networks:
  domjudge:
    name: domjudge

services:
  mariadb:
    container_name: mariadb
    image: mariadb:latest
    networks:
      domjudge:
        aliases:
          - ceits
    ports:
      - "13306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=rootpw
      - MYSQL_USER=domjudge
      - MYSQL_PASSWORD=djpw
      - MYSQL_DATABASE=domjudge

  domserver:
    container_name: domserver
    image: domjudge/domserver:latest
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    networks:
      - domjudge
    ports:
      - "12345:80"
    depends_on:
      - mariadb
    environment:
      - CONTAINER_TIMEZONE=Asia/Jakarta
      - MYSQL_HOST=mariadb
      - MYSQL_ROOT_PASSWORD=rootpw
      - MYSQL_USER=domjudge
      - MYSQL_PASSWORD=djpw
      - MYSQL_DATABASE=domjudge

  judgehost:
    container_name: judgehost
    image: domjudge/judgehost:latest
    privileged: true
    hostname: judgedaemon
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    networks:
      - domjudge
    depends_on:
      - domserver
    environment:
      - CONTAINER_TIMEZONE=Asia/Jakarta
      - DAEMON_ID=0
      - JUDGEDAEMON_PASSWORD=xx
```

## 4. Jalankan database & domserver

```bash
sudo docker compose up -d mariadb domserver
```

## 5. Ambil password untuk Judgehost

Jalankan perintah berikut untuk mendapatkan password daemon:
```bash
sudo docker exec -it domserver cat /opt/domjudge/domserver/etc/restapi.secret
```
Salin password pada baris `judgehost`, lalu edit file `docker-compose.yml` dan ganti:
```yaml
- JUDGEDAEMON_PASSWORD=xxx
```
dengan password yang didapatkan.

## 6. Jalankan semua layanan

```bash
#baru compose judgehostnya
sudo docker compose up -d judgehost
```

## 7. Akses DOMjudge

Buka browser dan akses:
```
http://localhost:12345
```

Untuk mendapatkan password admin:
```bash
sudo docker exec -it domserver cat /opt/domjudge/domserver/etc/initial_admin_password.secret
```

Login dengan:
- Username: `admin`
- Password: (hasil command di atas)

## Tips Manajemen Docker

- Cek container berjalan:
  ```bash
  sudo docker ps
  ```
- Hentikan semua layanan:
  ```bash
  sudo docker compose down
  ```
- Nyalakan kembali:
  ```bash
  sudo docker compose up -d
  ```
- Log tracing, untuk melihat jika ada error
  ```bash
  docker compose logs --tail=30 judgehost
  ```
