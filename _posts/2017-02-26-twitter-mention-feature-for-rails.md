---
layout: post
title: Twitter's mention feature for Rails 
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

<div class='block-code'>
{% highlight js %}
//= require jquery
//= require jquery.atwho
{% endhighlight %}
add these lines to app/assets/javascripts/application.js
</div>

<div class='block-code'>
{% highlight js%}
//=require jquery.atwho 
{% endhighlight %}
and this line to app/assets/stylesheets/applications.css
</div>

## Implementation  

To begin with you'll need to add a markdown function to the application_helper, this function will be used by the view to render your mention, decorated with markdown.
Redcarpet has a tonne of options and several different renderers that you can choose to enable, below is my setup.

<div class="block-code">
{% highlight ruby %}
# app/helpers/application_helper.rb
def markdown(text)
  renderer = Redcarpet::Render::SmartyHTML.new(filter_html: true, 
                                               hard_wrap: true, 
                                               prettify: true)
  markdown = Redcarpet::Markdown.new(renderer, markdown_layout)
  markdown.render(sanitize(text)).html_safe
end

def markdown_layout
  { autolink: true, space_after_headers: true, no_intra_emphasis: true,
    tables: true, strikethrough: true, highlight: true, quote: true,
    fenced_code_blocks: true, disable_indented_code_blocks: true,
    lax_spacing: true }
end
{% endhighlight %}
</div>

Now you can render content anywhere throughout your application with markdown applied, by simply passing the markdown function your content like this.

<div class="block-code">
{% highlight erb %}
<!-- app/views/posts/_post.html.erb -->
<li>
  <span class="content"><%= markdown(post.content) %></span>
  <span class="timestamp">
    Posted <%= time_ago_in_words(post.created_at) %> ago.
  </span>
  <% if current_user?(post.user) %>
    <%= link_to "delete", post, method: :delete,
                 data: { confirm: "You sure?" },
                 title: post.content %>
  <% end %>
</li>
{% endhighlight %}
</div>

Next you'll need to add this to your routes.rb file. 
This route is used to fire an AJAX request that will return a list of users, with usernames matching the characters typed into the text area.
I've added this function to the users_controller for simplicity. 

<div class='block-code'>
{% highlight ruby %}
# app/config/routes.rb
Rails.application.routes.draw do
  ...
  get 'mentions', to: 'users#mentions'
end
{% endhighlight%}
</div>

With the routing in place you can add the following code to your users_controller.rb, you'll notice that Mention.all is being passed a param, more on that in a bit .

<div class='block-code'>
{% highlight ruby %}
# app/controllers/users_controller.rb
def mentions
  respond_to do |format|
    format.json { render :json => Mention.all(params[:q]) }
  end
end
{% endhighlight %}
</div>

Next is the coffeescript, you can write this code in Javascript if you prefer. I went with coffeescript here as I like the syntax and wanted to practice my coffeescript.
The remoteFilter callback allows us to fire a request to our Users controller to fetch a list of users to mention. 
This will return a JSON object containing a list of usernames matching params[:q] and images for each user.
The "displayTpl" option is basically allowing us to specify what we do with the returned data.
Notice here I've added a CSS class called 'mention-item', this will allow you to tweak the look and feel of the dropdown menu.

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

Now you can add an after\_create callback on your Post model called add_mentions, this will be invoked after the creation of any new posts.

<div class="block-code">
{% highlight ruby %}
# app/models/post.rb
class Post < ActiveRecord::Base 
  
  after_create :add_mentions

  def add_mentions
    Mention.create_from_text(self)
  end
end
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
    # You should bring this user query into your User model as a scope
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
      # Don't forget to add your app's host here for production code!
      host = Rails.env.development? ? 'localhost:3000' : '' 
      text.gsub(/@#{mentionable.username}/i,
                "[**@#{mentionable.username}**](#{user_url(mentionable, host: host)})")
    end
  end
end
{% endhighlight %}
</div>

## Conclusion  
 
In my day job the mention feature was popular with the users working on our systems. 
They asked if I could extend the feature so they could mention teams as well as users. 
With this design it's easy to add another subclass to the Mention class, I added a class called TeamMention to mention teams.
So now instead of having a simple username mention we have a polymorphic mention for both teams and users. 

Let me know in the comment's or on [Twitter](https://twitter.com/sicklickwill) if you have any questions, comments or a creative implementation of the feature! 