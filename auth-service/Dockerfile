FROM golang:1.26-alpine AS base
WORKDIR /app
COPY . .
RUN go mod tidy

FROM base AS build
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

FROM scratch AS production
WORKDIR /app
COPY --from=build /app/main .
USER 10001:10001
EXPOSE 8081
CMD ["./main"]