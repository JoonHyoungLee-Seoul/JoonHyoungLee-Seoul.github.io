---
layout: single
title: "[Docker] 1.5 How Docker Runs Containers"
date: 2025-09-13
categories: Docker
tags: [Docker, Containers]
---

> **Source note:** This article is a study note based on Elton Stoneman, [*Learn Docker in a Month of Lunches*](https://github.com/gilbutITbook/080258). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Docker Workflow Overview

In the previous post, we learned that the build-share-run workflow is at the core of Docker. By applying this workflow, we can easily deploy software. When you create and share a container image, users simply need to run this image in Docker.

In this post, we'll explain Docker's internal operating principles to help you fully understand the process of running applications with Docker.

## Docker Architecture Components

The following diagram shows that many components are involved when Docker runs containers:

| ![1.5_1](/assets/images/docker/1.5_1.png) |
| :----------------------------------------------------------: |
| **Figure 1.The components of Docker** |

- **Docker Engine** is the component responsible for Docker's management functions. It manages the local image cache, downloading new images when needed and using existing images when available. It also works with the host operating system to create Docker resources such as containers and virtual networks. Docker Engine is a background process that runs continuously.

- **Docker Engine** performs its assigned functions through the **Docker API**. The Docker API is a standard HTTP-based REST API. By modifying Docker Engine's configuration, you can block or allow this API to be called from external computers over the network.

- **Docker Command-Line Interface (Docker CLI)** is a client of the Docker API. When we use `docker` commands, we're actually calling the Docker API through the CLI.

## API-Driven Architecture

The only way to interact with Docker Engine is through the API. The CLI serves as a conduit for forwarding requests to the API.

Since the Docker API is identical regardless of the operating system, you can control any Docker Engine from a Windows laptop, no matter where it's located.

## Container Management

Docker Engine manages containers through a component called **containerd**. Containerd creates containers (virtual environments) using features provided by the host operating system.

## Next Steps

The next post will cover Docker images and how they're structured and managed.
