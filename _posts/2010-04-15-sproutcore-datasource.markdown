---
layout: post
title: Sproutcore and DataSource - my take (part 1)
perex: Recently I have seen some confusion about DataStore and granularity of its usage in Sproutcore google group. Since I have been playing with this lately and usually to tidy things up in my head, it is best to put thoughts on paper, here is my take on the usage of DataSources.
---


Before we begin, let's just say, that this article is more trying to answer the why, than how. I assume, you are familiar with at least basic design of [SC.DataSource](http://docs.sproutcore.com/symbols/SC.DataSource.html#constructor) and interaction with [SC.Store](http://docs.sproutcore.com/symbols/SC.Store.html#constructor). If you are not, go, read the docs. This part is covered in great detail on [Sproutcore wiki](http://wiki.sproutcore.com/DataStore-About+DataSources). 

## The question

The question that sparked my mind to think about this a little bit more was "should I use one data source for record type?" When I first saw that question I sat back and wrote a response to the discussion roughly along these lines "No, use a data source instance per back end type". When I revise that answer again, I think, that more appropriate one could have been given. Sproutcore gives you little constraints about how to write, or not to write, a data source implementation, but as usual in programming, the objective should be elegant code, that will solve the problem at hand. If you write less code, there is less likelihood, that it will break and maybe it is even going to be elegant. So the best answer to the question posed in the beginning I was able to put together is "Write your data source in way, that allows you to abstract common patterns in a certain set of models". This is not implying anything about number of back end servers or number of record types per data source. But to prevent this notion to bring more disarray into your understanding, let's have a look at some examples.

## The ideal case
Though it never happened to me, I have heard, that it can actually happen. Imagine, you are writing an application and are free to design both server and client architecture. No legacy apps, no constraints. The application is not that important, but let's say, that it is something involving people, projects and tasks. Each project has several tasks and each person can be assigned to one project. There might be several people working on one project. Let's further assume, that the data coming from the REST API will look like this.

    # Person at /people/1
    {
        "id":           "/people/1"
        "first_name":   "John",
        "last_name":    "Doe",
        "age":          42,
        "project":      "/projects/1",
        
    }
    
    # Project at /projects/1
    {
        "id":           "/projects/1"
        "name":         "Project X",
        "members":      ["/people/1", "/people/2"]
        "tasks":        ["/tasks/1", "/tasks/2"]
    }
    
    # Task at /tasks/1
    {
        "id":           "/tasks/1"
        "name":         "Make it awesome",
        "project":      "/projects/1"
    }
    
    # Also, on uris /projects/, /tasks/ and /people/ there are expected arrays of specific objects
    
I hope, that these patterns I was talking about just a while ago is already started waving at you from the screen. These three types of objects look very similar. They are already flat, without any unnecessary stuffing, ready to be loaded to the store. Let's try writing a data source, that will tackle all these three resources in one go.
    
    // The method used fof actual xhr request, used by all GET related requests
    _getFromUri: function(uri, options) {
      var notifyMethod;
      notifyMethod = options.isQuery ? this._didGetQuery : this._didRetrieveRecords;
      
      SC.Request.getUrl(uri)
        .set('isJSON', YES)
        .notify(this, notifyMethod, options)
        .send();
      return YES;
    },
    
    // Method called by store, when retrieveing query, either local or remote
    fetch: function(store, query) {
      options = {
        store:    store,
        query:    query,
        isQuery:  YES
      };
      if (query === Md.PEOPLE_QUERY) {
        return this._getFromUri('/people', {type: Md.Person});
      }
      return NO;
    },
    
    // called when query is returned from server
    // basically, you just load the records, or their ids and notify store, the you have finished
    // there are three ways, you can handle the query, either it is local or remote
    // remote can be loaded with fully records or just with keys
    
    _didGetQuery: function(response, params) {
        var store     = params.store,
            query     = params.query, 
            type      = params.type,
            deffered  = params.deffered;

        if (SC.ok(response)) {
          if (query.get('isLocal')) {
              console.log("fetch local");
              var storeKeys = store.loadRecords(type, response.get('body'));
              store.dataSourceDidFetchQuery(query);
          } else if (deffered) {
            console.log("fetch remote deffered");
            var storeKeys = response.get('body').map(function(id) {
              return Md.Person.storeKeyFor(id);
            }, this);
            store.loadQueryResults(query, storeKeys);
          } else {
            console.log("fetch remote");
            var storeKeys = store.loadRecords(type, response.get('body'));
            store.loadQueryResults(query, storeKeys);
          }
        } else store.dataSourceDidErrorQuery(query, response);
    },
    
    // Method called by store when requesting individual record
    retrieveRecord: function(store, storeKey) {
      this._getFromUri(store.idFor(storeKey), {
        storeKey:       storeKey,
        store:          store,
        type:           store.recordTypeFor(storeKey)

      });
      return YES;
    },
    
    // handler called when record returns from server, this method will deal with individual records as well as arrays of records od the same type
    _didRetrieveRecords: function(response, params) {
      var store = params.store,
          type = params.type,
          data;
      if (SC.ok(response)) {
        data = response.get('body');
        console.log(data, type)
        store.loadRecords(type, data.isEnumerable ? data : [data]);
      } else store.dataSourceDidError(storeKey, response.get('body'));
    },

That is it. Let's go through some interesting points of this particular design.

1.   The design is agnostic to server URI space. Since the URI is provided by the server, client doesn't have to care. If this will ever change (which in my experience is happening more often than I would ever guess) (say /people/ to /people_space/employees/), this is one less place to care about.
2.   The only thing, that you have to know on the client is an entry point to your application. In my case  it is a PEOPLE_QUERY, which causes the server to return all people (yes, this is going to be very app/view specific, but there should be only certain not too big number of those entry points). All subsequent requests are initiated based on relations, thus the expected record type is known as is the URI, which is guid.
3.   You can decide if remote queries will be downloaded instantly, or on demand, with one flag change on the client (of course you have to deal with it on the server)
4.   Imagine, that new record type is introduced, say Note. Every task can have several notes. Now you have to do this
    1.   Add new Note model to your app, alter Task to have a relation
    2.   Design new REST API for notes and alter the one for tasks
    3.   Crete your controller/view layer

No mention about data source in point 4? Yes, as long as you will stick to the same rules, you used, when you created your other REST resources, there are no needed changes on the data source layer. Everything will just work, so you can clearly see, that we benefit from handling several models in one data source. The Note resource could look for example like this

    {
        "id":   "/notes/1",
        "text": "Some note text"
    }

## Hands on
If you want to play with this live, I have prepared a sample app, which can be cloned [here](http://github.com/fluke777/sproutcore_datasource_part1). Though it does not have a UI, it is pretty fine for some tinkering. Part of the example is fake back end server written in Sinatra, so I expect, you have sinatra installed. The server itself is in subdirectory `server` and you can run it with command 
    
    ruby demo.rb -p9393
    
Don't run it with shotgun or any other tools, that will force it to reload the source code on every request. This way, you would be able to save the stuff back to server and it will be persisted until you restart the server. In the example, the data source is the same as described above, moreover I have implemented one other method, so you can play with updating data to server. Sorry no creation or deletion, you can do that on your own. To get you up to speed, try

    l = Md.store.find(Md.PEOPLE_QUERY)
    p = l.objectAt(0)
    p.get('firstName')
    r = p.get('project')
    t = r.get('tasks')
    t.get('length')
    t1 = t.objectAt(0)
    t1.get('what')

Play with it, change it, break it.

##Multiple responses
Let's show one great advantage of the Sporutcore data store/data source architecture, that cannot be praised enough. I will illustrate it with another example. Let's say, that your hosting provider is a %^&*(%$#, he charges 100$ for each request to the server. Instead of going away to another provider, you decide to leverage great power of Sproutcore and load all your data in one request. You can afford to do this, because in your application there are not so many people/projects/tasks. So how, you would write this in the code? First the server side. Sinatra backand is already at your service and on uri `/all` awaits a collection of data, that looks like this.
    
    {
        "person": [
            {...}
        ],
        "task": [
            {...}
        ],
        "project": [
            {...}
        ]
    }

Notice, that the keys in response are not arbitrary. They are chosen, so that you can infer the record type class out of them. Now go to the client code and change these methods
    
    // Just changing the URI here, all the same otherwise
    fetch: function(store, query) {
        options = {
          store:    store,
          query:    query,
          isQuery:  YES
        };
        if (query === Md.PEOPLE_QUERY) {
          options['type'] = Md.Person;
          return this._getFromUri('/all', options);
        }
      return NO;
    },
    
    // gets the class of the record based on the key of data
    // and then load records
    _loadRecordsOfType: function(type, data, store) {
      var recordType = type.capitalize();
      store.loadRecords(Md.getPath(recordType), data);
    },
    
    // iterate over the keys in response
    _didGetQuery: function(response, params) {
        var store     = params.store,
            query     = params.query, 
            type      = params.type,
            deffered  = params.deffered,
            data      = response.get('body'),
            storeKeys,
            recordType,
            recordData;

        if (SC.ok(response)) {
          for (recordType in data) {
            if (data.hasOwnProperty(recordType)) {
              this._loadRecordsOfType(recordType, data[recordType], store);
            }
          }
          store.dataSourceDidFetchQuery(query);
        } else store.dataSourceDidErrorQuery(query, response);
    },

Again, let's break down some key features here
    
1. You created loader, that will inspect the result and load records of multiple types.
2. Since you used great power of *Convention over configuration* paradigm, you can add new record types (try with the Note object from above) and it will again work, without any code changes, since the record type is inferred from the result itself
3. You can afford this because the insulation, that is between data store and data source. Your application does not care how and when the data gets into the store, so data source is free to decide. You can evolve these two parts of an application without any dependency on each other. Very powerful.

## Wrap up
Well, this was a little stupid example, but you can hopefully see, that you can get smart about your data in many interesting ways, without affecting your other layers of application. As long as you will keep to the rules, you can get done a lot in just couple lines of code and benefit from similarities in your resources by tackling several models in one place. In the instant, you will start breaking those rules, you will need to add tons of conditional logic into your data source implementation, because you will have to deal with discrepancies and little inconsistencies in your data. That is a good sign, that you probably should start thinking about rearranging your data source differently, maybe even break it into pieces and create another one to deal with the odd parts.

In next article, we will do exactly that. We will tackle not so nice situation with legacy app thrown on your back. We will also look into cascading data sources and maybe more.