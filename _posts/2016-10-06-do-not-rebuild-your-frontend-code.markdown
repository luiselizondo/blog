---
layout: post
title: Do not rebuild your frontend code
date: '2016-10-06 05:59:44'
description: I was recently working on a frontend project that uses Angular2. We have 3 environments, development; staging and production so the frontend points to a different API url for each of the environments.
tags:
- angular-js
- angularjs
- docker
- javascript
---

I was recently working on a frontend project that uses Angular2. We have 3 environments, development; staging and production so the frontend points to a different API url for each of the environments.

The project didn't use Docker so one of the first things I did when I got in charge of the project was to Dockerize the whole thing. The problem was that there is no way the frontend, served by Nginx, detected the environment that it was running on.

The solution to this problems is usually to rebuild the whole project each time we promote from one environment to the next one, the problem with this approach is exactly that, you're rebuilding the project on each environment, causing potential problems that you might not be aware of.

The solution was rather simple, inject an environment variable telling the project to point to a different API on runtime, but this is not as simple as it sounds because the frontend cannot read the environment variable that says which environment is the code running on.

The solution, as it usually is, was very simple, create a script to run Nginx but before, replace some variables on all built files.

We have a configuration file like this:

```
// file config.js
config.url = "http://localhost:3000/api"
```

And our container will run this script each time it starts:

```
#!/bin/sh
# file /opt/start-nginx.sh

sed -i "s|http://localhost:3000/api|${API}|g" *.js

echo "Starting nginx in the foreground"
exec nginx -g 'daemon off;' "$@"
```

where ${API} is the url of the API we want to point to on each environment. Notice that we replace the string http://localhost:3000/api, which is our development url on each js file that we find. Also notice how we use " instead of ', this is so the variables are replaced correctly.

Finally, our Dockerfile looks like this:

```
FROM nginx:alpine
COPY dist /usr/share/nginx/html
WORKDIR /usr/share/nginx/html
CMD ["/opt/start-nginx.sh"]
```

This way, we don't rebuild the project each time. Problem solved.
