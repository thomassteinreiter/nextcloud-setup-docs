# Nextcloud Setup Documentation

## Disclaimer

This is a work in progress and currently very, very incomplete.


## Introduction

This is a documentation project for a [Nextcloud](https://nextcloud.com) setup. I invested a lot of time to get Nextcloud running the way I needed it to. I learned a lot on the way. This documentation contains the knowledge that is most helpful to me and most likely to others as well.


## Basic Architecture and Technologies

* [Ubuntu Server](https://ubuntu.com/download/server)
* [Apache HTTP Server](https://httpd.apache.org/) (as Reverse-Proxy/Gateway)
  * [Let's encrypt SSL](https://certbot.eff.org/instructions) (EFF certbot)
* [Docker](https://www.docker.com/get-started/)
  * [Nextcloud](https://hub.docker.com/_/nextcloud/)
  * [Redis](https://hub.docker.com/_/redis)
  * [MariaDB](https://hub.docker.com/_/mariadb)
  * [Elastic](https://hub.docker.com/_/elasticsearch)


### Architecture Motivation

There is a lot of documentation for Ubuntu Server available and I am already used to Ubuntu.

Regarding Docker: Nextcloud is a complex piece of software requiring countless dependencies. Technically, you can install and maintain (update) everything yourself, but with Docker, everything needed is packaged in an Docker image and there are less possibilites for stuff going wrong. Docker, in a certain way, decouples the OS maintenance (update/upgrade, backup/restore) from the Nextcloud software and data.

It is possible to directly use the nextcloud container, but I choose to put an Apache Web Server in front of for the following reasons: I have already an Apache Web Server running for different usecases, I am already used to Apache Web Server and I found it hard to get the Let's Encrypt certbot renewal working inside Docker. This is why I am using the Apache Web Server as an "Reverse Proxy" to forward the requests to the Docker. I then can use the "standard way" of setting up Let's Encrypt, which is good documented.

Nextcloud can (in an minimal setup) operate with an embedded Database, but since I am going for a setup with multiple Nextcloud users, I am using a dedicated MariaDB as database and Redis for session/locking handling.

Nextcloud has an integrated search feature, but this is limited to the names of files and folders, which is fine most of the time, but files are not named well and you have no idea where to look. Full text search is a game-changer in those situations. I set up Nextcloud with Elastic with full text search.


## Apache Web Server Setup

The following setup is compiled from the following sources:
 * https://docs.nextcloud.com/server/latest/admin_manual/office/example-docker.html#install-the-apache-reverse-proxy
 * https://docs.nextcloud.com/server/latest/admin_manual/configuration_files/big_file_upload_configuration.html#apache-with-mod-proxy-fcgi
 * https://docs.nextcloud.com/server/latest/admin_manual/installation/harden_server.html#enable-http-strict-transport-security


```
apt install apache2 # install web server

a2enmod proxy # install requirements for reverse-proxy
a2enmod proxy_http
a2enmod headers
a2enmod remoteip
```

Install SSL certificates. Follow: https://certbot.eff.org/

Create config file
```
/etc/apache2/conf-enabled/1-nextcloud.conf
```
with contents:
```
ProxyPreserveHost On
AllowEncodedSlashes NoDecode

# support slow/large uploads/downloads
ProxyTimeout 3600

# support large uploads via the sync clients
SetEnv proxy-sendchunked 1

# main reverse proxy rules
ProxyPass "/"  "http://127.0.0.1:11000/" nocanon
ProxyPassReverse "/"  "http://127.0.0.1:11000/"

# enable websocket proxying
RewriteCond %{HTTP:Upgrade} websocket [NC]
RewriteCond %{HTTP:Connection} upgrade [NC]
RewriteCond %{THE_REQUEST} "^[a-zA-Z]+ /(.*) HTTP/\d+(\.\d+)?$"
RewriteRule .? "ws://localhost:11000/%1" [P,L]

# Enable h2, h2c and http1.1
Protocols h2 h2c http/1.1

# hardening
TraceEnable off
<Files ".ht*">
  Require all denied
</Files>

# Support big file uploads
LimitRequestBody 0

# SSL hardening
Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
```
Enable the conf with:
```
a2enconf 1-nextcloud
```

To get WebDAV working, the following changes needed to be done, unfortunately directly in config file that contains the `<VirualHost *:443>`. For me that is `/etc/apache2/sites-enabled/000-default-le-ssl.conf`

Put this inside the `<VirualHost *:443>` block:
```
RewriteEngine On
RewriteRule ^/\.well-known/carddav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
RewriteRule ^/\.well-known/caldav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
```

## Docker Setup

### Install Docker Engine

Since I want to use the latest Docker images and feature, I am not using Docker from the default ubuntu repository, but use the official Docker repository instead.

https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

```
# https://docs.docker.com/engine/install/ubuntu/#install-using-the-convenience-script
# Not recommended for production systems
curl -fsSL https://get.docker.com -o get-docker.sh
sh ./get-docker.sh

apt install docker-compose
```

### Install Docker Compose
Docker Compose handles the downloading of Docker images, creating of containers, networks, volumes, starting and stopping of containers and so on. It is a very powerful tool that makes working with multiple Docker container that need to be connected together very easy. But since it can and will remove containers and volumes, it is very easy to accidentally delete all your data, be careful and always make backups before making changes on production systems.

```
apt install docker-compose
```

## Docker-Compose Setup

see [docker-compose-files/nextcloud/](docker-compose-files/nextcloud/) for the complete directory.


### [docker-compose.yml](docker-compose-files/nextcloud/docker-compose.yml)

#### database (db) section
```
...
  db:
    image: mariadb:11.1.2
    restart: always
    command: --log-bin=binlog --binlog-format=ROW
    volumes:
      - nextcloud_db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=myMySQLRootPassword
      - MYSQL_PASSWORD=mySqlPassword
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
...
```

The `command: --log-bin=binlog --binlog-format=ROW` command line arguments are required. see: https://docs.nextcloud.com/server/27/admin_manual/installation/system_requirements.html#database-requirements-for-mysql-mariadb

#### nextcloud (app) section
```
  app:
    build:
      context: nc
    image: custom-nextcloud:27.1.2-apache
    container_name: nextcloud_app_1
    restart: always
    depends_on:
      - db
      - redis
      - elastic
    ports:
      - 11000:80
    volumes:
      - nextcloud_data:/var/www/html
      - ./previews.config.php:/var/www/html/config/previews.config.php:ro
      - ./remoteip.conf:/etc/apache2/conf-available/remoteip.conf:ro
    environment:
      - PHP_MEMORY_LIMIT=1024M
      - NEXTCLOUD_TRUSTED_DOMAINS=localhost,myserver.mydomain.org
      - TRUSTED_PROXIES="localhost 192.168.1.2"
      - OVERWRITEHOST=myserver.mydomain.org
      - OVERWRITEPROTOCOL=https
      - MYSQL_PASSWORD=mySqlPassword
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db
      - REDIS_HOST=redis
```
##### custom build
this points to a Dockerfile in subdirectory [nc](docker-compose-files/nextcloud/nc)
```
    build:
      context: nc
    image: custom-nextcloud:27.1.2-apache
    container_name: nextcloud_app_1
```
additionally, set the `image` and `container_name` name. Since this is a custom build, it is good to have a recognizable image name. (e.g. `docker ps`). There seems to be a inconsistency in Docker-Compose with the auto-naming of the container, depending if it is a "initial" start of the container or a "update" of a running container. e.g. `nextcloud_app_1` vs. `nextcloud-app-1`. This can be problematic if we want to run commands automatically that require the container name. (e.g. cron) To workaround this, we can explicitly set the `container_name`.

##### increase memory
The default PHP memory value is 512MB. I ran into issues with thumbnail generation for a hand full of larger images. Increasing to 1024MB (`- PHP_MEMORY_LIMIT=1024M`) fixed this for me.

##### proxy config
Nextcloud has a automatic system and security check here: `https://myserver.mydomain.org/settings/admin/overview` that complains about several problems in the beginning that come from running Nextcloud behind a reverse proxy. To get a proper setup, the following properties must be set:
```
      - NEXTCLOUD_TRUSTED_DOMAINS=localhost,myserver.mydomain.org
      - TRUSTED_PROXIES="localhost 192.168.1.2"
      - OVERWRITEHOST=myserver.mydomain.org
      - OVERWRITEPROTOCOL=https
```

#### redis section
```
  redis:
    image: redis:6.2.13
    restart: always
    command: ["--databases", "1"]
    volumes:
      - nextcloud_redis:/data
```
the one adjustment is `command: ["--databases", "1"]`. Redis has 16 databases per default to improve performance. Since I run this setup only a few users and the hardware is not very powerful, I setting this to the minimal amount to save resources.

#### elastic section
```
  elastic:
    build:
      context: elastic
    image: custom-elasticsearch:8.10.2
    restart: always
    volumes:
      - nextcloud_elastic_data:/usr/share/elasticsearch/data
      - nextcloud_elastic_backup:/usr/share/elasticsearch/backup
      - nextcloud_elastic_logs:/usr/share/elasticsearch/logs
    environment:
      - "discovery.type=single-node"
      - "xpack.security.enabled=false"
      - "ES_JAVA_OPTS=-Xms2048m -Xmx2048m"
```

##### custom build
this points to a Dockerfile in subdirectory [elastic](docker-compose-files/nextcloud/nc)
```
    build:
      context: elastic
    image: custom-elasticsearch:8.10.2
```

##### elastic setup
```
    environment:
      - "discovery.type=single-node"
      - "xpack.security.enabled=false"
      - "ES_JAVA_OPTS=-Xms2048m -Xmx2048m"
```
We only need elastic on one node: `- "discovery.type=single-node"`. Since elastic version 8, authentication and other security featues are enabled per default, which complicates the setup. In my case, elastic runs inside an internal docker network (automatically created by docker-compose) so I can disable authentication to make the setup easier. The line `- "ES_JAVA_OPTS=-Xms2048m -Xmx2048m"` sets the max memory of elastic to 2 GiB

TODO: explain the files in detail

TODO: cron, thumbnailing,
