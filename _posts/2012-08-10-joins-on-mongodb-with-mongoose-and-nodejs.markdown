---
layout: post
title: Joins on MongoDB with Mongoose and NodeJs
date: '2012-08-10 20:26:00'
tags:
- joinsonmongodb
- mongodb
- mongoose
- nodejs
---

<p>If you don&#8217;t know, non relational databases (NoSQL) can&#8217;t do joins, that thing that helps you query two different tables in one simple query, something like:</p>
<p>SELECT posts.title, users.name FROM {posts} LEFT JOIN posts.userid = users.userid &#8230;.;</p>
<p>Document-oriented databases, like MongoDB, allows you to easily create collections and documents, and I thought the best way to do &#8220;joins&#8221; was to embed the user object in the post.author property inside the post object, but this presents a big problem, what happens if inside the user object you have fields like username or picture that the user can change? An update operation on all the posts would be crazy, and for each collection that you have that has a user object embedded in the model you would have to do an update just to change the picture. Ridiculous isn&#8217;t it? That&#8217;s what I thought.</p>
<p>Then, I spent some time saving just the user _id into the posts.author property, and every time I wanted to query the posts I would just do two queries, first, get the post, then with the post.author get the user. That&#8217;s also expensive and not very clean in your code. And if you&#8217;re doing loops (which I try to avoid because of the async/sync thing on Nodejs), you could run into problems really fast.</p>
<p>Introducing Mongoose and populate</p>
<p>It seems that with Mongoose (v3) is really, and I mean really easy to do &#8220;joins&#8221;. Yeah, I double quote joins because they are not really joins but from a result-oriented perspective (is there another one?) that&#8217;s what you&#8217;re doing.</p>
<p>Mongoose comes with a feature called &#8220;populate&#8221;. Let&#8217;s explain:</p>
<p>Suppose you have two models, Users and Posts, I&#8217;ll keep things simple for now.</p>
<p>var UserSchema = new Schema({</p>
<p>  username: String</p>
<p>})</p>
<p>var PostSchema = new Schema({</p>
<p>  title: String,</p>
<p>  author: {type: Schema.Types.ObjectId, ref: &#8220;User&#8221;}</p>
<p>});</p>
<p>That&#8217;s easy, we just define two schemas and in author we create a property that will be a reference to the User schema. On save, we will save the _id of the user object.</p>
<p>The fun part comes when doing a findOne or find:</p>
<p>var id = &#8220;some id of the post we want to query&#8221;;</p>
<p>Post.findOne({_id: id})</p>
<p>  .populate(&#8220;author&#8221;)</p>
<p>  .exec(function(error, result) {</p>
<p>    // do something with the result like render it</p>
<p>  })</p>
<p>What is nice about this is that Mongoose will do a query on the users collection and get the author of the post and embed it in the result. You&#8217;ve just done a join on MongoDB my friend.</p>
<p>More information about populate here: <a href="http://mongoosejs.com/docs/populate.html">http://mongoosejs.com/docs/populate.html</a></p>