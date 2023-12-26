# Final Project System Admin
## Web Cloud Menggunakan NextCloud
  Nextcloud adalah perangkat lunak peladen klien untuk membuat dan menggunakan layanan hos berkas. Secara fungsional mirip dengan Dropbox, meskipun Nextcloud gratis dan sumber terbuka, tetapi tak menutup kemungkinan untuk siapa pun menginstal dan mengoperasikannya di peladen pribadi. Tidak seperti Dropbox, Nextcloud tidak menawarkan hosting penyimpanan berkas di tempat.
  Berbeda dengan layanan seperti Dropbox, arsitektur perangkat lunak yang mendukung untuk menambah, peningkatan, dan penggantian komponen seperti penambahan fungsionalitas ke peladen dalam bentuk aplikasi memungkinkan pengguna untuk memiliki kontrol penuh atas data mereka.
## Service yang Perlu Diinstall
1. NGINX
2. PostgreSQL
3. PHP Module
4. OS Ubuntu 22.04
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
Lalu copy paste file dibawah ini
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

## cara penggunaan
