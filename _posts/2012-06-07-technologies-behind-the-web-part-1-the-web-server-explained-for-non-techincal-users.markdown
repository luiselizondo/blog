---
layout: post
title: 'Technologies behind the web. Part 1: The Web Server. (Explained for non techincal
  users)'
date: '2012-06-07 19:39:00'
tags:
- apache
- htmp
- http
- theweb
- web
- webserver
---

<p>The web is getting old, by know, you probably use it everyday, but the web is more than just your browser or that thing everyone talks about called Twitter. The web is a very complex infrastructure of web servers running software that makes really easy for you to just tweet or friend someone on Facebook.</p>
<p>Behind the web there&#8217;s a web server, who is responsible of serving your browser a web page, there&#8217;s also a database (something like Excel or Microsoft Access but more complex) to store everything you post or read, and finally there&#8217;s a program that interacts between the web server and the database.</p>
<p>So the web server serves your browser (like Firefox, Chrome, Safari, Opera or even Internet Explorer) with information, that information can be images, text, video, audio, and more. But here&#8217;s the interesting thing, what you really see on your screen, is not what the web server is actually sending to your browser, the web server is sending HTML code. This code will be interpreted by the browser and instead of code, you get text with nice fonts, links and more (to see how HTML code looks like go to <a href="http://www.sheldonbrown.com/web_sample1.html">http://www.sheldonbrown.com/web_sample1.html</a>).</p>
<p>A great example of a web server is <a href="http://httpd.apache.org/" title="Apache HTTP Project" target="_blank">Apache</a>, made by the <a href="http://apache.org/" title="Apache Foundation" target="_blank">Apache Foundation</a>. It is a great, and open source program that it is highly used. Of course, there are others, like <a href="http://nginx.org/" title="Nginx" target="_blank">Nginx</a> and Internet Information Services (IIS) made by Microsoft.</p>
<p>So next time you browse the web, you&#8217;ll know there&#8217;s this thing called a web server, probably Apache since it runs con more than half of the web servers in the world, that is the software responsible for giving you that page you want to see. The same is applied to services like Twitter or whatever app you open on your phone.</p>
<p>In Part 2 I will talk about databases and in Part 3 about the guys in the middle.</p>