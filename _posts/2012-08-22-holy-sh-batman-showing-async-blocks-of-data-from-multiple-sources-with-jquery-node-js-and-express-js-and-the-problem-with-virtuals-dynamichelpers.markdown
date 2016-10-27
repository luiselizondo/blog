---
layout: post
title: 'Holy sh$% Batman: showing async blocks of data from multiple sources with
  jQuery, Node JS and Express JS and the problem with Virtuals, DynamicHelpers'
date: '2012-08-22 20:46:00'
---

<p>I&#8217;ve been using Node.js lately, and it has some problems like everything on planet Earth because it&#8217;s not perfect. I&#8217;ll give you an example and my solution:</p>
<p>You have a page, that shows a list of something that comes from the database. This is just main content, could be a list of blogs or something. On the sidebar you have totally unrelated (or related) content that also comes from the database. The problem with Node.js (especially using Express JS) is that you have to build the render object before you pass it to res.render which in turn will pass it to the template file (jade or whatever). I&#8217;ll explain the two ways of doing this:</p>
<p>Using async</p>
<p>Remember you have to query the database to get &#8220;somedata&#8221; and &#8220;somextradata&#8221; to pass both of them to res.render:</p>
<p>res.render(&#8220;myview&#8221;, {listone: somedata, listtwo: someextradata});</p>
<p>Using async you&#8217;ll do something like this:</p>
<p>Easy. You just executed two queries in series, first one, then the other one, and when both finish, you execute a last callback which is gonna have a results object with the data from each of the callbacks you executed in series. With that object (the results object) you can do whatever you want, like passing it to res.render(&#8220;myview&#8221;, results);</p>
<p>The problem with this approach is that you don&#8217;t really have lots of reusable code and you have to wait until all queries finish.</p>
<p>Using callback hell</p>
<p>Love it or hate it, callback hell works like this:</p>
<p>What happened there? We first do the first query, save the results in the var someData and after we have the results we do the second totatlly unrelated query, when we have the results from the second query we save it in the var someExtraData. Since we only have two queries, we build an object &#8220;results&#8221; that includes someData and someExtraData and we pass both results to callback or even to res.render(&#8220;myview&#8221;, results)</p>
<p>The problem is the code, no wonder why they call it callback hell. And again, both ways are async. So, you can&#8217;t show the page to the user unless everything is finished building.</p>
<p>But, again, back to the original question, what if someExtraData is just a meaningless peace of content, that kind of content you use just to populate the right side of your page? You don&#8217;t really want to have all those problems do you? And what if it is dynamic? What if you don&#8217;t have to actually wait for the first query to return?</p>
<p>Well, I&#8217;m still trying to find a good approach to solve this problem, in the meantime I&#8217;m using jQuery. Yes, jQuery.</p>
<p>This is my approach:</p>
<p>First, build the important data into the main content area like you would do if you didn&#8217;t need someExtraData.</p>
<p>Then, create another app.get route to fetch only the someExtraData. Something like this</p>
<p>With that you can query <a href="http://example.com/someExtraData">http://example.com/someExtraData</a> and get a json object. This will be in a total different time from the main content. And with jQuery you just do:</p>
<p>Now, this approach works, and I think is pretty good, if you have any comments please let me know here will you?</p>
<p>Partials and dynamicHelpers (locals in 3.0)</p>
<p>Finally I don&#8217;t want to end without talking about those guys.</p>
<p>Partials are a thing of express/jade (don&#8217;t really know) in which you can reuse small peaces of jade code to do things you always do, the best thing is that it uses dynamic placeholders so you can pass inside jade a variable to the partial.</p>
<p>Something like this:</p>
<p>partial(&#8220;myMenu&#8221;, {menu: items})</p>
<p>Items will be a full collection of menu items. Inside the myMenu.jade partial file, you&#8217;re supposed to render &#8220;items&#8221;. The problem with partials is that partials need data but this data always come from the main .jade file, which in turn, comes from the same source. So, both the main.jade and partial.jade files use the same data source: res.render(&#8220;main&#8221;, {myData: myData})</p>
<p>Now, dynamicHelpers or locals in Express 3.0 are a different thing, those are like &#8220;global&#8221; variables. Let&#8217;s say you print the version of your site on every .jade file you have. You don&#8217;t want to do:</p>
<p>res.render(&#8220;somefile&#8221;, {version: getVersion}</p>
<p>everytime do you? Instead, you use dynamicHelpers to process the getVersion, and inside all your jade files you&#8217;ll have always #{version}. What it&#8217;s also nice is the fact that you have access to req and res arguments.</p>
<p>Virtuals</p>
<p>Virtuals in Mongoose are a special property of the object you query, the property doesn&#8217;t exists in your model, you create it &#8220;on-the-fly&#8221;. Let&#8217;s say you have a firstName and a lastName in your model, and you always want to use the fullName, the problem&#8217;s you don&#8217;t have it, you need to build it using firstName and lastName. You do that with virtuals.</p>
<p>The problem with virtuals and dynamicHelpers are almost the same, the async problem. Node.js won&#8217;t wait till dynamicHelpers/Virtuals query the database. Node.js already rendered the page to the user by the time you&#8217;re thinking about querying the database. So, whatever you query, it won&#8217;t show up in your page.</p>
<p>I will post the solution at Github: <a href="https://github.com/lelizondo/nodejs-async-blocks-example">https://github.com/lelizondo/nodejs-async-blocks-example</a></p>
<p>So, what do you think?</p>