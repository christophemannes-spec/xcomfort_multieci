ARG BUILD_FROM
FROM golang:1.24-bookworm AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    libusb-1.0-0-dev pkg-config && rm -rf /var/lib/apt/lists/*

WORKDIR /src
COPY src/ .

# GOFLAGS/GOPROXY left at defaults here: this stage builds on the CI/build
# host, which normally has full internet access. If you ever need to build
# in a similarly network-restricted environment, the same workaround used
# during development also works here:
#   GOPROXY=direct GOSUMDB=off go mod tidy
# (go.mod already carries `replace` directives for golang.org/x/net,
#  golang.org/x/sync and golang.org/x/text pointing at their github.com/golang
#  mirrors, needed only when golang.org itself is unreachable to resolve the
#  vanity-import redirect.)
RUN go build -o /xcomfortd-go .

FROM $BUILD_FROM

ENV LANG=C.UTF-8

RUN apk add --no-cache libusb

COPY --from=builder /xcomfortd-go /usr/bin/xcomfortd-go
COPY rootfs /

CMD [ "/init" ]
