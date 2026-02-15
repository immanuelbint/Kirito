# Bimasakti Devops Engineer Test

## Requirement

- Linux
- Golang
- Docker
- K8s cluster

## Prerequisite

1. Download Golang tarball latest version.

    ```bash
    wget https://go.dev/dl/go1.26.0.linux-amd64.tar.gz
    ```

2. Export and add Golang binary to bash_profile

    ```bash
    export PATH=$PATH:/usr/local/go/bin
    ```

3. Verify golang version

    ```bash
    [abint@l2-hostmaster-0 Kirito]# go version
    go version go1.26.0 linux/amd64
    ```

## Practical test

1. Make a Go project and install Gin on it. Make only one route on path root ( / ) and return
{‘msg’: ‘Hello World’} with header Content-Type: application/json. Then expose to port 8080, give the name of your project as “Kirito”

    ```bash
    mkdir Kirito
    ```

- Create main.go that return msg 'Hello World' inside Kirito folder

    ```go
    package main

    import (
    "net/http"

    "github.com/gin-gonic/gin"
    )

    func main() {
    // Create a Gin router with default middleware (logger and recovery)
    r := gin.Default()

    // Define a simple GET endpoint
    r.GET("/", func(c *gin.Context) {
        // Return JSON response
        c.JSON(http.StatusOK, gin.H{
        "message": "Hello World",
        })
    })

    // Start server on port 8080 (default)
    // Server will listen on 0.0.0.0:8080 (localhost:8080 on Windows)
    r.Run()
    }
    ```

- initialize Kirito

    ```bash
    [root@l2-hostmaster-0 Kirito]# go mod init Kirito
    go: creating new go.mod: module Kirito
    go: to add module requirements and sums:
            go mod tidy
    ```

    download gin and other required dependency

    ```bash
    go get -u
    ```

- compile the main.go to binary using `go build`
    ```bash
    [root@l2-hostmaster-0 Kirito]# go build -o kirito ./main.go
    [root@l2-hostmaster-0 Kirito]# ls -lrth
    total 20M
    -rw-r--r--. 1 root root  473 Feb 13 08:25 main.go
    -rw-r--r--. 1 root root 7.3K Feb 15 02:39 go.sum
    -rw-r--r--. 1 root root 1.6K Feb 15 02:39 go.mod
    -rwxr-xr-x. 1 root root  20M Feb 15 02:40 kirito
    ```

- run the kirito binary and check either via browser or curl command to make sure it's work

    ```bash
    [root@l2-hostmaster-0 Kirito]# ./kirito
    [GIN-debug] [WARNING] Creating an Engine instance with the Logger and Recovery middleware already attached.

    [GIN-debug] [WARNING] Running in "debug" mode. Switch to "release" mode in production.
    - using env:   export GIN_MODE=release
    - using code:  gin.SetMode(gin.ReleaseMode)

    [GIN-debug] GET    /                         --> main.main.func1 (3 handlers)
    [GIN-debug] [WARNING] You trusted all proxies, this is NOT safe. We recommend you to set a value.
    Please check https://github.com/gin-gonic/gin/blob/master/docs/doc.md#dont-trust-all-proxies for details.
    [GIN-debug] Environment variable PORT is undefined. Using port :8080 by default
    [GIN-debug] Listening and serving HTTP on :8080
    ```

    ```bash
    [root@l2-hostmaster-0 kirito]# curl -kv http://localhost:8080
    * About to connect() to localhost port 8080 (#0)
    *   Trying localhost...
    * Connected to localhost (localhost) port 8080 (#0)
    > GET / HTTP/1.1
    > User-Agent: curl/7.29.0
    > Host: localhost:8080
    > Accept: */*
    >
    < HTTP/1.1 200 OK
    < Content-Type: application/json; charset=utf-8
    < Date: Sun, 15 Feb 2026 02:47:38 GMT
    < Content-Length: 25
    <
    * Connection
    {"message":"Hello World"}
    ```

2. Make a Dockerfile from your project. The Dockerfile should have a multi stage build.
First stage must be compile process using Go official image and the second stage using
Alpine image running the binary of compiled project

- create Dockerfile that has multi stage build

    ```
    # syntax=docker/dockerfile:1
    FROM golang:alpine AS builder
    WORKDIR /src
    COPY <<EOF ./main.go
    package main

    import (
    "net/http"

    "github.com/gin-gonic/gin"
    )

    func main() {
    // Create a Gin router with default middleware (logger and recovery)
    r := gin.Default()

    // Define a simple GET endpoint, and returns hello world message in JSON format
    r.GET("/", func(c *gin.Context) {
        // Return JSON response
        c.JSON(http.StatusOK, gin.H{
        "message": "Hello World",
        })
    })

    // Start server on port 8080 (default)
    // Server will listen on 0.0.0.0:8080
    r.Run()
    }
    EOF
    RUN go mod init Kirito
    RUN go get -u
    RUN go build -o /bin/kirito ./main.go

    # Using Alpine image to run Go binary
    FROM alpine:3.14

    WORKDIR /

    COPY --from=builder /bin/kirito /go/kirito

    ENV PORT=8080
    ENV GIN_MODE=release
    EXPOSE 8080

    WORKDIR /go

    # Run the Go Gin binary.
    ENTRYPOINT ["/go/kirito"]
    ```

- push image do public docker registry

    ```
    docker login
    docker push imanbin/kirito:v1
    ```

3. Make a Kubernetes manifest to run and expose your image to the public. I'm going to use kind deployment and service.

- create deployment.yaml
    ```yaml
    # first version copied from
    # https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: kirito-go
    labels:
        app: kirito-go
    spec:
    replicas: 1
    selector:
        matchLabels:
        app: kirito-go
    template:
        metadata:
        labels:
            app: kirito-go
        spec:
        containers:
            - name: kirito-go
            image: imanbin/kirito:v1
            ports:
                - containerPort: 8080
    ```


- create service.yaml
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: kirito-go
    labels:
        app: kirito-go
    spec:
    ports:
        - port: 8080
    selector:
        app: kirito-go
    ```

4. I've include the `.gitlab-ci.yaml` file and the kubeconfig dummy in this repostory.
