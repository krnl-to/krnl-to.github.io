+++
title = "Linux Containers Internals: chroot Again"
date = "2023-02-11T17:31:07+03:00"
author = "Anthony Nandaa"
authorTwitter = "profnandaa" #do not include @
cover = ""
tags = ["linux", "containers", "lci-series"] #lci -> linux containers internals
keywords = ["linux", "containers", "chroot"]
description = "Part 2 of chroot, using a full file system as opposed to the hacky solution we used in part 1."
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

# Intro to Linux Containers Internals: `chroot` Again

In our [last post](../linux-containers-intro-chroot/) of the series [#lci-series](/tags/lci-series/), we scratched the surface of what `chroot` does. It was great to get some feedback on how this lit some light bulbs. We can now agree on the definition that _a container is a set of 1 or more processes isolated from the rest of the system._ We saw this when we could isolate `ls` and `bash` just within our `container-intro` directory. This was just one aspect of isolation - filesystem isolation.

> By the way, `chroot` was introduced in Unix around 1979. That's how old this concept is, and that marked the early beginnings of containers as we know them today. You can read more about the history here - [A Brief History of Containers: From the 1970s Till Now](https://blog.aquasec.com/a-brief-history-of-containers-from-1970s-chroot-to-docker-2016).

In this series, we would like to explore `chroot` further, this time with a _full_ filesystem, as opposed to the hacky approach we used previously.

On your Linux system, start by installing [`debootstrap`](https://manpages.debian.org/stretch/debootstrap/debootstrap.8.en.html).

```bash
sudo apt install -y binutils debootstrap
```

Next, let's create our directory for the _root filesystem_. If you are using WSL on Windows like me, make sure you are working from the linux filesystem and not the "Windows mount", i.e. `/mnt/c/...`. Therefore, you can `cd $HOME` and work from there.

```bash
# assuming you are in $HOME
sudo mkdir ubuntufs
```

Let's use `debootstrap` to create a Ubuntu 22.04 (`jammy`).

```bash
# jammy is the name for Ubuntu 22.04, see https://wiki.ubuntu.com/Releases
sudo debootstrap --arch amd64 jammy ./ubuntufs http://archive.ubuntu.com/ubuntu
# after running, the last lins should read something like:
# ...
# I: Base system installed successfully.
```

> ℹ️ **Note:** another way of getting a filesystem without using `debootstrap` is exporting a Docker container and then extracting it with `tar`, e.g. `docker export $(docker create ubuntu) | tar -C rootfs -xvf -`. However, we said we don't want to use Docker for demystification.

Now let's `chroot` into the directory:

```bash
sudo chroot ubuntufs
```

And we are in:

```
root@nandaa-x1:/# ls
bin boot dev etc home lib lib32 ...
```

Let's run `ps` to list processes, however, you will need to first _mount_ `/proc`.

> `/proc` is a virtual filesystem, sometimes referred to as a _process information pseudo-file system_; that contains runtime system information (e.g. system memory, devices mounted, hardware configuration, etc). You can [read more about it here](https://tldp.org/LDP/Linux-Filesystem-Hierarchy/html/proc.html).

```
root@nandaa-x1:/# ps
Error, do this: mount -t proc proc /proc
root@nandaa-x1:/# mount -t proc proc /proc

root@nandaa-x1:/# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   1744  1080 ?        Sl   15:27   0:00 /init
root         7  0.0  0.0   1752    68 ?        Ss   15:27   0:00 /init
root         8  0.0  0.0   1752    76 ?        S    15:27   0:00 /init
1000         9  0.0  0.0   6332  5368 ?        Ss   15:27   0:00 -bash
root     14351  0.0  0.0   8912  5564 ?        R+   15:55   0:00 sudo chroot co
root     14352  0.0  0.0   8912   880 ?        Ss   15:55   0:00 sudo chroot co
root     14353  0.0  0.0   5040  4000 ?        S    15:55   0:00 /bin/bash -i
root     14365  0.0  0.0   1752    68 ?        Ss   15:58   0:00 /init
root     14366  0.0  0.0   1752    76 ?        S    15:58   0:00 /init
1000     14367  0.0  0.0   6072  5180 ?        Ss+  15:58   0:00 -bash
root     14380  0.0  0.0   7784  3300 ?        R+   15:58   0:00 ps aux
```

In a new tab, if you ran `ps aux` on the main system, you should see a similar output. This is because we have not yet achieved _namespace isolation_, only _filesystem isolation_. We will cover _namespace isolation_ in our next episode.

> _Thanks for reading this far; here is [another small gift for you](https://www.youtube.com/watch?v=KFxPr-_v3AI)_ 🙂
