***
# 멀티 스테이지 빌드란?
멀티 스테이지 빌드는 Dockerfile 을 최적화하면서도 읽고 유지 관리하기 쉽도록 하는 데 어려움을 겪는 모든 사람에게 유용하다.

# 멀티 스테이지 빌드 사용법
멀티 스테이지 빌드를 사용하면 Dockerfile 에서 여러 개의 `FROM` 문을 사용할 수 있다. 각 `FROM` 문은 서로 다른 베이스를 사용할 수 있으며, 각 명령어는 빌드의 새로운 단계를 시작한다. 한 단계에서 다른 단계로 아티팩트를 선택적으로 복사할 수 있으며 최종 이미지에 필요한 것만 남길 수 있다.

다음 Dockerfile 은 두 개의 분리된 단계를 포함하고 있다. 하나는 바이너리를 빌드하는 단계이고, 다른 하나는 첫 번재 단계에서 바이너리를 복사하는 단계입니다.

```dockerfile
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
# RUN go build -o /bin/hello ./main.go
RUN go build -o /bin/hello ./main.go && rm -rf /usr/local/go/pkg/* /src/*


FROM scratch
COPY --from=0 /bin/hello /bin/hello
CMD ["/bin/hello"]

```

Dockerfile 하나만 필요하며 별도의 빌드 스크림트가 필요하지 않습니다. 그냥 `docker build` 명령어를 실행하면 됩니다.

```
docker build -t hello .
```
결과적으로 최종 이미지에는 빌드된 바이너리만 들어있는 작은 프로덕션 이미지가 생성됩니다. 애플리케이션을 빌드하는 데 필요한 도구들은 최종 이미지에 포함되지 않습니다.

## 검증
멀티 스테이징 빌드가 필요한 것만 남길 수 있다면 싱글 스테이지 빌드보다 경량화 된 모습이 확인되어야 합니다. 싱글 스테이지 빌드 환경을 구성해서 비교해봅시다.

``` dockerfile
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

```

```
docker build -t hello2 .
```

![[Pasted image 20240919180924.png]]
위 사진과 같이 멀티 스테이지 빌드는 2.13MB 를 차지하고, 싱글 스테이지 빌드는 869MB 를 차지하는 것을 볼 수 있다.


## 어떻게 작동하는가?
두 번째 `FROM` 지시문은 `scratch` 이미지를 베이스로 새로운 빌드 스테이지를 시작합니다. `COPY --from=0` 은 이전 스테이지에서 빌드된 아티팩트만 현재 스테이지로 복사합니다. Go SDK 와 중간 아티팩트들은 최종 이미지에 저장되지 않고 남겨집니다.

## 빌드 스테이지에 이름 지정하기
기본적으로 스테이지는 이름이 없으며, 첫 번째 `FROM` 지시문을 0부터 시작하는 정수로 참조합니다. 그러나 `FROM` 지시문에 `AS <NAME>` 을 추가하여 스테이지에 이름을 지정할 수 있습니다. 다음 예제는 이전 예제를 개선하여 스테이지에 이름을 지정하고, `COPY` 지시문에서 이름을 사용합니다. 이렇게 하면 Dockerfile 의 지시문 순서가 나중에 변경되더라도 `COPY` 는 문제없이 작동합니다.

``` dockerfile
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
```
## 특정 빌드 스테이지에서 중지하기
이미지를 빌드할 때, 반드시 Dockerfile 의 모든 스테이지를 빌드할 필요는 없습니다. 목표 빌드 스테이지를 지정할 수 있습니다. 다음 명령은 이전 Dockerfile 을 사용하면서 `build` 로 이름 지정된 스테이지에서 멈춥니다.

```
docker build --target build -t hello .
```

이 기능이 유용한 몇 가지 시나리오:
- 특정 빌드 스테이지를 디버깅할 때
- 모든 디버그 심볼 또는 도구를 활성화한 디버그 스테이지와, 간소화된 프로덕션 스테이지를 사용할 때
- 테스트 데이터를 채워 넣는 테스트 스테이지를 사용하면서, 실제 데이터를 사용하는 프로덕션 스테이지를 사용해 빌드할 때
## 외부 이미지를 스테이지로 사용하기
멀티 스테이지 빌드를 사용할 때, Dockerfile 에서 생성한 스테이지에만 국한되지 않습니다. `COPY --from` 지시문을 사용하여 별도의 이미지에서 복사할 수 있습니다. 로컬 이미지 이름, 로컬 또는 Docker 레지스트리에서 사용할 수 있는 태그 또는 태그 ID 를 사용할 수 있습니다. 필요한 경우 Docker 클라이언트는 이미지를 가져와 그곳에서 아티팩트를 복사합니다. 구문은 다음과 같습니다.

```dockerfile
COPY --from=nginx:latest /etc/nginx/nginx.conf /nginx.conf
```

첫 예제에서도 Go SDK  를 외부 이미지로 사용하는 모습을 볼 수 있습니다.
```dockerfile
FROM golang:1.23
```

## 이전 스테이지를 새 스테이지로 사용하기
`FROM` 지시문을 사용할 때 이전 스테이지가 중단된 곳에서 다시 시작할 수 있습니다. 예를 들어
```dockerfile
# syntax=docker/dockerfile:1

FROM alpine:latest AS builder
RUN apk --no-cache add build-base

FROM builder AS build1
COPY source1.cpp source.cpp
RUN g++ -o /binary source.cpp

FROM builder AS build2
COPY source2.cpp source.cpp
RUN g++ -o /binary source.cpp

```

이렇게하면 builder 를 기준으로 build1 과 build2 가 각각 빌드할 수 있게 됩니다.

## 레거시 빌더와 BuildKit 의 차이점
레거시 Docker 엔진 빌더는 선택한 `--target` 에 이르는 모든 스테이지를 처리합니다. 이는 선택한 대상이 해당 스테이지에 의존하지 않아도 스테이지를 빌드하는다는 의미입니다.

BuildKit 은 대상 스테이지에 의존하는 스테이지만 빌드합니다.

다음 Dockerfile 을 예로 들어 보겠습니다.
```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu AS base
RUN echo "base"

FROM base AS stage1
RUN echo "stage1"

FROM base AS stage2
RUN echo "stage2"
```

BuildKit 이 활성화된 경우, 이 Dockerfile 에서 `stage2` 대상 빌드 시 `base` 와 `stage2` 만 처리됩니다. `stage1` 에 대한 의존성이 없으므로 생략됩니다.
```bash
DOCKER_BUILDKIT=1 docker build --no-cache -f Dockerfile --target stage2 .
```

```bash
$ DOCKER_BUILDKIT=1 docker build --no-cache -f Dockerfile --target stage2 .
#0 building with "default" instance using docker driver

#1 [internal] load build definition from Dockerfile
#1 transferring dockerfile: 189B done
#1 DONE 0.0s

#2 resolve image config for docker-image://docker.io/docker/dockerfile:1
#2 DONE 1.6s

#3 docker-image://docker.io/docker/dockerfile:1@sha256:865e5dd094beca432e8c0a1d5e1c465db5f998dca4e439981029b3b81fb39ed5
#3 CACHED

#4 [internal] load metadata for docker.io/library/ubuntu:latest
#4 DONE 0.0s

#5 [internal] load .dockerignore
#5 transferring context:
#5 transferring context: 2B done
#5 DONE 0.0s

#6 [base 1/2] FROM docker.io/library/ubuntu:latest
#6 DONE 0.0s

#7 [base 2/2] RUN echo "base"
#7 0.323 base
#7 DONE 0.4s

#8 [stage2 1/1] RUN echo "stage2"
#8 0.267 stage2
#8 DONE 0.3s

#9 exporting to image
#9 exporting layers 0.1s done
#9 writing image sha256:bf74e2e21b477d89652cb54a01ac0d110dcf66984d28a7935c1388975d6fdb3c done
#9 DONE 0.1s

View build details: docker-desktop://dashboard/build/default/default/leeq2808la11op7rrkphw6q2t

What's Next?
  View a summary of image vulnerabilities and recommendations → docker scout quickview
```

반면, BuildKit 없이 동일한 대상을 빌드하면 모든 스테이지가 처리됩니다. 그리고 레거시 빌더가 deprecated 되었다는 알림을 볼 수 있습니다.

```bash
DOCKER_BUILDKIT=0 docker build --no-cache -f Dockerfile --target stage2 .
```

```bash
$ DOCKER_BUILDKIT=0 docker build --no-cache -f Dockerfile --target stage2 .
DEPRECATED: The legacy builder is deprecated and will be removed in a future release.
            BuildKit is currently disabled; enable it by removing the DOCKER_BUILDKIT=0
            environment-variable.

Sending build context to Docker daemon  2.048kB
Step 1/6 : FROM ubuntu AS base
 ---> edbfe74c41f8
Step 2/6 : RUN echo "base"
 ---> Running in 65b6aa46a6b6
base
 ---> Removed intermediate container 65b6aa46a6b6
 ---> 6de405e6b173
Step 3/6 : FROM base AS stage1
 ---> 6de405e6b173
Step 4/6 : RUN echo "stage1"
 ---> Running in 8a05cb170ab1
stage1
 ---> Removed intermediate container 8a05cb170ab1
 ---> 07b3d9163bb5
Step 5/6 : FROM base AS stage2
 ---> 6de405e6b173
Step 6/6 : RUN echo "stage2"
 ---> Running in 51544f6ad948
stage2
 ---> Removed intermediate container 51544f6ad948
 ---> b8a5a5355925
Successfully built b8a5a5355925
SECURITY WARNING: You are building a Docker image from Windows against a non-Windows Docker host. All files and directories added to build context will have '-rwxr-xr-x' permissions. It is recommended to double check and reset permissions for sensitive files and directories.

What's Next?
  View a summary of image vulnerabilities and recommendations → docker scout quickview
```

레거시 빌더는 `stage1` 의 빌드를 거치고 `stage2` 에 진입하는 차이가 있습니다.

그렇다면 기본 빌드는 BuildKit 으로 빌드하고 있을까요?

```bash
docker build --no-cache -f Dockerfile --target stage2 .
```

아래 처럼 `stage1` 이 동작하지 않는 점과 BuildKit 을 명시한 경우와 동일한 결과라는 점을 통해 BuildKit 으로 동작함을 확인할 수 있습니다.

```bash
$ docker build --no-cache -f Dockerfile --target stage2 .
#0 building with "default" instance using docker driver

#1 [internal] load build definition from Dockerfile
#1 transferring dockerfile: 189B done
#1 DONE 0.0s

#2 resolve image config for docker-image://docker.io/docker/dockerfile:1
#2 DONE 1.6s

#3 docker-image://docker.io/docker/dockerfile:1@sha256:865e5dd094beca432e8c0a1d5e1c465db5f998dca4e439981029b3b81fb39ed5
#3 CACHED

#4 [internal] load metadata for docker.io/library/ubuntu:latest
#4 DONE 0.0s

#5 [internal] load .dockerignore
#5 transferring context:
#5 transferring context: 2B done
#5 DONE 0.0s

#6 [base 1/2] FROM docker.io/library/ubuntu:latest
#6 CACHED

#7 [base 2/2] RUN echo "base"
#7 0.278 base
#7 DONE 0.3s

#8 [stage2 1/1] RUN echo "stage2"
#8 0.272 stage2
#8 DONE 0.3s

#9 exporting to image
#9 exporting layers 0.0s done
#9 writing image sha256:5fe18f4ced90a7932db7b35bcb350710174d5c6bd39f73397e89a47ba6f6f486 done
#9 DONE 0.1s

View build details: docker-desktop://dashboard/build/default/default/vw8lnc0g1wez01bwkanyy582s

What's Next?
  View a summary of image vulnerabilities and recommendations → docker scout quickview
```