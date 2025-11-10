# Wildcard SSL Let's Encrypt untuk Nginx
Panduan lengkap membuat sertifikat wildcard Let's Encrypt dan menggunakannya di Nginx.  
Domain contoh: `*.inf-dr.monitoring.cloudeka.id`

---

## 1. Persiapan Awal
Pastikan:
- Server menggunakan Ubuntu/Debian.
- Kamu punya akses ke DNS zone `monitoring.cloudeka.id`.
- Port 80 dan 443 terbuka di firewall.
- Nginx dan Certbot sudah terpasang.

### Instalasi Certbot dan Nginx
```bash
sudo apt update
sudo apt install -y nginx certbot
```

---

## 2. Membuat Wildcard Certificate Let's Encrypt
Wildcard hanya bisa dibuat menggunakan **DNS-01 challenge**, jadi kamu perlu menambahkan **TXT record** ke DNS.

Jalankan perintah:
```bash
sudo certbot -d "*.inf-dr.monitoring.cloudeka.id" -d inf-dr.monitoring.cloudeka.id --manual --preferred-challenges dns certonly
```

Certbot akan memberikan instruksi seperti ini:
```
Please deploy a DNS TXT record under the name
_acme-challenge.inf-dr.monitoring.cloudeka.id with the following value:

8nKfjhH2v9FJqRla_9a4YQhZC1A7bV8Hh7v7sF4H6Gs
```

---

## 3. Tambahkan DNS Record

Masuk ke DNS Manager tempat domain `monitoring.cloudeka.id` dikelola  
(contoh: Cloudflare, cPanel, Google Cloud DNS, Route53, dll).

Tambahkan record berikut:

| Type | Name / Host | Value | TTL |
|------|--------------|-------|-----|
| TXT | `_acme-challenge.inf-dr.monitoring.cloudeka.id` | `8nKfjhH2v9FJqRla_9a4YQhZC1A7bV8Hh7v7sF4H6Gs` | 300 |

### Verifikasi DNS
Gunakan perintah berikut untuk memastikan DNS sudah aktif:
```bash
dig TXT _acme-challenge.inf-dr.monitoring.cloudeka.id +short
```

Output yang benar akan menampilkan kode verifikasi yang sama seperti yang diberikan Certbot.

Setelah terverifikasi, tekan **Enter** di terminal Certbot untuk melanjutkan proses validasi.

---

## 4. Hasil Sertifikat

Jika berhasil, akan muncul pesan:
```
Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/inf-dr.monitoring.cloudeka.id/fullchain.pem
Your key file has been saved at:
/etc/letsencrypt/live/inf-dr.monitoring.cloudeka.id/privkey.pem
```

Lokasi file:
```
/etc/letsencrypt/live/inf-dr.monitoring.cloudeka.id/
├── cert.pem
├── chain.pem
├── fullchain.pem
└── privkey.pem
```

---

## 5. Konfigurasi Nginx

Buat file konfigurasi baru:
```bash
sudo nano /etc/nginx/sites-available/inf-dr.monitoring.cloudeka.id.conf
```

Isi dengan konfigurasi berikut:

```nginx
server {
    listen 80;
    server_name inf-dr.monitoring.cloudeka.id *.inf-dr.monitoring.cloudeka.id;

    # Redirect semua HTTP ke HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name inf-dr.monitoring.cloudeka.id *.inf-dr.monitoring.cloudeka.id;

    ssl_certificate /etc/letsencrypt/live/inf-dr.monitoring.cloudeka.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/inf-dr.monitoring.cloudeka.id/privkey.pem;

    # Pengaturan keamanan tambahan
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://127.0.0.1:8080; # Ubah sesuai aplikasi kamu
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    access_log /var/log/nginx/inf-dr.access.log;
    error_log /var/log/nginx/inf-dr.error.log;
}
```

---

## 6. Aktifkan Konfigurasi dan Uji Nginx

```bash
sudo ln -s /etc/nginx/sites-available/inf-dr.monitoring.cloudeka.id.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Jika muncul:
```
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
berarti konfigurasi sudah benar.

---

## 7. Otomatisasi Pembaruan Sertifikat

Let's Encrypt hanya berlaku **90 hari**, jadi kita tambahkan cronjob untuk auto renew:

```bash
sudo crontab -e
```

Tambahkan baris berikut:
```
0 3 * * * certbot renew --quiet --deploy-hook "systemctl reload nginx"
```

Ini akan memeriksa setiap hari jam 03:00, dan reload Nginx otomatis jika sertifikat diperbarui.

---

## 8. Pengujian HTTPS

Uji dengan:
```bash
curl -v https://inf-dr.monitoring.cloudeka.id
```

Atau buka di browser:
```
https://inf-dr.monitoring.cloudeka.id
```

Cek validitas SSL menggunakan:
👉 [https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)

---

## 9. (Opsional) Banyak Subdomain dengan Satu SSL
Wildcard ini otomatis melindungi semua subdomain, misalnya:
- `grafana.inf-dr.monitoring.cloudeka.id`
- `prometheus.inf-dr.monitoring.cloudeka.id`
- `api.inf-dr.monitoring.cloudeka.id`

Kamu tidak perlu membuat sertifikat tambahan — cukup satu wildcard.

---

## 10. Troubleshooting

| Masalah | Penyebab Umum | Solusi |
|----------|----------------|--------|
| `NXDOMAIN` saat verifikasi | TXT record belum dibuat di zone yang benar | Pastikan `_acme-challenge.inf-dr.monitoring.cloudeka.id` benar |
| `Timed out during DNS verification` | DNS belum propagasi | Tunggu 1–5 menit atau gunakan TTL kecil |
| Browser tetap HTTP | Nginx belum redirect atau sertifikat belum aktif | Pastikan block `listen 443` aktif dan reload Nginx |
| Renew gagal | Menggunakan mode manual | Gunakan plugin DNS (Cloudflare, Route53, dll) untuk otomatisasi |

---

## 11. Lokasi File Penting

| File | Lokasi | Keterangan |
|------|---------|------------|
| Fullchain | `/etc/letsencrypt/live/inf-dr.monitoring.cloudeka.id/fullchain.pem` | Sertifikat publik |
| Private Key | `/etc/letsencrypt/live/inf-dr.monitoring.cloudeka.id/privkey.pem` | Kunci privat |
| Konfigurasi Nginx | `/etc/nginx/sites-available/inf-dr.monitoring.cloudeka.id.conf` | Virtual host |
| Log Nginx | `/var/log/nginx/inf-dr.access.log`, `/var/log/nginx/inf-dr.error.log` | Log akses dan error |

---

## Selesai!
Sekarang domain `*.inf-dr.monitoring.cloudeka.id` sudah aman dengan HTTPS wildcard certificate dari Let's Encrypt 🚀
