---
path: the-great-shift-asp-net-to-rails
date: 2015-10-29T14:19:11.293Z
title: "The Great Shift — ASP.NET to Rails"
description: "Several months ago I started as the CTO of CompanyCam, a location based photo management solution for contractors. The founder hired a local dev shop to create a web, iOS, and Android app, this…"
---

Several months ago I started as the CTO of [CompanyCam](http://companycam.com/), a location based photo management solution for contractors. The founder hired a local dev shop to create a web, iOS, and Android app, this infrastructure became my new working code base. I am a Ruby advocate and so I began looking for ways to get this to the more nimble and open Ruby ecosystem.

In this post I will cover how I migrated our API from a .NET backend to a Rails backend, keeping functionality and using the existing SQL Server Database.

Some ASP.NET fans may be screaming at their Nokia Lumia’s right now “why would you ever move from _ASP.NET_ to _Rails_?”, so I will address this first.

## Why Switch?

For some people, this question may be a no-brainer, but for others it may not be so apparent. C# and ASP.NET is a powerful language and has _some_ great use cases. It is powerful and is used by many large sites such as [stackoverflow](http://stackoverflow.com/) and my previous employer [Hudl](http://hudl.com/). I needed to rewrite the existing API already since it was miserable to work with. My main gripes with ASP.NET for a startup are the following:

- Speed of Development
- Cost
- Windows Server
- Automated Deployments

I’ll address each one of these below.

## Speed of Development

One of my biggest complaints is constantly re-compiling my source code to test a single change. I’m aware you can work around it, but it isn’t as easy as it is in Ruby. Rails using _Spring_ to reload your code on each run during development, which is nice to say the least.

We integrate with several services, some of which had a NuGet package, but most did not. This required me to write several of my own API wrappers. On the Ruby side, most of the services provided a gem to use, making integration must faster and more stable.

Let’s face it, C# is verbose. For example, our image uploading and resizing code shrunk from several hundred lines of code to about 75 lines in Ruby. It was also easier to follow and read.

## Cost

Everyone _loves_ paying licensing fees right? Of course not. Now I know we can, and we did, get Microsoft BizSpark. This is only good for a year and if you meet certain requirement. After that expires we can enjoy expensive licenses for Visual Studio, Windows Server, and every other piece of the Microsoft pie.

The new backend runs on Ubuntu, Ruby, Rails, and uses several other open source pieces of software. For a _IDE/editor_ I use Sublime Text or Atom, the latter of which is free. Eventually we will migrate to Postgres for the database. The grand total for all of that **\$0**, now that I can get on board with.

## Windows Server

Some people love a [GUI](https://en.wikipedia.org/wiki/Graphical_user_interface) when managing a server. I personally prefer to have my server admin tasks automated using [Chef](https://www.chef.io/) and [Capistrano](http://capistranorb.com/). The only intervention I have with the server is adding SSL keys and setting enviroment variables.

## Automated Deployments

Deploying to Windows is a chore, from my research you basically write your own software to deploy your webapp, or manually FTP everything to the server and restart it when you’re finished. I would rather stand on the shoulders of giants and use Capistrano. With Capistrano I can run a single command to deploy code, migrate the database, concatenate and minify assets, and restart the items needing restarted upon completion.

## Preparation & Concerns

We wanted to Rails piece by piece, so we chose to move the API that drives our iOS and Android apps first. This allows us to ensure that everything is working well and after we feel comfortable we can begin porting the rest over bit-by-bit. Not only that, but once the API is setup, we can begin to consume the new API form the existing webapp using jQuery or Backbone!

My main concern was how I was going to use the SQL Server database from the Rails api. After a bit of digging I found a great gem [activerecord-sqlserver-adapter](https://github.com/rails-sqlserver/activerecord-sqlserver-adapter). The setup was basically non-existent, I had to change a few settings in the database.yml file and it was done.

After figuring out the database I wasn’t sure how I was going to integrate authentication, it turned out to be fairly easy. I chose to use [Sorcery](https://github.com/NoamB/sorcery) and a custom encryption provider. I chose Sorcery, because it adds just enough sugar without overwhelming your codebase like Devise.

I had to tweak a few settings in the sorcery initializer:

```ruby
Rails.application.config.sorcery.configure do |config|
  # We allow username or email address for auth
  user.username_attribute_names = [:Username, :EmailAddress]
  user.email_attribute_name = :EmailAddress
  user.crypted_password_attribute_name = :Passhash
  user.custom_encryption_provider = LegacyEncryptionProvider
  user.encryption_algorithm = :custom
end
```

The main take away from the code above is the LegacyEncryptionProvider which is a class that has encrypt and matches? class methods. Which are extremely simple:

```ruby
def self.encrypt(*tokens)
  Digest::SHA256.base64digest(tokens.join("").encode('UTF-16LE'))
end

def self.matches?(crypted, *tokens)
  crypted == Digest::SHA256.base64digest(tokens.join("").encode('UTF-16LE'))
end
```

The _UTF-16LE_ encoding is what C# uses when you use UnicodeEncoding#GetBytes

The .NET backend had some different user levels that acted as a permission set a.k.a authorization, so for this I decided to choose [Pundit](https://github.com/elabs/pundit). It gave me the flexibility required to ensure only proper user levels could perform certain actions in the controllers.

## To the Code Cave Batman

## Models

To begin I ran rake db:schema:dump to populate the _schema.rb_ file with the existing database structure. The tables and columns were all _CamelCased_, which I wanted to convert to _snake_case_ to match ruby and my personal preferences. I also needed to inform the model of its table name since Rails wasn’t going to be able to infer it automatically. An example model class looks like:

```ruby
class Image < ActiveRecord::Base
  self.table_name = "Images"
  self.primary_key = "ImageId"
  has_many :image_comments, foreign_key: :ImageId, inverse_of: :image
  has_many :image_gallery_images, foreign_key: :ImageId, inverse_of: :image
  has_many :image_galleries, through: :image_gallery_images
  belongs_to :location, foreign_key: :LocationId, inverse_of: :images
  belongs_to :company, foreign_key: :CompanyId, inverse_of: :images
  belongs_to :user, foreign_key: :UploadedById, inverse_of: :images

  accepts_nested_attributes_for :image_comments
  alias_attribute :id, :ImageId
  alias_attribute :active, :IsActive
  alias_attribute :filename, :Filename
  alias_attribute :date_uploaded, :DateUploaded
  alias_attribute :user_id, :UploadedById
  alias_attribute :notes, :Notes
  alias_attribute :location_id, :LocationId
  alias_attribute :lat, :Lat alias_attribute :lon, :Lon
  alias_attribute :company_id, :CompanyId
  alias_attribute :anonymous, :IsAnnonymousLocation
  alias_attribute :update_ticks, :UpdateTicks
  alias_attribute :url_large, :FullSizeUrl
  alias_attribute :url_medium, :WebSizeUrl
  alias_attribute :url_small, :MobileSizeUrl
  alias_attribute :image, :FullSizeUrl
  alias_attribute :date_received, :DateReceived

  before_validation :set_defaults, on: :create
  validates :filename, :date_uploaded, :user_id, :location_id, :lat, :lon, :company_id, :url_large, :url_medium, :url_small, presence: true

  private
    def set_defaults
      self.active = true
      self.date_received = DateTime.now.utc
    end
end
```

At the top you can see where I set the _table_name_ and _primary_key_ attributes for the model. You will also notice a call to alias_attribute for each column in the database.

One caveat of using alias_attribute is that some code will require you to reference the column by the name in the schema. For example, look at association at the top of the class.

## Controllers

Since we were going to be using this as an API first I wanted to namespace everything to make separating API controllers from the eventual view controller easier. I started by following most of [this great up to date post](https://labs.kollegorna.se/blog/2015/04/build-an-api-now/). This gave the basics for RESTful authentication and some default methods that allowed returning errors much more simple.

**Note:** If you also follow the guide above, I recommend choosing your version of ActiveModelSerializers carefully. Basically avoid version 0.9.x since it is a random version. Use either 0.8.x or master (0.10.x) since master builds upon 0.8.x and no 0.9.x.

## Routes

I wanted to use a subdomain for the api to use the speed of dns to resolve calls to the API versus a page request when we add those features in later.

To do so I setup a constraint in the routes file for the subdomain _api_ and a namespace for _v1_ to match it to our API controllers (covered in the article above).

```ruby
constraints subdomain: 'api' do
  namespace :v1 do resources :sessions, only: [:create]
  # omitted for brevity
  end
end
```

## Wrap Up

There are a lot of other items I chose not to cover here. I wanted to keep this digestable and cover only the main items to consider when migrating from ASP.NET with SQL Server to Ruby on Rails. However, I am happy to answer questions you may have. If you feel like something crucial is missing from this post, [let me know](http://www.chadwilken.com/contact).
