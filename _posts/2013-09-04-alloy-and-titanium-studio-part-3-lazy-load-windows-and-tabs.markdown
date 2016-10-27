---
layout: post
title: Alloy and Titanium Studio, Part 3. Lazy Load Windows and Tabs
date: '2013-09-04 08:42:55'
tags:
- alloy
- android
- ios
- lazyload
- tabgroup
- titaniumstudio
---

Speed. We don't wait for anything. In this short tutorial I'll show you my method to lazy load data as you need so your app bootstraps fast. Let's say you have an API you get data from, and you have a Tab with a Window inside, inside the Window you do all the magic and the data you get from your API shows up. I'm not gonna cover that part. You probably also have a login window, and once the user signs in, then you load the rest of the app. But, if you wait until your server gives you the data to bootstrap it's gonna take a few seconds to load the app. So, what do I do?

Let's code a simple tab with a window, our files will be myWindow.xml and myWindow.js

myWindow.xml
<code>
<Alloy>
  <Tab id="myTab" title="My tab">
    <Window id="myWindow">
      // Magic goes here
      <Label id="name">
    </Window>
  </Tab>
</Alloy>
</code>

As you can see, I'm giving my tab an id. That's all there is.

On the controller, you'd normally do the following: 

myWindow.js
<code>
function getData(callback) {
  // get data here and then return callback with data
  callback(data);
}

// Execute getData and set the text of name to what the server sent us
getData(function(data) {
  $.name.text = data.name;
});
</code>

But again, we have to wait. Instead, let's add an event listener to load the data once the tab is "focus", only then we'll load the data.

myWindow.js
<code>
function getData(callback) {
  // get data here and then return callback with data
  callback(data);
}

// Wait until we focus on this tab to get the data
$.myTab.addEventListener("focus", function() {
  // Execute getData and set the text of name to what the server sent us
  getData(function(data) {
    $.name.text = data.name;
  });
});
</code>

That's it, your app will load really fast. You may want to show and activity indicator so the user knows something is happening in the background.

You may ask, why is this different? the user will have to wait anyway, so why bother? Well, good question, except that when you have multiple tabs, Titanium will execute all code on all tabs, meaning that if you get data from the server on all tabs, you will make multiple requests and the tab group will not open until the server gets a response and displays the data, even if the user will only see one tab at a time.

So there you have it. Only one small note. I read (can't confirm) that this is not working on Android, but I tested it on iOS and it works great, I don't even need to test it, I can feel it's way faster, and I'm also not making multiple requests to my server almost at the same time.

Happy coding!

