---
layout:     post
title:      Dockerize your Golang App the Right way
date:       2019-11-29 20:08:00
summary:    Multiple approaches w/ method to slim down image size
categories: go docker container
---

Nowadays turning your applications into Docker images is a much more appreciated way of distributing and deploying new apps. I'm assuming that you are quite familiar with basic concepts of Docker and why you use it. (Otherwise you won't be looking at this post ;>) We'll walk through a few different ways to do it.

### A busted exec (Do Not Try This At Home)

Say you have a nice application inside  `$GOPATH/src/github.com/sanbaispeaking/dockerizego`
```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gorilla/mux"
)

func main() {
	r := mux.NewRouter()
	r.HandleFunc("/", hello)
	http.Handle("/", r)
	fmt.Println("Up & listening on 8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func hello(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintln(w, "Hello world!")
}
```

Application builds and runs fine locally.
```sh
$ go mod init
$ go build -o main .
$ ./main
```

Wrap it up in a *Dockerfile*
```dockerfile
FROM golang:latest

WORKDIR /app
COPY main /app

EXPOSE 8080
CMD ["./main"]
```

And build a image and created a container out of it
```sh
docker build -t dockerizego:busted .
docker run --rm -p 8080:8080 dockerizego:busted
```

And the container is busted
```
standard_init_linux.go:211: exec user process caused "exec format error"
```

That is because you are trying to run a binary not built for the target runtime platform. To explain that is a bit of off-topic, and generally you just don't build go app on host and hope it to run in container.

### Build App Within image

Move the build instructions inside the Dockerfile
```dockerfile
FROM golang:latest

WORKDIR /app

COPY . /app
RUN go build -o main

EXPOSE 8080
CMD ["./main"]
```

Build it and run it again
```sh
docker build -t dockerizego:v1 .
docker run -it -p 8080:8080 --rm dockerizego:v1
```
Now you can test it with `curl localhost:8080` to see the application up and running.


### Build App Outside the image

That `exec format error` we got at the beginning of the post is due to mismatched build and target runtime platform. What if we build the App first but within a container?
```sh
docker run --rm -v "$PWD:/go/src/github.com/sanbaispeaking/dockerizego" -w "/go/src/github.com/sanbaispeaking/dockerizego" golang go build -o main
```

That long command line could use some explanation. The `-v -w` part tells docker to mount host current directory, which is `$GOPATH/src/github.com/sanbaispeaking/dockerizego`, inside a container, and use that directory as *WORKDIR*, then build your application within. You will find the binary in your current directory.

Now get back to our first *busted* Dockerfile.
```
docker build -t dockerizego:busted .
docker run -it -p 8080:8080 --rm dockerizego:busted
```
This time it turns out OK.


### Build App With Multi-stage
```dockerfile
FROM golang:latest as base

WORKDIR /app

COPY . /app
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

FROM alpine:latest as runtime

WORKDIR /app
COPY --from=base /app/main .

EXPOSE 8080
CMD ["./main"]
```

That `golang:alpine` image is roughly a alpine image with only ca-certificate, which is highly recommend when images size needs to be as small as possible. As the cost to minimize size, you don't have tools like *git* built in, and without git, go tools are mostly broken.

It is worth nothing that alpine does not utilize *glibc*, instead, it does use *musl libc*. So you may need to ensure no use of *cgo* with env set to `CGO_ENABLED=0 GOOS=linux`. Otherwise you may run into a weird looking error `standard_init_linux.go:211: exec user process caused "no such file or directory"` due to absence of *glibc*.

Attach to a running container, and `ldd` reveals the difference:
```sh
ldd main
	/lib64/ld-linux-x86-64.so.2 (0x7f86281ab000)
	libpthread.so.0 => /lib64/ld-linux-x86-64.so.2 (0x7f86281ab000)
	libc.so.6 => /lib64/ld-linux-x86-64.so.2 (0x7f86281ab000)

# with CGO_ENABLED set to 0
ldd main
	/lib/ld-musl-x86_64.so.1: main: Not a valid dynamic program
```

### Time to a conclusion

As you can see, there are multiple ways to do this, some of which are quite straight forward, and others are a bit complicated but end up with a slim image. For a real world application, the extra steps of multi-stage build would make your images hundreds MBs smaller.


#### Reference
1.[Golang Official Images](https://hub.docker.com/_/golang)

2.[Confd binary will not run in Alpine, but will run in Ubuntu #78](https://github.com/gliderlabs/docker-alpine/issues/78)