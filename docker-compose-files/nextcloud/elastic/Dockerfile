ARG ELASTIC_VERSION
FROM elasticsearch:${ELASTIC_VERSION}
RUN /usr/share/elasticsearch/bin/elasticsearch-plugin install --batch ingest-attachment
