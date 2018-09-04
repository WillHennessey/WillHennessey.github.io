---
layout: post
title: Twitter's mention feature for Rails 
description: Implement your own version of Twitter's mention feature for Ruby on Rails applications using JQuery atwho / at.js
keywords: twitter, mention, ruby, rails, jquery-atwho-rails, at.js
image: /public/images/posts-example.png
comments: true
date: 2017-02-26 01:25:00 +0000
---
<p class='message'>
    <strong>@RailsDeveloper</strong> Want to implement your own version of twitter's mention feature? 
    In this post I'll show you how to implement the mention feature for posts, but don't feel limited to just posts. 
    You can easily adapt the code to work on comments, articles, tickets or whatever you like. 
</p>

![Mention Feature Example]({{ site.url }}/public/images/posts-example.png)

I've uploaded a sample Twitter style app that you guys can find on [my github](https://github.com/WillHennessey/sample-twitter-app). 
It's based off Michael Hartl's [Ruby on Rails tutorial](http://rails-4-0.railstutorial.org) which I highly recommend if you're just starting out with Ruby on Rails.
Just follow the readme on Github to try it out.

## Setup
First things first, you'll need to install and configure [jquery-atwho-rails](https://github.com/ichord/jquery-atwho-rails) and the [redcarpet](https://github.com/vmg/redcarpet) gem. 

Add `gem 'jquery-atwho-rails'`  and `gem 'redcarpet'` to your Gemfile then do a `bundle install`

The jquery-atwho-rails gem is going to handle the javascript that's invoked when a user types **@** in a text field.
I'm also using the redcarpet markdown gem to add markdown to a mention, this will display the mention as a bold clickable link. 

<script src="https://gist.github.com/WillHennessey/ecd9e024e71c1e7c3044f4b622210daa.js"></script>

<script src="https://gist.github.com/WillHennessey/5a51c68f2254b1d4e30b115a527d0d43.js"></script>

## Implementation  

To begin with you'll need to add a markdown function to the application_helper, this function will be used by the view to render your mention, decorated with markdown.
Redcarpet has a tonne of options and several different renderers that you can choose to enable, below is my setup.

<script src="https://gist.github.com/WillHennessey/6e83c887b8ce0fdf0fa18caf2db6d848.js"></script>

Now you can render content anywhere throughout your application with markdown applied, by simply passing the markdown function your content like this.

<script src="https://gist.github.com/WillHennessey/0ae26282e91d06aad595180de476ab8f.js"></script>

Next you'll need to add this to your routes.rb file. 
This route is used to fire an AJAX request that will return a list of users, with usernames matching the characters typed into the text area.
I've added this function to the users_controller for simplicity. 

<script src="https://gist.github.com/WillHennessey/7292f4fd8caf2809d24ea23d9f6c7543.js"></script>

With the routing in place you can add the following code to your users_controller.rb, you'll notice that Mention.all is being passed a param, more on that in a bit .

<script src="https://gist.github.com/WillHennessey/807d3f52df329e8bea48f3ccb4bf3f15.js"></script>

Next is the coffeescript, you can write this code in Javascript if you prefer. I went with coffeescript here as I like the syntax and wanted to practice my coffeescript.
The remoteFilter callback allows us to fire a request to our Users controller to fetch a list of users to mention. 
This will return a JSON object containing a list of usernames matching params[:q] and images for each user.
The "displayTpl" option is basically allowing us to specify what we do with the returned data.
Notice here I've added a CSS class called 'mention-item', this will allow you to tweak the look and feel of the dropdown menu.

<script src="https://gist.github.com/WillHennessey/faf3b4e77abe3976b7d55f35a1b82e41.js"></script>

Now you can add an after\_create callback on your Post model called add_mentions, this will be invoked after the creation of any new posts.

<script src="https://gist.github.com/WillHennessey/eee6c9ba1a480994dc7011f7f9933946.js"></script>

Finally the Mention class, this class will contain all the logic for finding and creating our mentions. 
Some of you might be thinking "why bother with a Mention class when you could just add this functionality to the User model". 
I extracted the functionality to this class for several reasons: 

- It's important to follow the single responsibility principle, this helps prevent ActiveRecord models from growing too complex and becoming a maintenance nightmare.
- It clearly defines the mention concept within the application.
- With this design it's easy to extend the functionality of the feature further.
 
<script src="https://gist.github.com/WillHennessey/c923e495bc529bc54ea50aaa12ea06aa.js"></script>

## Conclusion  
 
In my day job the mention feature was popular with the users working on our systems. 
They asked if I could extend the feature so they could mention teams as well as users. 
With this design it's easy to add another subclass to the Mention class, I added a class called TeamMention to mention teams.
So now instead of having a simple username mention we have a polymorphic mention for both teams and users. 

Let me know in the comment's or on [Twitter](https://twitter.com/sicklickwill) if you have any questions, comments or a creative implementation of the feature! 