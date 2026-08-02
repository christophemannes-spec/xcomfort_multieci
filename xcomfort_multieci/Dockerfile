ARG BUILD_FROM
FROM golang:1.24-alpine AS builder

RUN apk add --no-cache --update gcc musl-dev linux-headers pkgconfig libusb-dev

WORKDIR /src
COPY src/ .

RUN go build -o /xcomfortd-go .

FROM $BUILD_FROM

ENV LANG=C.UTF-8

RUN apk add --no-cache libusb

COPY --from=builder /xcomfortd-go /usr/bin/xcomfortd-go
COPY rootfs /
RUN chmod +x /etc/services.d/xcomfort_multieci/run
