---
layout: single
title: "[Docker] 2.5 Optimizing Dockerfile Scripts Using Image Layer Cachings" 
date: 2025-09-23
categories: Docker
header:
  overlay_image: /assets/images/docker-container-tech.svg
  overlay_filter: 0.5
  teaser: /assets/images/docker-container-tech.svg
  #caption: "Create Docker Image"
Typora-root-url: ../
---

# Optimizing Dockerfile Scripts Using Image Layer Caching

The web-ping image built in the previous post contains a JavaScript file with the application implementation.

When this file is modified and the image is rebuilt, a new image layer is created.

Assume that Docker image layers are arranged only in a specific order. Therefore, if a layer in the middle of the sequence is changed, layers that come above the changed layer cannot be reused.

This concept might be a bit difficult to understand, so let's understand it through practice.

Let's modify the `app.js` file in the `ch03-web-ping` directory. I made a simple change by adding a blank line at the end of the file:

```javascript
const https = require("https");

const options = {
  hostname: process.env.TARGET,
  method: process.env.METHOD
};

console.log(
  "** web-ping ** Pinging: %s; method: %s; %dms intervals",
  options.hostname,
  options.method,
  process.env.INTERVAL
);

process.on("SIGINT", function() {
  process.exit();
});

let i = 1;
let start = new Date().getTime();
setInterval(() => {
  start = new Date().getTime();
  console.log("Making request number: %d; at %d", i++, start);
  var req = https.request(options, res => {
    var end = new Date().getTime();
    var duration = end - start;
    console.log(
      "Got response status: %s at %d; duration: %dms",
      res.statusCode,
      end,
      duration
    );
  });
  req.on("error", e => {
    console.error(e);
  });
  req.end();
}, process.env.INTERVAL);

# !!I inserted new line here.!!
```

Let's rebuild the image after the modification:
```bash
docker image build -t web-ping:v2 .
```

Output:

| ![2.5](/images/$(filename)/2.5.png) |
| :----------------------------------------------------------: |
| **Figure 1.Building an image where layers can be used from the cache** |

In this image, steps 2 through 5 reused previously cached layers, but steps 6 and 7 created new layers.

Dockerfile script instructions have a 1:1 relationship with image layers. However, if the result of an instruction is the same as the previous build, it reuses the previously cached layer to reduce the waste of re-executing the same instruction.

Docker uses hash values to check if there are matching layers in the cache. Hash values are calculated from the Dockerfile script instructions and the contents of files copied by the instructions.

If there is no matching hash value in the existing image, a cache miss occurs and the instruction is executed. Once an instruction is executed, all subsequent instructions are unconditionally executed, even if they haven't been modified.


In the image above, when step 6's `COPY` instruction was modified, step 7's `CMD` instruction was also automatically executed.

To solve this problem, we should arrange Dockerfile script instructions that are rarely modified at the front, and instructions that are frequently modified at the back.

This way, we can reuse as many cached image layers as possible, which is an optimized script structure that can save time, disk space, and network bandwidth.

Now let's optimize the Dockerfile script for the `web-ping` image:

```bash
FROM diamol/node

CMD ["node", "/web-ping/app.js"]

ENV TARGET="blog.sixeyed.com" \
	METHOD="HEAD" \
	INTERVAL="3000"
	
WORKDIR /web-ping
COPY app.js .
```

Let's check the script before optimization for comparison:

```bash
FROM diamol/node

ENV TARGET="blog.sixeyed.com"
ENV METHOD="HEAD"
ENV INTERVAL="3000"

WORKDIR /web-ping
COPY app.js .

CMD ["node", "/web-ping/app.js"]
```

The changes for optimization are as follows:

1. Moved the `CMD` instruction, which is rarely modified, to the front.
2. Combined the three `ENV` instructions into one.

Through this optimization, we reduced seven instruction steps to five steps. When we modify the `app.js` file and rebuild the image, all layers except the last step are reused from the cache.

Considering such optimizations will be of great help when building images in the future.

## Next Steps

In the next post, we will review what we learned in this chapter by solving practice problems.

## Sources

- https://github.com/gilbutITbook/080258
- Learn Docker in a Month of Lunches by Elton Stoneman
  - [Amazon](https://www.amazon.com/-/ko/Elton-Stoneman/e/B0759TFV4F/ref=dp_byline_cont_book_1)
  - [PDF](https://pdfcoffee.com/learn-docker-month-lunches-4-pdf-free.html)
  - [Youtube](https://www.youtube.com/@EltonStoneman/playlists)