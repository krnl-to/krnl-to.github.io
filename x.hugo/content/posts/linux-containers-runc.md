+++
title = "Linux Containers Internals: runc"
date = "2023-03-10T11:14:02+03:00"
author = "Anthony Nandaa"
authorTwitter = "profnandaa" #do not include @
cover = ""
tags = ["linux", "containers", "lci-series"] #lci -> linux containers internals
keywords = ["linux", "containers", "runc", "oci", "opencontainers"]
description = "Introducing runc as a way of creating containers"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

# Intro to Linux Containers Internals: runc

At this point of our [#lci-series](/tags/lci-series/), we have seen how to _hack around_ and create containers. In this post, we will introduce some order and standardization. We aimed to demonstrate what goes on under the hood, just one abstraction layer below -- a scratch on the surface.

## Open Container Initiative (OCI)

The [OCI](https://opencontainers.org/) was set up as an open governance structure to create open industry standards around container formats and runtimes. It develops specifications for standards on Operating System and application containers.

OCI has two specs, an [_Image_ spec](https://github.com/opencontainers/image-spec) and a [_Runtime_ spec](https://github.com/opencontainers/runtime-spec). The _Image spec_ defines the archive format of OCI container images, which consists of a manifest, an image index, a set of filesystem layers and a configuration. The _Runtime spec_ specifies the configuration, execution environment and lifecycle of a container.

![oci](https://user-images.githubusercontent.com/261265/224251228-5dc72104-5f04-47a2-a00a-d28ba55fc63f.png)

## What is `runc`?

`runc` is a CLI tool for spawning and running containers according to the OCI specification. The diagram below demonstrates how `runc` fits in the whole "container platform" ecosystem.

![runc_eco](https://user-images.githubusercontent.com/261265/224251117-51a27866-41dd-4614-9d5e-a8105573c936.png)

## Create a container with `runc`

I know this is the moment you've been waiting for; let's get to it!

We will pick up from where we were in our last post. We used `debootstrap` to get a _full root filesystem_ that formed the basis of our container. We will re-use the same filesystem here.

> If you are joining us mid-journey, take a glance at the previous post on [_chroot_](../linux-container-intro-chroot-again/); it's a brief one.

So, I'm here where I have my `ubuntufs` directory within `$HOME`, looking like this and I'm on [WSL2](https://learn.microsoft.com/en-us/windows/wsl/about) on a Windows machine:

```bash
~/container-intro$ ls ubuntufs/
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media ...
```

We will first build `runc` from sources. So, first start by [installing Go](https://go.dev/doc/install) on your Linux system.

```bash
$ rm -rf /usr/local/go && tar -C /usr/local -xzf go1.20.2.linux-amd64.tar.gz
$ mkdir ~/.go # just my preference, you can name it as you wish
# update $PATH
# add this to your .bashrc (or equivalent file)
export PATH=$PATH:/usr/local/go/bin
# setup $GOPATH
export GOPATH=$HOME/.go

# chek that all is good
$ go version
```

I'm on a 1 version older Go by the time of this writing, but it shouldn't matter for now, at least in March 2023.

```bash
~/container-intro$ go version
go version go1.19.4 linux/amd64
```

Next, let's install `runc`:

```bash
$ go install github.com/opencontainers/runc@latest
# check version, it's marked unknown since we installed
# the latest from "dev" branch
$ $GOPATH/bin/runc --version
runc version unknown
spec: 1.0.2-dev
go: go1.19.4
```

Generate a template _runtime spec_ that will be used to spin off our container:

```bash
$ $GOPATH/bin/runc spec
```

A `config.json` file will be generated with the spec. I'll draw your attention to fields like `mounts` and `linux.namespaces`. Do you see some familiar stuff from what we have covered in the series?

In the `config.json` file, update `root.path` to the name of your filesystem directory, it is `ubuntufs` for my case.

Next, let's run a sample container:

```bash
$ sudo $GOPATH/bin/runc run my-cont
```

On a new tab on the host, you can run `sudo $GOPATH/bin/runc run list`, you should see the container listed:

![image](https://user-images.githubusercontent.com/261265/224249763-84856d2f-6bcb-4172-a956-55a47d7d7ebe.png)

This gets you in the container shell the same way we've been doing in the previous posts, but this time, it's in a standard and spec-compliant way. Go ahead and play around with it, knock yourself off with things like `ps`, `htop`, test the limits and more.

> So far, our filesystem is readonly, if you want to change that you can update `root.readonly` in the `config.json` file.

Get more details on [`runc` here](https://github.com/opencontainers/runc).

> _Sorry, no spoiler alerts on where our flight is headed next for this series. Maybe networking, maybe dissecting an OCI image, maybe explaining the whole trip of what happens when you run a command like `docker run`, how about `docker pull`? Stick with us!_

_For reading thus far, the custom dictates you have to [pick your gift here](https://www.youtube.com/watch?v=Z7eRU7C-ti8)_ 🙂 _Thanks for reading, see you next!_
