---
layout: single
title: "[Docker] 1.4 Using Containers to Host Websites" 
date: 2025-09-13
categories: Docker
header:
  overlay_image: /assets/images/docker-container-tech.svg
  overlay_filter: 0.5
  teaser: /assets/images/docker-container-tech.svg
  #caption: "Docker Container Architecture"
Typora-root-url: ../
---

# Using Containers to Host Websites

## Container Status Check

Before starting the practice, let's check the list of all containers regardless of their status with the following command:

```bash
docker container ls --all
```

Output:
```bash
CONTAINER ID     IMAGE        COMMAND       CREATED        STATUS

6b27a8190cde   diamol/base    "/bin/sh"    25 hours ago   Exited (0) 
```

You can see that all containers have the status `Exited (0)`. There are two important things to understand here:

- A container remains in running state only while the application inside the container is running. When the application process terminates, the container status becomes `Exited`. Terminated containers do not consume CPU resources or memory.
- Even when a container terminates, the container does not disappear. This allows you to restart the container later, check logs, or copy files to and from the container's file system. Additionally, the container's file system remains intact and continues to occupy disk space on the host computer.

## Running Background Containers

So how can we run a container and keep it running in the background? Since the main purpose of using Docker is to run server applications like websites, batch processes, and databases, this would be the primary use case.

Let's host a simple website in a container with the following command:

```bash
docker container run --detach --publish 8088:80 diamol/ch02-hello-diamol-web
```

When you run this command, it outputs the container ID. This container continues running in the background.

Output:
```bash
f71dd8d10b26cd32a34d7e33a5c444ce9f2140a65bbb21cfbf37445727e38bd9
```

Use the `docker container ls` command to verify that the newly created container status is `Up`.
The image used to create this container is `diamol/ch02-hello-diamol-web`.

This image contains an Apache web server and a simple HTML page. When you run this image as a container, web pages are served through an actual web server.

## Container Flags for Web Hosting

To make a container run in the background while listening on the network, you need to apply the following two flags to the `docker container run` command:

- **`--detach`**: Runs the container in the background and outputs the container ID
- **`--publish`**: Publishes the container's port to the host computer

## Network Isolation and Port Publishing

Containers are not exposed to the external environment by default. Each container has its own IP address, but this IP address belongs to Docker's internal virtual network and is not connected to the physical network that the host computer is connected to.

Publishing a container's port means that Docker listens on a port on the host computer and forwards incoming traffic on that port to the container.

In the example above, `8088:80` means that traffic coming to port `8088` on the host computer is forwarded to port `80` in the container. The following diagram helps you easily understand this process:

| ![스크린샷 2025-09-13 오후 3.30.47](/images/$(filename)/스크린샷 2025-09-13 오후 3.30.47.png) |
| :----------------------------------------------------------: |
| **Figure 1.The physical and virtual networks for computers and containers** |

To interpret this diagram: the computer is the host computer running Docker with IP address `192.168.2.150`. This address is from the physical network I'm using, assigned by my home router.

On this computer, one container is running through Docker. This container's IP address is `172.0.5.1`, which is an address on Docker's virtual network assigned by Docker.

Computers on the physical network cannot access this container's IP address. This is because this address exists only within Docker's internal network. However, since the container's port is published, traffic can be forwarded to the container.

## Testing the Web Server

Let's access the `http://localhost:8088` page in a browser. This `HTTP` request is sent from the local computer, but the `HTTP` response comes from the container.

| ![스크린샷 2025-09-13 오후 3.31.49](/images/$(filename)/스크린샷 2025-09-13 오후 3.31.49.png) |
| :----------------------------------------------------------: |
| **Figure 2. The web application served from a container on the local machine** |

This web page is packaged as an image along with the web server. Docker's portability and efficiency are demonstrated by the fact that no additional components are needed besides the image. Web developers can run the entire application stack by running just one container on their laptop.

## Monitoring Container Performance

You can check the status of running containers with the `docker container stats` command:

```bash
docker container stats f71
```

Output:
```bash
CONTAINER ID   NAME         CPU %     MEM USAGE / LIMIT    

f71dd8d10b26   sweet_kare   0.01%     8.094MiB / 3.828GiB 
 
MEM %     NET I/O        BLOCK I/O 

0.21%     1.7kB / 126B   4.49MB / 4.1kB   
```

## Cleanup

When you're done using containers, you can delete them with the following command. The `--force` flag allows immediate deletion even of running containers:

```bash
docker container rm --force $(docker container ls --all --quiet)
```

The `$()` syntax passes the output of the command inside the parentheses to another command. Therefore, the meaning of the entire command above is to create a list of all containers existing on the host computer and then command their complete deletion. It's convenient but should be used with caution.

## Next Steps

The next post will cover "Understanding how Docker runs containers"

## Sources

- https://github.com/gilbutITbook/080258
- Learn Docker in a Month of Lunches by Elton Stoneman
  - [Amazon](https://www.amazon.com/-/ko/Elton-Stoneman/e/B0759TFV4F/ref=dp_byline_cont_book_1)
  - [PDF](https://pdfcoffee.com/learn-docker-month-lunches-4-pdf-free.html)
  - [Youtube](https://www.youtube.com/@EltonStoneman/playlists)