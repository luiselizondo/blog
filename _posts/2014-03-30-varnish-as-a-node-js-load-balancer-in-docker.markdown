---
layout: post
title: Varnish as a Node.js Load Balancer in Docker
date: '2014-03-30 01:44:47'
tags:
- docker
- mongodb
- node-js
- varnish
---

Last week I covered <a href="http://luiselizondo.net/blogs/luis-elizondo/how-create-docker-nodejs-mongodb-varnish-environment">how to create a Docker environment using Node.js, MongoDB and Varnish as a load balancer</a>, but the post got too long and I didn't cover the Varnish part. In this post, I keep my promise and I'll talk about how to create multiple Node.js containers and a Varnish container on top of them acting as a load balancer.

You can take a look at the video or just keep reading. I'm sorry for the audio, I know it sucks.
<iframe width="480" height="360" src="//www.youtube.com/embed/4E47IJAgsZE" frameborder="0" allowfullscreen></iframe>

<p></p>

Creating the Dockerfile wasn't easy, since I wanted an easy way to add Node.js containers without having to manually add them to the Varnish configuration file. I managed to build a bash script that will parse the Node.js environment variables that are created when you link Node.js containers, and automatically create the default.vcl file that Varnish uses. However, the default.vcl file is rather simple and right now it does not add any of the other cool stuff that Varnish provides like reverse proxy.

Continuing where we left the last post, I'm just gonna clone the docker-varnish project that I created.

<code>
$ git clone https://github.com/luiselizondo/docker-varnish.git
</code>

After we clone it, we just need to build the image:

<code>
$ cd docker-varnish
$ docker build -t yourname/varnish .
</code>

By now, you should have your image, and you should have your MongoDB and Node.js containers up and running, but since we're gonna have a load balancer, it would be nice to at least, have two or maybe three Node.js containers. Let's create them:

<code>
$ docker run -itd --link mongodb:mongodb -p 8001:3000 yourname/nodejs
$ docker run -itd --link mongodb:mongodb -p 8002:3000 yourname/nodejs
</code>

With the previous commands, I'm creating two Node.js containers, the first container will redirect the port 3000 to the port 8001 on the host machine, and the same will apply for the second container, only that it will redirect the port 3000 to the port 8002. This is important since you don't want (and you can't) have two ports listening to the same thing. This is also important, because Varnish will use both of those ports to redirect traffic, but our Varnish container will actually be listening to the port 80.

<h3>The Dockerfile</h3>
Before we continue, let me explain the Dockerfile:

<code>
FROM        ubuntu
MAINTAINER  Luis Elizondo lelizondo@gmail.com
 
# Update apt sources
RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list

# Update the package repository
RUN apt-get -qq update

# Install base system
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y varnish vim git

# Make our custom VCLs available on the container
ADD default.vcl /etc/varnish/default.vcl

# Export environment variables
ENV VARNISH_PORT 80

# Expose port 80
EXPOSE 80

ADD parse /parse
ADD start /start

RUN chmod 0755 /start /parse

CMD ["/start"]
</code>

Putting aside the obvious first lines, I will focus on the last lines:

ENV VARNISH_PORT 80 is the actual port that Varnish will listen to, if you need to change it, don't forget to change the port we're exposing on EXPOSE 80.

The start and the parse files are the files we use to start Varnish. The start.sh file will call the parse file, and this one will auto-detect the environments that we pass (more on this later) and use them to create the /etc/varnish/default.vcl file.

<h3>Running the Varnish container</h3>
Before we run our Varnish container, we need to know the name of the Node.js containers that we're gonna link:

<code>
$ docker ps | grep nodejs
</code>

In the name column, select all the nodejs containers that you want to use in your load balancer. I have:

<code>
curious_torvalds
docker_rapid
</code>

Now I need to run the Varnish container and link my Node.js containers:

<code>
$ docker run -itd -p 8080:80 --link curious_torvalds:node1 --link docker_rapid:node2 yourname/varnish
</code>

A couple of things here to explain. First, the port mapping, Varnish will expose the port 80 and will be listening for requests on the port 80 inside the container, so we're mapping the port 80 inside the container to the port 8080 outside the container, this way, I will be using http://localhost:8080 to access my application through Varnish. Second, I'm linking my Node.js containers, but the "parse" file we talked about before, *needs* that the containers are named nodeN, so I'm using node1, node2 and if I add a new container, I'll have to name it node3. The naming is not consecutive, but the container needs the word "node" in the name, if you don't do this, the "parse" file will not detect the container and it will not be added to the default.vcl configuration file, therefore, it will not be added to the load balancer.

Finally, if I run some tests with Apache Benchmark, I can see the difference between serving my pages with a load balancer and not:

<a href="http://luiselizondo.net/sites/default/files/documents/Varnish-vs-Node.png"><img src="http://luiselizondo.net/sites/default/files/styles/square_thumbnail/public/documents/Varnish-vs-Node.png?itok=sknIxDRH"/></a>

(Click on the image to expand)

So that's it, I hope you have a good time using Docker. If you have any questions, please use the comments section.