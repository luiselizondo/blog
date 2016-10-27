---
layout: post
title: 'A production ready Docker workflow. Part 3: Orchestration tools'
date: '2015-03-27 16:07:54'
description: This is the third part of a series of posts about how we're using Docker in production at IIIEPE.
---

This is the third part of a series of posts about how we're using Docker in production at [IIIEPE](https://iiiepe.edu.mx). If you haven't, please read [Part 1](http://www.luiselizondo.net/a-production-ready-docker-workflow/) and [Part 2](http://www.luiselizondo.net/a-production-ready-docker-workflow-part-2-the-storage-problem/) before continuing. In this post, I'll discuss what orchestration tools we tested, which one we're using and why. I'll also explain how we are using Jenkins to do the hard work for us and how it's organised.

Using Docker is really cool, it solved several problems on our workflow, but it created others. Managing containers can be as hard, if not even harder, than managing a few VMs, so if you're not using an Orchestration tool to do it, you're doing it wrong. Once your containers start growing, it'll be really hard to manage.

All the base images that we open sourced at [Docker Hub](https://hub.docker.com/u/iiiepe) use Supervisor. Supervisor is a beautiful piece of software that manages processes, if a process dies, supervisor will restart it. Because the way Docker works, the container needs to have a process running in order for the container to stay alive. The process cannot be demonised and if the process dies, the container dies too and you'll have to find a way to restart the container. This is a problem you probably want to avoid in the first place. The great thing about supervisor is that it can handle several processes, so a container running a PHP application will run PHP-FPM, Nginx and Sendmail.

#### Orchestration tools
For about two weeks we did nothing but test a list of orchestration solutions. Our list included:

- Deis
- Shipyard
- Panamax
- Kubernetes
- Tsuru.io
- Decking.io
- Maestro-ng

I don't want to go over an extensive review of each of those solutions, so I'll be brief.

#### Deis
PasS solution that uses Docker. It is basically Heroku with Docker. Easy to install, but it's not really flexible when it comes to storage. This was the most attractive but it was not a solution for us because we use Drupal. Also, no UI.

#### Shipyard
We ended using Shipyard in the end but only as a viewer. Shipyard is still under heavy development and the biggest problem it has is that it doesn't provide a way to easily manage containers automatically. As I said, we use it only as a viewer to monitor the status of all of our containers and our Docker services. If a container crashes, instead of rebooting all of the containers of that application, we just restart the dead container with Shipyard.

#### Panamax
Promising but it wasn't ready when we needed it. It also depends heavily on some kind of templates which I personally didn't like. The lack of an agent when we tested it was a major blocker. Basically, with no agent we'd have to install Panamax on each server.

#### Kubernetes
PaaS solution, the hardest to install and configure of the list. It has many more features than we needed but it lacked the one feature that we needed, Kubernetes doesn't handle storage.

#### Tsuru.io
PasS solution, which states:

> ... Amazon S3 ... It’s the right way to store content files into tsuru.

We didn't even try to install it after reading that.

#### Decking.io
Is kind of a replacement for Fig, not having multi-host capabilities was the biggest issue.

#### Maestro-NG
Maestro-NG was the winner for several reasons, it was easy to use, has a CLI with very simple commands, does multi-host and everything is described as a YAML file.

We setup a server in which we installed Maestro-NG, since we needed to open the Docker port on each web node, only Maestro-NG can connect to Docker for security reasons. Then we organised all of the maestro files in a single git project. The project is organised using directories with the FQDN of the application, inside every directory there’s a single file, a maestro.yaml file.

```
/subdomain.example.com/maestro.yaml
/another-example.example.com/maestro.yaml
```

If we need to test a particular project (we don’t really do that kind of testing), we just create a new maestro file and push it, then we just treat it like any other project.

With Maestro-NG, our Continuous Delivery process is reduced to two single commands:

```
maestro pull ; maestro restart
```

Since we waste precious seconds doing that, we let Jenkins do this for us.

### Jenkins
We didn't use Jenkins before so CI and CD was not a practice we had implemented, everything was manual and error prone. Creating a new workflow was the perfect opportunity to bring it into the mix.

All of our projects have the same workflow:

1. A push is detected by Gitlab
2. Gitlab triggers a web hook and orders Jenkins to start a new Job.
3. Jenkins clones the latest version of the project.
4. Jenkins runs tests.
5. If tests are passing, then Jenkins starts to build a new docker image.
6. If the image finishes building, Jenkins pushes the image to our private registry.
7. Jenkins connects to the Maestro-NG server using SSH and runs the command ``` maestro pull ; maestro restart ```

This whole process takes small projects that are not tested less than 2 minutes, some projects are even faster, between 25 and 35 seconds. On the biggest project, which is a public project that gets pushed to Docker Hub, it takes about 6 minutes. We do have one exception, which is a project that takes 18-20 minutes to build, this is an old HTML website with lots of videos and big files, the whole project is about 1.8 GB in size so that's why it takes too long to build.

When we started to configure all the VMs we needed, we decided to install Jenkins on the same VM as Docker Registry. We did this for two reasons. The first reason is that this VM has lots of HD space in it, enough for both, while our web nodes are small in size, this VM is relatively bigger. The second reason to install both the Docker registry and Jenkins on the same VM was to reduce transfer times when pushing the image to the registry. This has worked well for us.

#### Jenkins tasks
For regular, non tested applications, Jenkins runs the following shell task:

```
docker build —tag=our-private-docker-registry/application_name --no-cache .
docker login --username="someusername" --password="somepassword" --email="someemail" https://our-private-docker-registry.fqdn
docker push our-private-docker-registry/application_name
```

For applications that are tested, Jenkins does the following:

```
make prepare-test
sleep 90
make install
make test
make clean-test
docker build --tag=iiieperobot/dashi3 .
docker login --username="iiieperobot" --email="someemail" --password="somepassword"
docker push iiieperobot/dashi3
```

In the example above, we push directly to the Docker Hub, but 99% of the tasks push to our private Docker Registry.

After the shell script is run, Jenkins connects to the Maestro-NG server using SSH and runs:

```
cd /path_to_maestro/application_fqdn ; maestro pull ; maestro restart
```

#### Re-building base images
When a new base image is rebuilt, we need to rebuild all images that depend on that base image, so we also have a task for each of the base images:

```
docker pull iiiepe/nginx-drupal
```

And we have a Post-build action to build all the projects that depend on that image.

### Testing
While I was in the middle of writing this post, I was asked if Docker was used to test our projects. You bet it is, when we need to. Sometimes we do test against a database instead of mocking, when we do this kind of testing, we use the process I described above, and to simplify it, we use Makefiles, so both Jenkins and developers can both run ```make test``` to run tests.

That's it for now, on the next and final post, I'll talk about Service Discovery and Load Balancing.

### Update
Part 4 is out, you can read it [here](http://www.luiselizondo.net/a-production-ready-docker-workflow-part-4-service-discovery-and-the-load-balancer/)
