FROM --platform=${TARGETPLATFORM:-linux/amd64} openfaas/of-watchdog:0.8.0 as watchdog
FROM --platform=${BUILDPLATFORM:-linux/amd64} golang:1.13-alpine3.11 as build

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories
RUN apk --no-cache add git

ENV CGO_ENABLED=0

COPY --from=watchdog /fwatchdog /usr/bin/fwatchdog
RUN chmod +x /usr/bin/fwatchdog

RUN mkdir -p /go/src/handler
WORKDIR /go/src/handler
COPY . .

# Run a gofmt and exclude all vendored code.
RUN test -z "$(gofmt -l $(find . -type f -name '*.go' -not -path "./vendor/*" -not -path "./function/vendor/*"))" || { echo "Run \"gofmt -s -w\" on your Golang code"; exit 1; }

ARG GO111MODULE="on"
ARG GOPROXY="https://goproxy.cn,direct"

RUN CGO_ENABLED=${CGO_ENABLED} GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build --ldflags "-s -w" -a -installsuffix cgo -o handler .
RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} go test handler/function/... -cover

FROM --platform=${TARGETPLATFORM:-linux/amd64} alpine:3.11
# Add non root user and certs
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories
RUN apk --no-cache add ca-certificates \
    && addgroup -S app && adduser -S -g app app \
    && mkdir -p /home/app \
    && chown app /home/app

WORKDIR /home/app

COPY --from=build /go/src/handler/handler    .
COPY --from=build /usr/bin/fwatchdog         .
COPY --from=build /go/src/handler/function/  .

RUN chown -R app /home/app

USER app

ENV fprocess="./handler"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:8082"

CMD ["./fwatchdog"]
