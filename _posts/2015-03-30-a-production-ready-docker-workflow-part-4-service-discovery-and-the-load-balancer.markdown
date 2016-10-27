---
layout: post
title: 'A production ready Docker workflow. Part 4: Service Discovery and the load
  balancer'
date: '2015-03-30 18:23:12'
description: This is the final part of a series of posts about how we're using Docker in production at IIIEPE
---

This is the final part of a series of posts about how we're using Docker in production at [IIIEPE](https://iiiepe.edu.mx). If you haven't, please read [Part 1](http://www.luiselizondo.net/a-production-ready-docker-workflow/), [Part 2](http://www.luiselizondo.net/a-production-ready-docker-workflow-part-2-the-storage-problem/) and [Part 3](http://www.luiselizondo.net/a-production-ready-docker-workflow-part-3-orchestration-tools) before continuing. In this post, I'll discuss how we configured Service Discovery and the load balancer.

### Service Discovery
There's plenty of Service Discovery solutions out there, but we only tested [etcd](https://github.com/coreos/etcd) and [Consul.io](https://consul.io/). Etcd is part of CoreOS, and while you can use it without CoreOS, you soon realise that you need to learn more than just etcd. Consul.io is really easy to use, runs with Docker and has a nice ecosystem.

Before I dig deeper, in case you don’t know what Service Discovery is, let me say that it has nothing to do with load balancing, is just a piece of software that knows about the current state of (in this case) the containers running across the infrastructure. That’s it, Service Discovery has nothing to do with how you send information to it.

Like any Service Discovery tool, Consul.io solves one problem, it stores the IP, port and state of an application. To register applications into Consul.io (the correct term is service) we used [Registrator](https://github.com/gliderlabs/registrator). Once Consul.io knows about the application something needs to happen, in our case, we need to reload the load balancer configuration, so we used [Consul-Template](https://github.com/hashicorp/consul-template).

Since both Consul.io and Registrator run inside Docker containers, implementing both was even easier than we thought.

As for Consul-Template, the hardest part was to figure out the template syntax.

#### Load balancer
We are using Nginx as a load balancer, since we’ve been using it for years, we know how to configure it and solutions like HAproxy just added a level of complexity to the whole workflow. Since this was the last piece of the puzzle, we just took the easy way out. Eventually, we'd like to replace Nginx with HAproxy but that will have to wait a few more weeks.

On a normal load balancing scenario, you configure Nginx like this:

```
upstream myapp {
  ip_hash;
  server 10.10.10.10:14001 fail_timeout=0;
  keepalive 64;
}

server {
  listen 80;
  server_name example.com;
  location / {
    proxy_pass          http://myapp;
  }
}
```

The problem arises when you want to dynamically assign a list of IPs and ports inside the upstream block. Using consul-template we created a template file at /etc/nginx/templates/template with blocks for each application we handle with this load balancer:

```
upstream myapp {
  ip_hash;
  {{range service "myapp"}}
  server {{.Address}}:{{.Port}} fail_timeout=0;
  {{end}}
  keepalive 64;
}

server {
  listen 80;
  server_name example.com;
  location / {
    proxy_pass          http://myapp;
  }
}
```

For websites using SSL the template is a little bigger:

```
# this part handles the list of IPs and ports
upstream myapp {
  ip_hash;
  {{range service "myapp"}}
  server {{.Address}}:{{.Port}} fail_timeout=0;
  {{end}}
  keepalive 64;
}

# this section handles requests to port 80, which we'll redirect to the port 445
server {
        listen  						80;
        server_name             		example.com;
        return 301 https://$host$request_uri;
}

server {
        listen  						443 ssl spdy;
        server_name             		example.com;
        keepalive_timeout 75 75;

        ssl                     		on;
        ssl_certificate         		/etc/nginx/cert/path_to_cert.crt;
        ssl_certificate_key     		/etc/nginx/cert/path_to_key.key;
        ssl_session_cache       		builtin:1000 shared:SSL:10m;
        ssl_protocols           		TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers             		HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
        ssl_prefer_server_ciphers       on;

        add_header              		Strict-Transport-Security max-age=31536000;

        location / {
                proxy_pass              http://myapp;
                proxy_set_header        Host $host;
                proxy_set_header        X-Forwarded-Proto $scheme;
                proxy_set_header        X-Real-IP $remote_addr;
                proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_connect_timeout   150;
                proxy_send_timeout      100;
                proxy_read_timeout      90;
                proxy_buffers           4 32k;
                client_max_body_size    500m;
                client_body_buffer_size 128k;
                proxy_redirect          http:// https://;
        }
}
```

Every time Consul-Template runs, it will query Consul.io for the service "myapp" and for each container found, it will add the IP and port to the upstream block.

Since we needed Consul.io and Consul-Template to start when the server reboots, we created an Upstart Job to handle this:

```
description	"Consul Template"
author		"Luis Elizondo"

start on filesystem or runlevel [2345]
stop on shutdown

script
	echo $$ > /var/run/consul-template.pid
	exec consul-template \
		-consul IP_OF_CONSUL:PORT_OF_CONSUL \
		-template "/etc/nginx/templates/template:/etc/nginx/sites-enabled/default:service nginx restart"
end script

pre-start script
	echo "[`date`] Consul Template Starting" >> /var/log/consul-template.log
end script

pre-stop script
	rm /var/run/consul-template.pid
	echo "[`date`] Consul Template Stoping" >> /var/log/consul-template.log
end script
```

The same for Consul.io:

```
description	"Consul"
author		"Luis Elizondo"

start on filesystem or runlevel [2345]
stop on shutdown

script
	echo $$ > /var/run/consul.pid
	exec docker start consul
end script

pre-start script
	echo "[`date`] Consul Starting" >> /var/log/consul.log
end script

pre-stop script
	rm /var/run/consul.pid
	exec docker stop consul
	echo "[`date`] Consul Stoping" >> /var/log/consul.log
end script
```

And for registrator too, only this time, Registrator runs not on the LB but on each of the web nodes:

```

description	"Registrator"
author		"Luis Elizondo"

start on filesystem or runlevel [2345]
stop on shutdown

script
	echo $$ > /var/run/registrator.pid
	exec docker start registrator
end script

pre-start script
	echo "[`date`] Registrator Starting" >> /var/log/registrator.log
end script

pre-stop script
	rm /var/run/registrator.pid
	exec docker stop registrator
	echo "[`date`] Registrator Stoping" >> /var/log/registrator.log
```

Before we started this journey in November 2014, we didn't find enough information about a complete workflow that we could adopt or adapt given our circumstances, that is my main motivation to share this with you. As a member of a team of two who did this, I realise our workflow is far from perfect and that with time there might be situations we didn't anticipate, right now there are things we'd like to improve like moving to HAproxy (nothing personal Nginx) among other things, but our workflow and infrastructure is configured in a way that helps us do our job (my main responsibility is being a developer, not as a sysadmin) instead of complicating our existence.

Testing, installing and implementing the new workflow, and then migrating all of our applications took us 2 and a half months, but it was worth it. We used Digital Ocean to anticipate most of the problems and situations before we moved everything and it was a great solution for us because it was really cheap (we spent about $5), doing it this way allowed us to document every step.

If you have any questions, please don't hesitate to post a comment here or mention me on Twitter [@lelizondo](https://twitter.com/lelizondo). You can also follow me on [Github](https://github.com/luiselizondo), fork and contribute to one of the projects of our [organisation](https://github.com/iiiepe) or use our [images](https://hub.docker.com/u/iiiepe). Thanks.
