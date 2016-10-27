---
layout: post
title: The thing no one mentions when you're learning AngujarJS
date: '2014-07-24 07:32:57'
tags:
- iwantedtokillmyselfbutnotanymore
- angular-js
- angularjs
- frustration
- javascript
---

So I decided to learn AngularJS, and it was HARD, I mean, frustrating. I couldn't make my router to work. I downloaded a free book, searched tutorials, watched videos, tried everything, until I found a deep and obscure post on SO that brought light.

There were two reasons why my app was failing:

<h3>Do not use the min.js version of angular</h3>
The reason for this was simple, you will get better error messages. At first, I said, Angular sucks, I don't understand the weird messages, it gives me way too much information about the error but I can't read it. And then I found out that the reason was simple, I was using the min.js file. So don't.

<h3>You will need more than one file</h3>
This was the other big reason, I assumed (I know, I'm banging my head against the wall while I write this) that angularjs included everything necesary to run Angular.js, you know, like the router library/component/module, but I found out that **it does not**, in fact, the angular.js file is probably useless on it's own, you'll need a file for the router, another file to handle HTTP requests using the resource module and so on.

The problem was not that I wasn't including the files, there was no files when downloading angular.js, so you have to go some obscure place where you download everything. As an example, this is what I'm using now:

<code>
<script src="//ajax.googleapis.com/ajax/libs/jquery/1.11.0/jquery.min.js"></script>
<script src="//ajax.googleapis.com/ajax/libs/angularjs/1.2.20/angular.js"></script>
 <script src="//ajax.googleapis.com/ajax/libs/angularjs/1.2.20/angular-route.js"></script>
<script src="https://code.angularjs.org/1.2.20/angular-resource.js"></script>
<script src="https://code.angularjs.org/1.2.20/angular-sanitize.js"></script>
<script src="vendor/angular-bootstrap/ui-bootstrap-tpls.js"></script>
<script src="//cdnjs.cloudflare.com/ajax/libs/showdown/0.3.1/showdown.min.js"></script>

<script src="js/vendor/bootstrap.min.js"></script>
<script src="js/main.js"></script>
</code>

As you can see, I'm loading 4 different angular files, all needed, but only the main one is provided when you download the package. So there you go, I hope it saves you hours of frustration.

If you're curious about the application, you can download the code at <a href="https://github.com/luiselizondo/angular-test-app">Github</a>. Turns out that after the 4 hours of frustration, I'm actually enjoying learning Angular.js.
