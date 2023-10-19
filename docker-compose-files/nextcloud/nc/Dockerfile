FROM nextcloud:27.1.2-apache

RUN set -ex; \
    apt-get update; \
    apt-get install -y \
        ffmpeg \
    ; \
    rm -rf /var/lib/apt/lists/*;
