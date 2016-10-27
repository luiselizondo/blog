---
layout: post
title: Capistrano saves you time
date: '2012-07-23 04:12:00'
tags:
- capistrano
- drupal
---

<p>Another quick post. I&#8217;ve been using Capistrano for a few months now, it is a time saver. If you don&#8217;t know what the hell capistrano does, I&#8217;ll give you a quick example.</p>
<p>I use two servers, one for production and another one for development and I use git to sync code. On a Drupal site, when you make important changes is important to clear the cache. So this is what I would do without capistrano after a code change.</p>
<p>ssh my-production-server.com</p>
<p>cd /var/www</p>
<p>git pull</p>
<p>drush cc all</p>
<p>And this is what I do with capistrano from my local pc</p>
<p>cap deploy cc</p>
<p>cap deploy will do a git pull and cap cc will do a drush cc all but you can call both tasks in one command. Easy and it saves time.</p>