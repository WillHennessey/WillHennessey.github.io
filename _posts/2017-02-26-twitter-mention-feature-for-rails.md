---
layout: post
title: Twitter's mention feature for Rails 
comments: true
date: 2017-02-26 01:25:00 +0000
---
<p class='message'>
    <strong>@RailsDeveloper</strong> Want to implement your own version of twitter's mention feature in your Rails app? 
    In this post I'll show you how to implement the mention feature for posts, but don't feel limited to just posts. 
    You can easily adapt the code to work on comments, articles, tickets or whatever you like. 
</p>

![Mention Feature Example]({{ site.url }}/public/images/posts-example.png)

## Setup
First things first, you'll need to install and configure the [jquery-atwho-rails](https://github.com/ichord/jquery-atwho-rails) gem. 

Add `gem 'jquery-atwho-rails'` to your Gemfile then do a `bundle install`

<div class='block-code'>
{% highlight js %}
//= require jquery
//= require jquery.turbolinks
//= require jquery.atwho
{% endhighlight %}
then add this to app/assets/javascripts/application.js
</div>

<div class='block-code'>
{% highlight css%}
//=require jquery.atwho 
{% endhighlight %}
and this to app/assets/stylesheets/applications.css
</div>

## Implementation  
<div class='block-code'>
<p>You'll need to add this to your routes.rb file, we will be using this route to fire an AJAX request for potential users we can mention.</p>
{% highlight ruby %}
# app/config/routes.rb
Rails.application.routes.draw do
  ...
  get 'mentions', to: 'users#mentions'
end
{% endhighlight%}
</div>

<p>
    Now you can add the following code to your users_controller.rb, you'll notice the function matches the route defined in the routes.rb file.
</p>
<div class='block-code'>
{% highlight ruby %}
# app/controllers/users_controller.rb
def mentions
  respond_to do |format|
    format.json { render :json =>Mention.all(params[:q]) }
  end
end
{% endhighlight %}
</div>

<p>Next is the coffeescript, you can write this code in Javascript if you prefer. I went with coffeescript here as I like the syntax and wanted to practice my coffeescript.
The remoteFilter callback allows us to fire a request to our Users controller to fetch a list of users to mention. 
This will return a JSON object containing a list of usernames matching params[:q] and images for each user.
The "displayTpl" option is basically allowing us to specify what we do with the returned data.
Notice here I've added a CSS class called 'mention-item', this will allow you to tweak the look and feel of the dropdown menu.</p>

<div class='block-code'>
{% highlight coffee-script %}
# app/assets/javascripts/posts.coffee
class @Post
  @add_atwho = ->
    $('#post_content').atwho
      at: '@'
      displayTpl:"<li class='mention-item' data-value='(${name},${image})'>${name}${image}</li>",
      callbacks: remoteFilter: (query, callback) ->
        if (query.length < 1)
          return false
        else
          $.getJSON '/mentions', { q: query }, (data) ->
            callback data

jQuery ->
  Post.add_atwho()
    {% endhighlight %}
</div>

Finally the Mention class, this class will contain all the logic for finding and creating our mentions. 
Some of you might be thinking "why bother with a Mention class when you could just add this functionality to the User model". 
I extracted the functionality to this class for several reasons: 

- It's important to follow the single responsibility principle, this helps prevent ActiveRecord models from growing too complex and becoming a maintenance nightmare.
- It clearly defines the mention concept within the application.
- With this design it's easy to extend the functionality of the feature further.
 

<div class='block-code'>
{% highlight ruby %}
# app/models/mention.rb
class Mention
  attr_reader :mentionable
  include Rails.application.routes.url_helpers

  def self.all(letters)
    return Mention.none unless letters.present?
    users = User.limit(5).where('username like ?',"#{letters}%").compact
    users.map do |user|
      { name: user.username, image: user.gravatar_for(size: 30) }
    end
  end

  def self.create_from_text(post)
    potential_matches = post.content.scan(/@\w+/i)
    potential_matches.uniq.map do |match|
      mention = Mention.create_from_match(match)
      next unless mention
      post.update_attributes!(content: mention.markdown_string(post.content))
      # You could fire an email to the user here with ActionMailer
      mention
    end.compact
  end

  def self.create_from_match(match)
    user = User.find_by(username: match.delete('@'))
    UserMention.new(user) if user.present?
  end

  def initialize(mentionable)
    @mentionable = mentionable
  end

  class UserMention < Mention
    def markdown_string(text)
      # add your app's host here!
      host = Rails.env.development? ? 'localhost:3000' : '' 
      text.gsub(/@#{mentionable.username}/i,
                "[**@#{mentionable.username}**](#{user_url(mentionable, host: host)})")
    end
  end
end
{% endhighlight %}
</div>

## Thoughts 

In my day job the mention feature was popular with the folks working on our systems. 
They asked if I could extend the feature so they could mention teams as well as users. 
With this design it's easy to add another subclass called TeamMention and extend the functionality to mention teams.
So now instead of having a simple username mention we have a polymorphic mention for both teams and users. 