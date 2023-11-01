---
title: "How to define a healthcheck for a Docker container"
date: "2023-11-01T19:38:47+01:00"
tags:
  - docker
  - nginx
  - observability
---

Today I learned that you can define a health check for a Docker container,
directly on the Dockerfile.

I was already familiar with the liveness and readiness checks of Kubernetes,
but somehow I had not yet bumped into the [HEALTHCHECK statement for the Dockerfile](https://docs.docker.com/engine/reference/builder/#healthcheck).

Example for Nginx, let's say you have some HTML content in a `content` directory,
then you can write a Dockerfile like this:

    FROM nginx

    COPY content /usr/share/nginx/html

    HEALTHCHECK --interval=5s --timeout=3s \
      CMD curl -f http://localhost/?healthcheck=1 || exit 1

If you build that image and then start a container with it:

    docker build . -t simple-nginx-docker  # Build the image
    docker run --rm -p 8000:80 simple-nginx-docker  # Run a container

Then, you'll see:

1. the incoming health check requests arriving in the container logs as per the configuration, every 5 seconds:

```
127.0.0.1 - - [01/Nov/2023:18:42:49 +0000] "GET /?healthcheck=1 HTTP/1.1" 200 411 "-" "curl/7.88.1" "-"
```

2. a health status in the output of `docker ps`, like these:

```
$ docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS                            PORTS                                   NAMES
0f7b75e705c1   simple-nginx-docker   "/docker-entrypoint.…"   4 seconds ago   Up 3 seconds (health: starting)   0.0.0.0:8000->80/tcp, :::8000->80/tcp   frosty_dewdney
```

```
$ docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS                   PORTS                                   NAMES
0f7b75e705c1   simple-nginx-docker   "/docker-entrypoint.…"   9 seconds ago   Up 8 seconds (healthy)   0.0.0.0:8000->80/tcp, :::8000->80/tcp   frosty_dewdney
```

**Good to know:** to debug a failing health check command, run `docker inspect CONTAINER_ID`
