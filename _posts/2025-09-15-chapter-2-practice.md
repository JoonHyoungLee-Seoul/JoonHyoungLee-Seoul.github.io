---
layout: single
title: "[Docker] 1.6 Exercise - Working with Container File Systems"
date: 2025-09-15
categories: Docker
tags: [Docker, Containers]
---

> **Source note:** This article is a study note based on Elton Stoneman, [*Learn Docker in a Month of Lunches*](https://github.com/gilbutITbook/080258). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Chapter Summary

Let's wrap up this chapter (Basic Docker Usage) by solving an exercise problem.

In this exercise, you'll run a website container and modify the web page content by replacing the index.html file.

This web page file is contained within the container's file system.

## Exercise Instructions

Please refer to the following hints:

- Use the `docker container` command to see a list of operations you can perform on containers
- Add the `--help` flag to any `docker` command to see help for that command  
- In the Docker image `diamol/ch02-hello-diamol-web`, the web page files are located at `/usr/local/apache2/htdocs`

## Solution

### Step 1: Run the Web Container

Run the web container from the chapter exercises:

```bash
docker container run --detach --publish 8088:80 diamol/ch02-hello-diamol-web
```

Output:
```bash
85caba75ad8fbf8a7ffff22f2c65aaff1460cb152c180a66d896006ffe100bea
```

### Step 2: Verify HTML File Location

Check the HTML page in the container is in the expected location:

```bash
docker container exec 85c ls /usr/local/apache2/htdocs
```

Output:
```bash
index.html
```

| ![1.6_1](/assets/images/docker/1.6_1.png) |
| :----------------------------------------------------------: |
| **Figure 1.Link before** |

### Step 3: Copy New HTML File

Now we know the HTML file is inside the container, so we can use `docker container cp` to copy a local file into the container. This will overwrite the `index.html` file in the container with the file in my current directory:

```bash
docker container cp index.html 85c:/usr/local/apache2/htdocs/index.html
```

Output:
```bash
Successfully copied 2.05kB to 85c:/usr/local/apache2/htdocs/index.html
```

The format of the `cp` command is `[source path] [target path]`. The container can be the source or the target, and you prefix the container file path with the container ID (`85c` here). You can use the same file path format with forward-slashes on Linux or Windows in the `cp` command.

### Step 4: Verify Changes

Browse to the published port on http://localhost:8088 - you'll see your new content.

| ![1.6_2](/assets/images/docker/1.6_2.png) |
| :----------------------------------------------------------: |
| **Figure 2.Link After** |

## Key Learning Points

- Container file systems can be modified while the container is running
- The `docker container exec` command allows you to run commands inside containers
- The `docker container cp` command enables file copying between host and container
- Changes to container file systems are temporary and will be lost when the container is removed

## Next Steps

The next chapter will cover "How to create Docker image".
