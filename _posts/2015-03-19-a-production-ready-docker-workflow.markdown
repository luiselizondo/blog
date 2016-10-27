---
layout: post
title: A production ready Docker workflow
date: '2015-03-19 20:03:48'
description: Docker is now 2 years old this week and here at IIIEPE have been using it in production for about 3 months. I want to share our experience and the workflow we had to design.
---

Docker is now 2 years old this week and here at [IIIEPE](https://iiiepe.edu.mx) have been using it in production for about 3 months. I want to share our experience and the workflow we had to design.

We run several websites using Drupal, PHP and Node.js among others, and the goal we had was to run all of our applications with Docker, so we designed the following workflow:

1. All developers use Docker to create the application.
2. Our Gitlab instance has Webhooks configured, so when a new push is detected, it will order Jenkins to run a task.
3. Each Jenkins task includes the same layout: clone the latest code from gitlab, run tests, login to our private Docker registry, build a new image with the latest code and then push the image to it.
4. Finally, Maestro-NG, our orchestration software, will deploy the new version of the image.
5. Our load balancer will detect the change and reload the new configuration.

Each of those steps required several days of planning, testing and work to design the basic guidelines.

The first thing we did was build base images that we could use for our own needs. The images are public at [Docker Hub](https://registry.hub.docker.com/repos/iiiepe/) and they include everything we need to run our applications except the application itself. Everytime we change one of the base images, we run a Jenkins task that will pull the new image and then trigger subsecuent tasks to rebuild all the images that depends on the modified base image.

After we created our images, we needed to define a standard structure for all of our applications. All of our applications are organized using the following structure:

```
/application
/logs
/files
Dockerfile
fig.yml.example
docker-compose.yml.example
Makefile
```

The _/application_ directory is the root folder of our application.

The _/logs_ and _/files_ directories are there for development purposes, just so the application has directories to write logs and files. Both directories are omitted by Git and completly excluded on production.

_Dockerfile_ is the file Jenkins uses to build the image, a developer will almost never have to interact with this file. More on this later.

_fig.yml.example_ and _docker-compose-yml.example_ are the files the developer uses to start the application. Neither of them are used on production and when a developer clones a project, he/she needs to copy the example file and fill it with his/her values.

_Makefile_ is the last piece of the puzzle, we use it so we could have a standard set of commands for all applications and at the same time, hide all kind of complexities to the developer.

###Dockerfile
The Dockerfile in each application is very similar from all of the other applications, the most important job of this file is to build the final image that includes all the code that will be deployed. Let's take a look at one example:

```
FROM iiiepe/nginx-drupal6

ENV MYSQL_ENV_MYSQL_DATABASE somedb
ENV MYSQL_ENV_MYSQL_USER root
ENV MYSQL_ENV_MYSQL_PASSWORD 123
ENV MYSQL_PORT_3306_TCP_ADDR localhost
ENV MYSQL_PORT_3306_TCP_PORT 3306
ENV BASE_URL http://example.com
ENV DRUPAL_ENVIRONMENT production

EXPOSE 80

RUN usermod -u 1000 www-data
RUN usermod -a -G users www-data

ADD ./application /var/www
RUN chown -R www-data:www-data /var/www
```

This Dockerfile depends on one of the base images we built, from there, it will only set some defaults for our environment variables, decleare the exposed port, and add the application code to /var/www.

Because of the way we built this image, the only difference between Jenkins and a developer is that while Jenkins will add the entire application directory to /var/www, a developer will only map the directories.

All this magic happens with docker composer (fig was used before it was depreciated):

````

mysql:
  image: mysql:latest
  expose:
    - "3306"
  ports:
    - "3307:3306"
  environment:
    MYSQL_DATABASE: database
    MYSQL_USER: root
    MYSQL_PASSWORD: admin123
    MYSQL_ROOT_PASSWORD: admin123
web:
  image: iiiepe/nginx-drupal6
  volumes:
    - application:/var/www
    - logs:/var/log/supervisor
    - files:/var/www/sites/default/files
  ports:
    - "80:80"
  links:
    - mysql:mysql
  environment:
    BASE_URL: http://local.iiiepe.net
    DRUPAL_ENVIRONMENT: development
```

This file is used by docker-compose to initialize. In this example, we're defining an application with two containers, a MySQL container and a Web container.

The MySQL container defines environment variables that will be used the the mysql image. We also map the port 3307 on the host to the port 3306 in the container. This allows us to access MySQL server using any client.

The web container uses the same image that will be used by Jenkins when building the final image (take a look at the Dockerfile above), but it will also share some volumes. The volumes shared between the host and the container are application, files and logs. This is actually the biggest change between the development environment and the production environment, on production the container will have the code inside the image, which allows us to start up a container on any server we want, and when developing, the directory is only shared, so any new files or changes to a file in the application folder are instantly reflected inside the container.

The BASE_URL variable points to http://local.iiiepe.net which is not a real address, it's just a way to standarize how we access the application. Since some of us use Macs and Boot2Docker, we had to use a standard address that each of us include in our /etc/hosts file.

My /etc/hosts on my Mac looks like:

```
127.0.0.1	localhost
192.168.59.103	local.iiiepe.net
```

On a Linux box, it will look like this:

```
127.0.0.1	localhost local.iiiepe.net
```

Finally, we define two more environment variables which are used to determine some settings inside the configuration file of the application.

#### Custom Settings
Drupal needs a settings.php to store database information, including passwords. This file is ignored by git so your password doesn't get commited, we decided to change this file so it uses environment variables and gets commited.

The following is the important part of the settings.php file of a Drupal 6 site:

```
$username = getenv("MYSQL_ENV_MYSQL_USER");
$password = getenv("MYSQL_ENV_MYSQL_PASSWORD");
$host = getenv("MYSQL_PORT_3306_TCP_ADDR");
$port = getenv("MYSQL_PORT_3306_TCP_PORT");
$database = getenv("MYSQL_ENV_MYSQL_DATABASE");

$db_url = 'mysql://' . $username . ':' . $password . '@' . $host . '/' . $database;
```

As you can see, no password is ever commited. Passwords and sensitive values are injected with ENV variables.

Some of our websites use ApacheSolr as a search engine, but when developing we don't want to be able to write to ApacheSolr, so we need an ENV variable like DRUPAL_ENVIRONMENT do things like the following in our settings.php file

```
$conf = array();
if(getenv("DRUPAL_ENVIRONMENT") === "development") {
	// Disable apache solr writting
	$conf["apachesolr_read_only"] = 1;
}
```

### Makefile
Using Docker can be hard because of the long commands, so docker-compose (fig) helps a lot with that. We went further to try and make things easier.

This is the Makefile we're using on a Drupal website:

```

CURRENT_DIRECTORY := $(shell pwd)

start:
	@fig up -d

clean:
	@fig rm --force

stop:
	@fig stop

status:
	@fig ps

cli:
	@fig run --rm web bash

log:
	@tail -f logs/nginx-error.log

cc:
	@fig run --rm web drush cc all

restart:
	@fig stop web
	@fig start web
	@tail -f logs/nginx-error.log

.PHONY: clean start stop status cli log cc restart
```

Using Makefiles is easier than using docker-compose or fig because we can make shortcuts like `make cc` to run very used commands like `drush cc all`.

One final note about Makefiles is that we're still using fig. Since docker-compose wasn't available when we designed this workflow, and some developers on our team still use it, we decided to just symlink docker-compose to fig, which is shorter and more practical to use anyway. After you install docker-compose, you can remove fig and create the symlink with:

```
sudo rm /usr/local/bin/fig
sudo ln -s /usr/local/bin/docker-compose /usr/local/bin/fig
```

#### Things we did wrong
We did several things wrong, and I will explain most of them on a different post, but I want to talk about one in particular. The greatest advantage of using Docker is that developers can run the application on the same environment as production with very little performance lost.

I've read some blog posts about people developing outside Docker and when they want to deploy they build the image and send it to production. If you're doing this you're doing it wrong, since the environment you're developing is not the same as the one running on production. Stop building the image everytime, instead, share volumes between the host and the container and let someone else build the image for you everytime you push.

#### To be continued...
This post is far from over but it's already too long. We still need to explain how we integrated Maestro-NG, configured Jenkins and specially how the Load Balancer works. So come back soon.

#### Update:
You can read part 2 at http://www.luiselizondo.net/a-production-ready-docker-workflow-part-2-the-storage-problem/
