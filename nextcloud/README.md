# Nextcloud

## docker-compose.yml

```yml
version: "3.7"

services:
  nextcloud:
    image: linuxserver/nextcloud:latest
    container_name: nextcloud
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=${TIMEZONE}
    volumes:
      - /data/nextcloud/config:/config
      - /data/nextcloud:/data
    restart: unless-stopped
    depends_on:
      - db
      - redis
    networks:
      - proxy
      - default
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.nextcloud.entrypoints=https"
      - "traefik.http.routers.nextcloud.rule=Host(`nextcloud.domain.com`)"
      - "traefik.http.routers.nextcloud.tls=true"
      - "traefik.http.services.nextcloud.loadbalancer.server.port=443"
      - "traefik.http.services.nextcloud.loadbalancer.server.scheme=https"
      - "traefik.http.middlewares.nc-header.headers.stsSeconds=31536000"
      - "traefik.http.routers.nextcloud.middlewares=nc-header"

  db:
    image: linuxserver/mariadb:latest
    container_name: nextcloud_db
    environment:
      - PUID=1000
      - PGID=1000
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PW}
      - TZ=${TIMEZONE}
      - MYSQL_DATABASE=${DB_NAME}
      - MYSQL_USER=${DB_USER}
      - MYSQL_PASSWORD=${DB_PW}
    volumes:
      - /data/nextcloud/config_db:/config
    restart: unless-stopped
    networks:
      - default

  redis:
    container_name: nextcloud-redis
    networks:
      - default
    image: 'redis:latest'
    restart: always
    volumes:
      - nc-redis:/data

volumes:
  nc-redis:

networks:
  proxy:
    external: true
```

## Set up

Go to your Nextcloud's URL, select MySQL and use `db` as URL for database. Password is `${DB_PW}`

### Set datadirectory

Add to `config/config.php`, usually `/data/nextcloud/config/www/nextcloud/config/config.php`.

```php
  'datadirectory' => '/data',
```

### Add a new trusted domain

Go to `config/config.php`, usually `/data/nextcloud/config/www/nextcloud/config/config.php`.

Initial config:

```php
  'trusted_domains' =>
  array (
    0 => '192.168.1.126:9443',
  ),
```

To add a new domain just add new entries by appending a new item to the PHP array:

```php
  'trusted_domains' =>
  array (
    0 => '192.168.1.126:9443',
    1 => 'nextcloud.domain.com',
  ),
```

To add redis:

```php
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'memcache.distributed' => '\\OC\Memcache\\Redis',
  'redis' =>
  array (
    'host' => 'redis',
    'port' => 0,
    'timeout' => 0.0,
  ),
```

`docker exec nextcloud-redis redis-cli monitor` should display something if redis is working.

### Remove password reset

Add to :`vim www/nextcloud/config/config.php`

```php
  'lost_password_link' => 'disabled',
  'auth.webauthn.enabled' => false, 
```

### Fix X_Forwarded_For

Go to `config/config.php`, usually `/data/nextcloud/config/www/nextcloud/config/config.php`.
Change `192.168.16.0/24`, with traefik's docker network, use: `docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' traefik`.

```php
  'trusted_domains' =>
  array (
    0 => 'nextcloud.domain.com',
  ),
  'trusted_proxies' =>
  array (
    0 => '192.168.16.0/24',
  ),
  'forwarded_for_headers' => array('HTTP_X_FORWARDED_FOR'),
```

### Filesystemcheck

For Nextcloud to check changes on file system:

```php
  'filesystem_check_changes' => 1,
```

### Memories app

[Support for high resolution images](https://github.com/pulsejet/memories/wiki/Configuration#imagevideo-support)

```php
  'preview_max_memory' => 4096,
  'preview_max_filesize_image' => 256,
```

### Fix region error

Add to :`www/nextcloud/config/config.php`

```php
  'default_phone_region' => 'ES',
```

### Enable OPCache

Add to: `config/php/php-local.ini`

```php
  opcache.enable=1
  opcache.interned_strings_buffer=10
  opcache.max_accelerated_files=10000
  opcache.memory_consumption=128
  opcache.save_comments=1
  opcache.revalidate_freq=1
```

### INotify

To install the INotify file watcher app, manually create `/config/scripts` folder on the docker container and add this script:

```
#!/bin/bash

apk update &&\
apk add --no-cache autoconf automake file g++ gcc make php82-dev php82-pear re2c zlib-dev inotify-tools &&\
wget -O /config/scripts/inotify-3.0.0.tgz https://pecl.php.net/get/inotify-3.0.0.tgz &&\
pear82 install /config/scripts/inotify-3.0.0.tgz &&\
echo "extension="inotify.so"" > /etc/php82/conf.d/00_inotify.ini &&\
apk del --purge &&\
occ app:install files_inotify
```

Remember to bind the folder to `/custom-cont-init.d:ro`, for example, add:
```
- /data/nextcloud/scripts:/custom-cont-init.d:ro
```

### Remove mail server warning

Two methods:

#### CLI

Add to: `config/php/php-local.ini`

```php
  'mail_smtpmode' => 'sendmail',
  'mail_sendmailmode' => 'pipe',
```

#### GUI

Set default mail on `Personal info > Administration > Basic settings` and change “Send Mode” to “Sendmail”, “Sendmail mode” to “pipe (-t)”, leave “From address” emty and hit “Send email”.

## Backup

First create folder /data/nextcloud/db_bak, then `crontab -e`

```
0 14 * * * docker exec nextcloud occ maintenance:mode --on && docker exec nextcloud_db mysqldump --single-transaction -h localhost -u ${DB_USER} -p${DB_PW} nextcloud > /data/nextcloud/db_bak/nextcloud-`date +"\%Y-\%m-\%d"`.sql && docker exec nextcloud occ maintenance:mode --off
```
