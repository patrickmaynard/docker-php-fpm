# docker-php-fpm

This is a docker php fpm image, based on the official php fpm image. It has the following additions:

- extensions
  - memcached (2.2.0 for php <7.0, 3.0.3 for php >=7.0)
  - gd (2.1.0)
    - png (1.2.50)
    - jpeg (6b)
    - webp
    - freetype (2.5.2)
    - gif create/read
    - wbmp
    - xpm
    - xbm
  - zip (1.11.0, lib: 0.10.1)
  - imagick
  - intl (1.1.0, icu: 52.1)
  - mysqli (5.0.11-dev)
  - pdo_mysql (5.0.11-dev)
  - xdebug (2.5.5 for php <7.2, 2.6.0alpha1 for php >=7.2)
  - opcache
  - soap
  - bcmath
  - xml
  - xsl
- composer cli (1.4.1)
- git cli (2.1.4)
- vim (7.4)
- mysql-client (mysql Ver 14.14 Distrib 5.5.58)
- rsync (3.1.1)
- sshpass (1.05)
- cron/crontab with `start-cron` executable
- possibility to override special php ini settings with environment variables: (see included php.ini for a full list, see [this blog post for reasons](https://dracoblue.net/dev/use-environment-variables-for-php-ini-settings-in-docker/)):
```ini
memory_limit=${PHP_MEMORY_LIMIT};
post_max_size=${PHP_POST_MAX_SIZE};
max_file_uploads=${PHP_MAX_FILE_UPLOADS};
upload_max_filesize=${PHP_UPLOAD_MAX_FILESIZE};
session.save_path=${PHP_SESSION_SAVE_PATH};
session.save_handler=${PHP_SESSION_SAVE_HANDLER};
max_input_vars=${PHP_MAX_INPUT_VARS};
```

## Usage "docker"

### Run PHP-FPM Server

```console
$ docker run --rm -p 9000:9000 -v `pwd`:/usr/src/app exozet/php-fpm:5.5.38
```

### Run PHP-CLI

If you want to launch a shell:

```console
$ docker run --rm -it -v `pwd`:/usr/src/app --user "${UID:www-data}:${GROUPS[0]:www-data}" exozet/php-fpm:5.5.38 bash
```

If you want to run a php command:

```console
$ docker run --rm -it -v `pwd`:/usr/src/app --user "${UID:www-data}:${GROUPS[0]:www-data}" exozet/php-fpm:5.5.38 php -v
PHP 5.5.38 (cli) (built: Aug 10 2016 21:09:37)
Copyright (c) 1997-2015 The PHP Group
Zend Engine v2.5.0, Copyright (c) 1998-2015 Zend Technologies
    with Zend OPcache v7.0.6-dev, Copyright (c) 1999-2015, by Zend Technologies
```

If you want to run composer:

```console
$ docker run --rm -it -v `pwd`:/usr/src/app --user "${UID:www-data}:${GROUPS[0]:www-data}" exozet/php-fpm:5.5.38 composer --version
Composer version 1.4.1 2017-03-10 09:29:45
```

## Using "Cron": Setting `CRONTAB_CONTENT`

You can define the crontab's content with an environment variable like this:

`docker-compose.yml`:
```yaml
services:
  import-data-cron:
    image: exozet/php-fpm:7.1.10
    command: start-cron
    environment:
      - 'CRONTAB_CONTENT=* * * * * php run-import.php >> /var/log/cron.log 2>&1'
    volumes:
      - ./:/usr/src/app:cached
```

It's very important to specify `/var/log/cron.log` as response for all outputs of your
cronjob, since crontab will otherwise try to send the response by email, which cannot work
in this docker setup.

We recommend to use **one** cronjob/container to ensure that your monitoring, restarting, recovery and
 so on works properly. Otherwise you don't **know**, which of your cronjobs is consuming which amount of
 resources.

## Alternative way to use "Cron": Mounting `/etc/cron.d` OR setting `CRON_PATH`

**Hint:** Please use this way only, if the previous way (setting `CRONTAB_CONTENT` Environment variable) does not work for your
project.

Create your crontab directory in project folder and put all your cron files in this directory.

`crontabs` directory:
```text
  - one-cron
  - other-cron
```
`one-cron` file:
```console
*/10 * * * * root php your-command/script >> /var/log/cron.log 2>&1
# Don't remove the empty line at the end of this file. It is required to run the cron job
```

Even though it's possible, we do not recommend to use **multiple** cronjob/container in one crontab file. This makes
monitoring the different cron jobs harder for your operation/monitoring/alerting tools.

Usage in your `docker-compose.yml`:
```yaml
services:
  crontab:
    image: exozet/php-fpm:7.1.10
    command: start-cron
    volumes:
      - ./:/usr/src/app
      - ./crontabs:/etc/cron.d
```

If your cron folder is already part or your project, you can override the
cron location with the `CRON_PATH` environment variable:

```yaml
services:
  crontab:
    image: exozet/php-fpm:7.1.10
    command: start-cron
    environment:
      - CRON_PATH=/usr/src/app/crontabs
    volumes:
      - ./:/usr/src/app
```

## Usage "docker-compose"

Create a `docker-compose.yml`:

```yaml
version: "2.1"

services:
  php-cli:
    image: exozet/php-fpm:5.5.38-sudo
    volumes:
      - ./:/usr/src/app
    user: "${UID-www-data}:${GID-www-data}"
    entrypoint: bash
    depends_on:
      - nginx
  php-fpm:
    image: exozet/php-fpm:5.5.38
    volumes:
      - ./:/usr/src/app
  nginx:
    image: nginx:1.11.10
    depends_on:
      - php-fpm
    ports:
      - "8080:8080"
    volumes:
      - ./:/usr/src/app
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
```

Create a `nginx.conf`:
```text
server {
    listen 8080 default_server;
    root /usr/src/app;

    location ~ \.php$ {
        fastcgi_pass php-fpm:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
    }
}
```

Create an `index.php`:

``` php
<?php

phpinfo();
```

Launch the php cli bash:

``` console
$ docker-compose run --rm php-cli
Creating network "dockerphpfpm_default" with the default driver
Creating dockerphpfpm_php-fpm_1
Creating dockerphpfpm_nginx_1
www-data@bf4eb5663c05:/usr/src/app$ ls -al
total 592
drwxr-xr-x 8 www-data www-data    510 Mar 17 13:05 .
drwxr-xr-x 3 root     root       4096 Mar 17 13:05 ..
-rw-r--r-- 1 www-data www-data    478 Mar 17 13:02 docker-compose.yml
-rw-r--r-- 1 www-data www-data    350 Mar 17 12:55 nginx.conf
-rw-r--r-- 1 www-data www-data     18 Mar 17 13:03 index.php
```

Open the nginx+php-fpm at <http://localhost:8080/>:

![Screenshot of localhost:8080](./screenshot-php.png)

## Fixing user permissions on linux

Since Docker for MacOS uses osxfs to map from the docker runtime to the osx
 filesystem, all files will be written with the hosts userid on the hots
 filesystem. This is actually not the case on linux.
 
To workaround this, we added:

```
    user: "${UID-www-data}:${GID-www-data}"
```

to the service definition in the `docker-compose.ym`. If you do this, it will
use the systems UID (e.g. 1000) or `www-data` as default user+group if `$UID`
is not set. But you can use a `.env` file with the following
content:

```text
UID=1000
GID=1000
```

and your local linux user (e.g. 1000 is the default uid on ubuntu) will own the files,
which are created in your docker container.

## LICENSE

The docker-php-fpm is copyright by Exozet (http://exozet.com) and licensed under the terms of MIT License.

