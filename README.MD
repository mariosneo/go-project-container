## **Prerequisites**

- Docker
- Github Actions build CI



## Retrieve the docker image digest prior to the build

Pulling the image using the image digest can prevent any interception during request while is pulling a docker image:

```
docker inspect --format='{{.RepoDigests}}' golang:alpine
```

The output should be similar to the one below:

```
golang@sha256:ec3acb156c891fe0aadd20ce4bec20808b2d53414b5f73e631698d0fab6b8d28
```



## Create an unprivileged user:group to run the go app

```
# Create unprivileged user 
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    "${USER}"
.
.
.

#run the go app as unprivileged user (appuser)    
USER appuser:appuser

```



## Use golang:alpine image as builder

Use golang:alpine image as builder and then copy over the application binary files to the scratch image. This will reduce the size of the final image:

```
FROM golang@sha256:ec3acb156c891fe0aadd20ce4bec20808b2d53414b5f73e631698d0fab6b8d28 AS builder

.
.
.

WORKDIR $GOPATH/src/mypackage/myapp/
COPY . .

.
.
.

FROM scratch

# Import from builder.
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group


COPY --from=builder /go/bin/main /go/bin/main
```



## Reduce the image size further 

It can be optimized further by removing debug informations and compile only for linux target and disabling cross compilation.

```
WORKDIR $GOPATH/src/mypackage/myapp/
COPY . .

RUN go env -w GO111MODULE=on

RUN go get -d -v

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags='-w -s -extldflags "-static"' -a \
    -o /go/bin/main .
```



### Build the final image on GitHub Actions

Build and push the final image upon commit with Github Actions

```
name: Docker Image CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - 
        name: Build the Docker image
      -
        name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - 
        uses: actions/checkout@v2
      - 
        name: Build the Docker image
        run: docker build -t mneorepo/go-project .
      - 
        name: Push Docker Image
        run: docker push mneorepo/go-project

```



 **The final size should be approximately 8MB**