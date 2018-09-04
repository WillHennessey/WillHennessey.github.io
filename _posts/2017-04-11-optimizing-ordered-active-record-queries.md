---
layout: post
title: Optimizing Performance of Ordered Active Record Queries
description: Improving the performance of ordered Active Record queries on a MySQL database.
keywords: order_by, Using Filesort, Filesort, Active Record, Rails, rails, ruby, Ruby, MySQL, Database, ordering, queries, query, order
comments: true
date: 2017-04-11 14:20:00 +0000
---

Active Record enables Rails developers to model, store and query their application's data, often without ever having to write a line of SQL.

While this is amazing for getting an app up and running quickly, once it's up and running, you will undoubtedly experience performance issues and find yourself examining SQL queries.
In this post I'll share with you an example of a performance issue that can creep in when using the ActiveRecord::QueryMethods [order](https://apidock.com/rails/ActiveRecord/QueryMethods/order) function and how to fix it.

## A Typical Rails Scenario

Lets take a blog app as an example and assume we have a Posts table and a Users table.
The relationship between these tables is a User has many Posts.
The Posts table is linked by the foreign key `user_id` to the Users table.

### Posts
![Posts table]({{ site.url }}/public/images/Posts_1.PNG)

### Users
![Users table]({{ site.url }}/public/images/User_2.PNG)

You're bound to see an ActiveRecord statement that's something like this:

<script src="https://gist.github.com/WillHennessey/5f197e2fcfcdddf3af19f589254e905a.js"></script>

With Active Record we can easily write a query to retrieve all the posts for a user and order them by when they were last updated.
Active Record will go and generate the following SQL query:

<script src="https://gist.github.com/WillHennessey/f2659e8fb66ced38b5c5d7ada5e15c16.js"></script>

However Active Record won't check this query's performance for you and when you go live with your blog you'll see a big difference ordering 1 post vs ordering 1000 posts.

Copy this generated SQL query into your preferred SQL editor and prepend the `EXPLAIN` keyword to the start of the query.

## EXPLAIN Yo Self

EXPLAIN is a must know for any serious Rails developer, it helps you understand your queries and identify possible performance issues.

There are already many good blog posts on EXPLAIN and it's usage, such as this one [here](https://www.sitepoint.com/using-explain-to-write-better-mysql-queries).
Below is the output of running EXPLAIN on our query.

![Explain Query 1]({{ site.url }}/public/images/explain_1.PNG)

As you can see this is a terrible query, it is using no indexes and doing a full table scan, luckily there's only 4 rows!

So the first thing most people will do in this case is add a foreign key constraint for user_id on the Posts table.
Lets do that first and re-run the explained query.

<script src="https://gist.github.com/WillHennessey/37e19fe066fddda417e21981c1abea0f.js"></script>

<script src="https://gist.github.com/WillHennessey/a3211816cf0c54d95897df699701f997.js"></script>

![Explain Query 2]({{ site.url }}/public/images/explain_2.PNG)

Things are looking a lot better now with the addition of the foreign key constraint.
If you look in the Extra column you'll notice that there are two statements "Using index" (good) and "Using filesort" (bad).

## What is Filesort then?

Using Filesort means you are performing a sort that can't use an index.
You might be thinking to yourself "I already added my foreign key so I do have an index".
While it's true that you have an index, you haven't created an index on the required columns for the sort.

This leads to:
* A full scan of the result set
* Using the filesystem to store chunks when the sort buffer in memory gets full
* A decrease in query performance as the result set grows

## How to fix it

Fortunately the hardest part of dealing with these types of performance issues is identifying them.
To solve the Filesort issue we simply need to add an index to the appropriate columns.
Going back to our Active Record generated SQL query, we can see that the columns we need to index are `user_id` and `updated_at`.

Lets add the index and EXPLAIN our query one last time.

<script src="https://gist.github.com/WillHennessey/c06b289cd1aa542b2a8cd9380f060b30.js"></script>

<script src="https://gist.github.com/WillHennessey/92e34fbb56e1b456231f788c8571f9c9.js"></script>

![Explain Query 3]({{ site.url }}/public/images/explain_3.PNG)

Finally we have a query using the new index `user_id_updated_at` and "Using Filesort" is a thing of the past.