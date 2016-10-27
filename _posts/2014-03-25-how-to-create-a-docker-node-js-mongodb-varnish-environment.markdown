---
layout: post
title: How to create a Docker + Node.js + MongoDB + Varnish environment
date: '2014-03-25 00:22:48'
tags:
- docker
- mongodb
- node-js
- vagrant
- varnish
---

<p>I started learning <a href="https://www.docker.io/" target="_blank">Docker.io</a> over the weekend and I must say that it's really cool, unfortunatelly, this is still a new project and you don't find lots of documentation and tutorials online. At first, I struggled a little to do what I'm gonna show you, but here's the whole explanation so you don't have to. We're going to create a simple blog application written in Node.js running on a MongoDB database with a Varnish Load Balancer (covered on a different post) on top of the Node.js instances. We're going to use Docker and I strongly recommend to use Vagrant too, so you don't mess around too much with your host. In this tutorial, I won't cover installing Docker or the basics, since there's enough of that out there.</p>

<p>First we need to create a new directory, in this directory I'll download 3 Git projects:</p>

<ul>
<li><a target="_blank" href="https://github.com/luiselizondo/blog-example">A simple Blog application made with Node.js and Express.js MVC</a></li>
<li><a target="_blank" href="https://github.com/luiselizondo/docker-nodejs">The Docker Node.js image</a></li>
<li><a target="_blank" href="https://github.com/luiselizondo/docker-mongo">The Docker MongoDB image</a></li>
</ul>

<p>
After you download everything, you can take a look at the Dockerfiles, I'll explain a little what are they doing and how to run them.</p>

<h3>Node.js Docker image</h3>

<blockcode>
FROM ubuntu:12.04
MAINTAINER Luis Elizondo "lelizondo@gmail.com"
RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list
RUN apt-get updateRUN apt-get install -y python-software-properties curl git
RUN add-apt-repository -y ppa:chris-lea/node.js
RUN apt-get -qq update
RUN apt-get install -y nodejs
RUN npm install -g expressjsmvc express nodemon bower
EXPOSE 3000
ADD start.sh /start.sh
RUN chmod +x /start.sh
CMD ["/start.sh"]
</blockcode>

This one has a Dockerfile, a README, a run.sh file and a start.sh file. The start.sh file is intended to be used *inside* the container so you won't really be using it but it's important that you don't modify it unless you know what you're doing.

The run.sh file is there so you can type 'sh run.sh' instead of the whole docker command which can get really long.

The Dockerfile will install nodejs, use npm to install expressjsmvc, express, bower and nodemon; and then it will expose the port 3000 before adding the 'start.sh' file to the container and then run it.

Now, I must explain something that I struggled with a little bit. When you're using Dockerfiles (by now you should know what are they) you basically build your image so you can run containers and this container will run as specified. What this basically means is that you can create a container that will run a command as soon as it's created *or* you can make this commands optional.

This is a huge difference and it really depends on the service you're configuring. When you use the ENTRYPOINT property in your Dockerfile basically you're telling the image to run that command as soon as the container is created, so if you do something like this:

Dockerfile:

<code>
ENTRYPOINT ["/start.sh"]
</code>

It means that the container will run the 'start.sh' file when it's created. If you later want to do something like: 

<code>
$ docker run -it luis/nodejs bash
</code>

To access the container, it will not work, since the container will just ignore everything and will run 'start.sh'

So in order to have both, sometimes you'll need to do CMD instead of ENTRYPOINT, this way, the container will run the command if you don't pass any commands, but if you do pass any commands, it will run them.

If I replace my Dockerfile with

<code>
CMD ["/start.sh"]
</code>

Then

<code>
$ docker run -it luis/nodejs bash
</code>

Will run bash, and:

<code>
$ docker run -it luis/nodejs
</code>

Will run start.sh

<h3>MongoDB Docker image</h3>
<code>
FROM ubuntu:12.04
MAINTAINER Luis Elizondo, lelizondo@gmail.com
RUN apt-get update

################## BEGIN INSTALLATION
####################### Install MongoDB Following the Instructions at MongoDB Docs
# Ref: http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/

# Add the package verification key
RUN apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10

# Add MongoDB to the repository sources list
RUN echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | tee /etc/apt/sources.list.d/mongodb.list

# Update the repository sources list once more
RUN apt-get update

# Install MongoDB package (.deb)
RUN apt-get install -y mongodb-10gen

# Create the default data directory
RUN mkdir -p /data/db

##################### INSTALLATION END #####################
# Expose the default port
EXPOSE 27017

# Default port to execute the entrypoint (MongoDB)
CMD ["--port 27017"]

# Set default container command
ENTRYPOINT /usr/bin/mongod
</code>

This one is easier, it will expose two ports and then run the mongod service using ENTRYPOINT, so you cannot really access the container unless you rewrite the entrypoint.

<h3>Build the images</h3>
The next step is to build your images, go to each directory containing a docker image and run:

<code>
$ docker build -t myname/image-name .
</code>

That command will build the image and tag it with a name, these are the commands I used:

<code>
$ docker build -t luis/nodejs .
$ docker build -t luis/mongodb .
</code>

If I want to list all my images, I just do:

<code>
$ docker images

REPOSITORY          TAG                 IMAGE ID                        
luis/nodejs         latest              3e9589892ef9                 
luis/mongodb        latest              79868a4506c7                 
</code>

<h3>How to link my docker containers?</h3>
By now, you should be able to run your containers really easy and they should work, but they're not linked, we need the Node container to access the MongoDB container. The first thing we need to do is to start a MongoDB container.

<code>
$ docker run -itd -p 27017 --name mongodb luis/mongodb
</code>

When we run this command, docker will create a new container with the image "luis/mongodb" and name that container as "mongodb", it will also link the port 27017 to whatever port the container exposes, and it will also run this container as a daemon. This is very important since we want the container to start and keep running.

 <h3>Wait, what about files?</h3>
Both MongoDB and your application need to read/write data on the HD, and you probably want to persist that data outside the container, which is disposable. The solution is to link volumes. First, let's do this for MongoDB.

By default, MongoDB, inside the container, will save the data in /data/db, and that's fine, we actually created that directory inside the container on the Dockerfile. MongoDB will think is saving the data in /data/db but in reality, it will be saving the data outside the container in a directory we specify.

Let's create a new folder to save the data *outside* the container.

<code>
$ sudo mkdir -p /var/mongodb
</code>

And now, let's fool MongoDB to save the data in /var/mongodb

<code>
$ docker run -itd -p 27017 -v /var/mongodb:/data/db --name mongodb luis/mongodb
</code>

What we're doing differently is to link /var/mongodb (outside the container) to /data/db (inside the container). 

The same principle will apply to your application files. 

<h3>Linking containers</h3>
Now we can go back to linking our containers. First, make sure you clone the application, in my case, I'm using a Node.js application that I created earlier, my files are at /home/luis/Docker/blog-example

Now, to run my node.js container linked to MongoDB all I have to do is:

<code>
$ docker run -itd -p 8000:3000 --name nodejs --link mongodb:mongodb -v /home/luis/Docker/blog-example:/var/www luis/nodejs
</code>

Let's explain what we're doing with that command. First, we know my container will expose the port 3000, so we're redirecting that port to the port 8000 (eventually we'll have to modify this but for now we're OK). Second, we set a name for the container. Third, we link the mongodb container, which basically allows the Node.js container to access the MongoDB container. Finally, we set the real path for /var/www, fooling our container for the real location of our files.

<h3>My application is not working</h3>
The application needs to install some stuff before running, so we're going to need to install the dependencies before we run it.

Let's kill the container first:

<code>
$ docker rm -f nodejs
</code>

And now, let's bash into it:

<code>
$ docker run -it -p 8000:3000 --link mongodb:mongodb -v /home/luis/Docker/blog-example:/var/www luis/nodejs bash
</code>

Notice that we're not daemonizing the container so we can actually access it.

If we list all the files in /var/www we'll see that our application is there:

<code>
$ ls -la /var/www
drwxr-xr-x  7 1000 1000 4096 Mar 25 05:09 .
drwxr-xr-x 20 root root 4096 Mar 25 05:20 ..
-rw-r--r--  1 1000 1000   34 Mar 25 05:09 .bowerrc
drwxr-xr-x  8 1000 1000 4096 Mar 25 05:09 .git
-rw-r--r--  1 1000 1000   44 Mar 25 05:09 .gitignore
-rw-r--r--  1 1000 1000 1377 Mar 25 05:09 app.js
-rw-r--r--  1 1000 1000  260 Mar 25 05:09 bower.json
drwxr-xr-x  4 1000 1000 4096 Mar 25 05:09 components
-rw-r--r--  1 1000 1000  169 Mar 25 05:09 expressjsmvc.json
drwxr-xr-x  2 1000 1000 4096 Mar 25 05:09 lib
-rw-r--r--  1 1000 1000  327 Mar 25 05:09 package.json
drwxr-xr-x  4 1000 1000 4096 Mar 25 05:09 public
-rw-r--r--  1 1000 1000  824 Mar 25 05:09 start.js
drwxr-xr-x  2 1000 1000 4096 Mar 25 05:09 views
</code>

<h3>Environment variables</h3>
Before I install everything, I want to show you a cool thing call environment variables. These are variables that are accessible by your system, do list them just do:

<code>
$ env
HOSTNAME=7921e0543e40
MONGODB_NAME=/focused_engelbart/mongodb
MONGODB_PORT_27017_TCP=tcp://172.17.0.2:27017
TERM=xterm
MONGODB_PORT=tcp://172.17.0.2:27017
MONGODB_PORT_27017_TCP_PORT=27017
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/
MONGODB_PORT_27017_TCP_PROTO=tcp
SHLVL=1
HOME=/
MONGODB_PORT_27017_TCP_ADDR=172.17.0.2
_=/usr/bin/env
</code>

As you can see, I have several variables that reference MongoDB, this is because of the link we created. If we take a look at our application we can see that we're using those variables.

<code>
var address = process.env.MONGODB_PORT_27017_TCP_ADDR;
var port = process.env.MONGODB_PORT_27017_TCP_PORT;
mongoose.connect("mongodb://" + address + ":" + port + "/blog");
</code>

Now let's install all dependencies:

<code>
$ cd /var/www
$ npm install ; expressjsmvc install
</code>

Now, let's just exit our container:

<code>
$ exit
</code>

And let's run it again:

<code>
$ docker run -itd -p 8000:3000 --name nodejs --link mongodb:mongodb -v /home/luis/Docker/blog-example:/var/www luis/nodejs
</code>

Now, let's see what's going on inside the container with:

<code>
$ docker logs nodejs
</code>

And finally, let's open the browser and go to http://localhost:8000

You should see out application up and running. You can add a new blog post going to http://localhost:8000/blogs/add

<h3>What about Varnish?</h3>
This blog post already got too long, so that's going to have to wait until the next post.
