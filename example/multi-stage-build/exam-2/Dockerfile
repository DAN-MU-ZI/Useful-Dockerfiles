# syntax=docker/dockerfile:1
FROM golang:1.23

WORKDIR /src

COPY <<EOF ./main.go
package main

import "fmt"

func main() {
  fmt.Println("hello, world")
}
EOF

# 애플리케이션 빌드
RUN go build -o /bin/hello ./main.go

# 컨테이너 실행 시 명령어
CMD ["/bin/hello"]
