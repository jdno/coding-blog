---
layout: post
title: How to set up Sinatra with ActiveRecord
identifier: 58bcf3ac33f0e0f0f08533b3828fc81488c10e2d6017ed5ab65a421bbac37642
---

In my series [Learning Docker](http://coding.jandavid.de/2016/01/17/learning-docker/),
I document my steps to use {{ site.links.sinatra }} with {{ site.links.docker }}.
Part of this journey is to set up {{ site.links.sinatra }} with a **database**,
and because I love {{ site.links.active_record }} I want to use it as well. In
this post, I am going to show you how to do that.

## Introduction

In case you are not familiar with my series and its goals, I'll quickly give you
an overview:

For one of my projects I decided to take a look at {{ site.links.docker }}, and
build my application using **services**. Most of these I want to build as **JSON
APIs** with {{ site.links.sinatra }}, since I like how lightweight yet
extensible it is. The first parts of the series dealt mostly with setting up a
very basic {{ site.links.sinatra }} app and getting {{ site.links.docker }} to
run.

If you want to inspect the source code of the application as it is now, follow
this link: [docker-sinatra-api@0.2.0](https://github.com/jdno/docker-sinatra-api/tree/0.2.0)

## Pulling in the dependencies

The first step to get {{ site.links.sinatra }} running with
{{ site.links.active_record }} is define and install the required dependencies.

Let's start by adding the gem `pg` to our `Gemfile`. This is the database
adapter for {{ site.links.postgres }}, which are are going to use in production.
We are also adding `sqlite3` for the development and test environment.

{% highlight ruby %}
group :production do
  # Use Postgresql for ActiveRecord
  gem 'pg'
end

group :development, :test do
  # Use SQLite for ActiveRecord
  gem 'sqlite3'
end
{% endhighlight %}

Next, we need to configure our application. As I mentioned, I am a big fan of
{{ site.links.active_record }}, and want to use it to handle my database
connection for me. While we are at it, we are also adding {{ site.links.rake }}.
{{ site.links.active_record }}'s gem includes all the tasks you might be used to
from {{ site.links.rails }}, and with {{ site.links.rake }} you can use them
in just the same way as in {{ site.links.rails }}:

{% highlight ruby %}
# Use ActiveRecord as the ORM
gem 'sinatra-activerecord', '~> 2.0'

# Use rake to execute ActiveRecord's tasks
gem 'rake'
{% endhighlight %}

Now that we have the right tools in place, let's look at the configuration.

## Configuring the database

To make everything as comfortable as possible for you, we are going to follow
the conventions of {{ site.links.rails }} and put the database configuration in
`config/database.yml`. With **SQLite** for the **development** and **test**
environment, and {{ site.links.postgres }} for **production**, our configuration
file looks like this:

{% highlight yaml %}
default: &default
  adapter: sqlite3
  pool: 5
  timeout: 5000

development:
  <<: *default
  database: db/development.sqlite3

test:
  <<: *default
  database: db/test.sqlite3

production:
  adapter: postgresql
  encoding: unicode
  pool: 5
  host: <%= ENV['DATABASE_HOST'] || 'db' %>
  database: <%= ENV['DATABASE_NAME'] || 'sinatra' %>
  username: <%= ENV['DATABASE_USER'] || 'sinatra' %>
  password: <%= ENV['DATABASE_PASSWORD'] || 'sinatra' %>
{% endhighlight %}

The **production** environment is configured via **environment variables**, what
allows us to keep our sensitive passwords out of the code base and out of
version control. In case the variables are not set, we fall back to default
values that are also going to configure in {{ site.links.docker }} as the
credentials for the database. This makes it easy to use the application with
{{ site.links.docker }} for development purposes.

## Configuring the application

The next step is to make {{ site.links.active_record }} available to our
application. While we have installed the gem and provided its configuration,
we are not yet able to use it within our code.

There are two things we need to do:

1. Add a `require` statement to pull in {{ site.links.active_record }}
2. Tell it where to find its configuration

Look at the beginning of our `app.rb` file to see how this is done:

{% highlight ruby %}
require 'sinatra'
require 'sinatra/json'
require 'sinatra/activerecord'

set :database_file, 'config/database.yml'
{% endhighlight %}

This is all that is necessary to get {{ site.links.active_record }} working
with a {{ site.links.sinatra }} application. You can now define your classes
just like you would in {{ site.links.rails }}.

See the next section for an example.

## Using ActiveRecord

Let's say we want to manage _Resources_. First, let's create a class for them.
Put it in `app.rb`, above the first `get` statement.

{% highlight ruby %}
class Resource < ActiveRecord::Base
  validates :name, presence: true, uniqueness: true
end
{% endhighlight %}

Now that we have our _Resource_ class, we can implement the
**[CRUD operations](http://guides.rubyonrails.org/active_record_basics.html#crud-reading-and-writing-data)**
for it. We start by extending the `#index` action:

{% highlight ruby %}
get '/' do
  json Resource.select('id', 'name').all
end
{% endhighlight %}

Next up is the `#show` action:

{% highlight ruby %}
get '/:id' do
  resource =  Resource.find_by_id(params[:id])

  if resource
    halt 206, json resource
  else
    halt 404
  end
end
{% endhighlight %}

Followed by `#create`:

{% highlight ruby %}
post '/' do
  resource = Resource.create(params)

  if resource
    json resource
  else
    halt 500
  end
end
{% endhighlight %}

`#update`:

{% highlight ruby %}
patch '/:id' do
  resource = Resource.find_by_id(params[:id])

  if resource
    resource.update(params)
  else
    halt 404
  end
end
{% endhighlight %}

And `#delete`:

{% highlight ruby %}
delete '/:id' do
  resource = Resource.find_by_id(params[:id])

  if resource
    resource.destroy
  else
    halt 404
  end
end
{% endhighlight %}

These are very basic implementations, just to show you that everything works
as expected. For an application that is aimed for **production**, I strongly
advice you to do a more robust implementation than this.

## Creating the table

Before we can do anything with this newly written code, we need to create the
table for our _Resource_ class. One of the nicest things about
{{ site.links.active_record }} is the support for **migrations**, which define
changes you make to the database as code that can be run in a reproducible way.
So let's create a migration. Remember {{ site.links.rake }}?

{% highlight bash %}
$ rake db:create_migration NAME=create_resources
{% endhighlight %}

The output of this command is the file path to your newly created migration.
Open it, and add the following instructions:

{% highlight ruby %}
class CreateResources < ActiveRecord::Migration
  def change
    create_table :resources do |t|
      t.string :name, null: false, default: ''

      t.timestamps, null: false
    end

    add_index :resources, :name, unique: true
  end
end
{% endhighlight %}

Just like you would in {{ site.links.rails }}, you can run this migration with
the following command:

{% highlight bash %}
$ rake db:migrate
{% endhighlight %}

The migration created the table `resources` for you, with an implicit field
`id`, the specified field `name` and timestamps containing the times of creation
and the last update.

You can now start the application and browse to its URL. The output from the
`#index` method should be `[]`, since we don't have created any resources yet.

**Congratulations!** You have successfully combined {{ site.links.sinatra }} with
{{ site.links.active_record }}.


## Summary

Setting up {{ site.links.sinatra }} with {{ site.links.active_record }} is not
difficult. We installed the necessary dependencies via our `Gemfile`, configured
the database connection in `config/database.yml`, created and ran a
**migration** with {{ site.links.rake }} and finally added a little bit of
configuration to our `app.rb` to make it use {{ site.links.active_record }}.

The provided code examples are **very basic** implementations of what is
possible, and I strongly recommend that you build a more solid API if you want
to try {{ site.links.sinatra }}. Two things that immediately come to mind when
thinking about areas to improve are **authentication** and **pagination**. Right
now, everyone can create and delete resources like he wants. That is almost
never what you want, though.

If you want to study the code more, have a look at the
[GitHub repository](https://github.com/jdno/docker-sinatra-api) for my series
on [Learning Docker](http://coding.jandavid.de/2016/01/17/learning-docker/).
The following version of the code is the result of this post:

[docker-sinatra-api@0.3.0](https://github.com/jdno/docker-sinatra-api/tree/0.3.0)

You might want to check out the series as well, since it described how to pack
everything we did today into a {{ site.links.docker }} container and make it
portable. Start with the first post:

If you have any questions or suggestions for improvement, feel free to leave a
comment below. I would love to hear from you.

Hope to see you in the next post!

_Jan David_
