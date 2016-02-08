---
layout: post
title: Dockerizing Sinatra
identifier: 479669aecad48436bd5bd580270aaa0595581b2cd516f60e479312555d8564a2
---

This is the first post in my series **Learning Docker with Sinatra**. We start
by building a simple **[Sinatra](http://sinatrarb.com)** app that just serves a
single endpoint with some static content. Then, we build a
**[Docker](https://docker.com)** container around it. In the next parts of the
series, we will be adding a database and orchestration.

> You can find the complete source code on GitHub: [jdno/docker-sinatra-api](https://github.com/jdno/docker-sinatra-api)

## A simple Sinatra app

My longterm goal with this series is to get a working web service that I can use
in a service-oriented web application. So I don't want to create just any
**[Sinatra](http://sinatrarb.com)** app, but one that is actually usable in the
long run.

### Defining dependencies

We start by creating the `Gemfile`. Obviously, our first dependency is
`sinatra`. Since we want to build an API, our app will only respond with
JSON formatted output. To do this, we need a second dependency called
`sinatra-contrib`, which provides the module `Sinatra::JSON`. This module makes
it really easy to return JSON.

Here is the complete `Gemfile`:

{% highlight ruby %}
source 'https://rubygems.org'

# Fix Sinatra version to ensure compatibility with upcoming releases
gem 'sinatra', '1.4.6'

# Use JSON extension provided by the Sinatra::Contrib project
gem 'sinatra-contrib', '~> 1.4.2'
{% endhighlight %}

Run `bundle install` to pull in the dependencies and get ready for the next
step.

### Creating the app

**[Sinatra](http://sinatrarb.com)** is a really nice framework, since it allows
us to create a web application with just a few lines of code. Create a file and
call it `app.rb`, and paste in the following code:

{% highlight ruby %}
require 'sinatra'
require 'sinatra/json'

get '/' do
  json 'Hello World!'
end
{% endhighlight %}

For everyone familiar with **[Sinatra](http://sinatrarb.com)**, this code should
be straightforward. Otherwise, check out the documentation:
[Getting started with Sinatra](http://www.sinatrarb.com/intro.html). It's really
good.

You can start the app with the following command: `ruby app.rb`

When browsing to [localhost:4567](http://localhost:4567), you can see the
output `"Hello World!"`

The beauty of **[Sinatra](http://sinatrarb.com)** is that you really don't need
anything else to get started. Later on, we will be adding a database, but
now this is the application we are working with.

### Preparing for Passenger

While this simplicity is great for development, it comes with a few limitations
with regards to a production environment. Especially from a performance
perspective, there is a lot to be gained by extending **Rack**, the bundled web
server, with a high-performance application server like **Passenger**, **Puma**,
**Unicorn** or **Thin**.

For all of these, you need to add a _rackup file_ called `config.ru`. This file
tells the application servers how to start **Rack**. Simply create it in the
same directory as your app, and paste the following lines in it:

{% highlight ruby %}
require 'sinatra'
require File.expand_path '../app.rb', __FILE__

run Sinatra::Application
{% endhighlight %}

Now we are ready to go to the next section and put everything in a container.

## Dockerizing Sinatra

The next step is to add **[Docker](https://docker.com)**. We want to be able to
run our app from within a Docker container, which allows us to later use the app
in an orchestration of multiple containers, e.g. to build a nice development
environment with a local database.

### Docker Hub - so many options

We won't be building our own base image, but instead rely on
**[Docker Hub](https://hub.docker.com)** and the images that are already
available there. But we still have to make a choice: What application server do
we want to use? We could just go with the built-in **Rack** server that we just
run locally. Or we could go with a dedicated application server like
**Unicorn**, **Puma**, **Thin** or **Passenger**...

For this project, I want to use
**[Passenger](https://www.phusionpassenger.com)**. It provides a great, scalable foundation for whatever we might want to do with the app itself.

When you start searching for `passenger docker` online, you will almost
immediately get to
[phusion/passenger-docker](https://github.com/phusion/passenger-docker). It's
the official GitHub repository for the Docker images made by
**[Phusion](http://phusion.nl)**, the company behind
**[Passenger](https://www.phusionpassenger.com)**.

**[Phusion](http://phusion.nl)** provides official Docker images for
**[Passenger](https://www.phusionpassenger.com)** that come with different
flavors. There are images for **Ruby**, **Node.js** and **Meteor**, and even
more if you want more customization. It should not come as a suprise that we
choose the **Ruby** version. When writing this post, the most current version
supported is **Ruby 2.2.3** in the `phusion/passenger-ruby22` image.

### Our first Dockerfile

Having picked a base image, we can start creating our `Dockerfile`. In the first
line, we say which base image we want to extend. The last few digits declare the
specific version of the image, and it is highly recommended that you do specify
them. Otherwise, you won't be able to build identical images from the same
`Dockerfile` once a new version gets released. And that kinda destroys the whole
purpose of it. Below, you find the first line of our `Dockerfile`:

{% highlight docker %}
FROM phusion/passenger-ruby22:0.9.18
{% endhighlight %}

The next few lines are taken directly from the documentation that is available
on [phusion/passenger-docker](https://github.com/phusion/passenger-docker). We
set the environment variable `HOME`, run through the base image's `init`
process, and enable **nginx** and **Passenger**:

{% highlight docker %}
# Set correct environment variables.
ENV HOME /root

# Use baseimage-docker's init process.
CMD ["/sbin/my_init"]

# Enable nginx and Passenger
RUN rm -f /etc/service/nginx/down
{% endhighlight %}

Next, we have to install our app and configure the application server. Before we
do that, though, I want to improve the directory structure. To configure
**nginx**, we need at least one configuration file. I don't want to put it in
the same folder as our app, but instead keep the app and the Docker
configuration separated. This is what I went with:

{% highlight bash %}
$ tree .
.
├── Dockerfile
├── app
│   ├── Gemfile
│   ├── Gemfile.lock
│   └── app.rb
└── docker
    └── vhost.conf
{% endhighlight %}

Having cleaned up the directory, we can continue with the configuration. In the
`Dockerfile`, we can add the following lines. The first deletes **nginx's**
default site, and the second adds our site.

{% highlight docker %}
# Remove the default site
RUN rm /etc/nginx/sites-enabled/default

# Create virtual host
ADD docker/vhost.conf /etc/nginx/sites-enabled/app.conf
{% endhighlight %}

Of course, we need to provide some configuration for the site. Right now,
`vhost.conf` is either empty or not existing. The following `server` block
tells **nginx** to listen on port 80, enables **Passenger** and points it to the
directory our app lives in.

{% highlight nginx %}
server {
    listen 80;
    server_name localhost;
    root /home/app/webapp/public;

    passenger_enabled on;
    passenger_user app;

    passenger_ruby /usr/bin/ruby2.2;
}
{% endhighlight %}

After configuring the application server, the only thing missing is our app.
The base image already includes a user account that has no elevated privileges,
and we will use this user to run our app. So we start by creating a directory in
the user's home:

{% highlight docker %}
# Prepare folders
RUN mkdir /home/app/webapp
{% endhighlight %}

Next, we want to copy the app and install its dependencies. To take advantage of
**Docker's** caching capabilities, we run `bundle` in its own layer. This approach
is taken directly from **Docker's** article on combining
[Compose and Rails](https://docs.docker.com/compose/rails/). The following lines
copy the files `Gemfile` and `Gemfile.lock` to the temporary folder, and execute
`bundle install` in it:

{% highlight docker %}
# Run Bundle in a cache efficient way
WORKDIR /tmp
COPY app/Gemfile /tmp/
COPY app/Gemfile.lock /tmp/
RUN bundle install
{% endhighlight %}

Now, the only thing missing is our app. This step is really straightforward,
since we are only going to copy the app to the destination directory. To be
absolutely sure all permissions are set correctly, we lastly change the owner of
the app to the user that used by **Passenger**.

{% highlight docker %}
# Add our app
COPY app /home/app/webapp
RUN chown -R app:app /home/app
{% endhighlight %}

There is one last thing we do, and that is clean up after `apt-get` and `bundle`
to decrease the image size.

{% highlight docker %}
# Clean up when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
{% endhighlight %}

If you put everything together, you end up with a nice `Dockerfile` that extends
the official image for **Passenger** and provides a performant and scalable
foundation for the future.

For the record, here is the complete file:

{% highlight docker %}
FROM phusion/passenger-ruby22:0.9.18

MAINTAINER Jan David <jandavid@awesometechnology.de>

# Set correct environment variables.
ENV HOME /root

# Use baseimage-docker's init process.
CMD ["/sbin/my_init"]

# Enable nginx and Passenger
RUN rm -f /etc/service/nginx/down

# Remove the default site
RUN rm /etc/nginx/sites-enabled/default

# Create virtual host
ADD docker/vhost.conf /etc/nginx/sites-enabled/app.conf

# Prepare folders
RUN mkdir /home/app/webapp

# Run Bundle in a cache efficient way
WORKDIR /tmp
COPY app/Gemfile /tmp/
COPY app/Gemfile.lock /tmp/
RUN bundle install

# Add our app
COPY app /home/app/webapp
RUN chown -R app:app /home/app

# Clean up when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
{% endhighlight %}

### Building the image

Now that we have everything in place, we can finally dockerize our app by
building an image. This is done with the following command (substitute the name
with your own name):

{% highlight bash %}
$ docker build -t jandavid/sinatra-example .
{% endhighlight %}

Running the command will give you a lot of output, which should end with a line
like this:

{% highlight bash %}
Successfully built b0442d244543
{% endhighlight %}

**Congratulations! You dockerized Sinatra.**

## Running the app

Let's quickly recap how we run the app previously: We executed the app locally
with the command `ruby app.rb`. By default, the built-in application server
started to listen on port 4567, and we where able to access the app by opening
the URL [localhost:4567](http://localhost:4567) in the browser. This resulted in
the JSON formatted output `"Hello World!"`.

Now, we want to achieve the same result by starting our container. If you take a
look at the **nginx** configuration, you can see that the server is listening on
port 80 for incoming connections. Hidden from you, the base image **exposes port
80 and 443**. This is **Docker** terminology and means that those ports are
accessible from outside the container. We still need to create a mapping,
though, from a port on the host system to the exposed port on the container.

You start containers with the `docker run` command. The command has the
following syntax:

{% highlight bash %}
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
{% endhighlight %}

Looking at the command and what we have built, the following traits have to be
kept in mind:

1. We need to map **port 80** on the container to a port on the host system.
2. We only run a **daemon** in the container, and therefore need no command.

Putting everything together, this gives us the following command to start our
app:

{% highlight bash %}
$ docker run -p 4567:80 jandavid/sinatra-example
{% endhighlight %}

Depending on your host system, you can now access the app at one of the
following URLs:

- On **OS X** and **Windows**, use the IP of the **docker-machine**. This is the
  virtualization layer between your operating system and **Docker**. By default,
  it uses the IP **192.168.99.100**, but you can check with the command
  `$ docker-machine ls`. Depending on the IP, you can access the app at the
  address [192.168.99.100:4567](http://192.168.99.100:4567).
- On **Linux**, you don't need the virtualization layer, and can directly go to
  [localhost:4567](http://localhost:4567).

Opening the URL gives you the same output as running the app locally, in our
case `"Hello World!"`. **Nice!**

## Wrapping up

This is it for the first part on my series on how to use **Docker** with
**Sinatra**. Looking back, we achieved quite a few things:

1. We built a simple **Sinatra** app.
2. We created a **Docker** image with our app.
3. We started a **Docker** container and made our app available to our host
   system.

This is a great result for the first day! In the next part, we will be looking
at **[docker-compose](https://docs.docker.com/compose/)** and use it to start
our app, and after that build a database container.

I hope you were able to take something away from this! If you have any questions
or know how to do things in a smarter way, please share them with me in the
comment section below. The same goes for bugs or errors you find in my code.

> You can find the complete source code on GitHub: [jdno/docker-sinatra-api](https://github.com/jdno/docker-sinatra-api)

Hope to see you in the [next part]({{ page.next.url }})!

_Jan David_
