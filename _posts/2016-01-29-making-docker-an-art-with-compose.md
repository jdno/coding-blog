---
layout: post
title: Making Docker an art with Compose
identifier: 4c56aca3e6bc32623821b7b52afa57f58e701ad2be154472ab54ab4834647d1e
---

After bundling {{ site.links.sinatra }} in a {{ site.links.docker }} container
in the first part of my
[Learning Docker](http://coding.jandavid.de/2016/01/17/learning-docker/) series,
let's look at _orchestration_ with {{ site.links.docker-compose }} now. This is
the foundation of the **multi-container environment** we want to build. Get this
right, and adding a database is just two lines of code. Sounds interesting,
right?

## Remember the goal

Before we dive into {{ site.links.docker-compose }}, let's quickly recap what we
set out to do:

The goal for this whole series is to create a {{ site.links.sinatra }} app that
serves a **JSON API**. Since it should be a template for something you can use
in _production_, we also wanted to add a **database** from which the app
retrieves its content.

There are two different ways how you could achieve this:

1. Run the **database** on your machine, and run the **app** inside the
   container.
2. Run both the **database** and the **app** inside a container.

The **first approach** is pretty straightforward. You might already be running a
**database** on your development machine, and accessing it from within the
container is simply a matter of your network configuration. However, the
downside to this approach is that you deprive yourself of the chance to build a
truyl portable and reproducable development environment.

Let's take a look at {{ site.links.docker-compose }} to see what I mean by that.

## Introducing docker-compose

Instead of trying to reinvent the wheel, I'm just going to quote the
documentation of {{ site.links.docker-compose }} to tell you what it does:

> Compose is a tool for defining and running multi-container Docker
> applications.

How does this help us?

Let's think about the different components our app has: From the standpoint of a
consumer, our app has an **API** that provides **content**. Where things get
interesting now is the fact that the content has to come from _somewhere_. This
could be a **database**, an **in-memory data store** ([redis](http://redis.io),
[memcached](http://memcached.org), ...) or simply a **file**.

While your first instinct might be to pack the data source in the same container
as the application (mine certainly was!), separating them into different
containers makes way more sense. First off, you can scale them independently if
you need to, which is awesome for _production_. Second, you apply the _principle
of single responsibility_, and create containers that serve a single very
specific purpose.

And thanks to {{ site.links.docker-compose }}, taking this approach is dead
simple.

## Defining our service(s)

Before we can use {{ site.links.docker-compose }} to start our application, we
need to configure it. This is done via a `docker-compose.yml` file. You can read
about all the available configuration parameters in the
[Compose file reference](https://docs.docker.com/compose/compose-file/), but for
now the [Overview](https://docs.docker.com/compose/#overview-of-docker-compose)
is more than enough.

We add the `docker-compose.yml` file to the top-level folder of our project.
The directory should look like this:

{% highlight bash %}
$ tree .
.
├── Dockerfile
├── app
│   ├── Gemfile
│   ├── app.rb
│   └── config.ru
├── docker
│   └── vhost.conf
└── docker-compose.yml
{% endhighlight %}

Let's open the file and start by defining our application as a new _service_. We
name it `api`. You define a service by adding it as a new top-level element in
the `docker-compose.yml` file:

{% highlight docker %}
api:
{% endhighlight %}

The next step is to tell {{ site.links.docker-compose }} how to build the
service. This is done with the `build` directive, which should point to the
service's `Dockerfile`. In our case, this would look like this:

{% highlight docker %}
api:
  build: .
{% endhighlight %}

Great job! You can now start the application by executing `docker-compose up`.
You should see a bunch of output that either tells you that the application got
build or that it got started.

There is one thing that doesn't work yet, and that is accessing the application.
While it starts and runs, the port the application uses inside the container is
not mapped to a port on either the {{ site.links.docker-machine }} or
**localhost**. We need to tell {{ site.links.docker-compose }} to do that with
the `ports` directive:

{% highlight docker %}
api:
  build: .
  ports:
    - "4567:80"
{% endhighlight %}

This tells {{ site.links.docker-compose }} to map **port 4567** on **localhost**
(or {{ site.links.docker-machine }}) to **port 80** on the **container**. After
restarting {{ site.links.docker-compose }}, you should be able to access the app
on the same URL as before and see its output `"Hello World!"`.

## Wrapping up

While this wasn't a very long or technial post, its results provide a very nice
foundation for our next endeavour: adding a database. This will make a true
_multi-container application_ out of your {{ site.links.sinatra }} app, and then
{{ site.links.docker-compose }} can show its full potential.

If you want to browse the code, you can find it on
[GitHub](https://github/jdno/docker-sinatra-api). For a snapshot of this
specific version, see [version 0.2.0](https://github.com/jdno/docker-sinatra-api/tree/0.2.0).

Hope to see you in the next part!

_Jan David_
