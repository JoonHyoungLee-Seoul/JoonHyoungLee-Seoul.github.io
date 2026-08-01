---
layout: single
title: "[Docker] 1.2 What is a Container?"
date: 2025-09-11
categories: Docker
tags: [Docker, Containers]
---

> **Source note:** This article is a study note based on Elton Stoneman, [*Learn Docker in a Month of Lunches*](https://github.com/gilbutITbook/080258). It summarizes and reorganizes concepts from the book; credit for the original material belongs to the author and publisher.

---

## Container Concept

Docker containers are like shipping containers that hold applications. The application is the cargo inside the container.

Consider the following diagram:

|        ![1.2_1](/assets/images/docker/1.2_1.png)        |
| :---------------------------------------------------: |
| **Figure 1. An app inside the container environment** |

The hostname, IP address, and file system are all virtual resources created by Docker. These components work together to create an environment where applications can run. The box in the diagram represents this environment.

## Container Isolation

Inside the container, you cannot see the external environment. However, this container runs on a computer that can execute multiple other containers simultaneously.

These containers maintain independent environments while sharing the computer's CPU, memory, and OS resources.

|           ![1.2_2](/assets/images/docker/1.2_2.png)            |
| :----------------------------------------------------------: |
| **Figure 2. Multiple containers on one computer share the same OS, CPU, and memory** |

## Key Benefits

This architecture satisfies Docker's two most important requirements: **isolation** and **density**.

- **Density**: Running the maximum number of applications that the computer's CPU and memory allow
- **Isolation**: Applications with different runtime versions and libraries need to run in separate environments due to compatibility requirements

## Efficiency Advantages

Docker containers achieve isolation and density through these efficiencies:

- Containers share the host computer's operating system, significantly reducing required resources
- This enables faster execution and supports approximately 5x more applications compared to virtual machines
- Containers provide independent environments from the outside world, achieving both density and isolation simultaneously

## Next Steps

The next post will cover hands-on practice for managing containers.
