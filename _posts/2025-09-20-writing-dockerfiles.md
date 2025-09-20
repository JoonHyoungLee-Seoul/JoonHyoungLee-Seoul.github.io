---
layout: single
title: "[Docker] 2.2 Writing Dockerfiles" 
date: 2025-09-20
categories: Docker
header:
  overlay_image: /assets/images/docker-container-tech.svg
  overlay_filter: 0.5
  teaser: /assets/images/docker-container-tech.svg
  #caption: "Create Docker Image"
Typora-root-url: ../
---

# Writing Dockerfiles

A Dockerfile is a simple script for packaging applications. Dockerfiles consist of instructions, and executing them results in the creation of Docker images.

The Dockerfile syntax is very flexible - commonly used tasks have dedicated commands, and you can write custom tasks directly using standard shell syntax (bash shell on Linux or PowerShell on Windows).

Here's the complete Dockerfile script for packaging the web-ping application:

```bash
From diamol/node

ENV TARGET="blog.sixeyed.com"
ENV METHOD="HEAD"
ENV INTERVAL="3000"

WORKDIR /web-ping
COPY app.js .

CMD ["node", "/web-ping/app.js"]
```

The instructions `FROM`, `ENV`, `WORKDIR`, `COPY`, and `CMD` used in this Dockerfile script are written in uppercase, but lowercase is also acceptable. Here's an explanation of each instruction:

- `FROM`: Every image starts from another image. This image specifies `diamol/node` as the starting point. The `diamol/node` image has Node.js installed, which is the runtime needed to run the `web-ping` application.

- `ENV`: An instruction for setting environment variable values. This script sets three environment variables.

- `WORKDIR`: An instruction that creates a directory in the container image filesystem and sets it as the working directory.

- `COPY`: An instruction that copies files or directories from the local filesystem to the container image. It follows the format (source path) (destination path).

- `CMD`: An instruction that specifies the command to execute when Docker runs a container from the image. In this script, it specifies `app.js` for the Node.js runtime to start the application.

With just these five instructions, you can package most applications with Docker.

Let's verify that all files needed to build an image are available using the GitHub example code by running the following command:

```bash
cd ch03/exercises/web-ping
ls
```

Output:
```bash
app.js Dockerfile README.md
```

With just these three files, you can build the image for the web-ping application.

## Next Steps

In the next post, we'll learn how to build a `container image`.

## Sources

- https://github.com/gilbutITbook/080258
- Learn Docker in a Month of Lunches by Elton Stoneman
  - [Amazon](https://www.amazon.com/-/ko/Elton-Stoneman/e/B0759TFV4F/ref=dp_byline_cont_book_1)
  - [PDF](https://pdfcoffee.com/learn-docker-month-lunches-4-pdf-free.html)
  - [Youtube](https://www.youtube.com/@EltonStoneman/playlists)