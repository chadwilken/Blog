---
path: postgres-json-column-type-in-rails-4
date: 2015-10-29T14:20:42.612Z
title: "Postgres JSON Column Type in Rails 4"
description: "In recent days I have been starting a new project using Rails of course and hit a unique issue. I was needing to store arbitrary data with each record, that would likely be different for each…"
---

In recent days I have been starting a new project using Rails of course and hit a unique issue. I was needing to store arbitrary data with each record, that would likely be different for _each_ record. I thought about using MongoDB for this project but didn’t feel that it was necessary since Postgres is always adding awesome new features.

At first glance ,_and Google search_, I thought about using the [hstore extension](http://edgeguides.rubyonrails.org/active_record_postgresql.html#hstore). However, this stores all values as strings and I needed to store all different types of data. Enter _JSON_, we all love JSON so why not store it in the database? Using JSON with Postgres doesn’t even require an extension and is supported out of the box with Rails 4+. Just use the json column type in your migration when creating or modifying tables. You can now store all sorts of complex data types as long as it is valid json.

```ruby
class User < ActiveRecord::Base
end
#let's say our json column is named settings
```

```ruby
me = User.create(name: 'Chad', settings: {notifications: [{email: true, sms:false}], profile_color: '#ccc'})
```

This will save the json as you provide it, most importantly keeping the _values_ as the same _data type_. Now when you retrieve the User’s settings from the you will be able to use all of the corresponding enumerable methods on them.

```ruby
me.settings.each_key { |key| puts me.settings[key] }
#Outputs {"email"=>true, "sms"=>false} #ccc
```

I’m curious about some interesting uses you have found for the _json_ column type, let me know about them in the comments.
