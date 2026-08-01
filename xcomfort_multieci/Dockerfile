ARG BUILD_FROM
FROM golang:1.24-bookworm AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    libusb-1.0-0-dev pkg-config && rm -rf /var/lib/apt/lists/*

WORKDIR /src
COPY src/ .

RUN go build -o /xcomfortd-go .

FROM $BUILD_FROM

ENV LANG=C.UTF-8

RUN apk add --no-cache libusb

COPY --from=builder /xcomfortd-go /usr/bin/xcomfortd-go
COPY rootfs /
RUN chmod +x /etc/services.d/xcomfort_multieci/run
