FROM alpine:latest

RUN apk add --update nginx && \
        rm -rf /var/cache/apk/* && \
        mkdir -p /tmp/nginx/ && \
        mkdir -p /run/nginx

COPY nginx.conf /etc/nginx/nginx.conf

ADD html.tar.gz /var/lib/nginx

EXPOSE 1234/tcp

ENV pueblo=Martos provincia=Jaen
ENV dni 26018864e

ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]

VOLUME /data
