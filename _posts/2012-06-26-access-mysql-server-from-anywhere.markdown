---
layout: post
title: Access MySQL Server from Anywhere
date: '2012-06-26 01:13:03'
---

<p>A common question I get asked is how to access MySQL from another host that is not the localhost? The process involves two steps, the first one is to create a user with access from the host you&#8217;ll be accessing, I do not recommend that you select Any host. In fact, I do not recommend this article and you should to this with care.</p>
<p>The second step is to edit the my.cnf (in Ubuntu you&#8217;ll find it on /etc/mysql/) and comment, remove or set the IP address to the IP you&#8217;ll be accessing from.</p>
<p>After that, just restart the server and test</p>
<p>mysql -u username -h host-where-your-database-is-located -p</p>