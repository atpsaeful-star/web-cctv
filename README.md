# Web-CCTV HG680P v2.5

Web CCTV modern, ringan, responsif mobile – khusus **STB Armbian HG680P (Amlogic S905X)** & PC / VPS.

HLS live player dengan **auto-reconnect**, transcode **H.265 → H.264**, peta OpenStreetMap, rekaman MP4 + scheduler, login 2 role, manajemen user.

Dibuat untuk Armbian Bionic / Bullseye / Bookworm di STB HG680P, B860H, dll.

---

## ✨ Fitur v2.5

- **Login 2 role**: `admin` & `public` (JWT, 7 hari)
  - admin: kelola kamera, user, rekaman, stream
  - public: lihat kamera publik
- **Live HLS** – hls.js 1.5.17
  - **Auto-reconnect**: retry 10x, backoff 3–15 detik, stall watchdog
  - **Fallback HLS CORS proxy** otomatis (`/api/hls-proxy`) – untuk stream eksternal seperti stream.serangkota.go.id
- **Transcode H.265 → H.264**
  - `libx264 ultrafast zerolatency`, 960x540 15fps 800kbps
  - Optimized HG680P: 1 stream ~35-55% CPU, RAM ~120 MB
- **Input kamera**:
  - RTSP IP Cam / NVR / DVR (Hikvision, Dahua, dsb – H.264/H.265 auto)
  - **HLS / HTTP Live .m3u8** – direct play, tanpa transcode
  - **YouTube Live Embed** – ID otomatis extract dari URL penuh
- **Live di Grid** (toggle)
  - Mode snapshot: thumbnail 480x270, refresh 15 detik – hemat CPU
  - Mode Live: HLS / YouTube langsung play di card (max 4 kamera)
- **Snapshot thumbnail** otomatis – `/api/snapshot/:id`
- **Peta OpenStreetMap + Leaflet** – pin lokasi kamera, status online/offline
- **Dashboard**: total kamera, online, offline, streaming, rekaman, user
- **Status Online/Offline** otomatis – ping RTSP / HLS tiap 8 detik, 1 kamera bergantian (ringan STB)
- **Rekaman MP4 + Scheduler**
  - Manual ● Rec di player
  - Otomatis cron mini: `* * * * *`, `*/15 * * * *`, `0 * * * *`
  - File: `records/{camera_id}/YYYY-MM-DDTHH-MM-SS.mp4`, faststart
  - Playback & download di browser
- **Manajemen User** – CRUD admin, tidak bisa hapus admin terakhir
- **Database SQLite (better-sqlite3)** – tanpa MySQL, file `cctv.db`, WAL mode, super ringan 1-2 GB RAM
- **Responsive Mobile** – HP Android / iPhone rapi
  - Topbar grid, nav scroll horizontal
  - Live grid: 1 kolom HP <600px, 2 kolom tablet, 3-4 kolom desktop
  - Modal player fullscreen di HP
- **Auto-start systemd** – mati lampu → nyala otomatis
- **Docker** + **.deb package** – siap deploy

---

## 🖥️ Spesifikasi yang diuji

- STB **HG680P** Armbian 23 / 24 (Amlogic S905X, 2GB RAM, 8GB eMMC)
- Node.js 18 / 20
- FFmpeg 4.1 – 6.x (Armbian repo)
- Browser: Chrome Android, Safari iOS, Firefox, Edge

Performa HG680P:
- Idle: ~45 MB RAM, 1-3% CPU
- 1 Live transcode: 35-55% CPU, total RAM ~130 MB
- 2 Live: 70-90% CPU – masih stabil
- 3 Live: >95% CPU – tidak disarankan
- **Saran**: pakai HLS direct / YouTube untuk kamera publik, RTSP transcode hanya saat dibuka

---

## 📦 1. Install Fresh – Armbian HG680P

Login SSH ke STB:

```bash
# 0. update sistem
sudo apt update && sudo apt upgrade -y

# 1. install dependensi
sudo apt install -y nodejs npm ffmpeg sqlite3 git build-essential python3 curl

# cek versi
node -v      # >= v16, rekom v18/20
ffmpeg -version
# pastikan ada:  --enable-libx264
ffmpeg -encoders | grep 264
# kalau tidak ada aac encoder, audio otomatis dimatikan (tidak masalah)
ffmpeg -encoders | grep aac
```

### 2. Ambil source

**Opsi A – git clone:**
```bash
cd /opt
sudo git clone https://github.com/atpsaeful-star/web-cctv
cd webcctv
# atau kalau file sudah di-upload manual:
cd /root/web-cctv
```

**Opsi B – upload manual (scp):**
dari PC:
```bash
scp -r ./web-cctv root@ip-stb:/root/
```
di STB:
```bash
cd /root/web-cctv
```

### 3. Install Node modules

```bash
cd /root/web-cctv
# atau /opt/webcctv

npm install --omit=dev
```
> Di HG680P proses bisa **3-10 menit**, karena `better-sqlite3` compile native. Tunggu sampai selesai.  
> Jika gagal compile:
> ```bash
> sudo apt install -y build-essential python3
> npm install --build-from-source --omit=dev
> ```

Jika masih error `Cannot find module 'express'` artinya `node_modules` belum terinstall – jalankan `npm install` lagi.

### 4. Init database

```bash
node init-db.js
```

Output:
```
✅ users created: admin/admin123  publik/publik123
✅ 3 sample cameras inserted
DB ready: /root/web-cctv/cctv.db
```

File database: `cctv.db` (SQLite, di root project)

### 5. Jalankan manual (test dulu)

```bash
node server.js
```

Output yang benar:
```
✓ SQLite: /root/web-cctv/cctv.db
FFmpeg AAC encoder: yes   (atau no – audio akan dimatikan)
🚀 Web-CCTV v2.5 http://0.0.0.0:3000
```

Buka browser:
```
http://IP-STB:3000
# contoh: http://192.168.1.50:3000
```

Login:
- **admin / admin123** – full akses
- **publik / publik123** – lihat saja

Jika live belum tampil, lihat:
```bash
cat logs/ff_1.log
# atau via API (login admin dulu):
# curl -H "Authorization: Bearer TOKEN" http://localhost:3000/api/stream/1/log
```

Ctrl+C untuk stop test manual.

---

## 🔌 6. Auto-start – Mati lampu nyala otomatis

Gunakan installer otomatis (rekomendasi):

```bash
cd /root/web-cctv
chmod +x install-autostart.sh
sudo bash install-autostart.sh
```

Script akan:
- copy aplikasi ke `/opt/webcctv`
- `npm install --omit=dev`
- buat data dir `/var/lib/webcctv`
  - DB: `/var/lib/webcctv/cctv.db`
  - Records: `/var/lib/webcctv/records`
- symlink `public/records` → `/var/lib/webcctv/records`
- buat `/opt/webcctv/.env`
- init DB
- install systemd `/etc/systemd/system/webcctv.service`
- `systemctl enable --now webcctv`

Selesai. Cek:
```bash
systemctl status webcctv
journalctl -u webcctv -f
```

**Perintah systemd penting:**
```bash
systemctl status webcctv        # cek jalan?
journalctl -u webcctv -f        # log live
systemctl restart webcctv       # restart setelah update file
systemctl stop webcctv
systemctl start webcctv
systemctl enable webcctv        # auto-start ON
systemctl disable webcctv       # auto-start OFF
```

> Mati lampu → STB nyala → service otomatis start dalam 20-40 detik setelah boot.

### Install manual service (tanpa installer)

Jika aplikasi tetap di `/root/web-cctv`:

```bash
sudo nano /etc/systemd/system/webcctv.service
```
Isi:
```
[Unit]
Description=Web CCTV HG680P
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/root/web-cctv
Environment=NODE_ENV=production
Environment=PORT=3000
Environment=DB_PATH=/root/web-cctv/cctv.db
ExecStartPre=/usr/bin/node /root/web-cctv/init-db.js
ExecStart=/usr/bin/node /root/web-cctv/server.js
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```
Lalu:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now webcctv
```

---

## ⚙️ 7. Konfigurasi .env

File `.env` (dibuat otomatis oleh installer):

```
PORT=3000
JWT_SECRET=cctv_hg680p_secret_ganti_ini_please
DB_PATH=/var/lib/webcctv/cctv.db

# Live transcode – HG680P optimized
VIDEO_SIZE=960x540
VIDEO_FPS=15
VIDEO_BITRATE=800k
VIDEO_AUDIO=0        # 1 = aktifkan audio (butuh ffmpeg aac)

# Recording
REC_SIZE=1280x720
```

**Mode super hemat HG680P (CPU <30% / stream):**
```
VIDEO_SIZE=640x360
VIDEO_FPS=10
VIDEO_BITRATE=500k
```

Setelah edit .env:
```bash
systemctl restart webcctv
```

---

## 📹 8. Tambah Kamera

Login admin → menu **Kamera** → form “Tambah / Edit Kamera”

### A. IP Camera / NVR / DVR (RTSP)
- **Nama**: Lobby
- **RTSP URL**: `rtsp://admin:pass@192.168.1.64:554/h265/ch1/main/av_stream`
- **Tipe**: IP Camera / NVR / DVR
- **Channel**: 1
- Centang Publik / Aktif
- Simpan

Contoh RTSP:
```
Hikvision NVR:
rtsp://admin:pass@ip:554/Streaming/Channels/101
# 101=ch1 main, 102=ch1 sub, 201=ch2 main …

Dahua NVR:
rtsp://admin:pass@ip:554/cam/realmonitor?channel=1&subtype=0
# subtype 0=main, 1=sub

Umum H.265:
rtsp://admin:pass@ip:554/h265/ch1/main/av_stream
```

H.265 otomatis di-transcode ke H.264 540p 15fps.

### B. HLS / HTTP Live (tanpa transcode – ringan!)
- **Nama**: Kaligandu Serang
- **RTSP URL**: `https://stream.serangkota.go.id/LiveApp/streams/kaligandu.m3u8`
- **Tipe**: **HLS / HTTP Live**
- Simpan

> URL kotor seperti  
> `https://stream.serangkota.go.id/.../kaligandu.m3u8?token=undefined&subscriberId=undefined&subscriberCode=undefined?token=undefined...`  
> otomatis dibersihkan server menjadi `...kaligandu.m3u8`

Player akan direct HLS, jika CORS error otomatis fallback ke `/api/hls-proxy`.

### C. YouTube Live
- **Tipe**: YouTube Live
- **YouTube Video ID / URL**: bisa isi salah satu:
  - `dQw4w9WgXcQ`
  - `https://www.youtube.com/watch?v=dQw4w9WgXcQ`
  - `https://youtu.be/dQw4w9WgXcQ`
  - `https://www.youtube.com/embed/dQw4w9WgXcQ`
- ID otomatis di-extract
- Thumbnail otomatis dari `img.youtube.com`

### Lokasi Map
Isi Lat / Lng di form kamera, misal Jakarta:
```
Lat: -6.2088
Lng: 106.8456
```
Pin muncul di menu **Map**.

---

## ▶️ 9. Cara Pakai – Live

1. Buka **Live**
   - Default: mode **snapshot** – thumbnail update tiap 15 detik (hemat CPU)
   - Toggle **“Live di grid” ON**: HLS / YouTube langsung play di card (max 4 kamera)
2. Klik card kamera → buka **player modal**
   - HLS auto-reconnect, status tampil: `connecting… / live / reconnect 2/10`
   - Tombol **● Rec** (admin) → rekam manual
3. Mobile HP:
   - Grid otomatis 1 kolom
   - Modal player fullscreen
   - Swipe nav menu horizontal

Jika video tidak tampil:
- Cek F12 → Console / Network → cari `.m3u8` – harus 200
- Test HLS proxy manual:
  ```
  http://ip-stb:3000/api/hls-proxy?url=https://stream.serangkota.go.id/LiveApp/streams/kaligandu.m3u8
  ```
  harus keluar `#EXTM3U`
- Log FFmpeg: `cat /opt/webcctv/logs/ff_1.log` atau `journalctl -u webcctv -f`

---

## 📼 10. Rekaman

- **Manual**: buka player → tombol ● Rec → masukkan durasi detik
- **Otomatis / Scheduler**: di form Kamera → centang “Aktifkan rekam terjadwal”
  - `record_duration`: 300 (detik)
  - `record_schedule` cron mini:
    - `* * * * *` – tiap menit (tes)
    - `*/15 * * * *` – tiap 15 menit
    - `0 * * * *` – tiap jam pas menit 0
    - `0 0 * * *` – tiap tengah malam
- File: `/var/lib/webcctv/records/{camera_id}/2025-06-21T14-30-00.mp4`
- Menu **Rekaman** → Play / Download / Hapus

---

## 👥 11. Manajemen User

Menu **User** (admin only):
- Tambah user: username, password, role `admin` / `public`
- Edit: kosongkan password jika tidak ganti
- Tidak bisa hapus admin terakhir

Default:
```
admin  / admin123   → role admin
publik / publik123  → role public
```
**Segera ganti password setelah install!**

---

## 🐳 12. Docker

```bash
docker compose up -d --build
# http://localhost:3000
```
Volume:
- `./data:/data` → `cctv.db`
- `./records:/app/public/records`
- `./streams:/app/public/streams`

Edit ENV di `docker-compose.yml`.

---

## 📦 13. .deb Package (Armbian / Debian arm64 / armhf)

Build di STB / PC Debian:
```bash
cd /opt/webcctv
./build-deb.sh
# output: webcctv_2.0.0_arm64.deb
```

Install:
```bash
sudo dpkg -i webcctv_2.0.0_arm64.deb
sudo systemctl status webcctv
```

Uninstall:
```bash
sudo dpkg -r webcctv
```

Package install ke:
- App: `/opt/webcctv`
- DB: `/var/lib/webcctv/cctv.db`
- Records: `/var/lib/webcctv/records`
- Service: `/etc/systemd/system/webcctv.service` (auto enable)

---

## 🗂️ 14. Struktur File

```
server.js              # Express + FFmpeg HLS + Recorder + Proxy (SQLite)
init-db.js             # bikin cctv.db + seed users
package.json
.env.example
public/
  index.html           # SPA: Dashboard / Live / Map / Rekaman / Kamera / User
  app.js               # hls.js auto-reconnect, grid live, snapshot
  style.css            # dark modern, mobile-first
  streams/             # output HLS .m3u8 + .ts (tmp)
  snapshots/           # jpg thumbnail 480x270
  records/ -> /var/lib/webcctv/records  (symlink saat install)
logs/
  ff_1.log             # ffmpeg live log per kamera
  ff_snap_1.log        # snapshot error log
deploy/
  webcctv.service      # systemd
install-autostart.sh   # installer auto-start 1-klik
Dockerfile
docker-compose.yml
build-deb.sh
README.md
server.mysql.js        # backup versi MySQL
database.mysql.sql
```

---

## 🔌 15. API Ringkas

```
POST /api/login {username,password} → {token,role}

# Cameras
GET    /api/cameras
POST   /api/cameras                 (admin)
PUT    /api/cameras/:id             (admin)
DELETE /api/cameras/:id             (admin)
GET    /api/cameras/status          (auth optional)
POST   /api/cameras/:id/ping        (admin)

# Stream
POST /api/stream/:id/start         (auth optional – HLS/YouTube public boleh)
POST /api/stream/:id/stop          (admin)
GET  /api/stream/:id/status
GET  /api/stream/:id/log           (admin)

# HLS CORS proxy
GET  /api/hls-proxy?url=https://...m3u8

# Snapshot
GET  /api/snapshot/:id             → image/jpeg

# Record
POST   /api/record/:id/start {duration} (admin)
POST   /api/record/:id/stop          (admin)
GET    /api/records?camera_id=
DELETE /api/records/:id              (admin)

# Users
GET    /api/users                   (admin)
POST   /api/users                   (admin)
PUT    /api/users/:id               (admin)
DELETE /api/users/:id               (admin)

# Dashboard
GET /api/dashboard                 (auth optional)
```

HLS file lokal:
```
/streams/1/index.m3u8
```

---

## 🩺 16. Troubleshooting HG680P

**1. `Error: Cannot find module 'express'`**
```bash
cd /opt/webcctv
npm install --omit=dev
```
`node_modules` hilang, install ulang.

**2. Stream exit code 8 / `Unrecognized option 'stimeout'`**
– Sudah di-fix v2.1+, pastikan pakai `server.js` terbaru (tanpa `-stimeout`).

**3. FFmpeg exit, live hitam**
```bash
journalctl -u webcctv -f
cat /opt/webcctv/logs/ff_1.log
```
Biasanya:
- RTSP user/pass/IP salah
- Kamera H.265 high profile – tetap bisa, karena di-transcode
- FFmpeg Armbian tidak ada aac → audio otomatis dimatikan (`VIDEO_AUDIO=0`)
Test manual:
```bash
ffmpeg -rtsp_transport tcp -i "rtsp://admin:pass@ip:554/..." -c:v libx264 -preset ultrafast -tune zerolatency -s 960x540 -r 15 -an -f hls -hls_time 2 /tmp/test.m3u8
```

**4. YouTube / m3u8 tidak tampil**
- Pastikan tipe kamera = `hls` atau `youtube`
- YouTube ID bisa URL penuh – otomatis extract
- HLS eksternal CORS error → player otomatis fallback ke `/api/hls-proxy`
- Test proxy:  
  `curl "http://localhost:3000/api/hls-proxy?url=https://stream.serangkota.go.id/LiveApp/streams/kaligandu.m3u8"`

**5. Grid Live HP acak-acakan**
– Pastikan pakai `style.css` v2.5, hard refresh: Chrome HP → ⋮ → “Muat ulang” tahan → “Kosongkan cache”

**6. CPU 100%**
- Max 2 stream RTSP transcode bersamaan di HG680P
- Gunakan substream kamera
- Turunkan di `.env`:
  ```
  VIDEO_SIZE=640x360
  VIDEO_FPS=10
  VIDEO_BITRATE=500k
  ```
- Untuk kamera publik, pakai tipe HLS / YouTube (tanpa transcode)

**7. Port 3000 sudah dipakai**
```bash
sudo lsof -i :3000
# ganti PORT di .env, lalu systemctl restart webcctv
```

**8. Database terkunci / corrupt**
```bash
systemctl stop webcctv
rm /var/lib/webcctv/cctv.db /var/lib/webcctv/cctv.db-*
cd /opt/webcctv
DB_PATH=/var/lib/webcctv/cctv.db node init-db.js
systemctl start webcctv
```

---

## 🔒 17. Keamanan

- Ganti `JWT_SECRET` di `.env` dengan string acak panjang
- Ganti password default admin segera, via menu User
- Jika expose ke internet, pasang reverse proxy Nginx + HTTPS (Let's Encrypt)
- `rtsp_url` kamera RTSP tidak di-expose ke publik non-login (hanya HLS/YouTube yang boleh)
- Rate-limit / fail2ban disarankan jika publik

Contoh Nginx:
```
server {
  listen 80;
  server_name cctv.domain.com;
  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
  }
}
```

---

## 📝 Lisensi

MIT – bebas modifikasi untuk proyek CCTV desa / kota / internal.

---

## 📞 Versi

- **v2.5 – 21 Jun 2026**
  - HLS auto-reconnect + stall watchdog
  - Live di grid (real-time HLS/YouTube)
  - Mobile UI fix total
  - HLS CORS proxy
  - YouTube ID extractor robust
  - install-autostart.sh 1-klik
- v2.3 – Dashboard + online/offline + user CRUD
- v2.2 – Snapshot thumbnail
- v2.0 – SQLite + Recorder scheduler
- v1.0 – MySQL, basic HLS

Dibuat untuk **STB Armbian HG680P** – hemat resource, modern UI.
Butuh kustom? – tambah AI detection, Telegram alert, multi-tenant, silakan fork.
