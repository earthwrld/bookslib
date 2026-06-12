FROM golang:1.22-alpine AS base
WORKDIR /app
COPY . .
RUN go mod tidy

FROM base AS test
RUN go test -v ./...

FROM base AS build
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

FROM alpine:latest AS production
RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001 -G appgroup
WORKDIR /app
COPY --from=build --chown=appuser:appgroup /app/main .
USER appuser
EXPOSE 8081
CMD ["./main"]