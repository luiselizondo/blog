---
layout: post
title: Alloy and Titanium Studio, Part 1
date: '2013-04-06 23:20:13'
tags:
- alloy
- backbone-js
- titaniumstudio
- underscore-js
---

<p>Titanium Studio introduced Alloy, an MVC Framework. I'm not gonna cover the installation or basic concepts of an MVC because you can find enough resources on the web about them, I'll just focus on what's not on the web, how to use it and the things you can do with it. But I do  have to cover some basics.</p><ul><li>You don't touch the Resources directory</li><li>Controllers go in the app/controllers folder and are just javascript files</li><li>Models go in the app/models folder and are javascript files</li><li>Styles go in app/styles and are a mix between JSON and CSS. All files have the tss extension, which means Titanium Style Sheets</li><li>Views to in app/views and are xml files. You can use ids and classes to create elements, everything should be wrapped in</li><li>Styles, Models, Controllers and Views are all tied together by the name of the file, that means there's a relationship between the files and that relationship is created by the name of the file. So, if you create a new View called "loginScreen.xml", you should name the controller "loginScreen.js" and the style loginScreen.tss</li><li>Alloy is Backbone.js and Underscore.js compatible/based, so if you know how to use Backbone.js then you won't start from zero, if you don't it's higly recommended that you spend a few days learning it.</li><li>Forget pretty much about everything you know about Titanium Studio before Alloy. You can still use the old way to code, but it's not recommended. Anyway, once you start using Alloy, you'll love it.</li></ul><p>Let's do something easy. I'll create two Windows. In one window, you'll enter a value in a textfield, once you click a button, we'll open a new window and do something with the entered value. This is pretty much what we do with a login screen.</p>

<h3>Code</h3>
So let's code, first, we'll create the two XMLs, the first one is index.xml and it's the first window that will be loaded unless you override this behaviour. For now, I will just forget the tss files ut keep in mind that we use ids and classes. We use ids too in controllers to reference a window or objects in the XML file, like buttons, labels, etc.

File index.xml:
<code>
<Alloy>
    <Window id="urlWindow">
        <Label>Enter the URL of the site you want to connect</Label>
        <TextField id="textField" />
        <Button id="urlRegisteredButton" onClick="urlRegistered">Save</Button>
    </Window>
</Alloy>
</code>

So, what are we doing here? We create a Window and give it an id (urlWindow). Then inside, we add three elements, a label with a text, a textfield with an id (you'll se later why) and a button. Notice that the button has an onClick event, the property will load a callback in the controller file.

File index.js:
<code>
function urlRegistered(e) {
    var valueEntered = $.textField.getValue();
    $.urlWindow.close();
    $.urlWindow = null;
    var args = {
        data : "This value will be passed to loginWindow and be inserted in a label",
        value : valueEntered,
    }
    // Create a new Window by loading the controller loginWindow
    // Pass args as arguments, we'll catch them up in loginWindow using arguments[0]
    var loginWindow = Alloy.createController("loginWindow", args);

    // Get the view of the controller and open it
    loginWindow.getView().open();
}

$.urlWindow.open();

</code>

Remember the callback we'll execute when we click on the button? That's the first thing we do. Notice that $.urlWindow means something like "In this context, use urlWindow" which is the id of the Window we defined in the XML file. This is really cool because we can use this objects declared in our views right into the controller files.

Inside the urlRegistered function, which by now you know it will be executed when we click on the button, we get the value of the Textfield declared in the view, notice that "textField" is just the id we used in our view and can be any other value. We then close the urlWindow, declare it null to free memory and then we define an args object with some properties. 

Finally, we create a new controller in loginWindow. Notice that we use Alloy.createController and we pass both the name of the controller we want to use and the args object. We'll create that file in a few moments, the important thing here is that we create the controller and then in the next line we load the view using loginWindow.getView() and open it.

Now, let's define the loginWindow, which we'll open when we click on the Button. We need two files, the xml and the js file. 

File: loginWindow.xml
<code>
<Alloy>
    <Window id="loginWindow">
        <Label id="connectionLabel">You'll connect to:</Label>
        <Button id="loginButton" onClick="clickLoginButton">Login</Button>
    </Window>
</Alloy>
</code>

Again, we create a window with a different id and add two elements. Notice that I'm assigning an id to the label because I'm gonna change the value of the label in the controller.

File: loginWindow.js
<code>
function clickLoginButton(e) {
    alert("Login button pressed");
}

var args = arguments[0] || {};

$.connectionLabel.text = "data here: " + args.data + " " + args.value;
</code>

Now we define a new callback to be executed when the loginButton is clicked. We keep things simple and just alert the user with a message. The important thing is the next line because we define an arguments object and we take the first argument we passed to the controller in the index.js file. In case you forgot, in index.js we did:

Alloy.createController("loginWindow", args);

And in loginWindow.js we use that argument with:

var args = arguments[0] || {};

That way, we'll just do args.data or args.value or whatever property we defined in args inside the index.js file.

Finally, the next line is very simple, we take the connectionLabel id and assign the text value to use the args object.

$.connectionLabel.text = "data here: " + args.data + " " + args.value;

I hope you now get an idea of how to interact with multiple controllers and views using Alloy and Titanium Studio. Is not that hard isn't it? If you have any questions, please leave a comment.
