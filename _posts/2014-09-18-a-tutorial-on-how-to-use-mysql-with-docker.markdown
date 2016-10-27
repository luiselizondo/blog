---
layout: post
title: A tutorial on how to use MySQL with Docker
date: '2014-09-18 05:06:51'
tags:
- docker
- mysql
---

<p><span style="line-height: 1.6em;">So you're learning how to use Docker and you want to install Wordpress, Drupal or any project that needs MySQL? Well, you've come to the right place. Docker can be hard sometimes, and when you add complexity to something that it's already hard (at least for beginners), you end up drinking more coffee than your body can take.</span></p>

<p>The thing about Docker is that you can run any service, like MySQL or Apache inside a container. You can run more than one service inside the container but it's not a good practice. You must also know that containers are pretty much disposable, and that you can run multiple containers on a machine, each one doing just one thing at a time.&nbsp;</p>

<p>MySQL as you might know is a database server, when you run the MySQL server process, you need to get into MySQL using the MySQL client and create a database, a user, grant permissions, etc. How to do this while using Docker? Well, <em>you need to create a Docker container running MySQL server and then create another container running MySQL client that can connect to the container running MySQL Server</em>.</p><p>Before I continue let me explain images. Images are like templates for your containers, there's hundreds of images for pretty much anything you can imagine. With an image everything is installed and configured for you so you can create one or many containers using the same image. You can also create your own images, but I recommend that you go step by step.</p><p>In this case we'll be using this <a href="https://registry.hub.docker.com/_/mysql/">MySQL image</a>. Remember, with the same image, we'll be running two containers, one with MySQL server running and another one with MySQL client talking to the container running the server.</p><p>The command to run MySQL server is the following:</p>

<code>docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=mysecretpassword -d mysql</code>

Let's explain each part of that command:

<ul><li><em>docker</em> is the command so there's not too much to say about it</li><li><em>run</em> is the first argument and is used to tell docker that we want to run a container</li><li><em>--name some-mysql</em> is the name argument following the name&nbsp;of the container. This is not really neccesary but it's very helpful. Since we can use the same image to run multiple containers of the same image, we need to name them. Docker is smart enough to name the containers for us in case we don't name it, but the names are random.</li><li><em>-e</em> The "-e" part is just telling docker to pass and environment variable to the container with a name and a value. With environment variables we pass information to the container that the container will use to do something. In this case to set the root password.</li><li><em>MYSQL_ROOT_PASSWORD=mysecretpassword</em> is the environment variable name and the value of the variable. You can pass any variable that you want, some containers need a variable to do something and are waiting for it. Others don't need any variables, it really depends on the image and what you want to do with the variable inside the container. In this case, as I mentioned earlier, we use them to set automatically the root password.</li><li><em>-d</em> The "-d" argument is just to tell docker that we want to daemonize the container and leave it running until it dies or until we kill it.</li><li><em>mysql</em> is the name of the image and if it doesn't exists on our computer docker will pull it from the docker public repository.</li></ul><p>Ok that part was easy, if you run that command you'll have a docker container running MySQL server. But how do we connect to it? We still need to create a database. Well, you can connect to the container using the following command:</p>

<code>docker run -it --link some-mysql:mysql --rm mysql sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'</code>

That's a little bit more complicated but all we're doing is using the same mysql&nbsp;image to create an interactive container and then&nbsp;run the command "exec mysql" with a bunch of arguments and environment variables.

You might be asking right now if you need to enter that command every time you want to connect to MySQL server. The answer is yes, but there's a better way.

Last night, I created Docker-MySQL-Scripts, a collection of 4 scripts written in python&nbsp;to helps you&nbsp;interact with a dockerized MySQL. 

You can download them at https://github.com/luiselizondo/docker-mysql-scripts

<h4>dmysql-server</h4>

Replaces the first command I explained earlier. With it you can run a MySQL container really easy. All you have to do is pass the container name and the root password:

<code>dmysql-server myappdb 123</code>

In this case, the name of the container will be "myappdb" and the root password will be 123. Easy right?

<h4>dmysql-client</h4>
Replaces the second command I mentioned earlier. With it you can run MySQL client and connect to a container running MySQL. All you have to do is pass the name of the container running MySQL Server that you want to connect to:

<code>dmysql myappdb</code>

In this case, it will connect to the MySQL server running on the container named "myappdb"

<h4>dmysql-create-database</h4>
The name says what it does, it will create a database inside the container. All you have to do is pass the MySQL&nbsp;container name you want to connect to and the name of the database you want to create.

<code>dmysql-create-database myappdb myblog</code>

In this case it will connect to the container "myappdb" and issue the command CREATE DATABASE myblog

<h4>dmysql-import-database</h4>
Again, the name says what it does, it will take a file and import it to a database. This is a little more complicated than the rest of the commands but it's easier than using a docker command. You have to pass the name of the container, the SQL file you want to import (right now it only accepts *.sql files so you'll have to ungzip them first) and the database you want to import the file into. The database is optional since the sql file can create one for you.

<code>dmysql-import-database myappdb /Users/me/myblog-monday-backup.sql --database myblog</code>

In this case, it will import the file myblog-monday-backup.sql into the database myblog running on the container myappdb.

That's it. Using those simple commands you can save yourself hours of frustration. If you need help or if you have an idea please leave a comment or even better, you can fork the project and submit a pull request.</p><p><strong>Update 10/29/2014</strong></p><p><span style="line-height: 1.6em;">The latest version of the official MySQL image now supports creating a Database user with a password and a database if the environment variables are passed. The scripts are still working but now you have another option to create a database.</span></p>