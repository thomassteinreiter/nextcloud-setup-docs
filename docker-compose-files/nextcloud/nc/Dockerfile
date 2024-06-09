ARG NEXTCLOUD_VERSION
FROM nextcloud:${NEXTCLOUD_VERSION}-apache

RUN set -ex; \
    apt-get update; \
    apt-get install -y \
        ffmpeg \
    ; \
    rm -rf /var/lib/apt/lists/*;
