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

see [docker-compose-files/nextcloud/](docker-compose-files/nextcloud/)

TODO: explain the files in detail
