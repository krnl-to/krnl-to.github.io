+++
title = "Linux Containers Internals: Namespaces"
date = "2023-02-22T12:28:31+03:00"
author = "Ishuah Kariuki"
authorTwitter = "ishuah_" #do not include @
cover = ""
tags = ["linux", "containers", "lci-series"] #lci -> linux containers internals
keywords = ["linux", "containers", "chroot", "unshare", "namespaces"]
description = "Introducing the concept of Linux namespaces"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

# Intro to Linux Containers Internals: namespaces

In our [last post](../linux-container-intro-chroot-again/) of the series [#lci-series](/tags/lci-series/), we ran a container using chroot and a complete filesystem. We managed to isolate `ls` and `bash` within our `container-intro` directory. However, `ps aux` lists all the processes on the host environment because we didn't implement `namespace` isolation. This article will focus on namespaces. Why they exist and how to implement them in our container.

## Working Without Namespaces

We'll start by demonstrating the perils of using `chroot` without namespaces. If you don't have it running already, fire up the container from [Linux Containers Intro: Chroot Again](../linux-container-intro-chroot-again/) and mount the `proc` filesystem.

```bash
$ sudo chroot ubuntufs
# once you're in the container, mount proc with the next command
$ mount -t proc proc /proc
```

Let's start a foreground process on your host environment.

```bash
# run this on the host
# you can tail any of the log files in /var
# there's nothing special with bootstrap.log :)
tail -f /var/log/bootstrap.log
```

Run `ps aux` on your container.

```bash
root@container:/# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   2276  1536 ?        Sl   08:38   0:00 /init
root         4  0.0  0.0   2276     4 ?        Sl   08:38   0:00 plan9 --control-socket 5 --log-level
root         7  0.0  0.0   2280   104 ?        Ss   08:38   0:00 /init
root         8  0.0  0.0   2296   108 ?        R    08:38   0:00 /init
1000         9  0.0  0.0   6340  5280 ?        Ss   08:38   0:00 -bash
root        47  0.0  0.0   2280   104 ?        Ss   08:47   0:00 /init
root        48  0.0  0.0   2296   108 ?        S    08:47   0:00 /init
1000        49  0.0  0.0   6340  5392 ?        Ss   08:47   0:00 -bash
root     13975  0.0  0.0   8920  5340 ?        R+   08:58   0:00 sudo chroot ubuntufs
root     13976  0.0  0.0   8920   884 ?        Ss   08:58   0:00 sudo chroot ubuntufs
root     13977  0.0  0.0   5040  4048 ?        S    08:58   0:00 /bin/bash -i
1000     14184  0.0  0.0   3236  1100 ?        S+   12:33   0:00 tail -f /var/log/bootstrap.log
root     14185  0.0  0.0   7784  3396 ?        R+   12:37   0:00 ps aux
```

As we saw in the last post, this command lists all the processes in the host environment. One of these processes is the foreground process we started on the host system. In my example, the process has a PID of `14184`. Let's kill this process from the container.

```bash
# run this from the container
kill 14184
```

What happens next? The tail process in the host environment gets terminated! Are you scared yet? There's more. Let's check the hostname on your host environment:

```bash
# run this command on your host
ishuah@h0st-5ys:~$ hostname
h0st-5ys
```

Running the same command in your container will yield the same hostname. Let's change the hostname in the container.

```bash
# run these commands in your container
root@container:/# hostname
h0st-5ys
root@container:/# hostname newh0stname
root@container:/# hostname
newh0stname
```

We've successfully changed the hostname in the container to `newh0stname`. Now check the hostname on the host system.

```bash
# run this command on your host
ishuah@h0st-5ys:~$ hostname
newh0stname
```

What happened? Changing the hostname on the container affected the host env - they share the same hostname!

```bash
# run this in the container and the host
# the namespace ids are the same in both envs
# i.e 'ipc:[4026532271]' -> 'namespace:[id]'

$ ls -l /proc/$$/ns
total 0
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 16:37 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 16:37 ipc -> 'ipc:[4026532271]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 16:37 mnt -> 'mnt:[4026532282]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 16:37 net -> 'net:[4026531840]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 16:37 pid -> 'pid:[4026532284]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 16:37 pid_for_children -> 'pid:[4026532284]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 16:37 user -> 'user:[4026531837]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 16:37 uts -> 'uts:[4026532283]'
```

> Namespaces are a Linux kernel feature that partitions kernel resources to create isolated environments within a Linux system. [You can read more about namespaces here](https://man7.org/linux/man-pages/man7/namespaces.7.html).

The host env and the container share [namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html). That's why you can see the host's PIDs ([`pid` namespace](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)) and change the hostname ([`uts` namespace](https://man7.org/linux/man-pages/man7/uts_namespaces.7.html)) from the container.

## Implementing Namespaces with `unshare`

Close your container and create a new one with the following command:

```bash
sudo unshare --mount --uts --ipc --net --pid --user --fork --map-root-user chroot ubuntufs
```

[`Unshare`](https://man7.org/linux/man-pages/man1/unshare.1.html) is a Linux utility package that allows you to run a program with some namespaces 'unshared' from its parent. The command above creates a container similar to previous runs, except now we have isolated namespaces. The `--mount --uts --ipc --net --pid --user` flags instruct unshare to execute chroot in new namespaces, see summary details below from the Linux docs. I didn't include the `cgroup` namespace here - we'll go through it in the next installment of this series.

![image](https://user-images.githubusercontent.com/321040/220844132-79460e05-4370-4557-8fbd-8eb995bd7078.png)

Let's check the namespaces in both environments.

```bash
# on the host
ls -l /proc/$$/ns
total 0
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 17:06 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 17:06 ipc -> 'ipc:[4026532271]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 17:06 mnt -> 'mnt:[4026532282]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 17:06 net -> 'net:[4026531840]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 17:06 pid -> 'pid:[4026532284]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 17:06 pid_for_children -> 'pid:[4026532284]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 17:06 user -> 'user:[4026531837]'
lrwxrwxrwx 1 ishuah ishuah 0 Feb 22 17:06 uts -> 'uts:[4026532283]'
```

```bash
# on the container
# mount the proc filesystem first
 mount -t proc proc /proc
 
 # then check the namespaces
ls -l /proc/$$/ns
total 0
lrwxrwxrwx 1 root root 0 Feb 22 14:05 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Feb 22 14:05 ipc -> 'ipc:[4026532328]'
lrwxrwxrwx 1 root root 0 Feb 22 14:05 mnt -> 'mnt:[4026532326]'
lrwxrwxrwx 1 root root 0 Feb 22 14:05 net -> 'net:[4026532330]'
lrwxrwxrwx 1 root root 0 Feb 22 14:05 pid -> 'pid:[4026532329]'
lrwxrwxrwx 1 root root 0 Feb 22 14:05 pid_for_children -> 'pid:[4026532329]'
lrwxrwxrwx 1 root root 0 Feb 22 14:05 user -> 'user:[4026532325]'
lrwxrwxrwx 1 root root 0 Feb 22 14:05 uts -> 'uts:[4026532327]'
```

The two environments don't share namespaces anymore! (Except for `cgroup`, we'll get into that later).

Running the `ps` and `hostname` demos will now yield different results.

On the container, `px aux` now only lists processes within that environment.

```bash
ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   5040  3820 ?        S    16:17   0:00 /bin/bash -i
root        11  0.0  0.0   7476  3092 ?        R+   16:19   0:00 ps aux
```

Try running our `hostname` demo:

```bash
# run these commands in the chroot's env
root@container:/# hostname newh0stname
root@container:/# hostname
newh0stname
```

Then check the host env hostname:

```bash
# run this command on your host
ishuah@h0st-5ys:~$ hostname
h0st-5ys
```

Hostname changes on the container are limited to that environment. The host is not affected.

We've created a namespaced container - Which means we have **namespace isolation** for the processes running in the container.

### Next Episode: Spoiler Alert

The isolated environments still have access to all physical resources on your computer. That means your namespaced container can hog all the available RAM and CPU cycles with reckless abandon. In the next episode, we'll use `cgroups` to isolate resource usage for a group of tasks.

> _Thanks for reading this far. As culture dictates, here is a [small gift for you](https://www.youtube.com/watch?v=jiiFzKfuPMk)!_ 😀


