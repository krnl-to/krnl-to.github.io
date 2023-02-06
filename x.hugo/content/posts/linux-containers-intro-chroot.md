+++
title = "Linux Containers Intro: chroot"
date = "2023-02-06T14:03:52+03:00"
author = "Anthony Nandaa"
cover = ""
tags = ["linux", "containers", "lci-series"]
keywords = ["linux", "containers", "chroot"]
description = "Introduction to Linux Containers: chroot"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

_Edited by [Ishuah Kariuki](https://ishuah.com/)_

# Intro to Linux Containers Internals: `chroot`

## Pitch

By the end of this post, we will appreciate the idea of a _container_, within the Linux context. For most people, when you mention containers, the first thing that comes to mind is Docker. What if I told you that containers have nothing to do with Docker; instead it is the other way round.

Actually, this series will introduce Linux container internals without ever using Docker, just to pass the point across.

> ℹ️ **NOTE:** all the examples below are based on `Ubuntu 22.04.1 LTS` running on [Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/about) (WSL2). However, you can still run this on any Linux distro (VM or machine).
> ![image](https://user-images.githubusercontent.com/261265/216800823-0f522fc5-eea1-4fc4-98c0-302c951cf1a6.png)



## Enter `chroot`

Create a directory, e.g. `container_intro`:

```bash
mkdir container_intro
```

Add a simple file to the directory:

```bash
echo "CONTAINER" > container_intro/CONTAINER_ROOT
```

Let's try to `chroot` into the directory:

```bash
sudo chroot container_intro
```

You will get this error: `chroot: failed to run command ‘/bin/bash’: No such file or directory`. This means that we are can get into this directory and use as our root `/`, when we try to execute `bash` which should be in `/bin/bash`, it is not found.

**First hack:** let's copy `/bin/bash` (and one more command `ls` that we will run too afterwards) from the current system to our directory:

```bash
mkdir container_intro/bin
cp /bin/bash /bin/ls container_intro/bin
```

Now back to `chroot`:
```
sudo chroot container_into
```

Still not working! This is because `bash` has dependancy on other libraries. Let's view them:

```bash
ldd /bin/bash
```
```
    linux-vdso.so.1 (0x00007ffe20ff5000)
    libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6 (0x00007fdbf1438000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fdbf1210000)
    /lib64/ld-linux-x86-64.so.2 (0x00007fdbf15d2000)
```

Let's go ahead and copy them into our directory:

```bash
mkdir container_intro/lib{,64} # creates both lib and lib64
cp /lib/x86_64-linux-gnu/libtinfo.so.6 /lib/x86_64-linux-gnu/libc.so.6 container_intro/lib
cp /lib64/ld-linux-x86-64.so.2 container_intro/lib64
```

Let's do the same for `ls`:

```bash
ldd /bin/ls
```
```
    linux-vdso.so.1 (0x00007ffcccf2f000)
    libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1 (0x00007f932a5fc000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f932a3d4000)
    libpcre2-8.so.0 => /lib/x86_64-linux-gnu/libpcre2-8.so.0 (0x00007f932a33d000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f932a653000)
```

```bash
cp /lib/x86_64-linux-gnu/libselinux.so.1 /lib/x86_64-linux-gnu/libc.so.6 \
    /lib/x86_64-linux-gnu/libpcre2-8.so.0 container_intro/lib
cp /lib64/ld-linux-x86-64.so.2 container_intro/lib64
```

Finally, back to our `chroot`:

```bash
sudo chroot container_intro
```
And we are in! We should see the prompt:
```
bash-5.1#
```

Let's run `ls`:
```
bash-5.1# ls
CONTAINER_ROOT  bin  lib  lib64
bash-5.1#
```

Now, essentially, we are **contained** within that directory as our whole world (file system). As far as we are concerned, we can't jump out of it.

## Next Episode: Spoiler Alert

We have looked at a hacky solution for getting only two processes: `bash` and `ls` up and running. In the next episode, we will look at setting up a full _filesystem_. For the eager learners, you can check out [`debootstrap`](https://manpages.debian.org/stretch/debootstrap/debootstrap.8.en.html). See you next!

> _Thanks for reading this far. If you followed this walkthrough hands-on, here is a [small gift for you](https://www.youtube.com/watch?v=kIjExHdnx2E)!_ 😀

<hr/>

**Footnote / Further Reading:**

- https://medium.com/100-days-of-linux/chroot-a-linux-wonder-fc36ed08087e
- https://btholt.github.io/complete-intro-to-containers/chroot
