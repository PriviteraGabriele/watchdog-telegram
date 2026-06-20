FROM --platform=$BUILDPLATFORM golang:alpine AS builder

ARG TARGETOS
ARG TARGETARCH

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build \
    -ldflags="-s -w" \
    -o watchdog-telegram .

FROM alpine:3.23

LABEL maintainer="Gabriele Privitera <prigab.zucchetti@gmail.com>" \
  org.label-schema.name="Watchdog Telegram Plugin" \
  org.label-schema.vendor="thecorps" \
  org.label-schema.schema-version="1.0"

LABEL org.opencontainers.image.source=https://github.com/thecorps/watchdog-telegram
LABEL org.opencontainers.image.description="Enhanced Drone/Woodpecker CI Telegram notifications"
LABEL org.opencontainers.image.licenses=MIT

RUN apk add --no-cache ca-certificates && \
  rm -rf /var/cache/apk/*

COPY --from=builder /app/watchdog-telegram /bin/

ENTRYPOINT ["/bin/watchdog-telegram"]
