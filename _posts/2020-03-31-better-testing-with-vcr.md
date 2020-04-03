---
layout: post
title: Better Testing with VCR
description: How you can improve your tests written around HTTP requests by using VCR. 
keywords:  python, python3, vcrpy, vcr, testing, VCR, API, HTTP Testing, HTTP VCR, API Testing
comments: true
date: 2020-03-30 13:38:00 +0000
---
> In this post I want to share a Ruby gem that is so good it's crossed over into Python. 
> VCR allows you to record your test suite's HTTP interactions and replay them during future test runs for fast, 
> deterministic and accurate tests.

## Hello Old Friend
I've recently changed jobs and with this change came a new language (Python) and some new challenges.
One thing that hasn't changed though is the need to interact with APIs, on my very first ticket I was working with a REST API.

Building APIs and testing them has been a core part of my job for years now, but that was with Ruby.
As I began to design a client for this API I was also searching Google for the "Python way" to design and test an API client.

My heart sank as I saw examples of people mocking APIs, I really didn't want lose time messing around with mocks.
On a whim I Googled "VCR for Python" and boom up comes [vcrpy](https://github.com/kevin1024/vcrpy) 
the Python version of my old friend [vcr](https://github.com/vcr/vcr).

## Install, Record and Replay
Installing VCR is simply `pip install vcrpy` or `pip3 install vcrpy` if you're using Python3.


OK so lets say for example you're working on a client for an API like Github, a basic version would look something like this.


<script src="https://gist.github.com/WillHennessey/0b17be8f146581375cdc419320ac605e.js"></script>

Now we can use our client to fetch data about the user, lets take a simple use case and just get my name.

<script src="https://gist.github.com/WillHennessey/cd7719e32912793034a01399e0b997b8.js"></script>

So far we've got a function using our client class to fetch the username value, but no tests.

Someone could be working on the Github client class in a few weeks time, make some changes and now my `get_my_name()` 
function is broken. We can't have that, so we'll have to add a test.

<script src="https://gist.github.com/WillHennessey/6821dace8f9989e6cadecd15fbcce777.js"></script>

Now we have a test it's job done right? Well not quite...

Lets say our suite of tests is running on Jenkins or Gitlab a week from now and there's a failure,
 the build is broken and people are losing their minds.
 
It's `get_my_name_test()` that's broken the build, but nobody knows how. Then you hear someone say the famous words: 
> Must be networking issues because it's passing now.

You even catch yourself muttering:
> But it ran fine on my machine....

The problem could well be due to networking issues, but networking shouldn't affect your unit tests
, they should be able to run and pass offline on any machine. 

This is where VCR comes in, with only one extra line of code, we add a `vcr.use_cassette` block. 
So when `get_my_name()` uses our `Client` class to make a request to Github's API, all the HTTP request and response will be recorded.
 
 This means that the VCR cassette will record and replay all HTTP interactions that happen within the execution of the `with` block. 

<script src="https://gist.github.com/WillHennessey/3d4a4f7c487eafd5f03639bb9fbb7148.js"></script>

After the first run of this test, VCR will create a YAML file at the path we've provided, this is the VCR cassette.

This cassette has captured all HTTP interactions and can be used to replay them for future runs of this test.

As you can see in the example below, the cassette captures all details of the request and response such as: headers,
 authorization, response codes and server details.

<script src="https://gist.github.com/WillHennessey/62c9ee8b89d81d70830102c6dca623f9.js"></script>

## Now Where's my Walkman?
So hopefully by now you've now seen how amazing VCR is and why the Ruby gem crossed over into the Python world. 

I think it's an incredible testing tool and highly recommend every developer use it when testing code that has HTTP interactions.

That's all folks, as always I hope you enjoyed this post and if you've any questions feel free to leave me a comment below.