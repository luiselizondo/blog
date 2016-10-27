---
layout: post
title: NavigationWindow problem in Titanium
date: '2014-07-18 18:22:12'
tags:
- appcelerator
- navigationwindow
- tabgroup
- titanium
---

<p>Before I start or you jump right to the comments to tell me how general and irrelevant is the Title of this post, I know, I couldn't do better, because the problem is not easy to explain, at least not in a title. So what's the problem?</p>

<p>I'm developing an iOS app using Titanium, and to create that endless navigation of windows (basically when you open a window and then tap something and another window opens) you have two different ways of doing with Titanium out-of-the-box. NavigationWindow and TabGroups. </p>

<h3>TabGroups</h3>
<p>TabGroups are easy, you have your tabs at the bottom of the screen and if you want to create a new window you just:</p>

<js>
$.tabGroup.activeTab.open(newWindowToOpen);
</js>

<p>The problem is that most of the time, you don't have access to the tabGroup in the current window, where you'll open the next window. This is because the object $.tabGroup is only available on the file where you define it. So if you define the a TabGroup in a file mainWindow.xml it will not be available in someOtherWindow.js. This is why, it's a good practice to create a global object to access the tabGroup where and when you want it.</p>

<p>In alloy.js you'll do something like:</p>

<js>
Alloy.Globals.tabGroup = null;
</js>

<p>And in the file you define the TabGroup (mine is mainWindow.xml / mainWindow.js) you do: </p>

<js>
Alloy.Globals.tabGroup = $.tabGroup;
</js>

<p>And now, you can easily access it from any window like<p>

<js>
Alloy.Globals.tabGroup.activeTab.open(newWindowToOpen);
</js>

<p></p>

<h3>NavigationWindow</h3>
<p>
The other method to open windows is using a NavigationWindow. What are those? Suppose you need to open a new window, without the tabGroup and outside it's scope. This new window that opens can also be the start of a new navigation group, in which you open new windows one after the other, but remember, you don't want the tabGroup controlling the navigation because then you'll have to show the tabGroup at the bottom of the screen and that's maybe something you don't like.
</p>

<p>
To give you an example of this, if you have a search button that opens a new window, in it, you are searching for a person, and when the person shows up in a table, you click on it and a new window opens with some information about that person. In that process, you'll have two windows, the initial window (the search), which also defines the navigation window, and the next window, the profile of the person.
</p>

<p>
In the first window you define the navigation window, so you can access it, but in the second one, you can't access the navigation window if you want. So you need to do almost the same thing that you did with the tabGroup, create a new Global object. But this time, let's do a library with a class.
</p>

<p>Inside a folder lib/window create a new file navigationWindow.js and paste the following:</p>

<js>
/**
 * Controls a navigationWindow so we can access it
 * outside the same window in a global scope
 * We instantiate this class in alloy.js as
 * Alloy.Globals.NavigationWindow and we set the NavigationWindow
 * according to the context
 */
function NavigationWindow() {
var navigationWindow;

this.get = function() {
return this.navigationWindow;
};

this.set = function(window) {
this.navigationWindow = window;
};

this.reset = function() {
this.set(null);
};
}

module.exports = NavigationWindow;
</js>

And then, inside alloy.js you'll instantiate this class:

<js>
var NavigationWindow = require("window/navigationWindow"); // omit the lib
Alloy.Globals.navigationWindow = new NavigationWindow();
</js>

Now when you define the navigation window, in the same file you set the Alloy.Globals.navigationWindow object:

<code>
Alloy.Globals.navigationWindow.set($.nav); // assuming your navigation window has an id of nav
</code>

Now, if you need to access the navigationWindow from any other window, you just do:

<js>
var navigationWindow = Alloy.Globals.navigationWindow.get();
navigationWindow.openWindow(someOtherWindow);
</js>

That's it, I hope it saves you three hours.