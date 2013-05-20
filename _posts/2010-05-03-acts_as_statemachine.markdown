---
layout: post
title: Acts as state machine
perex: Recently I have been posed a task in our company to think about redesigning our invitation and registration infrastructure to enable incorporation of new use cases. Since I have got my feet wet with Sproutcore, I am now seeing patterns and problems on client, I used to thought are server side only. Since I used to to do stuff with Rails, I decided to steal something from them and hack it quickly in JavaScript. Our company uses primarily YUI 3 now, so this is meant to work right in the YUI environment, though it probably will be not much hassle to port it to any library out there.
---

The problem I have encountered is that in the scenario described above, I found a lots of objects, that are in certain states and during transitions between these states, something interesting might happen. This was a recurring pattern, so I thought it might be nice to wrap it in some kind of a module for reuse. Since Ruby people are crazy about all the DSL and DRY stuff and Ruby itself lend itself well for these purposes, someone already created an acts_as_statemachine plugin for ActiveRecord library. Although Javascript itself is not able of such neat DSL constructs, it didn't take long to create an implementation, that acheived something similar and was pleasant to look at. 

First, lets have a look on user's states, events and transitions. The idea behind a state chart is pretty simple. Object can be in one state at a time. Each state has defined some events, that it responds to. The response is to go to another state and maybe do something interesting, some side effect. As you can see, not all events are defined for all states.

 
![User state chart](https://dl.dropbox.com/sh/rrbxfbxjpdn9jjx/rQKVpT2GyV/images/user_state_chart.png?token_hash=AAHrKDH8pO2ILgP_I-cWd2fE4UrkyC8JICZKbOoi8WH8CQ "User state chart")

Lets see, how User described in the statechart could be expressed in code

    UNVERIFIED  = "unverified",
    ACTIVE      = "active",
    CLOSED      = "closed",
    EXPIRED     = "expired",
    AWAITING    = "awaiting",
    CANCELLED   = "cancelled",
    ACCEPTED    = "accepted",
    FAILED      = "failed"
  
    var User = function(userValues) {
      
      // define states
      this.addState(UNVERIFIED)
      this.addState(ACTIVE)
      this.addState(DELETED)
      this.addState(EXPIRED)
      
      // define transisions
      this.addEvent("verify", function(event) {
        event.transition({from: UNVERIFIED, to: ACTIVE});
        event.transition({from: EXPIRED, to: ACTIVE});
      })
    
      this.addEvent("expire", function(event) {
        event.transition({from: UNVERIFIED, to: EXPIRED});
      })
    
      this.addEvent("close", function(event) {
        event.transition({from: UNVERIFIED, to: CLOSED});
        event.transition({from: ACTIVE, to: CLOSED});
      })
      
      // define attributes
      var attributeConfig = {
        firstName: {
          value: undefined
        },
        lastName: {
          value: undefined
        }
      };
      this.addAttrs(attributeConfig);
      this.setAttrs(userValues)
    
    }
    
    // mix in attrbute and acts as state machine mixins
    Y.augment(User, ActsAsStateMachine)
    Y.augment(User, Y.Attribute);

Ok, this is our User object defined, all state machine parts are neatly wrapped in a mixin, so you just need to augment your class with it. I am also using Attribute here to provide KVO and KVC. Now you have your user, you can transfer him between states and the class definition looks kinda tidy and clean.

Let's try something more difficult. Imagine, you have a user, that just filled a form on your registration page. You decided, that the flow will be as follows. After filling this form, you create an account for this user immediately, though it will have some constraints. User will thus be able to log in, but You will also send him an email, that he will use later to confirm, that the credentials he registered with, are valid. If he does not do so in come amount of time, his account will be expired. And he will have to verify it in order to use it further.

Pretty straightforward. You can probably smell, that not only user here has a state, but also registration object could be thought about as a stateful machine. Lets have a look at another state chart, that will depict states and transitions of the registration object.

![Registration state chart](https://dl.dropbox.com/sh/rrbxfbxjpdn9jjx/HI_khazyp0/images/registration_state_chart.png?token_hash=AAHrKDH8pO2ILgP_I-cWd2fE4UrkyC8JICZKbOoi8WH8CQ "Registration state chart")

Let's define our Registration class. This one should implement some kind of workflow and on important places do something to the user.

    var Registration = function(userValues) {
      
      this.addState(AWAITING)
      this.addState(ACCEPTED)
      this.addState(CANCELLED)
      this.addState(EXPIRED)
      this.addState(FAILED)
      
      this.addEvent("resend", function(event) {
        event.transition({from: AWAITING, to: AWAITING});
      })
      
      this.addEvent("cancel", function(event) {
        event.transition({from: AWAITING, to: CANCELLED});
      })
      
      this.addEvent("expire", function(event) {
        event.transition({from: AWAITING, to: EXPIRED});
      })
      
      this.addEvent("accept", function(event) {
        event.transition({from: AWAITING, to: ACCEPTED});
      })
      
      // these hooks will be called in transitions
      awaiting_accepted: function() {
        var user = this.get('user');
        user.verify()
        console.log("User ", user.get('firstName'), "was accepted")
      },
      
      this.setAttrs(userValues);
    }
    
    Y.augment(Registration, ActsAsStateMachine)
    Y.augment(Registration, Y.Attribute);
    

Now we can use our code.

    user = new User({
      state:      UNVERIFIED,
      firstName:  "Tomas",
      lastName:   "Svarovsky"
    });
    
    registration = new Registration({
      state:  AWAITING,
      user:   user
    })
    
    user.get('firstName')
    >> Tomas
    user.get('state')
    >> unverified
    registration.get('state')
    >> awaiting
    registration.accept()
    registration.get('state')
    >> accepted
    user.get('state')
    >> active

So far I could have been pulling your leg. All this API is nice (hopefully :-)), but since I didn't give you implementation of ActsAsStateMachine, those examples would not work. Good news is, that there is a real implementation. The bad news is, that I hacked it together in like one hour, so it probably is rough around edges. But we will get back to this issue later, lets have a look.
    
    
    var ActsAsStateMachine = function(userValues) {
      var attributeConfig = {
        // current state, object is in
        state: {
            value: undefined
        },
        // all states, object has
        states: {
            value: []
        },
        // events, object knows, with associated rules for transitions
        events: {
          value: {}
        }
      };
      this.addAttrs(attributeConfig, userValues);
    }

    ActsAsStateMachine.prototype = {
      addState: function(state) {
        var states = this.get('states');
        states.push(state);
        this.set('states', states);
      },
      addEvent: function(event, fun) {
        var events = this.get('events');
        
        // Simple hacks, allows us to have nicer API,
        // maybe might be implemented as an object, rather than giving each instance a method, don't know
        var rules = []
        rules.transition = function(rule) {
          this.push(rule)
        }
        
        fun.call(this, rules);
        
        events[event] = rules;
        this.set('events', events);
        
        this[event] = function() {
          console.log("event ", event, 'occured');
          this._makeTransitions(event)
        }
      },
      _makeTransitions: function(event) {
        var transitions = this.get('events')[event] || [],
            currentState = this.get('state'),
            currentTransition,
            funName;
        
        transitions = transitions.filter(function(t) {
          return t.from === currentState
        });

        if (!transitions.length) {
          throw "NoTransitionDefined"
        } else if (transitions.length === 1) {
          currentTransition = transitions[0];
          funName = currentTransition.from + '_' + currentTransition.to
          this[funName] && this[funName].call(this);
          this.set('state', currentTransition.to);
        } else {
          throw "MoreTransitionsFound"
        }
      }
    }
    
The mixin is pretty simple. It provides only two public methods, addState and addEvent, with the former, you define the known states of the state machine, with the latter, you define the possible transitions. As a by product, this method will create a method, that bear the same name as event name, that you can call, to invoke the transitions to happen and machine to start turning.

###What is not done yet?
Now back to my statement, that it is rough around edges. There are several problems. First, as with any mixin, there is problem with name clashes. That is probably part, that you will have to accept, because there are not many ways to work around that, that I know of and like. Still, If you do think, that I have picked up really stupid names, that will definitely clash with YUI, let me know. Second, I do not know, if the invocation through created methods is the right way to go. But I kinda like, how you can write the code with it and also wanted to proof, that we also have some meta programming cards up our sleeves. Third, I haven't come up with reasonable way, how to set up initial state, because as a mixin, you do not have very much guaranteed, when your constructor is going to get called. So this is probably up to you. Last issue I can think about i that there is maybe too much state exposed to the user. You can set up state from outside an break the whole machine. So far, for me it worked very well. In near future, I would love to push this to YUI gallery, if anyone would be interested. In the meantime, you can find this on github [http://github.com/fluke777/acts_as_statemachine](http://github.com/fluke777/acts_as_statemachine).

Have fun.