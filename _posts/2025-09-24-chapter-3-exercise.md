---
layout: single
title: "[Docker] 2.6 Lab Exercise - Creating Docker Image without Dockerfile Script" 
date: 2025-09-24
categories: Docker
header:
  overlay_image: /assets/images/docker-container-tech.svg
  overlay_filter: 0.5
  teaser: /assets/images/docker-container-tech.svg
  #caption: "Create Docker Image"
Typora-root-url: ../
---

# Lab Exercise - Creating Docker Image without Dockerfile Script

## Lab Question

The goal here is to answer this question: how do you produce a Docker image without a Dockerfile? The Dockerfile is designed to automate the deployment of your application, but you can't always automate everything. Sometimes you need to run the application and complete some steps manually, and those steps can't be scripted.

This lab is a much simpler version of that scenario. You're going to start with an image on Docker Hub: diamol/ch03-lab. That image has a file at the path /diamol/ch03.txt. You need to update that text file and add your name at the end. Then produce your own image with your changed file. You're not allowed to use a Dockerfile.

Here are some hints to get you started:

- Remember that the `-it` flags let you run a container interactively.
- The filesystem for a container still exists when it is exited.
- There are many commands you haven't used yet. `docker container --help` will show you two that could help you solve the lab.

## Lab Answer

First, let's take a look at the current state of the `ch03.txt` file:

``` txt
Lab solution, by: 
```

### Step 1

First, we need to run the container with the `-it` and `--name` options:

``` bash
docker container run -it --name ch03lab diamol/ch03-lab
```

- `docker container run`: runs a new container 
- `-it`: enables interactive mode and connects the terminal
- `--name ch03lab`: assigns the container name as `ch03lab`
- `diamol/ch03-lab`: specifies the image to use

This command runs the `ch03lab` container using the `diamol/ch03-lab` image.

### Step 2

Now, let's modify the `ch03.txt` file:

``` bash
echo LEE >> ch03.txt

exit
```

- `echo LEE >> ch03.txt`: appends `LEE` to the `ch03.txt` file
- `exit`: exits the container shell and stops the container

### Step 3

Let's save the container as a new image:

```bash
docker container commit ch03lab ch03-lab-soln
```

- `docker container commit`: saves the current container state as an image
- `ch03lab`: the source container name
- `ch03-lab-soln`: the new image name

This command creates a new image called `ch03-lab-soln` from the modified state of the `ch03lab` container.

### Step 4

Let's run the new image `ch03-lab-soln` and check the result:

``` bash
docker container run ch03-lab-soln cat ch03.txt
```

Output:
``` bash
Lab solution, by: LEE
```

Here, we can see that `ch03.txt` has been updated with the name at the end.


## Sources

- https://github.com/gilbutITbook/080258
- Learn Docker in a Month of Lunches by Elton Stoneman
  - [Amazon](https://www.amazon.com/-/ko/Elton-Stoneman/e/B0759TFV4F/ref=dp_byline_cont_book_1)
  - [PDF](https://pdfcoffee.com/learn-docker-month-lunches-4-pdf-free.html)
  - [Youtube](https://www.youtube.com/@EltonStoneman/playlists)