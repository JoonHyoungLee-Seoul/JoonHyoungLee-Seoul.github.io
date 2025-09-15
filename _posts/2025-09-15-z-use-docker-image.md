---
layout: single
title: "[Docker] 2.1 Using Images Shared on Docker Hub" 
date: 2025-09-15
categories: Docker
header:
  overlay_image: /assets/images/docker-container-tech.svg
  overlay_filter: 0.5
  teaser: /assets/images/docker-container-tech.svg
  #caption: "Create Docker Image"
Typora-root-url: ../
---

# Using Images Shared on Docker Hub

## Chapter Introduction

In the previous chapter, we learned how to run and manage containers using Docker. With containers, you can run and manage applications in the same way regardless of the application's technology stack. In this chapter, we'll learn how to create Docker images.

In the previous chapter, when using the `docker container run` command, we saw the process of downloading images when the required image wasn't available on the local computer. This process was possible because software distribution functionality is completely built into the Docker platform.

## Downloading Images with Docker CLI

Let's explicitly download a desired image through the Docker CLI:

```bash
docker image pull diamol/ch03-web-ping
```

Output:
```bash
Using default tag: latest
latest: Pulling from diamol/ch03-web-ping
0362ad1dd800: Pull complete 
b09a182c47e8: Pull complete 
39d61d2ed871: Pull complete 
b4e2115e274a: Pull complete 
f5cca017994f: Pull complete 
f504555623f6: Pull complete 
Digest: sha256:2f2dce710a7f287afc2d7bbd0d68d024bab5ee37a1f658cef46c64b1a69affd2
Status: Downloaded newer image for diamol/ch03-web-ping:latest
docker.io/diamol/ch03-web-ping:latest
```

## Docker Hub Registry

The image name is `diamol/ch03-web-ping`. This image is stored in **Docker Hub**, the repository that Docker accesses first when looking for images. The repository that provides images is called a registry, and Docker Hub is a free public registry.

## Image Layers

When downloading an image, you can see multiple files being downloaded simultaneously. Each of these files is called an **image layer**.

Docker images are physically composed of multiple small files, and Docker assembles these files to create the container's internal file system. Once all layers are downloaded, the entire image becomes available.

## Running Containers from Downloaded Images

Let's run a container from the downloaded image and check the application's functionality:

```bash
docker container run -d --name web-ping diamol/ch03-web-ping
```

Output:
```bash
6fa28d68d04d9e292a4f062fa5bc4aeb315ea2c99ef76f952c7b4735f885a706
```

The `-d` flag is an abbreviation for `--detach`, so this container runs in the background. Since this container doesn't receive requests through the network, there's no need to publish ports externally.

Using the `--name` flag, you can give the container a desired name and refer to the container by this name. We gave this container the name `web-ping` for easy identification.

## Examining Application Logs

Let's examine the application logs collected through Docker:

```bash
docker container logs web-ping
```

Output:
```bash
** web-ping ** Pinging: blog.sixeyed.com; method: HEAD; 3000ms intervals

Making request number: 1; at 1757906463163
Got response status: 200 at 1757906463302; duration: 139ms
Making request number: 2; at 1757906466170
Got response status: 200 at 1757906466203; duration: 33ms
Making request number: 3; at 1757906469178
Got response status: 200 at 1757906469215; duration: 37ms
Making request number: 4; at 1757906472185
Got response status: 200 at 1757906472217; duration: 32ms
Making request number: 5; at 1757906475195
Got response status: 200 at 1757906475238; duration: 43ms
Making request number: 6; at 1757906478204
```

This application is repeatedly sending HTTP requests to `blog.sixeyed.com`. The application allows custom configuration of the target URL, request interval, and request type. It reads configuration values from the system's environment variables.

## Environment Variables

Environment variables are key-value pairs provided by the operating system. Docker containers can have separate environment variables, which are not inherited from the host operating system but are assigned by Docker, like the container's hostname or IP address.

The `web-ping` image includes default values for these environment variables. When you run a container, Docker applies these default values to the container for use by the application. Changing environment variable values changes the application's behavior.

## Using Custom Environment Variables

Let's delete the running container and run a new container with a different value for the TARGET environment variable:

```bash
docker rm -f web-ping
docker container run --env TARGET=google.com diamol/ch03-web-ping
```

Output:
```bash
** web-ping ** Pinging: google.com; method: HEAD; 3000ms intervals

Making request number: 1; at 1757907330710
Got response status: 301 at 1757907331258; duration: 548ms
Making request number: 2; at 1757907333715
Got response status: 301 at 1757907334141; duration: 426ms
Making request number: 3; at 1757907336722
Got response status: 301 at 1757907337151; duration: 429ms
Making request number: 4; at 1757907339730
Got response status: 301 at 1757907340094; duration: 364ms
```

Although this container was created from the same image as before, this time it sends requests to `google.com`.

## Configuration Flexibility

Docker images are packaged with default configuration values, but you should be able to change these configuration values when running containers. Environment variables provide a simple way to implement this. The `web-ping` application code looks for an environment variable called `TARGET`. Just as we changed from `blog.sixeyed.com` to `google.com`, you can specify different values using the `--env` flag.

The following diagram shows containers with various configuration values beyond the default values included in the image:

| ![2.1](/images/$(filename)/2.1.png) |
| :----------------------------------------------------------: |
| **Figure 1.Environment variables in Docker images and containers** |

Containers only have environment variables assigned by Docker. The important point in the diagram above is that both containers have the same application and use the same image, but their behavior differs due to configuration values.

## Next Steps

In the next post, we'll learn how to create Docker images directly by writing a `Dockerfile`.

## Sources

- https://github.com/gilbutITbook/080258
- Learn Docker in a Month of Lunches by Elton Stoneman
  - [Amazon](https://www.amazon.com/-/ko/Elton-Stoneman/e/B0759TFV4F/ref=dp_byline_cont_book_1)
  - [PDF](https://pdfcoffee.com/learn-docker-month-lunches-4-pdf-free.html)
  - [Youtube](https://www.youtube.com/@EltonStoneman/playlists)