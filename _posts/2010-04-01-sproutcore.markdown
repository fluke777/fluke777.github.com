---
layout: post
title: Sproutcore and NodeJS are stars and comets
perex: During spending a lots of time with YUI3, I felt like it is a great set of tools and abstractions, but it is still too rough and low level for real application development. YAHOO gave me the hammer and nails, but never told me, how to build a house. Sproutcore on the other hand is much more mature in this regard and IMHO solves a lots of problems, I have encountered in current web application development. I have been playing with it a little lately, so let's have a look, how NodeJS and Sproutcore, two latest and greatest in client and server side development, are playing together.
---

> __Note__: This was done on Sproutcore version 1.0 and NodeJS version v0.1.28. Since   NodeJS API is quite unstable, if you encounter any problems, let me know and I will try to update this article. 

Sproutcore (if you are new, start [here](http://www.sproutcore.com/get-started/)) has couple of notable features. One of those I like the most is a DataStore. It is a great abstraction, that helps you, to deal with your application data. Search them, alter them, query them and much more. It is very cleanly designed and abstracts you away from your transport layer, so there are tons of ways, how you can get data into it, without noticing in your UI code. After some tinkering I was wondering, how difficult it would be, to make some comet style pushing data to the client from server. Since some time ago, I have been playing with NodeJS, which seemed to be nice for the task, I gave it a try.

## Sproutcore app

Well let's start with a Sproutcore app, though a very simple one. Let's say, you want to have a list of messages, that people all around the world are watching and it would be very nice to have them updated in realtime. Open your terminal and type.

    sc-init Comet

this will create your Sproutcore application, as usual. Next, we are going to need a Message model.

    sc-gen model Comet.Message
    
and also a controller

    sc-gen controller Comet.messagesController

That is enough for now. Start your development environment

    sc-server
    
and fire up your favorite editor. 

###The face
First we will make some simple UI and test, that everything is hooked up all right. Open main_page.js in you resources directory. It should contain something like this
    
    Comet.mainPage = SC.Page.design({

      mainPane: SC.MainPane.design({
        childViews: 'labelView'.w(),

        labelView: SC.LabelView.design({
          layout: { centerX: 0, centerY: 0, width: 200, height: 18 },
          textAlign: SC.ALIGN_CENTER,
          tagName: "h1", 
          value: "Welcome to SproutCore!"
        })
      })
    });

Since we need to display a list of messages, on of the possibilities is to use a ListView. So alter the file to look like this.

    mainPane: SC.MainPane.design({
      childViews: 'messagesView'.w(),
  
      messagesView: SC.ListView.design({
        layout: { centerX: 0, top: 0, bottom: 0, width: 400},
        contentBinding: "Comet.messagesController.arrangedObjects",
        contentValueKey: 'message'
        
      })
    })

We got rid of the LabelView and put a messagesView in its place. We made some positioning, so it should stretch over whole screen from top to bottom, should stay centered and 200 pixels wide. You can change it as you see fit. This line

    contentBinding: "Comet.messagesController.arrangedObjects"

will bind the list to our controller, which we will soon fill with data. The last line will tell the listView, which attribute on our objects, we would like to see in the list. Since there might be several (author, data, rating), we need to provide a hint.

###Controller
Open file `messages.js` from `controllers` directory. Only slight change is required here, since by default the generated controller is an ObjectController and since we need to work with a collection, we would like to have an instance of ArrayController. So change the file to look like this.

    Comet.messagesController = SC.ArrayController.create(
    /** @scope Comet.messagesController.prototype */ {

      // TODO: Add your own code here.

    });

Done here.

###Fixtures
Now, for some data. Since we are in development stage and want to keep it as simple as possible, we will leverage the fixtures for messages. Open message.js from fixtures directory and fill it with some data, like this;

    sc_require('models/message');

    Comet.Message.FIXTURES = [
        {
            guid: 1,
            message: "Is this thing on?"
        },
        {
            guid: 2,
            message: "Seems so"
        }
    ];

One last thing, wee need, before we can check out our application for the first time is to fill some data to our controller. Since our data store will contain these two messages, we just defined, we can make a query to get them. Open main.js and inside the main function add these lines

    var messages = Comet.store.find(Comet.Message);
    Comet.messagesController.set('content', messages);

Ok, that is it, fire up your browser and go to `http://localhost:4020/comet`

You should see a list with two items in it, only one little glitch, there is not the text. After you are happy with what you see, go to the message.js from fixtures directory and delete those hashes again, we will not need them anymore.

##Server

To keep it simple, this is by no means a very complex server implementation. It is not even a real comet server implementing Bayeux protocol. It is just a long polling scheme, neverheless it works as an example.

![Long polling server scheme](https://dl.dropbox.com/sh/rrbxfbxjpdn9jjx/F6GrZnXcdw/images/long_polling_server.png?token_hash=AAHrKDH8pO2ILgP_I-cWd2fE4UrkyC8JICZKbOoi8WH8CQ "Long polling server scheme")
[Omni Graffle source file](/sources/long_polling_server.graffle)

The point here is, that client opens a connection to the server and server will keep it open only for a certain period of time and then send a response and close it. Client will process the response and immediately opens a new connection to the server. This goes on and on. The period of time, that the connection is opened can be divided into three cases

1. on connection server has some messages, that the client doesn't have. In this case, response is send immediately with these new messages and connection is closed.

2. server has no new messages, so it waits until new message arrives. Then it sends it to the client and closes a connection.

3. server has no new messages, so it waits until new message arrives. But it does not for some time, so server returns an empty response and closes the connection.


Pretty simple hm? How we could implement it in NodeJS?

    var sys  = require('sys'), 
        http = require('http'),
        repl = require('repl');

    messages = [];
    message_queue = new process.EventEmitter();


    addMessage = function(message) {
        messages.push({
            guid: messages.length,
            message: message,
            time: Date.now()
        });
        sys.puts("Message added " + message);
        message_queue.emit("newMessage");
    };

    getMessagesSince = function(lastTimeAsked) {
        if (!lastTimeAsked) {
          return messages;
        }
        return messages.filter(function(m) {
          return m.time > lastTimeAsked;
        });
    };

    routeToAction = function(route, req, res) {
      if (route === '/messages') {
        getMessagesAction(req, res);
      } else if (route === '/add') {
        addMesssageAction(req, res);
      }
    };

    addMesssageAction = function(req, res) {
      var message = req.url.split('?')[1];

      res.sendHeader(201, {'Content-Type': 'text/plain', "Connection": 'Close'});

      addMessage(message);
      res.finish();

    };

    getMessagesAction = function(req, res) {
      var lastTimeAsked = req.url.split('?')[1];

      res.sendHeader(200, {'Content-Type': 'text/plain', "Connection": 'Close'});
      var m = getMessagesSince(lastTimeAsked);
      if (m.length) {
          res.sendBody(JSON.stringify(m));
          res.finish();
      } else {
          var timeout = setTimeout(function() {
              res.sendBody(JSON.stringify([]));
              res.finish();
              message_queue.removeListener('newMessage', listener);
          }, 10000);
          var listener = message_queue.addListener("newMessage", function(e) {
              var m = getMessagesSince(lastTimeAsked);
              res.sendBody(JSON.stringify(m));
              res.finish();
              clearTimeout(timeout);
              message_queue.removeListener('newMessage', listener);
          });
      }
    };


    server = http.createServer(function (req, res) {
        // "/messages"
        // "/add"
        var route = req.url.split('?')[0];
        routeToAction(route, req, res);
    });
    server.addListener("connection", function(s) {
        sys.puts("someone has connected " + s);
    });
    server.listen(8000);
    sys.puts('Server running at http://127.0.0.1:8000/');
    repl.start('>');

Lets have a closer look on what means what here

    messages = [];
    message_queue = new process.EventEmitter();

`messages` is the place, we will be stashing our messages into

    message_queue = new process.EventEmitter();

is just a fancy name for something, that can fire events. Other parts of the application will be listening here. Nothing new for a seasoned ui developer like you.

    addMessage = function(message) {
        messages.push({
            guid: messages.length,
            message: message,
            time: Date.now()
        });
        sys.puts("Message added " + message);
        message_queue.emit("newMessage");
    };

this method will store a message object and let others know, that we have added new message, so they can react. Notice, that we are saving time of message arrival, so we can later determine, which messages are new.

    getMessagesSince = function(lastTimeAsked) {
        if (!lastTimeAsked) {
          return messages;
        }
        return messages.filter(function(m) {
          return m.time > lastTimeAsked;
        });
    };

A helper method, which will return messages, that are new, based on lastTimeAsked variable. If lastTimeAsked s not provided, it will return all messages. This feature is used on the first connection of the client, to sync up.

    server = http.createServer(function (req, res) {
        // "/messages"
        // "/add"
        var route = req.url.split('?')[0];
        routeToAction(route, req, res);
    });
    
This one will create an http server. As a parameter a function is provided, that has request and response variables. This function is called on every request. It calls a routeToAction function, which will dispatch a request to one of two actions based on url.

    routeToAction = function(route, req, res) {
      if (route === '/messages') {
        getMessagesAction(req, res);
      } else if (route === '/add') {
        addMesssageAction(req, res);
      }
    };

You can see here, which actions are supported one is intended, to provide a convenient way to add new messages. The other is meant to get the messages.

    addMesssageAction = function(req, res) {
      var message = req.url.split('?')[1];
      
      res.sendHeader(201, {'Content-Type': 'text/plain', "Connection": 'Close'});
      
      addMessage(message);
      res.finish();
    };

Parses out a message and saves it with a function, we have seen before. Then it responds with a 201.

    getMessagesAction = function(req, res) {
      var lastTimeAsked = req.url.split('?')[1];

      res.sendHeader(200, {'Content-Type': 'text/plain', "Connection": 'Close'});
      var m = getMessagesSince(lastTimeAsked);
      if (m.length) {
          res.sendBody(JSON.stringify(m));
          res.finish();
      } else {
          var timeout = setTimeout(function() {
              res.sendBody(JSON.stringify([]));
              res.finish();
              message_queue.removeListener('newMessage', listener);
          }, 10000);
          var listener = message_queue.addListener("newMessage", function(e) {
              var m = getMessagesSince(lastTimeAsked);
              res.sendBody(JSON.stringify(m));
              res.finish();
              clearTimeout(timeout);
              message_queue.removeListener('newMessage', listener);
          });
      }
    };

The most complex function here. This one is implementing the 3 rules mentioned in the beginning of this section. Let's have a look on the key parts.

    var lastTimeAsked = req.url.split('?')[1];
    var m = getMessagesSince(lastTimeAsked);
    
    if (m.length) {
        res.sendBody(JSON.stringify(m));
        res.finish();
    }
    
We get the number of new messages based on the timestamp from the client. This will tell us, when did client asked us last time. Based on this, we will grab new messages. If there are some, we apply rule number one and return them immediately.

    } else {
        var listener = message_queue.addListener("newMessage", function(e) {
            var m = getMessagesSince(lastTimeAsked);
            res.sendBody(JSON.stringify(m));
            res.finish();
            clearTimeout(timeout);
            message_queue.removeListener('newMessage', listener);
        });
        var timeout = setTimeout(function() {
            res.sendBody(JSON.stringify([]));
            res.finish();
            message_queue.removeListener('newMessage', listener);
        }, 10000);
    }

in the other case we register on newMessage event (which is fired, when new message is created) and start a timer, which will fire in 100 seconds here. If timer will fire first, we send an empty array as a response and unregister the listener, because there is no reason to wait again. If we receive a newMessage event first, we clear a timer, unregister a listener and send a message as a response.

    server.addListener("connection", function(s) {
        sys.puts("someone has connected " + s);
    });
    server.listen(8000);
    sys.puts('Server running at http://127.0.0.1:8000/');
    repl.start('>');

Finally we add listener on connection, so an info message is thrown on console, when new connection is received. We hook up server to port 8000. Finally we start a repl, so you can tinker with server on a CLI. This is a reason, why some variables are defined as global.

###Ignition
Ok, put the stuff to some file, say server.js and fire it up with

    node server.js

As I have promised, repl is fired up with prompt `>`. You can check the messages on the server, or add some.

    addMessages('My new message');
    messages

Other way, you can add a record is through http and our second action, we have prepared. You can use for example curl. Type in your terminal

    curl http://localhost8000/add?Message

Now, lets get back to our Sproutcore app, and wire these two together. Technically, we will use some simple XHR request to keep making request to the server. After each response, we will tell DataStore, that we have new records and we let the store handle the rest. The cool thing is, that we don't need to change a single line in the code, we have already written. Yes, I told you, Sproutcore is good.

Again, fire your editor and open main.js and add these lines to the main function.

    var lastTimeAsked = null;
    var getMessages = function(lastTimeAsked) {
      var newTimeAsked = Date.now();
      SC.Request.getUrl('/messages?' + (lastTimeAsked || ""))
          .set('isJSON', YES)
          .notify(this, notifyMethod)
          .send();
      return newTimeAsked;
      console.log(lastTimeAsked);
    };

    var notifyMethod =  function(response, params) {
      if (SC.ok(response)) {
        var data = response.get('body');
        Comet.store.loadRecords(Comet.Message, data.isEnumerable ? data : [data]);
        console.log(response.get('body'));
        lastTimeAsked = getMessages(lastTimeAsked);
      } else {
        // handle the error
      };
    };

    lastTimeAsked = getMessages(lastTimeAsked);

Let's have a look at these once again (I promise, we are almost there).

    var lastTimeAsked = null;

We initialize lastTimeAsked with null value, it is the first time, we want all the server has.

    var getMessages = function(lastTimeAsked) {
      var newTimeAsked = Date.now();
      SC.Request.getUrl('/messages?' + (lastTimeAsked || ""))
          .set('isJSON', YES)
          .notify(this, notifyMethod, {})
          .send();
      return newTimeAsked;
      console.log(lastTimeAsked);
    };

This method just make an XHR call and registers `notifyMethod` as a callback. Then it returns the new lastTimeAsked for the next XHR call.

    var notifyMethod =  function(response) {
      if (SC.ok(response)) {
        var data = response.get('body');
        Comet.store.loadRecords(Comet.Message, data);
        console.log(response.get('body'));
        lastTimeAsked = getMessages(lastTimeAsked);
      } else {
        // handle the error
      };
    };

Here we get the body of a response and use a store.loadRecords helper method. This method will take a constructor of a model object, we would like to create from the data and an array of data. Method will take care of instantiating the records and lets the datastore know, that there are new records available. Than fire new request again.

    lastTimeAsked = getMessages(lastTimeAsked);

Fire up the whole process.

##Wrap up

Since we are in the dev mode and are running our Sproutcore app from server on port 4020 and our comet is sitting on 8000, we cannot use it right away, because XHR cross site policy would stop us. Open Buildfile, which should sit somewhere in you application directory and add this line to the bottom.

    proxy '/messages', :to => '127.0.0.1:8000'

Our Sproutcore dev server will act as a proxy and every request sent to '/messages' will be relayed to our comet server. After it returns, it will be send back to our Sproutcore application. Don't forget, that after such change, you need to restart the sc-server, that we started at the beginning.

Go to browser, refresh it on the address `http://localhost:4020/comet`, if you have your Firebug or Safari console open, you should see XHR requests in progress. Add some messages to the server with techniques described above and you should see the lists to be populated almost immediately. Fire app in another browser window. Should work as well.

###What is missing
Yes, yes I can hear you, there are certainly some things, that are not very nice

* server is not very RESTy
* adding message should be a POST and server doesn't care that much
* code communicating with the comet server should be wrapped inside some module/class and not directly in the main function

But you are a great programmer, so I am sure you can mend all these things and make it much more awesome. This is meant just to get you started and show you, how easy is to do something like this, when such great and well thought out tools like NodeJS and Sproutcore are available. Though, I am eager to hear about mistakes and things I have overlooked, so let me know, if something serious is going on.

Till next time.
