---
layout: single
title: "[Docker] 2.3 Building Container Images" 
date: 2025-09-21
categories: Docker
header:
  overlay_image: /assets/images/docker-container-tech.svg
  overlay_filter: 0.5
  teaser: /assets/images/docker-container-tech.svg
  #caption: "Create Docker Image"
Typora-root-url: ../
---

# Building Container Images

## Building Images from Dockerfile

To build an image, you need to specify the image name and the path to the files required for packaging, in addition to the Dockerfile script.

Run `cd ch03/exercises/web-ping` to navigate to the directory containing all the necessary files, then build an image from the Dockerfile script using the following command:

```bash
docker image build --tag web-ping .
```

The argument for `--tag` (web-ping) is the image name, and the following argument (.) is called the **context**, which in this script is set to the current working directory.

Output:
```bash
[+] Building 4.4s (9/9) FINISHED                           docker:desktop-linux
 => [internal] load build definition from Dockerfile                       0.0s
 => => transferring dockerfile: 191B                                       0.0s
 => [internal] load metadata for docker.io/diamol/node:latest              4.2s
 => [auth] diamol/node:pull token for registry-1.docker.io                 0.0s
 => [internal] load .dockerignore                                          0.0s
 => => transferring context: 2B                                            0.0s
 => [1/3] FROM docker.io/diamol/node:latest@sha256:dfee522acebdfdd9964aa9  0.0s
 => => resolve docker.io/diamol/node:latest@sha256:dfee522acebdfdd9964aa9  0.0s
 => => sha256:dfee522acebdfdd9964aa9c88ebebd03a20b6dd5739 1.41kB / 1.41kB  0.0s
 => => sha256:6467efe6481aace0c317f144079c1a321b91375a828 1.16kB / 1.16kB  0.0s
 => => sha256:8e0eeb0a11b3a91cc1d91b5ef637edd153a64a3792e 5.66kB / 5.66kB  0.0s

 => [internal] load build context                                          0.0s
 => => transferring context: 881B                                          0.0s
 => [2/3] WORKDIR /web-ping                                                0.0s
 => [3/3] COPY app.js .                                                    0.0s
 => exporting to image                                                     0.0s

 => => exporting layers                                                    0.0s
 => => writing image sha256:cea5b9b6e8731e81ee0f7510ff81f899891ab16181a71  0.0s
 => => naming to docker.io/library/web-ping                                0.0s
 
View build details: docker-desktop://dashboard/build/desktop-linux/desktop-linux/ptkuaf9gm33nk2xahl9qce3pr

**What's next:**
    View a summary of image vulnerabilities and recommendations → docker scout quickview
```

## Verifying the Built Image

If you successfully built the image, check the image list using the tag name:

```bash
docker image ls 'w*'
```

Output:
```bash
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE

web-ping     latest    cea5b9b6e873   5 minutes ago   75.5MB
```

You can now see the web-ping image in your local image repository.

## Running Containers from Custom Images

Let's run a container from the newly built image to send requests to the Docker website every 5 seconds:

```bash
docker container run -e TARGET=docker.com -e INTERVAL=5000 web-ping
```

Output:
```bash
** web-ping ** Pinging: docker.com; method: HEAD; 5000ms intervals

Making request number: 1; at 1758422726483
```

In the output log, you can see that the target is set to `docker.com` and the request interval is set to `5000` in the first line.

## Application Packaging Process Summary

This demonstrates how to package a simple application as an image and run it with Docker. The process involves three key steps:

1. **Write the packaging process in a Dockerfile script** - Define how the application should be built and configured
2. **Collect the resources to include in the image** - Gather all necessary files and dependencies
3. **Decide how to allow users to configure the application's behavior** - Use environment variables for runtime configuration

## Next Steps

In the next post, we'll study the concepts of Docker images and image layers in more detail.

## Sources

- https://github.com/gilbutITbook/080258
- Learn Docker in a Month of Lunches by Elton Stoneman
  - [Amazon](https://www.amazon.com/-/ko/Elton-Stoneman/e/B0759TFV4F/ref=dp_byline_cont_book_1)
  - [PDF](https://pdfcoffee.com/learn-docker-month-lunches-4-pdf-free.html)
  - [Youtube](https://www.youtube.com/@EltonStoneman/playlists)