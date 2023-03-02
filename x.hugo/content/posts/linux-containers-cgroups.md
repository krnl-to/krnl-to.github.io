+++
title = "Linux Containers Internals: Cgroups"
date = "2023-03-01T11:14:02+03:00"
author = "Ishuah Kariuki"
authorTwitter = "ishuah_" #do not include @
cover = ""
tags = ["linux", "containers", "lci-series"] #lci -> linux containers internals
keywords = ["linux", "containers", "chroot", "unshare", "namespaces", "cgroups"]
description = "Introducing the Linux Control Groups"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

# Intro to Linux Containers Internals: cgroups

This post marks the fourth entry in the [#lci-series](/tags/lci-series/). In our [last post](../linux-containers-namespaces/), we achieved namespace isolation for the processes running in our container. However, these processes have access to all the available physical resources. For instance, a long-running process can use up all the available memory. This article will explore how to sandbox our container to a limited set of physical resources (RAM, CPU).

## Working Without Control Groups

Let's first understand why cgroups exist. We'll start by installing and running htop on our host.

```bash
# run this on the host
$ sudo apt-get install htop
# after htop is installed, run it as root
# this is so that you can kill rogue processes on the container
$ sudo htop
```

If your container is not running, start it by running this command in a separate terminal session. (This is the command from [Linux Containers Internals: Namespaces](../linux-containers-namespaces/))

```bash
$ sudo unshare --mount --uts --ipc --net --pid --user --fork --map-root-user chroot ubuntufs
# once you're in the container, mount proc with the next command
$ mount -t proc proc /proc
```

Next, let's start a pointless, long-running process (in our container) that will hog memory and CPU cycles.

```bash
# run this command on your container
$ cat /dev/urandom | tr -dc 'a-z' | head -c 1048576000 | grep '[a-z]'

# the command explained:
# cat /dev/urandom - read a stream of random data from /dev/urandom
# tr -dc 'a-z' - remove all but the characters specified in 'a-z' from the first command
# head -c 1048576000 - read the first 1 GB of the preceding output
# grep '[a-z]' - match at least one letter from the preceding output
```

Back on `htop` (running on your host), sort processes by CPU percentage usage. You should see `grep --color=auto [a-z]` at the top:

![image](https://user-images.githubusercontent.com/321040/222208515-72218846-ba02-4729-abf6-bdf12e35b91c.png)

You'll notice memory consumption grows steadily, depending on your RAM size. This process uses about 1.7% of my memory when left to run for 187 seconds. This usage grows until you either run out of memory or terminate the process. For sanity's sake, kill the process. You can do this on htop or by running `kill -9 <LONG-RUNNING_PROCESS_PID>`.

## Implementing cgroups

We've experienced ungovernable chaos - let's fix it with `cgroups`.

> [`Cgroups` (aka `Control Groups`)](https://man7.org/linux/man-pages/man7/cgroups.7.html) is a Linux kernel feature that isolates, limits, and tracks resource usage of groups of processes. [You can read more about cgroups here](https://kernel.googlesource.com/pub/scm/linux/kernel/git/glommer/memcg/+/cpu_stat/Documentation/cgroups/cgroups.txt).

On your host, quit htop (confirm you've killed the long-running process first) and install `cgroup-tools`.

```bash
# run this command on your host
$ sudo apt-get install -y cgroup-tools
```

Let's create a new cgroup called sandbox and add our container to this new cgroup.

```bash
# run these commands on your host
# create new cgroup
$ sudo cgcreate -g cpu,memory,blkio,devices,freezer:/sandbox
# get the PID of our container bash process
# that's the PID of the child bash process under unshare (1552)
$ pstree -p
init(1)─┬─init(4)───{init}(5)
        ├─init(512)───init(513)───bash(514)───sudo(1549)───sudo(1550)───unshare(1551)───bash(1552)
        ├─init(570)───init(571)───bash(572)───pstree(1695)
        └─{init}(6)
$ sudo cgclassify -g cpu,memory,blkio,devices,freezer:sandbox 1552
# confirm that our container has been added to the sandbox cpu group
$ cat /sys/fs/cgroup/cpu/sandbox/tasks
1552
```

The `cgcreate` command creates a new control group with cpu, memory, and several more resource controllers. [You can read more about cgroup resource controllers here](https://kernel.googlesource.com/pub/scm/linux/kernel/git/glommer/memcg/+/cpu_stat/Documentation/cgroups).

Great! We have attached our container to our new cgroup. Now we can set resource limits. We'll start by limiting memory. I have about 23.4 GB of memory available on my Linux host. Let's set the limit to 128 MB, approximately ~0.5% of my total memory.

```bash
# run this command on your host
$ sudo cgset -r memory.limit_in_bytes=128M sandbox
# confirm that the limit is set
$ cgget -r memory.stat sandbox | grep memory_limit
hierarchical_memory_limit 134217728
# 134217728 bytes = 128 MB
```

Let's restart the experiment - run htop on the host, then start the long-running process on the container.

```bash
# run this command on your container
$ cat /dev/urandom | tr -dc 'a-z' | head -c 1048576000 | grep '[a-z]'
```

```bash
# run this command on your host
$ sudo htop
```

These are the results from my `htop`:

![image](https://user-images.githubusercontent.com/321040/222223829-708202fc-a49d-4d75-8c8e-5980ffb7c78d.png)

Memory usage never goes above 0.5%. Memory limit achieved! CPU usage started at 100% and then dropped to 15%-20% as soon as the process maxed the memory limit. Let's limit CPU usage to 5%.

> Note: In the htop output above, `cat /dev/urandom` has 4.1% CPU usage, and `tr -dc a-z` has 2.7% CPU usage.

```bash
# run this command on your host
$ sudo cgset -r cpu.cfs_period_us=100000 -r cpu.cfs_quota_us=5000 sandbox
```

Let's break down how the above command accomplishes 5%. The parameter cpu.cfs_period_us sets the duration of a cgroup's CPU scheduling period. The cpu.cfs_quota_us sets the maximum CPU time that all the tasks in a cgroup can run during one scheduling period. It doesn't matter if there are two processes or a hundred; they will all adhere to this limit. Using these two parameters, you can summarize the CPU percentage limit as follows:

```
cpu.cfs_quota_us \ cpu.cfs_period_us = CPU percentage limit.
```

Substituting the values in the above command, we get:

```
5000 \ 100000 = 0.05 == 5/100 == 5%
```

Let's run the experiment one last time - run htop on the host, then start the long-running process on the container. Here's my `htop` output:

![image](https://user-images.githubusercontent.com/321040/222227792-7b92c2c9-fd2b-4dad-bfcc-a02da9fa84a3.png)

Let's examine the four container processes:

```bash
process                     CPU usage
grep --color=auto '[a-z]'   2.7 %
cat /dev/urandom            1.3 % 
tr -dc 'a-z'                0.7 %
head -c 1048576000          0.0 %
---------------------------------
total                       4.7 %
```

CPU percentage usage never goes above 5%! CPU limit achieved! We have successfully limited the CPU and memory resources available to our container.

### Next Episode: Spoiler Alert

It has taken four posts and several command line utilities to set up a resource-isolated, namespaced container. In the next episode, we'll speed up our container creation process using `runc`.

> _Thanks for reading this far. We hope you enjoy this [small gift](https://www.youtube.com/watch?v=FE1vSjikzEQ)!_ 😀
