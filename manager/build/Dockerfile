FROM python:3.11-alpine3.21

RUN apk add --update --no-cache \
    iproute2 \
    libffi-dev \
    build-base \
    docker-cli \
    docker-cli-compose \
    bridge-utils \
    && rm -rf /tmp/* /var/tmp/* /var/cache/apk/* /var/cache/distfiles/*

COPY ./requirements.txt /requirements.txt
RUN pip install --no-cache-dir -r /requirements.txt
