---
layout: post
title: "Book Review: MapReduce Design Patterns"
published: true
perex: Review of one of the books that I enjoyed recently a lot about Hadoop MapReduce framework and distributed systems in general.
---

![Alt text](http://akamaicovers.oreilly.com/images/0636920025122/cat.gif)

When I first opened this book and skimmed the contents I thought "I already know this stuff" but after I read it from cover to cover I must say that I have really enjoyed reading it and I realized that I did not know the topic that well.

The book is written by 2 guys that work at Stack Owerflow and their focus is on types of tasks that are so common in SQL world like sorting, filtering and summarization that you probably do not consider them worth explaining or patterns. The fact is that when we switch from non-distributed systems to those that span multiple nodes the rules kinda change. They are showing you what exactly is happening in the Hadoop MapReduce framework so you can see that what used to be trivial to do in a non-distributed system can be fairly difficult (even on M/R) to implement and run in a distributed manner efficiently. They not only present you the code for M/R but also show you how the same thing is implemented in other frameworks built on top of M/R like Pig or Hive (SQL) so you can compare and appreciate how much easier it can get if you use something more higher level. In majority of cases the code that you write in Java for M/R is exactly the same what makes the other flavors tick under the hood so you get pretty good feeling for what is possible and what is not.

What I value most in this book is that they show you how the most fundamental things work and how they are supposed to be written in M/R. Majority of the cases is pretty trivial but sometimes you get the aha moment that reveals you more of the underlying implementation of Hadoop. Over the course of this book you hopefully develop sense why certain operations have to be done in map or reduce phase. This helps you when you look on a more involved job in a higher level abstraction like Cascalog you know how many  phases there has to be and get the ballpark estimation if this is an easy job or not.

I have only two smaller problems with this book. First is the title. I suspect that the authors wanted to attract a wider audience by using the word "pattern" in the title but I do not see the majority of topics treated here to be patterns. Only later in the book they talk about problems connected with executing more tasks together as dependent jobs which I think fits the bill. When I bought the book I also thought that it talks about patterns that are used in M/R framework. But this can be easily fixed by reading the TOC upfront :-).

Second problem is that some of the chapters are really trivial. If you do understand map reduce algorithm mapper only tasks or even summarization tasks are kinda obvious Moreover I would like to see some other non SQL like tasks like tree traversal and many others. Authors admit this and there is a brief discussion at the end of the book about omitted or other interesting so I am looking forward to more books on similar topics.

###The Good
* advanced book that is focused on usage of M/R not the framework itself.
* good explanations of typical tasks and showing the right implementation.
* comparison to high level implementations in other languages

###The Bad
* misleading title
* some topics are fairly trivial

###Overall
Definitely a must have book for anybody who wants to gain deeper understanding of Hadoop M/R behavior and learn the way to use it.