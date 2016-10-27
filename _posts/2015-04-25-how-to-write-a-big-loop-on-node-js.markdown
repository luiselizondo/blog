---
layout: post
title: How to write a big loop on Node.js
date: '2015-04-25 21:42:18'
description: I'm working on a project on Sails.js and I need to insert 144,034 records into a MySQL database. Sails provides Waterline, a database abstraction mechanism which is outside of the scope of this post.
---

I'm working on a project on Sails.js and I need to insert 144,034 records into a MySQL database. Sails provides Waterline, a database abstraction mechanism which is outside of the scope of this post.

The problem I faced is that when doing a big loop, Node.js runs out of memory. Before I continue, let me say that if the same thing happened to you is probably not because you found a bug on Node.js, is likely that there's a problem with your code.

There's many good posts about loops on Node.js, I read most of them, and to keep things simple, I usually use the async module, which is great. But, async fails when doing a loop this big.

The process I'm trying to accomplish is actually very simple.

1. Read a JSON file.
2. Loop the array in the JSON file and for each of the objects found...
3. Insert it into the database.
4. Continue

The problem as I said, was with the magnitude of the array, on smaller arrays I never had problems. But, around 20,000+ records, Node.js starts complaining about memory.

I tried async, a for loop, using process.nextTick, Fabric, and all kind of hacks. And then I found ["The potentially asynchronous loop"](https://blog.jcoglan.com/2010/08/30/the-potentially-asynchronous-loop/) which is a very simple approach (simple solutions usually work best) to the problem. Basically, it just waits until the current operation is completed before trying the next one. Yes, it's slower, but I'm not trying to fool myself, I know inserting 144,304 records on a MySQL is not going to take a few seconds. And I'm running this as a script, is not technically part of the application.

Enough words, this is the code:

```
var filePath = process.argv[4];
var fs = require("fs");
var include = require("include");
var file = include(filePath);


function init(callback) {
	var total = file.length;
	var counter = 0;
	console.log("Inserting " + total);
	file.asyncEach(function(item, resume) {
		insert(item, function(err, result) {
			counter++

			if(counter < total) {
				resume();
			}
			else {
				return callback();
			}
		});
	});
}
```

The insert method is not really special, is just the operation to insert a single record into the DB so I'm going to omit the explanation.

The init method will read the file "file" and then loop the elements using the "asyncEach" method. It will call the insert method which will insert the record into the DB. When the operation is completed, it will check if the array has finished processing and return the callback, if is not, then it will call resume(), meaning it work on the next item of the array.

I call this method like this:

```
init(function() {
			process.exit();
		})
```

The cool part of this is the asyncEach method:

```
Array.prototype.asyncEach = function(iterator) {
  var list    = this,
      n       = list.length,
      i       = -1,
      calls   = 0,
      looping = false;

  var iterate = function() {
    calls -= 1;
    i += 1;
    if (i === n) return;
    iterator(list[i], resume);
  };

  var loop = function() {
    if (looping) return;
    looping = true;
    while (calls > 0) iterate();
    looping = false;
  };

  var resume = function() {
    calls += 1;
    if (typeof setTimeout === 'undefined') loop();
    else setTimeout(iterate, 1);
  };
  resume();
};
```

This code was taken from the post I mentioned before.

So there you have it, a very easy solution to handle a big loop.
