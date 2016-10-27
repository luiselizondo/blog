---
layout: post
title: Alloy and Titanium Studio, Part 2. Controllers, Views, Collections and Models
  and how to put it all together.
date: '2013-05-07 04:44:41'
tags:
- alloy
- android
- collections
- controllers
- ios
- models
- titaniumstudio
---

When you're working with an MVC model you need to understand the relationships within, that's a huge part of programming this way. In <a href="http://luiselizondo.net/blogs/luis-elizondo/alloy-and-titanium-studio-part-1">Part 1</a>, I covered how to create Windows, and pass data between them, in this tutorial I'll show you:

1. Create model to save data
2. How to create collections (of items), interact with them and show them on a table view.
3. How to go from a table view showing a collection to a detail window showing the item (model) that was selected.
4. In between, I'll cover how controllers and views interact with the data (models and collections).
5. I'll use a "Things" collection, so we're just gonna be describing things. I do this because it's really easy, we're just gonna name a thing and describe it, this makes it a really easy example if we compare it with a traditional Todo app where we have states and extra code that complicates everything.
6. This will all be made using Android.

The first step is defining how we'll be organizing the data, we'll have:

- a model we'll name: "thing"
- a collection we'll name "thing"

The collection "thing" is composed by multiple "thing", notice I'm using "thing" insted of using "things". The reason I'm doing it is to keep Titanium from collapsing since basically once you name a controller, model, view or collection one way, you have to use the same name for each and everyone of the other components, so I must have a thing.js file for the controller, a thing.js file for the model, a thing.js file for the view, a thing.tss for the styles and also my collection must be named "thing". It also helps to keep things simple, although it doesn't help with how humans name stuff.

Let's start easy. Let's create the model first using the Titanium Studio wizard for creating models:

<code>
// File models/thing.js
exports.definition = {
  config: {
      columns: {
          "name": "string",
          "description": "text"
      },
      adapter: {
          type: "sql",
          collection_name: "thing"
      }
  },        
  extendModel: function(Model) {        
      _.extend(Model.prototype, {
          // extended functions and properties go here
      });
      
      return Model;
  },

   extendCollection: function(Collection) {        
      _.extend(Collection.prototype, {
          // extended functions and properties go here
      });
      
      return Collection;
  }
}
</code>

Now, let's create a new Collection, and because we want it to be globally accessible we'll do it in the alloy.js file. In case you don't know, we can use the alloy.js to create methods or properties that will be available through the entire app.

<code>
// File alloy.js
Alloy.Collections.thing = Alloy.createCollection("thing");
</code>

Now, let's create our controller. The controller will do basically two things, it will add some initial data to the collection by creating models, and then it will react when we select a thing that we want to see in detail. We'll add more functionality later on.

<code>
// File controllers/thing.js

// create a variable to reference our collection
var things = Alloy.Collections.thing;

// our data object it's really simple, it has only one thing
var data = {
  "name": "Pencil",
  "description": "You use a pencil to write things down"
}

// This is our model
// we pass data to our model "thing"
var thing = Alloy.createModel("thing", data);

// Add the "thing" model to the "things" collection
things.add(thing);

// Save our model to the SQL database
thing.save(thing);

// Finally, fetch the collection items
things.fetch();
</code>

Next, we'll create a View.

<code>
// File: view/thing.xml

<Alloy>
<Window id="thingsWindow" class="container">
    <TableView id="tableview" dataCollection="thing">
      <TableViewRow id="row" dataId="" model="{alloy_id}">
        <Label class="rowName" text="{name}">
      </TableViewRow>
    </TableView>
</Window>
</Alloy>
</code>

Let's analyse line by line what we're doing here. First, we create a Window and give it an id and a class. Nothing special here. Then, things get interesting, since we add a TableView and we add a class but we also add a new property named dataCollection with a value of "thing". With this property we're telling the controller to fill the View rows with data from the collection "thing". Then we add a TableViewRow, and as model we set the {alloy_id} token which will be replaced automatically with a value determined by the model. Notice that we won't be seeing any of this, but we can display it if we want. One of the nice things about this is that we'll always have a unique ID for each row. Finally, we create a label, give it a class and again, we use the {name} token to reference the property "name" in our model.

Again, that seems like a lot of text, but basically we're just telling Titanium to populate our TableView with data from the collection "thing" and to replace some values in both the TableViewRow and the Label.

Finally, since we want to load the window when we launch the application, we edit the index.js file and tell Titanium to launch the window that we want.

<code>
// File: index.js
function doClick(e) {  
   alert($.label.text);
}

// $.index.open();

var thingWindow = new Alloy.createController("thing").getView();
thingWindow.open();
</code>

Save everything and run your app. Everything should be working but there's a catch, if we run the app a second time, we'll notice that the line:

<code>
// Save our model to the SQL database
thing.save(thing);
</code>

will add to the database the model every time, we'll deal with that situation later, for now, let's just comment it.

<code>
// Save our model to the SQL database
// thing.save(thing);
</code>

<h2>Events!</h2>

Now we must have some pencils in our table, but nothing happens when we click on them, and we do want a new window to open and showing us some detail of our "thing".

Add the following code to the controllers/thing.js file

<code>
$.tableview.addEventListener("click", function(e) {
  var detailWindow = Alloy.createController("detailWindow", {
 
  });

  detailWindow.getView().open();
});
</code>

Nothing really special here, we're just adding an event listener to the tableview (defined by the class tableview in thing.xml). The event will trigger a function that will create a controller named "detailWindow" and open the window. We're not passing any data to the new detailWindow yet.

Now, create a new controller and name it "detailWindow". Don't enter any code in detailWindow.js for now, but add the following code to views/detailWindow.xml

<code>
<Alloy>
<Window id="detailWindow" class="container">
   <View>
    <Label text="Hello, this is the detailWindow" />
   </View>  
</Window>
</Alloy>
</code>

Again, we're not doing nothing really special here, we just create a Window named "detailWindow" that will open. Inside we have a View and a Label.

Run the code and see what happens.

<h2>Passing data from the parent to the child window</h2>

By now, you have a basic navigation system, if you click on a table row, a new window will open but it'll show nothing but a label with static text. We need to change that, we'll pass the "thing" selected and then we'll show the name and description.

Let's go back to the eventListener we added in controllers/thing.js. If you remember correctly, in the last episode of this tutorials, we learned how to pass data from one window to another, we're doing the same with a slightly different approach because we'll be sending the whole model to the new window.

Modify your event listener in controllers/thing.js so it looks like this one:

<code>
$.tableview.addEventListener("click", function(e) {
var send = things.get(e.rowData.model);
var detailWindow = Alloy.createController("detailWindow", {
   data: send,
   "$model": send
});

 detailWindow.getView().open();
});

</code>
Now modify the views/detailWindow.xml file so it looks like this:
<code>

<Alloy>
<Window id="detailWindow" class="container">
   <View>
    <Label class="nameLabel" text="{name}" />
    <Label class="descriptionLabel" text="{description}" />
   </View>  
</Window>
</Alloy>
</code>

Let's go through both files. In our controller, we're creating a new variable, we called it send, and we're using our Collection (things) to get the model selected. We're using the "model" property from the TableViewRow (check the views/thing.xml file), the one with the token {alloy_id} as value. The controller will get the model for us using the ID. Then we create a new controller to open the detailWindow and we send an object along with two properties, "data" and "$model". The data property will be used in our detailWindow controller file (we created it but it's still empty) and the $model property will be used in our view file detailWindow.xml with some magic.

If you take a look at the detailWindow.xml file you'll see we're using tokens for name and description in our labels. Alloy will magically transform $model into tokens for each property we define in our model file (models/thing.js).

Run the code again to see the changes and click on a row.

You probably don't see both labels because we haven't give them any style, but we can do that really quickly in our styles/detailWindow.tss file

<code>
// File: styles/detailWindow.tss
".container": {
  backgroundColor: "white"
},
".nameLabel": {
  top: "10px",
  left: "10px",
  font: {
    fontSize: '24px'    
  },
  color: "#000"
},

".descriptionLabel": {
  top: "38px",
  left: "10px",
  font: {
    fontSize: '18px'    
  },
  color: "#666"
}
</code>

Finally, I'm just gonna show you how to use the model inside the detailWindow.js controller file. Remember that we also passed data as a property? Well, enter this code.

<code>
// File: controllers/detailWindow.js

var args = arguments[0] || {};
alert(args.$model.attributes.name);
alert(args.data.attributes.name);
</code>

We're just defining here an args variable that will get the first argument we send to this window when we created the controller in the things.js controller. Then, we can either use args.$model or args.data and walk the object. Remember that we need to add attributes to get to the actual properties of the model, just like in Backbone.js

By now hopefully you understand the relations between Controllers, Views, Models and Collections. If you have any comments please let me know. In the next episode I'll cover more about Alloy, some things still need clarification are navigation groups (specially for iOS devices), how to interact with APIs to get data, manipulate data, and maybe some other common use cases.

I'm uploading the source code for the app at https://github.com/lelizondo/titanium-alloy-tutorial-part-2
