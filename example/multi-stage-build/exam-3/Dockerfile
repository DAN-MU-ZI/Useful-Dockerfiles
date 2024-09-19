# syntax=docker/dockerfile:1
# 빌드 스테이지 네이밍 -> AS build
FROM golang:1.23 AS build
WORKDIR /src
COPY <<EOF /src/main.go
package main

import "fmt"

func main() {
  fmt.Println("hello, world")
}
EOF
RUN go build -o /bin/hello ./main.go

FROM scratch
# 빌드 스테이지 이름 참고 -> --from=build
COPY --from=build /bin/hello /bin/hello
CMD ["/bin/hello"]
