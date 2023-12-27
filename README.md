# Final Project System Admin
## Web Cloud Menggunakan NextCloud
  Nextcloud adalah perangkat lunak peladen klien untuk membuat dan menggunakan layanan hos berkas. Secara fungsional mirip dengan Dropbox, meskipun Nextcloud gratis dan sumber terbuka, tetapi tak menutup kemungkinan untuk siapa pun menginstal dan mengoperasikannya di peladen pribadi. Tidak seperti Dropbox, Nextcloud tidak menawarkan hosting penyimpanan berkas di tempat.
  Berbeda dengan layanan seperti Dropbox, arsitektur perangkat lunak yang mendukung untuk menambah, peningkatan, dan penggantian komponen seperti penambahan fungsionalitas ke peladen dalam bentuk aplikasi memungkinkan pengguna untuk memiliki kontrol penuh atas data mereka.
## Service yang Perlu Diinstall
1. NGINX
2. PostgreSQL
3. PHP Module
4. Cockpit remote server
5. OS Ubuntu 22.04
   
## Langkah-langkah Install NextCloud on Ubuntu
### Step 1 : Download NectCloud on Ubuntu 22.04
```
wget https://download.nextcloud.com/server/releases/nextcloud-24.0.0.zip
```
Install unzip utility
```
sudo apt install unzip
```
Buat direktori /var/www/ dan extrack archive file
```
sudo mkdir -p /var/www/
sudo unzip nextcloud-24.0.0.zip -d /var/www/
```
### Step 2: Buat Database dan User untuk Nextcloud di PostgreSQL
Nextcloud compatible menggunakan PostgreSQL,MariaDB/MySQL, dan SQLite. Nextcloud berjalan lebih cepat menggunakan PostgreSQL, jadi disini saya menggunakan PostgreSQL.
```
sudo apt install -y postgresql postgresql-contrib
```
Masuk ke PostgreSQL sebagai postgres user
```
sudo -u postgres psql
```
Buat database Nextcloud
```
CREATE DATABASE nextcloud TEMPLATE template0 ENCODING 'UNICODE';
```
Buat user dan set password
```
CREATE USER nextclouduser WITH PASSWORD 'nextclouduser_password';
```
Beri ijin ke database user
```
ALTER DATABASE nextcloud OWNER TO nextclouduser;
GRANT ALL PRIVILEGES ON DATABASE nextcloud TO nextclouduser;
```
Tekan ```Ctrl+D``` untuk keluar dari PostgreSQL console
### Step 3: Buat Nginx Virtual Host untuk Nextcloud
Install Nginx web server.
```
sudo apt install nginx
```
Buat file nextcloud.conf di directory /etc/nginx/conf.d/
```
sudo nano /etc/nginx/conf.d/nextcloud.conf
```
Lalu masukkan file dibawah ini
```
server {
    listen 443;
    listen [::]:443;
    server_name cloud.doom.my.id;

    # Add headers to serve security related headers
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header X-Download-Options noopen;
    add_header X-Permitted-Cross-Domain-Policies none;
    add_header Referrer-Policy no-referrer;

    #I found this header is needed on Ubuntu, but not on Arch Linux. 
    add_header X-Frame-Options "SAMEORIGIN";

    # Path to the root of your installation
    root /var/www/nextcloud/;

    access_log /var/log/nginx/nextcloud.access;
    error_log /var/log/nginx/nextcloud.error;

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # The following 2 rules are only needed for the user_webfinger app.
    # Uncomment it if you're planning to use this app.
    #rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
    #rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json
    # last;

    location = /.well-known/carddav {
        return 301 $scheme://$host/remote.php/dav;
    }
    location = /.well-known/caldav {
       return 301 $scheme://$host/remote.php/dav;
    }

    location ~ /.well-known/acme-challenge {
      allow all;
    }

    # set max upload size
    client_max_body_size 512M;
    fastcgi_buffers 64 4K;

    # Disable gzip to avoid the removal of the ETag header
    gzip off;

    # Uncomment if your server is build with the ngx_pagespeed module
    # This module is currently not supported.
    #pagespeed off;

    error_page 403 /core/templates/403.php;
    error_page 404 /core/templates/404.php;

    location / {
       rewrite ^ /index.php;
    }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
       deny all;
    }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
       deny all;
     }

    location ~ ^/(?:index|remote|public|cron|core/ajax/update|status|ocs/v[12]|updater/.+|ocs-provider/.+|core/templates/40[34])\.php(?:$|/) {
       include fastcgi_params;
       fastcgi_split_path_info ^(.+\.php)(/.*)$;
       try_files $fastcgi_script_name =404;
       fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
       fastcgi_param PATH_INFO $fastcgi_path_info;
       #Avoid sending the security headers twice
       fastcgi_param modHeadersAvailable true;
       fastcgi_param front_controller_active true;
       fastcgi_pass unix:/run/php/php8.1-fpm.sock;
       fastcgi_intercept_errors on;
       fastcgi_request_buffering off;
    }

    location ~ ^/(?:updater|ocs-provider)(?:$|/) {
       try_files $uri/ =404;
       index index.php;
    }

    # Adding the cache control header for js and css files
    # Make sure it is BELOW the PHP block
    location ~* \.(?:css|js)$ {
        try_files $uri /index.php$uri$is_args$args;
        add_header Cache-Control "public, max-age=7200";
        # Add headers to serve security related headers (It is intended to
        # have those duplicated to the ones above)
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Robots-Tag none;
        add_header X-Download-Options noopen;
        add_header X-Permitted-Cross-Domain-Policies none;
        add_header Referrer-Policy no-referrer;
        # Optional: Don't log access to assets
        access_log off;
   }

   location ~* \.(?:svg|gif|png|html|ttf|woff|ico|jpg|jpeg)$ {
        try_files $uri /index.php$uri$is_args$args;
        # Optional: Don't log access to other assets
        access_log off;
   }
}
```
Ubah kepemilikan direktori ini ke ```www-data``` jadi web server nginx dapat membaca ke direktori ini
```
sudo chown www-data:www-data /var/www/nextcloud/ -R
```
Test Nginx configuration
```
sudo nginx -t
```
Jika sukses reload nginx
```
sudo systemctl reload nginx
```
### Step 4: Install dan Enable PHP module
Install paket PHP
```
sudo apt install imagemagick php-imagick php8.1-common php8.1-pgsql php8.1-fpm php8.1-gd php8.1-curl php8.1-imagick php8.1-zip php8.1-xml php8.1-mbstring php8.1-bz2 php8.1-intl php8.1-bcmath php8.1-gmp
```
### Step 5: Buat Domain dan Setting Cloudflare
Beli domain di website yang terpercaya
![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/13e13162-b84c-4787-8cf7-2a48b220e323)

Buat akun cloudflare, lalu masuk ke menu add site

![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/281108ff-8ad8-488b-87a8-9d5ecf8a4f48)

Masukkan domain yang sudah dibeli

![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/a2579b6b-5848-4d24-9587-9aeb60173118)

Lalu copy Name server cloudflare ke website hosting yang kita beli, tunggu 24 untuk pengaktivan.

![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/a7b1b06c-2b92-43e7-b403-3d0c4b6ae064)

Masuk kemenu Zero trust lalu pilih tunnel

![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/24cf1d80-0924-4f04-804a-4f76b3cc1e06)

Masukkan nama tunnel 

![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/b3976d28-36bc-4f77-a6e7-3f3c398068e1)

Lalu pilih os debian dan install konector cloudflare di ubuntu

![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/8b6b2e6e-c2d9-410d-b1b0-04349483039f)

kemudian buat publick hostname baru

![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/0098dab5-3a11-4f0c-b891-9824fc1bd91a)

buat subdomain yang sama seperti file nextcloud.conf dan masukkan port yang sama juga

![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/a4fa6d2d-1ca4-43b5-beab-598e1c84b56e)


### Step 6: Enable HTTPS
jika website tidak bisa muncul maka kamu butuh membuka port 80 di firewall
```
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
```
dan juga port 443
```
sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT
```

### Step 7 : Setting Nextcloud di Web Browser
Buat folder untuk penyimapan file nextcloud
```
sudo mkdir /var/www/nextcloud-data
```
Beri ijin agar bisa dibaca nginx user
```
sudo chown www-data:www-data /var/www/nextcloud-data -R
```
kemudian login ke nexcloud menggunakan browser di windows dengan link yang sudah dibuat
```
cloud.doom.my.id
```
Lalu isi seperti dibawah ini untuk passwar menggunakan passwaord user database yang sudah kita buat di awal, dan port 5432 adalah port default dari PostgreSQL. untuk username dan pasword bagian atas bisa diisi bebas( ini digunakan untuk login sebagai admin nextcloud)
![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/2c30b121-c192-4602-9838-799330892a01)

Maka akan muncul tampilan seperti ini jika setelah selesai
![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/86e317c0-27c6-425c-8f6f-a1aeb876257b)

### Step 8 : Add User Nextcloud
1. login nextcloud sebagai admin
2. klik logo kanan atas, masuk ke menu user
   ![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/b53868e6-cd5c-47ea-94f3-d5afb6c767b3)

3. klik ```+ new user``` kemuadian masukkan user baru
4. ![image](https://github.com/Fajarkhakimm/FPOS/assets/147434983/52147a92-5b85-4f00-b4f5-578b1ec310c6)

