---
layout: post
title: Synchronous JavaScript using Promises. Solving the callback hell.
date: '2015-07-04 01:26:00'
description: Let's face it, one of the most difficult things to understand when learning Javascript is the asynchronous stuff. If you want to learn how to solve it (almost), keep reading, I'll show you how to use promises in an easy way, step by step.

---

Let's face it, one of the most difficult things to understand when learning Javascript is the asynchronous stuff. If you want to learn how to solve it (almost), keep reading, I'll show you how to use promises in an easy way, step by step.

![alt](https://luiselizondo.ghost.io/content/images/2015/07/meme-functions.jpg)

We'll be creating a blog object, very basic stuff, and we're going to see how to use callbacks, how not to use callbacks, and finally how to work with promises. Since I don't want to complicate things, I wrote a FakeDB class that simulates a very slow database, you'll need it so here it is:

```
function FakeDB() {
	this.blog = {};
}

FakeDB.prototype.get = function get(next) {
	var self = this;
	setTimeout(function() {
		return self.blog();
	}, 1000);
}

FakeDB.prototype.set = function set(data) {
	this.blog = data;
}

FakeDB.prototype.validate = function validate(data) {
	var error = false;

	if(!data.title) {
		return "Title is required";
	}

	if(!data.body) {
		return "Body is required";
	}

	return error;
}

FakeDB.prototype.save = function save(next) {
	var self = this;

	var error = self.validate(self.blog);
	if(error) {
		return next(error);
	}

	setTimeout(function() {
			return next(false, self.blog);
	}, 1000);
}

module.exports = FakeDB;
```

To use this fake database, you just instantiate the class, pass a data object to the set method, and then call save. It will basically return the same object, but it will do it after 1 second, and it has a validation mechanism just in case.

## A note
Every time I run the program, I will be using three functions, I will not repeat them every time. The first one is the *uslessCallback* which will only return a message and log it to the console, this is a synchronous function. The second function, *saveAsCallback* is a wrapper around the class, it will create a new blog by instantiating a class, creating an object and then saving it, this function is called with 3 arguments, the title of the blog, the body of the blog and a callback. The third function, *saveAsPromise* is the same as the second one, it does the same, but it uses promises.

```
var Database = require("./fakedb");

function saveAsCallback(title, body, next) {
	console.log("New database instantiated");
	var db = new Database();

    // create a new object with the values from the arguments
	var data = {
		title: title,
		body: body
	}

    // set the data to be saved
	console.log("Setting data");
	db.set(data);

    // execute the method save and then pass a callback
    // the callback will be executed after 1 second
    // if there's an error, it will return the callback with an error
    // if not, then it will return the result object.
	console.log("About to save data");
	db.save(function(err, result) {
		if(err) {
			console.log("Error: " + err);
			return next(err);
		}
		else {
			console.log("Data saved with title " + result.title)
			return next(err, result);			
		}
	});
}

// This doesn't do much
function uslessCallback(message) {
	console.log(message);
	return message;
}
```

## How not to use callbacks

```

function usingCallbacksWrong() {
	// save one blog
	saveAsCallback("Hello World 1", "This is my first blog", function(err, result) {
		console.log("---");
		console.log(result);
		console.log("---");
	});

	uslessCallback("--- This message should show after the first object was created but it won't ---");

// save a second blog
	saveAsCallback("Hello World 2", "This is the second blog", function(err, result) {
		console.log("---");
		console.log(result);
		console.log("---");
	});

	uslessCallback("--- This message should show after the second object was created but it won't ---");

	return;
}

usingCallbacksWrong();
```

This is not the way you should use callbacks. The results will be unpredictable. In this case, we don't really do much with the results of the blog once it's saved, but this will execute the following way:

```
$ node usingCallbacksWrong.js         
New database instantiated
Setting data
About to save data
--- This message should show after the first object was created but it won't ---
New database instantiated
Setting data
About to save data
--- This message should show after the second object was created but it won't ---
Data saved with title Hello World 1
---
{ title: 'Hello World 1', body: 'This is my first blog' }
---
Data saved with title Hello World 2
---
{ title: 'Hello World 2', body: 'This is the second blog' }
---
```

As you can see, the second blog is instantiated before the first one finishes saving. The messages reflect where they should appear, but you can see they are not in the place they should be. This is because the second time we execute saveAsCallback the first callback hasn't finished. This is not how you should be writing Javascript.

## Using callbacks the right way

```

function usingCallbacksRight() {

	// save one
	saveAsCallback("Hello World 1", "This is my first blog", function(err, result) {
		console.log("---");
		console.log(result);
		console.log("---");

		uslessCallback("--- This message should show after the first object was created ---");

		saveAsCallback("Hello World 2", "This is the second blog", function(err, result) {
			console.log("---");
			console.log(result);
			console.log("---");

			uslessCallback("--- This message should show after the second object was created ---");

			return;
		});
	});
}

usingCallbacksRight();
```

This is the way you should be working with callbacks, but as you can see, it creates a callback hell, where you just keep indenting to the right. However, the results show in the right order:

```
$ node usingCallbacksRight.js
New database instantiated
Setting data
About to save data
Data saved with title Hello World 1
---
{ title: 'Hello World 1', body: 'This is my first blog' }
---
--- This message should show after the first object was created ---
New database instantiated
Setting data
About to save data
Data saved with title Hello World 2
---
{ title: 'Hello World 2', body: 'This is the second blog' }
---
--- This message should show after the second object was created ---
```

As you can see, everything works the way it should work. But, callback hell is something we're trying to avoid.

## Callbacks with errors
Sometimes error happens, and you want to react to it, just for the sake of it, I'm going to use almost the same code as the previous one, but this time, I will not pass the body of the blog, which will cause the program to interrupt and the second time I call "saveAsCallback" it will not be executed because of the error.

```
function usingCallbacksRightWithError() {

	// save one
	saveAsCallback("Hello World 1", null, function(err, result) {
		if(err) {
			console.log("An error is thrown, the rest of program will not be executed");
			return err;
		}
		else {
			uslessCallback("--- This message should show after the first object was created ---");

			saveAsCallback("Hello World 2", "This is the second blog", function(err, result) {
				console.log("---");
				console.log(result);
				console.log("---");

				uslessCallback("--- This message should show after the second object was created ---");

				return;
			});
		}

	});
}

usingCallbacksRightWithError();
```

The result is the following:

```
$ node usingCallbacksRightWithError.js
New database instantiated
Setting data
About to save data
Error: Body is required
An error is thrown, the rest of program will not be executed
```

# Promises
Before we can use promises, we must re-write the saveAsCallback function as a promise, we'll also rename it.

```
var Q = require("q");
var Database = require("./fakedb");

function saveAsPromise(title, body) {
	var d = Q.defer();
	console.log("New database instantiated");
	var db = new Database();

	var data = {
		title: title,
		body: body
	}

	console.log("Setting data");
	db.set(data);

	console.log("About to save data");
	db.save(function(err, result) {
		if(err) d.reject(err);
		else {
			console.log("Data saved with title " + result.title)
			d.resolve(result);
		}
	});

	return d.promise;
}
```

A few things to notice here. I removed the callback from the arguments. I'm also adding the line 	<code>var d = Q.defer();</code> and <code>return d.promise;</code> outside of any callback hell. Remember that my fakeDB class hasn't changed, the save method still needs a callback. The only difference is that instead of returning another callback, I use <code>d.reject()</code> and <code>d.resolve()</code>. Very easy.

## Using promises
One of the main advantages of using promises is that we can simulate synchronous code, we do it using <code>.then()</code> but the important thing to remember here is that every time I use <code>.then()</code> I have access to whatever the previous step returned.

```

function usingPromises() {

	saveAsPromise("Hello Promises", "This is the first blog using a promise")
	.then(function(blog1) {
		console.log("---- THEN NUMBER 1 --------");
		console.log("Printing the title of blog1 since I have access to it: " + blog1.title);
		return blog1;
	})
	.then(function(blog1) {
		console.log("---- THEN NUMBER 2 --------");
		uslessCallback("Second step, I still have access to the blog1 that the previous returned");
		uslessCallback("Using the function uslessCallback to print the title of blog1: " + blog1.title);
		return;
	})
	.then(function() {
		console.log("---- THEN NUMBER 3 --------");
		uslessCallback("I don't have access to blog1 in this step since the previous step didn't return anything");
		uslessCallback("But in this step I will call saveAsPromise to create the blog2, and since saveAsPromise returns");
		uslessCallback("an object (the blog object) it will be available to the next step");
		return saveAsPromise("Hello Promises 2", "This is the second time I call a method that will return a promise");
	})
	.then(function(blog2) {
		console.log("---- THEN NUMBER 4 --------");
		uslessCallback("As promised, I called saveAsPromise in the previous step and I now have access to blog2");
		uslessCallback("Printing the title and the body");
		uslessCallback("Title: " + blog2.title);
		uslessCallback("Returning the blog2 so I can access it in the next then");
		return blog2;
	})
	.then(function(blog2) {
		console.log("------ THEN NUMBER 6 ------");
		uslessCallback("Since I have access to the blog2, I can finish printing the body");
		uslessCallback("Body: " + blog2.body);
		console.log("Finishing");
		return;
	})
	.fail(function(err) {
		console.log(err);
		return;
	});
}

usingPromises();

```

The code is easy to read and self explanatory because of the use of the uslessCallback. Let's run it:

```
$ node usingPromises.js               
New database instantiated
Setting data
About to save data
Data saved with title Hello Promises
---- THEN NUMBER 1 --------
Printing the title of blog1 since I have access to it: Hello Promises
---- THEN NUMBER 2 --------
Second step, I still have access to the blog1 that the previous step returned
Using the function uslessCallback to print the title of blog1: Hello Promises
---- THEN NUMBER 3 --------
I don't have access to blog1 in this step since the previous step didn't return anything
But in this step I will call saveAsPromise to create the blog2, and since saveAsPromise returns
an object (the blog object) it will be available to the next step
New database instantiated
Setting data
About to save data
Data saved with title Hello Promises 2
---- THEN NUMBER 4 --------
As promised, I called saveAsPromise in the previous step and I now have access to blog2
Printing the title and the body
Title: Hello Promises 2
Returning the blog2 so I can access it in the next then
------ THEN NUMBER 6 ------
Since I have access to the blog2, I can finish printing the body
Body: This is the second time I call a method that will return a promise
Finishing
```

As you can see, all the code executed in order. Pay special attention to the THEN NUMBER 3, because I'm returning and calling a method as I normally do it in a synchronous language. You can also assign it to a variable and return the value of the variable. It will work, even if we know that the function is calling the database.

## What about errors?
Errors happen, so we need to know what to do with them, in this case, we'll just log the error, but the execution of the program will stop.

```

function usingPromisesWithError() {

	saveAsPromise("Hello Promises", "This is the first blog using a promise")
	.then(function(blog1) {
		console.log("---- THEN NUMBER 1 --------");
		console.log("The blog1 was created, in the following step, an error will be thrown after this step");
		return blog1;
	})
	.then(function(blog1) {
		console.log("---- THEN NUMBER 2 --------");
		return saveAsPromise("Hello Promises 2");
	})
	.then(function(blog2) {
		console.log("---- THEN NUMBER 3 --------");
		console.log("This is never executed");
		return;
	})
	.fail(function(err) {
		console.log("This step is executed when something fails and the following line should print an error");
		console.error(err);
		return;
	});
}

usingPromisesWithError();
```

And the result is the following:

```
node usingPromisesWithError.js
New database instantiated
Setting data
About to save data
Data saved with title Hello Promises
---- THEN NUMBER 1 --------
The blog1 was created, in the following step, an error will be thrown after this step
---- THEN NUMBER 2 --------
New database instantiated
Setting data
About to save data
This step is executed when something fails and the following line should print an error
Body is required
```

You can see there's a whole step that is never executed.

## A promise inside a promise
Our function *usingPromisesWithError* is called only once and is written to be called only once. If we want to call it multiple times we must use callbacks, or promises. Let's transform that function and call it twice. Remember, this is a function that is calling a promise, but it will also return another promise.

```

function usingPromisesAsAPromise() {
	var d = Q.defer();

	saveAsPromise("Hello Promises", "This is the first blog using a promise")
	.then(function(blog1) {
		console.log("---- THEN NUMBER 1 --------");
		console.log("Printing the title of blog1 since I have access to it: " + blog1.title);
		return blog1;
	})
	.then(function(blog1) {
		console.log("---- THEN NUMBER 2 --------");
		uslessCallback("Second step, I still have access to the blog1 that the previous step gave me");
		uslessCallback("Using the function uslessCallback to print the title of blog1: " + blog1.title);
		return;
	})
	.then(function() {
		console.log("---- THEN NUMBER 3 --------");
		uslessCallback("I don't have access to blog1 in this step since the previous step didn't return anything");
		uslessCallback("But in this step I will call saveAsPromise to create the blog2, and since saveAsPromise returns");
		uslessCallback("an object (the blog object) it will be available to the next step");
		return saveAsPromise("Hello Promises 2", "This is the second time I call a method that will return a promise");
	})
	.then(function(blog2) {
		console.log("---- THEN NUMBER 4 --------");
		uslessCallback("As promised, I called saveAsPromise in the previous step and I have now access to blog2");
		uslessCallback("Printing the title and the body");
		uslessCallback("Title: " + blog2.title);
		uslessCallback("Returning the blog2 so I can access it in the next then");
		return blog2;
	})
	.then(function(blog2) {
		console.log("------ THEN NUMBER 6 ------");
		uslessCallback("Since I have access to the blog2, I can finish printing the body");
		uslessCallback("Body: " + blog2.body);
		console.log("======================= Finishing one execution of usingPromisesAsAPromise ===============================");
		d.resolve();
	})
	.fail(function(err) {
		console.log(err);
		d.reject(err);
	});

	return d.promise;
}

// calling usingPromisesAsAPromise twice
usingPromisesAsAPromise()
.then(usingPromisesAsAPromise)
.fail(function(err) {
	console.log(err);
});
```

I added a few lines so we can easily see how many times this function is called.

```
$ node usingAPromiseInsideAPromise.js
New database instantiated
Setting data
About to save data
Data saved with title Hello Promises
---- THEN NUMBER 1 --------
Printing the title of blog1 since I have access to it: Hello Promises
---- THEN NUMBER 2 --------
Second step, I still have access to the blog1 that the previous step gave me
Using the function uslessCallback to print the title of blog1: Hello Promises
---- THEN NUMBER 3 --------
I don't have access to blog1 in this step since the previous step didn't return anything
But in this step I will call saveAsPromise to create the blog2, and since saveAsPromise returns
an object (the blog object) it will be available to the next step
New database instantiated
Setting data
About to save data
Data saved with title Hello Promises 2
---- THEN NUMBER 4 --------
As promised, I called saveAsPromise in the previous step and I have now access to blog2
Printing the title and the body
Title: Hello Promises 2
Returning the blog2 so I can access it in the next then
------ THEN NUMBER 6 ------
Since I have access to the blog2, I can finish printing the body
Body: This is the second time I call a method that will return a promise
======================= Finishing one execution of usingPromisesAsAPromise ===============================
New database instantiated
Setting data
About to save data
Data saved with title Hello Promises
---- THEN NUMBER 1 --------
Printing the title of blog1 since I have access to it: Hello Promises
---- THEN NUMBER 2 --------
Second step, I still have access to the blog1 that the previous step gave me
Using the function uslessCallback to print the title of blog1: Hello Promises
---- THEN NUMBER 3 --------
I don't have access to blog1 in this step since the previous step didn't return anything
But in this step I will call saveAsPromise to create the blog2, and since saveAsPromise returns
an object (the blog object) it will be available to the next step
New database instantiated
Setting data
About to save data
Data saved with title Hello Promises 2
---- THEN NUMBER 4 --------
As promised, I called saveAsPromise in the previous step and I have now access to blog2
Printing the title and the body
Title: Hello Promises 2
Returning the blog2 so I can access it in the next then
------ THEN NUMBER 6 ------
Since I have access to the blog2, I can finish printing the body
Body: This is the second time I call a method that will return a promise
======================= Finishing one execution of usingPromisesAsAPromise ===============================
```

That's it, you can see the code on [Github](https://github.com/luiselizondo/Synchronous-JavaScript-Using-Promises). I hope you understand now the advantages of using promises. If you have any questions, please leave a comment or follow me on [Twitter](http://twitter.com/lelizondo).
