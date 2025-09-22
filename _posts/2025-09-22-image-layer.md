---
layout: single
title: "[Docker] 2.4 Understanding Docker Images and Image Layers" 
date: 2025-09-22
categories: Docker
header:
  overlay_image: /assets/images/docker-container-tech.svg
  overlay_filter: 0.5
  teaser: /assets/images/docker-container-tech.svg
  #caption: "Create Docker Image"
Typora-root-url: ../
---

# Understanding Docker Images and Image Layers

Docker images contain all the files we included in our packaging. These files later form the container's file system. In addition to files, images also contain various metadata information about themselves.

This information includes a simple history of how the image was built. Using this information, you can find out what each layer that makes up the image is and what commands were used to build these layers.

Let's check the history of the web-ping image:

```bash
docker image history web-ping
```

When you run this command, information about each layer is displayed line by line:

Output:
```bash
IMAGE          CREATED        CREATED BY                                       

cea5b9b6e873   22 hours ago   CMD ["node" "/web-ping/app.js"]                  

<missing>      22 hours ago   COPY app.js . 

<missing>      22 hours ago   WORKDIR /web-ping                                
```

The `CREATED BY` field shows the Dockerfile script instruction that created that layer. Dockerfile instructions and image layers have a 1:1 relationship.

Let's understand image layers properly to use Docker efficiently.

## Understanding Image Layers

A Docker image is a logical object composed of image layers. Layers are files physically stored in the Docker engine's cache. Image layers are shared among multiple images and containers. For example, if you run multiple containers running Node.js applications, all these containers share the image layer containing the Node.js runtime.

| ![2.4](/images/$(filename)/2.4.png) |
| :----------------------------------------------------------: |
| **Figure 1.How image layers are logically built into Docker Images** |

The `diamol/node` image includes a minimal operating system layer and the Node.js runtime. Since our web-ping image uses `diamol/node` as the base image, it includes all layers from the base image.

This is the meaning of the `FROM` instruction in the Dockerfile script.

What is the total size of the web-ping image?:

```bash
docker image ls
```

Output:
```bash
REPOSITORY                     TAG       IMAGE ID       CREATED        SIZE

web-ping                       latest    cea5b9b6e873   22 hours ago   75.5MB

hello-world                    latest    ca9905c726f0   6 weeks ago    5.2kB

mjerabek/vivado                latest    1bc51e69e359   3 years ago    499MB

diamol/ch02-hello-diamol-web   latest    40ba417b2294   4 years ago    54.6MB

diamol/base                    latest    e65ed3e91e57   4 years ago    6.89MB

diamol/ch03-web-ping           latest    bfce5d697312   5 years ago    75.5MB

diamol/ch02-hello-diamol       latest    c65623b61864   5 years ago    5.3MB
```

Here, it appears that the two images `diamol/ch03-web-ping` and `web-ping` occupy similar capacity (75.5MB). However, this is not true. The `SIZE` field in the image list above shows the logical size of the image, not the actual disk space occupied by that image. When layers are shared with other images, they take up much less disk space than the numbers shown here.

First, the sum of the above `SIZE` values is approximately 716MB.

To verify this, let's run the following command:

```bash
docker system df
```

Output:
```bash
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE

Images          7         3         635.5MB   580.9MB (91%)
```

Looking at the output of this command, you can see that the actual capacity of the image cache occupies 635MB. Therefore, approximately 81MB is shared between images as layers.

This approach saves disk space.

## Layer Immutability

However, if image layers are shared by multiple images, the shared layers must be immutable. If an image layer is modified, that modification would affect other images that share the layer.

## Next Steps

In the next post, we'll learn how to optimize Dockerfile scripts using image layer caching.

## Sources

- https://github.com/gilbutITbook/080258
- Learn Docker in a Month of Lunches by Elton Stoneman
  - [Amazon](https://www.amazon.com/-/ko/Elton-Stoneman/e/B0759TFV4F/ref=dp_byline_cont_book_1)
  - [PDF](https://pdfcoffee.com/learn-docker-month-lunches-4-pdf-free.html)
  - [Youtube](https://www.youtube.com/@EltonStoneman/playlists)