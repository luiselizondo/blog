---
layout: post
title: 'A production ready #Docker workflow. Part 2: The Storage Problem'
date: '2015-03-23 18:39:56'
description: A few days ago I wrote about how we at IIIEPE started using Docker in production. It really changed the way we develop and deploy web based applications. In this series of posts I'll keep writing about our experience and the problems we faced and how we solved them.
---

A few days ago I [wrote](http://www.luiselizondo.net/a-production-ready-docker-workflow/) about how we at [IIIEPE](https://iiiepe.edu.mx) started using Docker in production. It really changed the way we develop and deploy web based applications. In this series of posts I'll keep writing about our experience and the problems we faced and how we solved them.

The last post didn't have as much context as you'd probably needed to understand why we did what we did, but if you're still reading, allow me to explain.

IIIEPE is a Government Institute and we love [Open Source](https://github.com/iiiepe). We develop most of our projects using Drupal and Node.js and we have our own Datacenter. As of this writing, we have 25 web applications, but only 8-10 of them receive commits on a daily basis. Our previous workflow was very 2005, we developed on a totally different environment than the one in production (sounds familiar?), so we usually ran the application on a testing environment to detect problems. The funny thing about testing environments is that testing VMs are never the same as production VMs.

As we began planning the migration and the new workflow, we created a wish list which included replication of 100% of our websites to reduce downtime to a minimum during server maintenance, even the HTML websites that don't receive too much traffic would be running on at least 2 instances running on different VMs. Having an elastic cloud was another item in the wish list since we have some traffic spikes every now and then.

Since many of our websites are made with Drupal, this really presented us with many challenges when we tested the migration, the biggest of them was the _Storage Problem_

We do not have a storage solution like AWS S3 nor we have the money to pay Amazon for GBs of data so we tested [LibreS3](http://www.skylable.com/products/). LibreS3 is a great API-compatible S3 clone once you sort the DNS problems.

But Drupal sometimes will give you a hard time. Drupal just won't play nice with S3 so even if we wanted to use S3, we had to invest months tweaking Drupal and we just didn't have the time nor the patience to do it.

Before we moved everything to production, we used DigitalOcean to test PasS solutions like [Deis](http://deis.io) and orchestration software like [Panamax](http://panamax.io/). Deis was a not too hard to configure and it was really easy to use. The biggest problem with Deis and some of the other orchestration solutions we tested was the Storage problem, actually, the heroku-like ephemeral filesystem. Panamax was not ready for us at the time because it didn't have an agent to support a multi-host environment.

## GlusterFS
Before GlusterFS we used NFS, but when we configured it years ago we made the mistake of only configuring one NFS VM. We also made the mistake of never upgrading the OS of the machine to the next major version so by the time we wanted to do it was too late. We heard nice things about GlusterFS and since now it was necessary to reinstall NFS, we decided to try GlusterFS before.

GlusterFS was surprisingly easy to configure. We ended up with 2 GlusterFS VMs, one is a replica of the other and if one goes down, the other immediately and transparently will start to serve files. If we need to expand our storage solution in the future, it's just a matter of creating new replicas with bigger space and then shutting down the old ones.

With GlusterFS configured and our storage layer figured out, we just had to connect each of the web nodes, which was really easy. Then we just had to introduce Docker into all of this.

## Volumes
Drupal writes all kinds of files in the directory _sites/default/files_. The problem with this is that the files are inside the web application. On Drupal, the main index file is at _/var/www/index.php_ and the files are at _/var/www/sites/default/files_, as you can see, there's no separation of concerns, files and code living side by side with just a couple of directories in the way. Even the .gitignore that comes with Drupal includes this directory so the files aren't committed.

Moving _sites/default/files_ out of /var/www was not really an option because Drupal will fight you, if you want to try it, save yourself the trouble and the time.

So we had to share.

I wrote about each application having a Dockerfile, and this Dockerfile uses a base image. When we develop an application, we share the _application_ folder so the code lives in _/var/www_ and when Jenkins builds the image it copies the same _application_ folder to _/var/www_. From the perspective of the container, it's just the same code when we share or copy and from the perspective of of the developer, it's the same environment as production.

To solve the _/var/sites/default/files_ problem, we just had to share it with the host, so we added one line to the Dockerfile

```
VOLUME ["/var/www/sites/default/files"]
```

and made sure not to include the files directory with our project. Docker will just create the directory and map it to where we tell it to. Since Maestro-NG makes it really easy to do this (I'll discuss this on the next post), the rest was a walk in the park.

## GlusterFS + Docker
To made things even easier and to standardize further our workflow, the GlusterFS volume has an structure like the following:

```
/storage/files/subdomain.domain.com
/storage/logs/subdomain.domain.com
```

And each container maps the files directory like this::

```
container -> host
/var/www/sites/default/files -> /storage/files/subdomain.domain.com
```

Each of the web nodes reads and writes to this directory and since each application has one single directory to write and read files, it makes things easier to backup. All of this while fooling Drupal because it thinks it's reading and writing to _/var/www/sites/default/files_ when in fact it's reading and writing from _/storage/files/subdomain.domain.com_. Voilà!

Docker 1 - Drupal 0.

## Developer
We don't really need to download files to develop any of the websites we have, we treat them as content so they stay on the server, but we do have to recreate this structure when we boot a container. I'll talk about how we handle our databases in another post. The way we do it is by including a files directory on the root of the project and we include this directory in .gitignore so we won't commit it.

Using docker-compose (or fig) it would look like this:

```
volumes:
  - application:/var/www
  - files:/var/www/sites/default/files
```

So our application is mapped to _/var/www_ and drupal has a place where it can write files happily.

Thanks for reading, please share your thoughts or questions. On the next post I'll talk about how we configured Maestro-NG and how we let Jenkins do the hard work for us. On the final post I'll discuss what service discovery solution we're using and the issues we had with the load balancer.

##### Update
Part 3 is out, you can read it [here](http://www.luiselizondo.net/a-production-ready-docker-workflow-part-3-orchestration-tools/)
