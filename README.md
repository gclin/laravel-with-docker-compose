# laravel-with-docker-compose

Step by Step 的說明，如何在 Laravel 的專案內使用 Docker Compose

## Step 1. 建立 Laravel 專案

```sh
laravel new laravel-with-docker-compose
cd laravel-with-docker-compose
```

## Step 2. 加入 Docker Compose 及 Service 的設定

### 產生 `docker-compose.yml`

裡面會包含 `nginx` 、 `php` 、 `mariadb` 和 `phpmyadmin` 4 個服務的設定，服務名稱可以隨便設定，但如果有更動要連同 `links` 及有用服務名稱連線的設定，都要一併修改。

```sh
cat << 'EOF' > docker-compose.yml
version: "3.1"

services:
  nginx:
    image: nginx:1.17
    working_dir: /laravel/www
    ports:
      - 80:80
    volumes:
      - ./:/laravel/www:cached
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    links:
      - php
      - phpmyadmin

  php:
    image: php:7.2-fpm-stretch
    working_dir: /laravel/www
    volumes:
      - ./:/laravel/www:cached
      - ./docker/php/php-fpm.d/custom.conf:/usr/local/etc/php-fpm.d/custom.conf
    environment:
      DB_HOST: mariadb
    links:
      - mariadb

  mariadb:
    image: mariadb:10.4
    ports:
      - 3306:3306
    volumes:
      - ./docker/mysql/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
      # 有要把 db 資料導出的加入這行
      #- ./docker/mysql/data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: I2U0Z6TddCFEq3uW

  phpmyadmin:
    image: phpmyadmin/phpmyadmin:fpm-alpine
    environment:
      PMA_HOST: mysql
      PMA_ABSOLUTE_URI: http://127.0.0.1/pMA/
EOF
```

### Nginx Config

```sh
mkdir -p docker/nginx
cat << 'EOF' > docker/nginx/default.conf
server {
    listen 80;
    listen [::]:80;

    server_name _;

    set $base /laravel/www;
    root $base/public;

    index index.php;
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /pMA {
        return 302 http://$host/pMA/;
    }

    # reverse proxy
    location ~ \/pMA {
        rewrite ^/pMA(/.*) $1  break;
        proxy_pass http://phpmyadmin;
        proxy_http_version   1.1;
        proxy_cache_bypass   $http_upgrade;

        proxy_set_header Upgrade            $http_upgrade;
        proxy_set_header Connection         "upgrade";
        proxy_set_header Host               $host;
        proxy_set_header X-Real-IP          $remote_addr;
        proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_set_header X-Forwarded-Host   $host;
        proxy_set_header X-Forwarded-Port   $server_port;

    }
 
    # handle .php
    location ~ \.php$ {
        # 404
        try_files $fastcgi_script_name =404;

        # default fastcgi_params
        include fastcgi_params;

        # fastcgi settings
        fastcgi_pass          php:9000;
        fastcgi_index         index.php;
        fastcgi_buffers       8 16k;
        fastcgi_buffer_size   32k;

        # fastcgi params
        fastcgi_param DOCUMENT_ROOT     $realpath_root;
        fastcgi_param SCRIPT_FILENAME   $realpath_root$fastcgi_script_name;
        fastcgi_param PHP_ADMIN_VALUE   "open_basedir=$base/:/usr/lib/php/:/tmp/";
    }

    location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        include        fastcgi_params;
    }

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
EOF
```

### PHP Config

這邊是 php-fpm 的設定，也可以用來覆寫 php.ini 的設定

```sh
mkdir -p docker/php/php-fpm.d
cat << 'EOF' > docker/php/php-fpm.d/custom.conf
[www]
access.format = "%R - %u %t \"%m %r%Q%q\" %s %f %{mili}d %{kilo}M %C%%"
php_flag[expose_php] = off
EOF
```

### MariaDB Config

以前是用 MySQL，現在改用 MariaDB，想找其他資料庫系統，可以到 Docker Hub 上看看有沒有。

以 MariaDB 來說，Container 裡的 `/docker-entrypoint-initdb.d` 目錄，在 Container 啟動時會去執行這個目錄裡的東西，例如：在裡面放個 `initial.sql`，用來建立專案所需的資料庫及帳號。

```sh
mkdir -p docker/mysql/docker-entrypoint-initdb.d
cat << 'EOF' > docker/mysql/docker-entrypoint-initdb.d/initial.sql
EOF
```

### 啟動 Container

```sh
# 啟動
docker-compose up -d

# 查看 Log
docker-compose logs -f
```

### 開啟網站

打開這個網址 [http://127.0.0.1/](http://127.0.0.1/) 應該要可以看到 Laravel 的預設首頁。