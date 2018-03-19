---
title: "ARM Your Golang Containers"
date: 2018-03-19T11:52:57+05:30
project_url: "https://github.com/tchaudhry91/mail-proxy-svc"
categories: ["Ops", "Dev"]
tags: ["golang", "docker", "arm", "raspberrypi", "scaleway"]
---

So then, here is the deal. I write tiny microservices that may or may not serve any purpose. Despite how meaningless they might be, I give them the perfect operations treatment.

Here is an example of such a service: [hash-svc] (https://github.com/tchaudhry91/hash-svc). At it's core, all it does is take a string and returns it hash.
```
tchaudhr:cmd/ (master✗) $ ./hash-svc -serverAddr :12000
{"addr":":12000","msg":"Started HTTP Server"}
{"err":null,"input":"123","method":"hashsha256","output":"a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3","took":"41.472µs"}

tchaudhr:~/ $ curl http://localhost:12000/hash\?s\=123                                                                    
{"v":"a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3"}
```

Now, to please my heart, I put this in a container, added a Makefile, hooked up my CI and started pushing out containers with every commit. Check the Makefile, Dockerfile and the travis.yml in this [commit] (https://github.com/tchaudhry91/hash-svc/tree/a6bccb829f18e6d1f9e6f9592e0700d14ad03f4b) to see how I achieved that.

There's a problem. I couldn't run this image on my Pi or new bare metal 2.99$/month ARM server on [scaleway] (https://scaleway.com).

Docker officially supports ARMv7, Golang cross-compiles quite well. Shouldn't be that hard.

First up, modify the Dockerfile to cross-build for ARM instead.
```
ENV     CGO_ENABLED=0
ENV     GOOS=linux
ENV     GOARCH=arm
```
This should now produce binaries that would run on ARMv7-linux (i.e RaspberryPi). CGO is disabled by default when cross-compiling, but in a multi-stage docker build it can cause problems, so we disable it for the non-ARM version as well.

Next up, modify the final image to be ARM compliant.
Replace `FROM alpine` with an ARM-based image like [arm32v7/ubuntu](https://hub.docker.com/r/arm32v7/ubuntu/) or the more lightweight `FROM hypriot/rpi-alpine` from the guys over at [Hypriot] (https://blog.hypriot.com/post/setup-simple-ci-pipeline-for-arm-images/)

Add the targets to the [Makefile] (https://github.com/tchaudhry91/hash-svc/blob/master/Makefile) and now you should have a build mechanism for ARM containers.
There is, however, one problem. While the service itself is cross-compiled and runs on ARM, the final image builder that we use also uses other commands to prepare the image. An example would be `RUN apk update && apk add --no-cache ca-certificates`. This command fails to run on my x86 machine because `apk` itself is an ARM binary.
This means we need a processor emulator to be able to build them on TravisCI. Fortunately, the cool guys at [Hypriot] (https://blog.hypriot.com/post/setup-simple-ci-pipeline-for-arm-images/) have documented an easy method to get that done. Read the blog in full if you need more details, but here the command you need to run in your travis.yml to get this to work.

`docker run --rm --privileged multiarch/qemu-user-static:register --reset`

This will register the QEMU processor emulator for the ARM binaries (via binfmt_misc).
And that's all, custom service containers on your Pi! You can check out the full code over here in this [repo] (https://github.com/tchaudhry91/hash-svc)
