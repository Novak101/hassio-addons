# ARG declared before the first FROM is globally scoped, so it's available
# to populate the *second* stage's `FROM $BUILD_FROM` below -- an ARG
# declared partway through the file (as this originally was) only scopes to
# the one stage it's declared in and can't be used in a later FROM line.
ARG BUILD_FROM

# Stage 1: build the donetick backend from source (Novak101/donetick fork)
# instead of downloading upstream's prebuilt release binary, which doesn't
# contain the per-chore icon feature or the local-storage URL-signing fix.
# --platform=$BUILDPLATFORM pins this stage to the host (native) architecture
# even when building for other target platforms, so `go build` cross-compiles
# natively via GOOS/GOARCH instead of running (much slower) under emulation --
# same approach donetick's own upstream Dockerfile uses.
FROM --platform=$BUILDPLATFORM golang:1.24-alpine AS builder

WORKDIR /usr/src/app
RUN apk --no-cache add ca-certificates git

RUN git clone --depth 1 https://github.com/Novak101/donetick.git .

RUN go mod download && go mod verify

ARG TARGETOS
ARG TARGETARCH
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -ldflags="-s -w" -buildvcs=false -o /donetick .

# Stage 2: the actual add-on image, based on the Home Assistant base image
# (s6-overlay/bashio, required for Supervisor add-ons) -- this is NOT where
# the Go build happens, just where the compiled binary ends up running.
ARG BUILD_FROM
FROM $BUILD_FROM

WORKDIR /app

RUN mkdir -p /app/core
COPY --from=builder /donetick /app/core/donetick
COPY --from=builder /usr/src/app/config /app/core/config
RUN chmod +x /app/core/donetick

COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

EXPOSE 2021

CMD ["/app/entrypoint.sh"]
