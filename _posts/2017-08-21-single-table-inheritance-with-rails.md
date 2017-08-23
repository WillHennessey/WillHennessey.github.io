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
If not you may need to consider another design or even separate tables for each model.

Remember that STI is a great way of DRYing up models that share the same attributes, OO characteristics and database structure. 

It's not supposed to be used to compress many similar tables into one big table.

## A real world example 

I'm going to stick with the JIRA example given above, so the first thing we'll do is generate a migration to create our ticket table. 

Here's an example migration for creating the ticket table.

<div class = "block-code-expanded">
{% highlight ruby %}
class CreateTickets < ActiveRecord::Migration
  def change
    create_table :tickets do |t|
      t.integer :priority, unsigned: true, null: false, default: 4
      t.string :type, limit: 255, null: false
      t.string :abstract, limit: 255, null: false
      t.text :details, limit: 65535, null: false
      t.timestamps
    end
  end
end
{% endhighlight %}
</div>

The `type` column we've defined will be used by STI, to store the sub-model name.

Next we'll need to define our Ticket model and it's sub-models. 
A simple Ticket model will look like this.

<div class = "block-code-expanded">
{% highlight ruby %}
# app/models/ticket.rb
class Ticket < ActiveRecord::Base
  MAX_ABSTRACT_LENGTH = 255
  MAX_DETAILS_LENGTH = 65535
  
  validates :abstract, presence: true, length: { minimum: 10, maximum: MAX_ABSTRACT_LENGTH }
  validates :details, presence: true, length: { minimum: 10, maximum: MAX_DETAILS_LENGTH }
  validates :priority, numericality: { only_integer: true, greater_than_or_equal_to: 0 }
  
  after_create :set_default_priority
  
  def ticket_description
    "This is a #{type.downcase} ticket."
  end
end
{% endhighlight %}
</div>

So we've defined our base Ticket model with a few validations and an after_create action.
Notice that we have not defined the method `set_default_priority` yet, this will be done in the sub-models. 

<div class = "block-code-expanded">
{% highlight ruby %}
# app/models/defect.rb
class Defect < Ticket

  def set_default_priority
    update(priority: 1)
  end
  
  def due_at
    created_at + 1.day + 1.hour
  end
end
{% endhighlight %}
</div>

<div class = "block-code-expanded">
{% highlight ruby %}
# app/models/story.rb
class Story < Ticket

  def set_default_priority
    update(priority: 2)
  end
  
  def due_at
    created_at + 1.day + 3.hours
  end
end
{% endhighlight %}
</div>

<div class = "block-code-expanded">
{% highlight ruby %}
# app/models/issue.rb
class Issue < Ticket
  
  def set_default_priority
    update(priority: 3)
  end
  
  def due_at
    created_at + 1.day + 6.hours
  end
end
{% endhighlight %}
</div>

<div class = "block-code-expanded">
{% highlight ruby %}
# app/models/request.rb
class Request < Ticket

  def set_default_priority
    update(priority: 4)
  end
  
  def due_at
    created_at + 1.day + 12.hours
  end
end
{% endhighlight %}
</div>

All of the sub-models we've defined have a unique implementation of the `set_default_priority` function.
This is to illustrate that by using STI we can have sub-models share the same database table, yet have behave differently.

Using the rails console we test that STI is working as expected.
<div class = "block-code-expanded">
{% highlight ruby %}
# Open the console by typing 'rails console' in your terminal
ticket = Ticket.create(abstract: 'Testing abstract', 
                       details: 'Testing details', 
                       type: 'Defect')    
                                         
puts ticket.description
=> "This is a defect ticket"

puts ticket.priority
=> "1"
{% endhighlight %}
</div>