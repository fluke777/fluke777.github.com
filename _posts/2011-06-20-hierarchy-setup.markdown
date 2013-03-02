---
layout: post
title: How to set up a user hierarchy in GoodData
---

*In a recent article I have discussed several options for expressing hierarchies. Though it discussed the problem in context of GoodData BI platform it actually did not show you how to do that exactly. Now it is time to roll up our sleeves and try to hack it together.*

First a word of caution. This is work in progress. I would like this to be the foundation of something bigger and more polished and it serves currently mainly to sort out my ideas and for you to play with it. Feedback is highly appreciated at svarovsky.tomas@gmail.com.

##What you should know.

The tools are written in Ruby so I am expecting some familiarity with these tools and tools surrounding ruby development. Some programming experience is probably required. Thought change this is planned for the future currently only 1.8.x version of ruby is supported. I am assuming you have Ruby and Rubygems installed. Rubygems are packaging manager for Ruby. Also I am assuming you are using some unix flavoured system. Although it should be totally possible to use it on windows based system I have no experience with that so you are on your own here.

##Installing the software

What we are going to do here is that we will install couple of necessary packages. We will clone them from github since this will enable us very easily to follow any changes that will be done in the future on those tools. There are 4 parts that will be interacting together.

* GoodData ruby tool -> This is wrapper over GoodData REST API
* user_hierarchies -> convenience tool that will allow parse us and handle data that express user hierarchies
* variable_uploader -> tool that encapsulates the painful process of uploading attribute variables into project
* variable_uploader_template -> just a template that will enable you to just fill in some values and hopefully have a working sample

    mkdir gooddata_sandbox
    cd gooddata_sandbox

cloning all those 4 repositories mentioned earlier

    git clone git://github.com/fluke777/gooddata-ruby.git
    git clone git://github.com/fluke777/user_hierarchies.git
    git clone git://github.com/fluke777/variable_uploader_template.git
    git clone git://github.com/fluke777/variable_uploader.git

Now we will install all the dependencies but for that we need the right tool - Bundler.

    sudo gem install bundler

Now we can install the dependencies

    cd variable_uploader_template
    bundle install

This should take a while since this should grab all we are going to need from the internet. After it is finished fire up your favorite text editor and let's have a look around. There are two files you should take care about. First is upload.rb which itself takes care of defining what and where it is going to upload and transform.rb that has a definition of how to transform your raw data to something that uploader understands. Lets walk through each of those.

##Transform and adjacency list
One basic form how to express a tree hierarchy in a database is through adjacency list. For each user it has an id of his manager. Data can look like this

    Id,Email,ManagerId
    1,John@example.com,3
    2,Jake@example.com,3
    3,Joe@example.com,
    4,Mike@example.com,2

This data describes hierarchy on the picture.

![User Hierarchy](/images/hierarchies/user_hierarchy.png "User Hierarchy")

This is not a very pleasant format to work with in a databases but it is very well suited to handle in any language that uses a recursion. User hierarchies gem is a little tool that aims to free you from using recursion itself and enable you to express handling of hierarchies with simple DSL.

So let's have a look at a simple transformation. Let's assume that our goal is the to load a list of subordinates for each user including himself. Also lets assumed that there is a file with the above outlined data in the file data/data.csv (which it actually is). Such a transformation is defined in file transformations/transformation.rb. This file reads the input from csv file and prints out the output to console.

    GoodData::UserHierarchies::UserHierarchy::read_from_csv(File.expand_path("../../data/data.csv", __FILE__), {  
      :user_id => "Id",
      :manager_id => "ManagerId",
      :description_field => "Email"
    }) do |hierarchy|
        hierarchy.users.each do |user|
          puts "#{user.Email} => #{([user] + user.all_subordinates).map {|s| s.Email}.join(", ")}"
          # puts "#{user.Email} => #{user.all_subordinates.size}"
          # puts "#{user.Email}" if user.is_manager?
        end
    end

The output is something like this

    John@example.com => John@example.com
    Jake@example.com => Jake@example.com, Mike@example.com
    Joe@example.com => Joe@example.com, John@example.com, Jake@example.com, Mike@example.com
    Mike@example.com => Mike@example.com

Lets break it line by line.

On the first line we actually just say it to load from csv. The strange formulation
    
    File.expand_path("../../data/data.csv", __FILE__)

just says "from fuke data/data.csv" relative to the location of current file without regard of the working directory of the time of execution. The other three lines are configuration for the program to know how to reconstruct the hierarchy. You need to tell in which column there is a user's id and id of his manager. Next we pass a block to this function and into this function we get a hierarchy

    do |hierarchy|
        hierarchy.users.each do |user|
          puts "#{user.Email} => #{([user] + user.all_subordinates).map {|s| s.Email}.join(", ")}"
        end

we grab array of users from this hierarchy and for each of the users we print some string. If you are unfamiliar with ruby blocks check out  for example [Understanding Ruby blocks](http://www.robertsosinski.com/2008/12/21/understanding-ruby-blocks-procs-and-lambdas/). Basically it is variation on lambda/anonymous functions with some syntactic sugar.

    puts "#{user.Email} => #{([user] + user.all_subordinates).map {|s| s.Email}.join(", ")}"

With this line first we print users email. Notice that I am calling a method Email on the user.

    user.Email

This method is actually not defined anywhere and it is actually result of my csv having the column Email. Though this is very convenient there must be little caution not to interfere with any predefined method on Ruby object (I know there is Clean Slate, stay tuned). The last line we are going to look at is 

    ([user] + user.all_subordinates).map {|s| s.Email}.join(", ")

Here we are just joining all subordinates of user with user to form an array. Then we are iterating over the array and creating new array consisting of emails of those subordinates. A lot happened inside those couple of lines.

This is not everything that this tool can do. Since ruby is very expressive a lots can be done easily. You can find some examples on my github page. I would like to implement direct integration with SF since a lots of people are salesforce customers.

##Variables
I have mentioned a variable a couple of times. Since I assume that you are a seasoned GoodData user you are probably familiar with this terms but for the others just couple of sentences. Variable is a thing that holds values. There are two types. Attribute variables and numeric (scalar) variables. The former means basically enumeration of values for certain attribute. The latter means a number. The useful trick is that any user can be set specific value for the variable. Please notice that the current revision of the tool can only work with attribute variables. This is not such an issue atribute variables are used much more often. You can create a variable in application in section Manage > Data > Variables.

##Loading values inside project
We have now prepared the values for prompts and we need to load those into the project. First let's talk about what does it actually mean. In these examples we are considering only attribute filters so let's talk about attributes first. Imagine for example an attribute person - sales rep. Attribute is a logical entity in your project. It may be given multiple names. For example it might have ID (as in SF this is unique) or it might be given a name like Thomas. It might be the case that we have multiple entities of Sales Rep called Tomas in out organization. So the name we use to label the entity does not have to be unique. We call these labels labels and an entity might have multiple labels. These labels are attached to the particular entity that is identified by a key which is unique and it is completely hidden to the end user. When we are setting the Variable value for a user we are basically saying "Find me an entity key for which a label does have value X". This is ok as long as the attribute has only one label. If it has more we need to specify which one should be searched to obtain the key. It is best to choose some label that is unique for that attribute. The underlying structure illustrates the picture below

![Attribute and Labels](/images/hierarchies/attribute.png "Attribute and Labels")

So to specify the loading process we need to tell our script several things . We need to say which variable we will load the data into. If we use other than default label we need to specify which one exactly. We also need to tell it where are the values we would like to load. The values format should be in csv in form

    loginA,value1,value2
    loginB,value3,value53,value30
    ..
    .

For starters I have createda simple DSL for specifying these things in a Ruby syntax. Let's walk through a default script that is in the file upload.rb

    require 'rubygems'
    require 'bundler/setup'
    require 'variable_uploader'

    include GoodData::VariableUploader::DSL

    Project.update :login => 'login', :pass => 'passwrod', :pid => 'pid' do
      upload({
        :file => 'values/data.csv',
        :variable => '/gdc/md/PID/obj/variable_ID',
        :label =>"/gdc/md/PID/obj/label_ID"
      })
    end

First couple of lines is just loading some libraries. The first interesting line is

    Project.update :login => 'login', :pass => 'passwrod', :pid => 'pid

here we provide credentials for using for particular project. To this method we also pass a block inside which we can use several upload statements. Each of these expects file, variable and label arguments. The label is not mandatory and in such case the program finds the default one for the variable. When you are happy with the program fill in the right credentials and run the command

    ruby upload.rb

The program should run and load the values that match for user that do exist in the project.

##How to obtain URIs
In the article there are couple of place where I am refferring either PID or a URI of an entity. You might not be aware of where to get those. Read on.

### PID - project ID
This is easy. When you log in the GoodData and you switch to your project the pid will appear as a part of the URL in location bar. For example your URL might look like this

    https://secure.gooddata.com/#s=/gdc/projects/e3yg9dwo45vx3khl7gsto3jr4i3v24sv|projectDashboardPage|/gdc/md/e3yg9dwo45vx3khl7gsto3jr4i3v24sv/obj/632|aUqCTYr0fLvX

This part 

    /gdc/projects/e3yg9dwo45vx3khl7gsto3jr4i3v24sv

is the project URI and this part

    e3yg9dwo45vx3khl7gsto3jr4i3v24sv

is project ID.

### Object URI
When I am looking for an URI of a particular object I usually recommend to find it inside application in Manage > Data. Let's look how to find URI of an attribute and URi of its label.

![Step 1](/images/hierarchies/manage_attributes.png "Step 1")
![Step 2](/images/hierarchies/manage_attribute_name.png "Step 2")

We can have a look into the definition of the obj in something we inside GoodData call Grey Pages. The URL is simple. https://secure/gooddata.com/ + obj URL

![Step 3](/images/hierarchies/gdc_gray.png "Step 3")

Here you can see that this attribute has 3 labels (internally they are called display forms). We can find URI of our desired one by name. If we are not sure we can check the values that are in that particular attribute by clicking on the link named elements.

##Conclusions
We have walked through the process of creating a variable, transforming the values to be more suitable for our problem, defining an upload plan and uploading the values. As I have stated before this is just a start. I have several ideas how we could improve this so the usage would be more easy. It is also possible that the API and the docs here will change. The advantage of the setup we made is that it is very easy to follow the latest and greatest code through pulling from github. In the meantime try it and tell me what did you like or hate. Any feedback is appreciated on email svarovsky.tomas@gmail.com or here in comments.