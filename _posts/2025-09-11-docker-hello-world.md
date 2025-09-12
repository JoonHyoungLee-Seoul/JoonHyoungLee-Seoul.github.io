---
layout: single
title: "[Docker] 1.1 Running Hello World with Containers" 
date: 2025-09-11
categories: Docker
header:
  overlay_image: /assets/images/docker-container-tech.svg
  overlay_filter: 0.5
  teaser: /assets/images/docker-container-tech.svg
  #caption: "Docker Container Architecture"
Typora-root-url: ../
---

# Running Hello World with Containers

## Verify Docker Installation

Check if Docker is installed correctly:

```bash
docker version
```

Expected output:

```bash
Client:
 Version:           28.3.3
 API version:       1.51
 Go version:        go1.24.5
 Git commit:        980b856
 Built:             Fri Jul 25 11:33:03 2025
 OS/Arch:           darwin/arm64
 Context:           desktop-linux
  
Server: Docker Desktop 4.45.0 (203075)
 Engine:
  Version:          28.3.3
  API version:      1.51 (minimum version 1.24)
  Go version:       go1.24.5
  Git commit:       bea959c
  Built:            Fri Jul 25 11:35:32 2025
  OS/Arch:          linux/arm64
  Experimental:     false
```

## Run Hello World Container

Execute the Hello World example:

```bash
docker container run diamol/ch02-hello-diamol
```

Output:

```bash
Unable to find image 'diamol/ch02-hello-diamol:latest' locally
latest: Pulling from diamol/ch02-hello-diamol
941f399634ec: Pull complete 
93931504196e: Pull complete 
d7b1f3678981: Pull complete 
Digest: sha256:c4f45e04025d10d14d7a96df2242753b925e5c175c3bea9112f93bf9c55d4474
Status: Downloaded newer image for diamol/ch02-hello-diamol:latest
---------------------
Hello from Chapter 2!
---------------------
My name is:
b4605429addb
---------------------
Im running on:
Linux 6.10.14-linuxkit aarch64
---------------------
My address is:
inet addr:172.17.0.2 Bcast:172.17.255.255 Mask:255.255.0.0
---------------------
```

## Understanding the Command

The `docker container run` command executes an application as a container using a pre-packaged **image** named `diamol/ch02-hello-diamol`.

The initial "unable to find image locally" message indicates Docker is downloading the image before running it.

## Key Output Elements

Each container run shows unique identifiers:

- **Container Name**: `b4605429addb` (changes each run)
- **OS**: `Linux 6.10.14-linuxkit aarch64`
- **IP Address**: `172.17.0.2` (may change each run)

## Why Values Change

Running the same command again produces different output:

```bash
My name is: 308904538fd6  # Different container name
```

Each container execution creates a new, isolated instance with unique identifiers. This behavior will be explained in detail in the next section.

## Sources

- https://github.com/gilbutITbook/080258
- Learn Docker in a Month of Lunches by Elton Stoneman
  - [Amazon](https://www.amazon.com/-/ko/Elton-Stoneman/e/B0759TFV4F/ref=dp_byline_cont_book_1)
  - [PDF](https://pdfcoffee.com/learn-docker-month-lunches-4-pdf-free.html)
  - [Youtube](https://www.youtube.com/@EltonStoneman/playlists)