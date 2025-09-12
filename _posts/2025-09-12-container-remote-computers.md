---
layout: single
title: "[Docker] 1.3 Using Containers as Remote Computers" 
date: 2025-09-12
categories: Docker
header:
  overlay_image: /assets/images/docker-container-tech.svg
  overlay_filter: 0.5
  teaser: /assets/images/docker-container-tech.svg
  caption: "Docker Container Architecture"
Typora-root-url: ../
---

# 1.3 Using Containers as Remote Computers

Docker allows you to package tools and scripts into a single image, enabling you to run scripts directly as containers without additional installation or configuration work.

You can run an interactive container from an image using the following command:

```bash
docker container run --interactive --tty diamol/base
```

- The `--interactive` flag connects you to the container.
- The `--tty` flag means you want to control the container through a terminal session.

If you see a terminal session like this, you have successfully connected to the inside of the container:

```bash
/ #
```

You can now execute commands as if you were using a command-line interface inside the container.

```bash
/ # hostname

6b27a8190cde

/ # date

Fri Sep 12 04:45:48 UTC 2025
```

This means two things:
- A local terminal session is open.
- The connected computer is the currently running container.

Docker containers share the host computer's operating system, so if the host computer is a Linux machine, you'll get a Linux shell; if it's a Windows machine, you'll get a Windows command prompt.

Docker itself operates identically regardless of the host computer's architecture or operating system, but applications inside containers can distinguish between operating systems and architectures. Regardless of what's inside the container, the way you handle containers is the same across all environments.

Open a new terminal session and check information about all running containers with the following command:

```bash
docker container ls
```

output:
```bash
CONTAINER ID   IMAGE           COMMAND        CREATED          STATUS

6b27a8190cde   diamol/base     "/bin/sh"      12 minutes ago   Up 12 minutes
```

You can see that the Container ID matches the hostname we checked inside the container.
Docker generates and assigns a random ID value each time a container runs, and part of this ID becomes the hostname.
If you want to specify a particular container, you can specify the first few characters of the container ID.

Since the current `CONTAINER ID` is `6b27a8190cde`, let's specify up to `6b`:

```bash
docker container top 6b
```

output:
```bash
UID                 PID                 PPID                C
root                2956                2934                0   

STIME               TTY                 TIME                CMD
04:39               ?                   00:00:00            /bin/sh
```

The `logs` command outputs all logs collected from the target container:

```bash
docker container logs 6b
```

output:
```bash
/ # hostname

6b27a8190cde

/ # date

Fri Sep 12 04:45:48 UTC 2025
```

The `inspect` command shows detailed information about the target container:

```bash
docker container inspect 6b
```

output:
```bash
[

    {

        "Id": "6b27a8190cde6d4261c6070532205d4191e3ba83abbc95880060db97deabdaa2",

        "Created": "2025-09-12T04:39:26.662005215Z",

        "Path": "/bin/sh",

        ....
     }
]
```

In addition to the above information, useful information for tracking application issues is provided, such as the container's virtual file system paths and commands running in the container. This information is also generated in `JSON` format, making it advantageous for automated processing.

There's one more lesson to learn from the exercises so far: when using Docker, all containers are the same.
Applying Docker adds one management layer on top of all applications, which manages any application in the same way:

- Use the `run` command to execute applications.
- Use the `logs` command to output logs.
- Use the `top` command to check the process list.
- Use the `inspect` command to check detailed container information.

## Next Steps

Hosting websites using containers.

## Sources

- https://github.com/gilbutITbook/080258
- Learn Docker in a Month of Lunches by Elton Stoneman
  - [Amazon](https://www.amazon.com/-/ko/Elton-Stoneman/e/B0759TFV4F/ref=dp_byline_cont_book_1)
  - [PDF](https://pdfcoffee.com/learn-docker-month-lunches-4-pdf-free.html)
  - [Youtube](https://www.youtube.com/@EltonStoneman/playlists)