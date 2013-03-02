---
layout: post
title: User hierarchies in GoodData
---

*Trees are interesting structures and lots of fun and despair can arise from them. Recently I had a chance to work on an interesting problem at my day job at [GoodData](http://gooddata.com) so I wrote down some notes for others.*

Recently a lot of our customers asked us to solve an interesting problem. They have an army of sales people that sell some goods. On top of classic reporting (questions like "How much did we sell last year?", "How are we doing this year against quotas?" etc.) they wanted to ask questions like "How much did my subordinates sell?". They would also love if they could invite every single person in their company and each would have a constrained view of the warehouse. Seeing only deals themselves and their subordinates made. During couple of last weeks we have tried a lots of ways how to do that. As it often is with computers there is nothing like a free lunch so lets have a brief look on each of those methods and lets talk about their pros and cons.

##What users want
I have seen couple of requests so lets summarize them here. Customer would like to

1. constrain the data each user can see.
2. be able to get answer to question "How much all my subordinates sell together".
3. be able to answer the questions like "How much sells John and all his subordinates". This basically means how difficult is to create a report with filter on particular user
4. be able to answer the question "How much subordinates  of each of my subordinates sell". This is generalization of the number 3 and is basically something like hierarchical sum.

Note, that not every method can provide all of the above. And of course not every customer is interested in all the bullets.

##How is that achieved in GoodData

Generally, to constrain the users view of the data we somehow need to express the user hierarchy in the data. Then we use this hierarchy and a mechanism called variables to constrain the view for each user. Variable is a list of values (basically a filter) or a number. Key part here is that it can be set separately for each user. To be able to do the reports conveniently (3 and 4) we need to somehow represent the Sales Rep and Manager objects in the model.

If you know anything about SQL like  you probably know that it might be pain to express a tree structure in the database. Various SQL based product compete with this problem in various ways. A lots of these issues apply on GoodData as well so if you want to get better understanding of those problems you can look into literature out there. For example in [Joe Celko's Trees and Hierarchies for Smarties](http://www.amazon.com/Hierarchies-Smarties-Kaufmann-Management-Systems/dp/1558609202/ref=sr_1_4?ie=UTF8&qid=1308603757&sr=8-4)

##Naive and often correct approach

For starters let's consider this simple model.

![Data Model](/images/hierarchies/basic_model.png "Basic data model")

and this hierarchy

![User Hierarchy](/images/hierarchies/user_hierarchy.png "User Hierarchy")

The first one is the most straightforward approach anybody can think of. Just enumerate every person's subordinates in the variable. For our model it would look like this

    Joe     => [Joe, Jake, John, Mike]
    Jake    => [Jake, Mike]
    John    => [John]
    Mike    => [Mike]

There are no needed changes in the model. You just need to create a variable, fill it with data and you can directly enjoy the benefits.

There are several advantages of this approach. It is easy to understand since there is no change needed to the model and no need to change any metrics. Every report that should be constrained according to the user signed in just needs to have a variable filter in it and that is all. The performance is very good since under the hood there is simply an attribute filter and that is the case GoodData engine is optimized for. However, there are also numerous disadvantages. The most prominent one is the fact that variable is currently constrained to the number of 200 values in it so it is unusable for larger hierarchies. Though this limitation might change in the future there most probably always will be some upper bound. The other problem is the fact that it is very difficult to crate a report that says how much John's department sold. It is possible but what I would need to do is to create a report and enumerate all people in John's department. It certainly is doable but not pleasant and very error prone. It is straight impossible to create a report where would be all the departments in one report unless I would like to create special metrics for each of those numbers. Since we have no notion of manager in the model (just general sales rep). If I would break the sum of all sales by Sales Rep I would get a report which would tell me how much each individual Sales rep sold. If Joe would not sell anything by himself (not uncommon for managers in larger corporations) he would not be in the report at all. Not what we wanted.

###Let's sum it up

####Pros
* No changes to the underlying model structure
* Easy to understand
* No changes to the underlying metric. The reports that should be constrained just need the variable in the report.
* No changes in the cardinality of either Users or Opportunities
* Performance should be very good

####Cons
* Unsuitable for larger hierarchies due to 200 values limit in variable values
* Hard to provide number 3.
* Unable to provide number 4

##Closure table

So how we can change this approach? There is a way how to express a tree hierarchy called closure table. It is a table that for each manager it has all his subordinates (not only direct ones). These are exactly the same values we put into variables in the previous method but this time we will put that hierarchy into data. I created a picture to illustrate this.

![Closure Table Hierarchy](/images/hierarchies/closure_table_hierarchy.png "Closure Table Hierarchy")

New datasets in our model are Manager and Subordinates. This has several implications. We no longer can claim that we did not change the model but this is not all the changes we need to do. Lets observe the picture for a moment. The Sales Rep dataset is the same as in the previous model. In the Subordinate dataset there is stored the relationship of Manager and his subordinates. The Manager dataset contains the exact same data as Sales Rep dataset. The most important thing happens in the Opportunity table. Notice that each opportunity is now duplicated several times. The duplication factor depends on the number of managers of that particular rep. Opportunity 1234 is there twice one for John who actually sold it and once for his manager Joe.

Lets talk about sizes of Subordinates and Opportunity datasets for a moment. The size of the Subordinates dataset depends on the size of Sales Reps and the shape of your hierarchy. In the worst case if everybody is manager of everybody this would be n^2. For larger amount of reps this could result in a huge table but reality is rarely that bad. Especially in sales companies where the organizational structure is usually flat. The sizes I have experienced is n*k where k is some small one digit number. Factor k actually represents the average number of managers each person has. Ok, so no problem here. The situation is much worse in the case of opportunities. First guess might be that the factor would be the same here as with Subordinates table (mine was) but this appears not to be true. Like I said each opportunity is duplicated so many times the rep has managers. The duplication factor depends on where in the hierarchy lies the people that realize the sales. Since they tend to be at the bottom it is usually a higher number. Because the Opportunities is a fact table that is usually the biggest part of the project  we might get into trouble here.

These are not all the bad news. One is that the metrics need to be designed with this duplication in mind which brings some overhead. The other one is that some ETL step is necessary for this type of hierarchy. You need to duplicate the opportunities and generate the artificial keys for subordinates. The duplication has one last sacrifice and that is the possibility to unknowingly introduce double counting of opportunities into the report.

Now some of the good news. The report can be broken down either by Sales Rep or Manager. In former case it would be the same result we were able to get in previous method. In the latter it is actually performing the hierarchical sum of each manager's subordinates. This is something we were unable to do before.

Lets set the variables. The situation here is similar as in case before. The variable should filter on the Manager attribute. Since we want to break on managers (for people higher in the hierarchy) we again need to enumerate all the subordinates with one slight change which is that we can omit the leaf users.

    Joe     => [Joe, Jake]
    Jake    => [Jake]
    John    => [John]
    Mike    => [Mike]

This will cause that the leaf users will not be included in the report when we will break by manager. This might not be that bad since his number will be included in the report if we break by Sales Rep.

###Let's sum it up

####Pros
* Easy to user
* No changes to the underlying metric. The reports that should be constrained just need the variable in the report.
* Easily enables number 3 and 4

####Cons
* Changes needed to the underlying model structure
* Unsuitable for larger hierarchies due to 200 values limit in variable values thought it is better than previous method
* Changes in the cardinality of Subordinates and Opportunities which might be prohibitive

##Nested Sets

There is one more way how to express a tree hierarchy. Suppose that we will assign two numbers (lets call it left/right) to each user in a way that the following is true. For each subordinate his left is greater than left of his manager and at the same time subordinate's right is lesser than his manager's right. We will than add one fact to the users dataset either left or right. The model then looks like this.

![Left Right Model](/images/hierarchies/left_right_model.png "Left Right Model")

And we create two variables left and right that for each user we fill with the respective values. This image shows the sample values for our hierarchy.

![Left Right Hierarchy](/images/hierarchies/left_right_hierarchy.png "Left Right Hierarchy")

This solution has some unique properties. For arbitrarily deep hierarchy we fill only two numbers for each user. It also does not affect the cardinality of Sales Rep and Opportunities dataset.

    LEFT
    Joe     => 1
    Jake    => 2
    John    => 6
    Mike    => 3
    
    RIGHT
    Joe     => 8
    Jake    => 5
    John    => 7
    Mike    => 4

Unfortunately it has some disadvantages as well. The metrics need to be altered so every one of those contains a definition similar to this (this is pseudo code not actual MAQL but it is very similar).

    WHERE $Left < Left_Fact AND $Right > Left_Fact

This scheme is also problematic in that it is not that straightforward for users to understand although this might be improved by naming things in a smarter way. There are also some problems with performance with this kind of model in GoodData (in relative terms to the previous methods). Since we are not filtering directly on the foreign key in Opportunities dataset the engine needs first to join the left fact from the sales rep table to perform the filtering. We can try to move the fact down the hierarchy directly to the Opportunities dataset but this might be cumbersome if there is more than one fact table. The filtering is still not that fast as in case of other methods since GoodData internally do not create indexes on facts so in any case full table scan is performed.

###Let's sum it up

####Pros
* Usable for arbitrarily deep trees
* Preserves cardinality of Sales Reps and Opportunities

####Cons
* Not easy for user to grasp
* The condition needs to be expressed on metric level
* Works only for trees


## Your Method ....
This list is by no means complete. I have seen some more experiments but they were usually variation on the methods mentioned above. If you have more experience or you have come up with some clever mashup let me know. I will be happy to publish here any interesting ideas.

## Conclusion

When I set up to write this article I was hoping that I would be able to pick a clear winner that will bring the most value for the customer and we will be able to hide most of its complexity under the hood and deal with it during ETL or not at all. Unfortunately each of these solutions have different advantages so each project has to be assessed individually. Particularly size of the project, customer's requirements and users' experience must be taken into account. In GoodData we are now sticking to the variation of first solution where we try to use it on bigger hierarchies by sacrificing some clarity (read: we add more variables). Well, business like usual. Till next time.