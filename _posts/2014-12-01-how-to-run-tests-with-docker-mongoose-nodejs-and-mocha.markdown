---
layout: post
title: 'How to run tests with #Docker, #Mongoose, #Nodejs and #Mocha'
date: '2014-12-01 18:38:17'
description: I thought about posting here how I'm testing a Node.js application that needs to be connected to a Database running MongoDB with Mongoose. I'm using Mocha as a testing framework and it can be tricky to configure everything to get you up and running.
---

I thought about posting here how I'm testing a Node.js application that needs to be connected to a Database running MongoDB with Mongoose. I'm using Mocha as a testing framework and it can be tricky to configure everything to get you up and running.

You will need:

1. [Docker](http://docker.com)
2. [Fig](http://www.fig.sh/)
3. [Mocha.js](http://mochajs.org/)
4. [Should.js](https://github.com/shouldjs/should.js)
5. [Node.js docker image](https://registry.hub.docker.com/u/luis/nodejs/)
6. [MongoDB docker image](https://registry.hub.docker.com/u/luis/mongodb/)
7. [Factory Lady](https://github.com/petejkim/factory-lady)
8. [Chance.js](http://chancejs.com/)
9. [Mongoose](http://mongoosejs.com/)
10. [Patience](http://en.wikipedia.org/wiki/Patience)


### /test/mocha.opts


		--require should
		-R spec
		--ui bdd
		--timeout 90
		--colors

### /test/db.js
This is the file I'm using to connect to the database


		/**
		 * @file
		 * hooks to run before and after tests
		 * it does not need to define a module.exports since
		 * mocha will run the hooks just by requiring the file
		 */

		'use strict';

		var mongoose = require("mongoose");
		var config = require("express-config");

		/**
		 * Util function to clear the database
		 * @param  {Function} callback A callback to return when done
		 */
		function clearDB(callback) {
		 for (var i in mongoose.connection.collections) {
		   mongoose.connection.collections[i].remove(function() {});
		 }
		 return callback(false);
		}

		/**
		 * Run before everything
		 * Checks the state of the connection, if is is connected,
		 * disconnect and then try again
		 */
		before(function() {
			if(mongoose.connection.readyState === 1) {
				mongoose.disconnect();
			}
			return mongoose.connect(config.mongodb.uri);
		});

		/**
		 * Before each test is run, we clear the DB
		 */
		beforeEach(function (done) {
			clearDB(function() {
				done();
			});
		});

		/**
		 * After all tests are done, disconnect
		 * @param  {Function} done [description]
		 * @return {[type]}        [description]
		 */
		after(function (done) {
		 mongoose.disconnect();
		 return done();
		});


### /test/factory.js
I'm using Factory Lady, it helps you to interact directly with Mongoose by creating an object based on a Model definition. This is the file I'm using.


		var Factory = require("factory-lady");
		var Chance = require("chance");
		var chance = new Chance();
		var include = require("include");

		// Models
		var User = include("app/models/User");

        var counter = 1;

		Factory.define('user', User, {
        	username: function(cb) { cb( 'user' + counter++); },
            fullname: chance.name(),
            password: chance.hash({length: 8}),
            email:  function(cb) { cb( 'user' + counter++ + "@example.com"); },
            gender: chance.gender().toLowerCase(),
            locale: 'en-US'
        });

### /test/unit/UserRepository.js
And now my test:

		'use strict';

		var assert = require("assert");
		var should = require("should");
		var include = require("include");
		var db = include("test/db");
		var Factory = include("test/factory");
		var Chance = require("chance");
		var chance = new Chance();
		var User = include("app/repositories/UserRepository");

		describe("UserRepository", function() {
			describe("Save", function() {
				it("Should fail when data is not provided", function(done) {
					var data = {};
					User.save(data, function(err, result) {
						err.should.equal("Input is empty");
						done();
					});
				});

				it("Should fail to create a user when not providing a username", function(done) {
					Factory.build("user", {username: null}, function(data) {
						User.save(data, function(err, result) {
							result.should.be.false;
							err.should.equal("A username is required, please enter one");
							done();
						});
					});
				});

			});
		});

### /Makefile
I'm using a Makefile to run my application, the test command assumes that the application is installed, which means that a the MongoDB and Redis containers should be running before you run the tests (I'm doing this with make install). Since I'm also using fig, it will create a container with a name using the convention *CurrentDirectoryName_ServiceName_N*, so, if the directory of my application is webapp, fig will create webapp_mongodb_1 and webapp_redis_1, you'll need to replace those:


		CURRENT_DIRECTORY := $(shell pwd)

		test:
			@docker run --rm -e NODE_ENV=test -v $(CURRENT_DIRECTORY):/var/www -p 3999:3000 --link current_directory_name_mongodb_1:mongodb --name mocka luis/nodejs npm test

		start-all:
			@fig start

        clean:
			@fig stop

		install:
			@fig up -d

		start:
			@fig start web

		stop:
			@fig stop web

		restart:
			@fig stop web
			@fig start web

		.PHONY: test start-all clean install start stop restart

### /fig.yml
To use Fig you need a fig configuration file, this is the one I'm using:

    mongodb:
      image: luis/mongodb
      expose:
        - "27017"
      volumes:
        - /var/data:/data/db
    web:
      image: luis/nodejs
      links:
        - mongodb:mongodb
      ports:
        - "3000:3000"
      environment:
        NODE_ENV: development
      volumes:
        - .:/var/www
        - ./docker:/etc/supervisor/conf.d
        - /var/log/docker:/var/log/supervisor


### /docker/supervisord-nodejs.conf
The luis/nodejs image by default will run Node.js using supervisor and it will run the start.js file, if you want to change it, here is my supervisor configuration file, just change the file you want to start with node.js:

    [supervisord]
    nodaemon=true

    [program:nodejs]
    directory=/var/www
    command=nodejs app.js
    autorestart = true
    stdout_logfile=/var/log/supervisor/%(program_name)s.log
    stderr_logfile=/var/log/supervisor/%(program_name)s.log

## Notes
- The db.js file does not export a function, mocha will execute it when is required in the test file.
- I'm using express-config to load the configuration from a /config/test.js file, inside that file is the configuration needed to connect Mongoose to MongoDB.

## Run with:

		make test
