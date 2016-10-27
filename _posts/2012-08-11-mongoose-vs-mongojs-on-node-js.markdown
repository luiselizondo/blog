---
layout: post
title: Mongoose vs mongojs on Node.js
date: '2012-08-11 09:48:52'
tags:
- mongodb
- mongojs
- mongoose
- node-js
- nodejs
---

<p>Quick post. I&#8217;ve been using Node.js a lot, to be honest, I love it, along with Node, I&#8217;ve been using MongoDB which, as you may know if you read my last post, doesn&#8217;t support joins.</p>
<p>My first choice on a project was to use mongojs because it was plain simple. It uses the same structure you use on MongoDB directly with a very few exceptions, the main one is the ability to pass a callback as an argument.</p>
<p>I moved really fast with mongojs, but then I hit a wall. Joins. Is just not possible to do it with MongoJS for the same reason you can&#8217;t do it with MongoDB, you have to use some sort of black magic to do them, and this is when Mongoose excells.</p>
<p>To be honest, I tried to avoid Mongoose because I thought it was hard to use, defining a model was just &#8220;a waste of time&#8221; when with MongoJS you don&#8217;t need to. I was wrong.</p>
<p>Bottom line. If your project is really simple, no joins, no complicated features, go with MongoJS, is really easy but limited. If you&#8217;re trying to save the world with your crazy idea and you need more powers than Superman, spend some time learning Mongoose and use it, it will take you there.</p>
<p>Trust me, I just spent two days rewriting everything using Mongoose, I wish someone told me before.</p>