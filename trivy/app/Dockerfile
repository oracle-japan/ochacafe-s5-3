#Stage-1
FROM arm64v8/golang:1.17.7 as builder
COPY ./main.go ./
RUN go build -o /sample-go-app ./main.go

#Stage-2
FROM gcr.io/distroless/base
EXPOSE 8080
COPY --from=builder /sample-go-app /.
ENTRYPOINT ["./sample-go-app"]
