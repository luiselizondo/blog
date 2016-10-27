---
layout: post
title: A production ready Docker workflow. One Year Later.
---

Is been a year since I originally wrote this series of posts which had great success in terms of traffic. One year is a lot of time so a user on Twitter suggested that I should give an update.

I must say that I no longer work for the company where I originally implemented this, but AFAIK they're still using the same model, the same applications and everything is still working.

The Docker community has grown a lot the past year and the tools are more mature. Fig is now docker-compose, docker-machine didn't exist the last time I wrote about this and neither docker swarm.

The ecosystem grew as well and other tools consolidated. I don't really know what happened with Kubernetes and if they solved the storage problem and some of the UX issues I originally complained about.

One of the new players, who in my opinion got right the orchestration is [Mesosphere](#https://mesosphere.com/). They are the creators of [Marathon](https://github.com/mesosphere/marathon) which is an incredible and simple to use orchestration system for Docker.

Marathon is build on top of Apache Mesos, which is also Open Source. It has a great UI and great features like a REST API to manage the containers. Marathon also takes care of restarting containers if they die so you don't need Supervisor anymore running inside your container.

The storage problem is no longer an issue since Marathon can share volumes between the host and the container but also you can limit where to run a container by IP very easily.

Service Discovery is also something [Marathon has](https://mesosphere.github.io/marathon/docs/service-discovery-load-balancing.html) so you won't be needing Consul anymore.

As for the workflow, still applies, Jenkins can build an image and send it to the registry and then Marathon can do the rest, you just need to let Marathon know you want to start containers using the REST API.

So that's it. If I had to do it al over again I would be using Apache Mesos and Marathon because of the simplicity of the approach. The only downside is that you need the resources to run Apache Mesos Master, Apache Mesos Slaves, Marathon and Zookeeper. If you have enough compute power, go for it.