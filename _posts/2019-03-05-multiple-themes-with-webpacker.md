---
layout: post
title: Multiple Bootstrap4 Themes with Webpacker
description: A practical example of how to implement multiple Bootstrap themes using Rails and webpacker.
keywords:  rails webpacker bootstrap, rails multiple themes, webpacker multiple themes, bootstrap themes webpacker, bootstrap themes, bootstrap4 themes, bootstrap, webpacker, rails
comments: true
date: 2019-03-05 15:03:00 +0000
---

> The goal of this post is to share a flexible way of supporting multiple Bootstrap4 themes, using webpacker.
If you hate reading blog posts or just want to jump straight in to the code, I've created a demo app you can pull from Github [here](https://github.com/WillHennessey/webpacker_themes_app).

## From Sprockets to Webpacker
Recently I began the process of migrating a Rails app from sprockets to webpacker.
I hit many snags along the way, but one of the most awkward ones was supporting multiple themes.
The Rails application I was working on had a "default" theme and a "dark" theme.
Both were compiled and served with some "Rails magic" and I did'nt really have to worry too much about how sprockets worked behind the scenes.
As the application grew, our dev team decided to move to webpacker, so we could take advantage of JS libraries like React. 
I figured that since Bootstrap 4 uses SCSS and variable overriding, it'd be a pretty simple aspect of the migration.
Needless to say I was wrong.

If you're a Rails developer who's grown very used to the "Rails way" of doing things, webpacker and it's configuration can seem a little daunting.
Sprockets handles the asset pipeline with little to no configuration. Webpacker does require some basic configuration and understanding of javascript importing, but it's worth it in the long haul.


## Getting started
The first thing you'll need to do is change how your application loads in javascript and stylesheets.
<script src="https://gist.github.com/WillHennessey/ab5cf4a04ac31747713fc67b44a96561.js"></script>
So here we're telling our application to use the webpacker helpers: `javascript_pack_tag` and `stylesheet_pack_tag` to load our assets or 'packs'.

The `current_theme` variable needs to return a theme string, for example: `'cerulean'` or `'darkly'`. This will be the name of our theme's pack.
A good idea for `current_theme` is to add a string column to your uses table called 'theme', where users theme selection can be saved.
Then write a UserHelper method to access this in the view for the current_user.
In the demo app I've just stored the variable as a simple cookie.

The next place we need to look is `app/javascript/packs/application.js`. This is the default entry point for webpacker when it is compiling your assets.

So what does that mean? Basically everything that you want webpacker to compile must be imported here, this is fairly new to sprockets veterans.
<script src="https://gist.github.com/WillHennessey/11d47a9d07a074455a74626b960e57b0.js"></script>

In my `application.js` I'm importing some standard libraries like bootstrap, then I'm importing all of my themes.

## Structure is Key
An important thing to note here is my directory structure, it's very important to keep your packs clearly separated from an early stage.
This will allow us to dynamically request a specific theme in our view code.
<script src="https://gist.github.com/WillHennessey/90300a3a4dec2325c5e5b45515fbdfa3.js"></script>
Lets focus on the theme `app/javascript/packs/cerulean`, this theme contains a index.js file that you can use to import scss files.
<script src="https://gist.github.com/WillHennessey/e527bb83f05f2a7b88adb3a1d4d63d3a.js"></script>
Then in our cerulean.scss file we can import our custom theme variables, bootstrap and any other styles needed for our theme.
<script src="https://gist.github.com/WillHennessey/477495e370bb56c5fc9c29ad0a4aa776.js"></script>
The most important thing to note here is the order of the imports. Here we take advantage of Bootstrap4 using SCSS, by importing our variables first to overwrite the bootstrap default variables.
This is what allows us to have custom colors, fonts etc.

## Creating your own theme

As you can see the rest of our theme's packs have exactly the same structure and exactly the same importing strategy.
This gives us a consistent and predictable folder structure for our code, but also allows fully flexible theming.
With this approach you can add a new theme in minutes!

So now all you have to do is create a `_variables.scss` and override bootstrap to your liking.
<script src="https://gist.github.com/WillHennessey/d6bf0ac62e1e4817062aad76b384f7ff.js"></script> 
If you're like me and don't have time to come up with a totally unique theme by yourself there's lots of places online where you can buy themes, or just download free ones.
For my demo app, I used [BootSwatch](https://bootswatch.com/) themes, you should check them out for ideas and color schemes.