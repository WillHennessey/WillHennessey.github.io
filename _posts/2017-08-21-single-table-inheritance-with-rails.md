---
layout: post
title: Single Table Inheritance with Rails
description: A practical example of single table inheritance with Rails.
keywords: single table inheritence, single table, inheritence, Active Record, Rails, rails, ruby, Ruby, MySQL, Database, tickets, JIRA, ticketing system, kanban
comments: true
date: 2017-08-11 15:03:00 +0000
---

Recently my team was tasked with building an in house Kanban Ticketing system, similar to JIRA.
At it's core the system would create and manage tickets, which are assigned to people to work on.

We needed to support several different types of tickets, using the JIRA example you could have: story, defect, request or epic tickets.
It became apparent to the team early on that all these tickets would have the exact same database structure, but would need to behave differently.

## When to use Single Table Inheritance
Before diving into how to implement STI, I want to point out that this design choice only makes sense in certain circumstances.
You should ask yourself the following questions: 
* Does your application have several sub-models that inherit from a parent model class?
* Do the sub-models all have the same attributes and database columns as each other?
* Do the sub-models need to behave differently from the parent model class?
 
If you answered yes to all of the above, then you've come to the right place. 
If not you may need to consider another design like polymorphism, or even separate tables for each model.
Having attributes that are not used by all sub-models can lead to a tonne of nulls in your database.
 
Remember that STI is a great way of DRYing up models that share the same attributes, OO characteristics and database structure. 

## A real world example 

I'm going to stick with the JIRA example given above, so the first thing we'll do is generate a migration to create our ticket table. 

Here's an example migration for creating the ticket table.

<script src="https://gist.github.com/WillHennessey/10a19a49318eba05112e08c6f789a01b.js"></script>

The `type` column we've defined is the default column used by STI, to store the sub-model name.
If you want to use a different column you just need to declare it your base model. 

`self.inheritance_column = :my_cool_column`

Next we'll need to define our Ticket model and it's sub-models. 
A simple Ticket model will look like this.

<script src="https://gist.github.com/WillHennessey/54045668a3f5c57f9e7e33af23e138a3.js"></script>

So we've defined our base Ticket model with a few validations and an after_create action.
Notice that we have not defined the method `set_default_priority` yet, this will be done in the sub-models. 

<script src="https://gist.github.com/WillHennessey/8d41f72f18f6c99b657d7e4b002e1aa9.js"></script>

All of the sub-models we've defined have a unique implementation of the `set_default_priority` function.
This is to illustrate that by using STI we can have sub-models share the same database table, yet have behave differently.

Using the rails console we can test that STI is working as expected.

<script src="https://gist.github.com/WillHennessey/eb817758a287776c164ec501017ab57d.js"></script>

Now we know STI is working as expected, we can begin to build our ticketing system on our ticket model.
We can add our routes and some controller actions. 

<script src="https://gist.github.com/WillHennessey/043b684457b7ac2b88b14e74ea051e38.js"></script>

<script src="https://gist.github.com/WillHennessey/e910fc6477612a88d3536ef8605fa1e7.js"></script>

## Wrapping up 
Through the use of Single Table Inheritance we've been able to design our sub-models to be flexible on a code level, while sharing the same database table.
I hope this post has given you a good understanding of how and when to implement Single Table Inheritance. 
I've created a simple toy app with all the code shown above that you can find [here on Github](https://github.com/WillHennessey/ticketing-system/tree/single-table-inheritance-branch).
Feel free to clone it and experiment yourself!