---
layout: post
title: Debugging docker no space left on device problem
date: '2017-07-20 11:20:44'
description: A few weeks ago I started seeing an error that was preventing the Jenkins (docker) slave from running the tests of one of our projects.
tags:
- docker
- linux
- jenkins
---

At Rever we use Docker on Docker to build our images and distribute them to production. In a nutshell, Jenkins is in charge of creating a new slave using Docker, then, we start the Docker daemon inside the Jenkins (docker) slave, where we pull all the images of a project, run tests and then we build the final image that we send to our Docker Registry.

A few weeks ago I started seeing an error that was preventing the Jenkins (docker) slave from running the tests of one of our projects.

```
time="2017-07-20T03:52:58.422032126Z" level=error msg="Handler for POST /v1.24/containers/create returned error: Error processing tar file(exit status 1): write /usr/libexec/gcc/x86_64-alpine-linux-musl/5.3.0/cc1obj: no space left on device"
docker: Error response from daemon: Error processing tar file(exit status 1): write /usr/libexec/gcc/x86_64-alpine-linux-musl/5.3.0/cc1obj: no space left on device.
See 'docker run --help'.
make: *** [test] Error 125
```

This error was really annoying since when going to the Jenkins VM it showed that we have more than enough hard drive space. I started to explore the option that this was a RAM issue and I increased the RAM on our VM but nothing happened. I found on internet several posts that talked about iNodes, but that was not our problem.

After two days exploring the internet and a lot of debugging I came up with a post that point me in the right direction. Turns out that when Docker creates a new container, assigns a virtual partition with only 10 GB of space. When running Docker on Docker, we pull Docker images inside the Docker container and the device is left with no space on the virtual partition.

I ran some tests and found that that when the container starts, the `/dev/mapper/docker*` partition has 10G and 9+ GB of free space. But after pulling some images, I'm left with less than 10% of free space.

```
root@c85b9b0acbbb:~# df -h
Filesystem                                                                                        Size  Used Avail Use% Mounted on
/dev/mapper/docker-202:1-152516-66f9feb4b4c712acf1f74beee40cce2675127de8ea33ef7de213348f53325ca2   10G  934M  9.1G  10% /
tmpfs                                                                                             3.9G     0  3.9G   0% /dev
tmpfs                                                                                             3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/xvda1                                                                                         79G  7.9G   68G  11% /keys
shm                                                                                                64M     0   64M   0% /dev/shm
```

The solution was simple, I had to increase the partition and I'll get to that in a moment. Increasing the partition was not enough, since I had to remove and rebuild all the images on the main Docker daemon.

First, I had to pass the following options to the Docker Daemon:

```
DOCKER_OPTS="--storage-opt dm.basesize=20GB --storage-opt dm.min_free_space=20%"
```

Then, I started to get a weird error about Docker not running. I found out after reading the logs that the `%` on `20%` was not being parsed because systemd considers the character `%` a special character.

The final configuration I had to enter on `/etc/systemd/system/docker.service.d/docker.conf` was:

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --storage-opt dm.basesize=20GB --storage-opt dm.min_free_space=20%%
```

Notice the double %% at the end of the ExecStart configuration

I had to stop and start docker by running the following commands:

```
systemctl stop docker.service
systemctl daemon-reload
systemctl start docker.service
```

And then, I had to remove all images with:

```
docker rmi $(docker image -qa)
```

And rebuild the images that I had.

Once I had that, I tested that the configuration was working by running:

```
$ sudo docker run alpine:latest df -h
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
88286f41530e: Pull complete
Digest: sha256:1072e499f3f655a032e88542330cf75b02e7bdf673278f701d7ba61629ee3ebe
Status: Downloaded newer image for alpine:latest
Filesystem                Size      Used Available Use% Mounted on
/dev/mapper/docker-202:1-152516-94c4aa9da75e3faaa8237047f51255ddf370e849d6eb82a75d4d2323a0421b59
                         20.0G     38.3M     20.0G   0% /
tmpfs                     3.9G         0      3.9G   0% /dev
tmpfs                     3.9G         0      3.9G   0% /sys/fs/cgroup
/dev/xvda1               78.6G      7.3G     68.0G  10% /etc/resolv.conf
/dev/xvda1               78.6G      7.3G     68.0G  10% /etc/hostname
/dev/xvda1               78.6G      7.3G     68.0G  10% /etc/hosts
shm                      64.0M         0     64.0M   0% /dev/shm
tmpfs                     3.9G         0      3.9G   0% /proc/kcore
tmpfs                     3.9G         0      3.9G   0% /proc/timer_list
tmpfs                     3.9G         0      3.9G   0% /proc/timer_stats
tmpfs                     3.9G         0      3.9G   0% /proc/sched_debug
tmpfs                     3.9G         0      3.9G   0% /sys/firmware
```

Voila! The partition was increased to 20G! and no more problem.
