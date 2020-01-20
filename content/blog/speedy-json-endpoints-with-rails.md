---
path: speedy-json-endpoints-with-rails
date: 2017-07-26T16:05:15.824Z
title: "Speedy JSON Endpoints with Rails"
description: "CompanyCam is a service to store photos by projects for contractors and other industries that work at a lot of different places. Our goal is to provide as much information as possible to our users…"
---

CompanyCam is a service to store photos by projects for contractors and other industries that work at a lot of different places. Our goal is to provide as much information as possible to our users at a glance. This often means pulling in information from multiple models, potentially filtered based on the current user’s permissions or settings.

When most people think of the performance issues of a large object graph, they think about the cost to query the information out of the database. They pay special attention to avoid things like n+1 queries or loading too much information, which can slow the app to a halt. Yes, this is useful, but in our case, that wasn’t the bottleneck — our issue was Active Model Serializers which had mediocre caching and wouldn’t use the eagerly loaded associations.

In this post I will discuss our decision to move from Active Model Serializers to the [jb gem](https://github.com/amatsuda/jb). The jb gem allowed us to vastly improve caching, reduction of n+1 queries, and most importantly our response time.

## Some Context

Let’s get a better understanding of the object graph and why some things were so slow when using AMS. Take a look at the main screen of our mobile app.

<figure>

![](/assets/speedy-json-endpoints-with-rails/screentshot.jpeg)

<figcaption>Project Feed Cards</figcaption></figure>

In case it isn’t apparent at first glance, we are pulling in the project name, number of photos at the project, the 6 most recent photos at the project, who took the last photo, and on some screens we show how many photos have been taken that day. Then repeat that 50 times (one for each card in the page of projects). The query was difficult to get optimized (we can save that for another post), but the serialization was still taking an average of 500–700ms and some all the way up to 2,500ms. Using [Scout](http://scoutapp.com/) we were able to see that this was because Active Model Serializers was causing a n+1 query when rendering instead of using the eager loaded associations.

I did a bit of research and stumbled on a issue thread on GitHub where user’s were saying that sometimes caching made things _slower_. I happened to stumble across an article talking about [jb](https://github.com/amatsuda/jb), a gem that uses Rails view rendering with partials to make it faster than other options like JBuilder. The added benefit is that it allows easily caching partials and will use eagerly loaded associations instead of triggering additional queries. It also provides access to all of your view helpers, so methods like `current_user` are accessible to your JSON view and you don’t have to provide a `serialization_scope` like you do with AMS.

## How It Works

The gem adds itself as a render handler so that it will check in the directory matching the controller path with the `json.jb` extension. If you want to render a template that doesn’t follow the Rails lookup default you can specify `render ‘path/to/your/template.json.jb` To start, let’s look at our `index.json.jb` template to see how we get render a collection of projects.

We simply call `map` on projects, which means that this view will return an Array of projects. For each project we first check to see if it has a cached version we can load to avoid unnecessary rendering. You will notice we are adding ‘v1’, ‘recent’, and then whether or not the user can see all users photos (a permission) to the cache key so that this project is used only in this context since the contents are unique. Then for each project we render the `project` partial. Notice we are following the Russian doll caching method.

The beauty of this is that you are simply constructing and returning an object that can be used as valid JSON, in our case this is a Hash. This gives you nearly unlimited freedom on how you construct your JSON object.

## So What’s the Result?

With these few simple changes we were able to reduce our average request time for the controller from 500ms to just _50ms!_ That is _10x_ faster and only took 30 minutes to implement*.* Needless to say, I am extremely pleased with these results. Not to mention the reduced load on the database is a free bonus.

## Where to Now?

It is so simple migrating an Active Model Serializers class to the new jb view that anytime I see a controller with slow responses due to AMS I can spend 15 minutes converting it. I have already moved a handful of other controller endpoints to the new method and have noticed significant increase in speed and reduction in load on the database.

If you have any stories about complete object graph rendering I would love to hear about them in the comments below.
