---
layout: post
title: Dockerized Databases with Sinatra
identifier: c61e7606355d6723263a20d12eddbaa518eab5558a5d37d929fe7d1a5f0efbff
---

In this part of my
[Learning Docker](http://coding.jandavid.de/2016/01/17/learning-docker/) series,
we are going to look at **databases** and see how they can be used with
{{ site.links.docker }}. The goal is to create a **development** environment
that can be used with our {{ site.links.sinatra }} app.

> You can find the complete source code on GitHub:  
>[jdno/docker-sinatra-api@0.4.0](https://github.com/jdno/docker-sinatra-api/tree/0.4.0)

## Requirements

Almost any decent application nowadays has the demand for some **persistent**
data storage - be it for **content**, **session information** or
**configuration**. This is also true for the service we developed in the
[last posts]({{ page.previous.url }}). In this specific case, we need to store
the **content** of the application in a persistent manner.

For this, we want to use a **database** and make it available to the
application. While there are different strategies how you could do this, putting
the database in a {{ site.links.docker }} container and adding it to our
orchestration is the most sensible approach. This allows us to take full
advantage of {{ site.links.docker-compose }} and its support for _multi-
container applications_, and makes it increadibly easy to start the development
environment on another machine.

## Choosing a Database

For this project, we want to use {{ site.links.postgres }} as our **database
management system**. Before going into the details on how to create the
{{ site.links.docker }} container for the database, let me quickly reason why I
only want to use this container for **development**.

When we deploy the application to **production**, certain needs arise with
regards to the data stored within the database. For example, it must be
**persistent** and **available**. While this sounds easy enough, preventing and
dealing with **data losses** and **failures** is an art in itself. Essentially,
you need **backups** to protect against **data loss**, and you need **replicas**
to ensure **availability**.

While you could take care of this yourself, and many people do, I prefer to
**outsource** this to specialists who know what they are doing. I love building
applications, so my knowledge of database maintenance is adequate at best, and
quite frankly I don't like it. With a hosted service like
**[Amazon RDS](https://aws.amazon.com/rds/)** (for example), I get a highly
available database management system that is maintained by professionals and
that takes care of almost everything out-of-the-box.

## Getting the Database

Thanks to {{ site.links.docker-hub }}, this step is incredibly easy. I bet that
whatever database you want to use, an image already exists on
{{ site.links.docker-hub }}. So go there and search for the **database
management system** you want to use. For {{ site.links.postgres }}, you probably
want to use the [offical image](https://hub.docker.com/_/postgres/).

Instead of downloading and building the image manually with
{{ site.links.docker }}, we automate the process and use
{{ site.links.docker-compose }}. Open up the `docker-compose.yml` file, and add
a new service:

{% highlight yaml %}
db:
  image: "postgres"
{% endhighlight %}

While this would be all that {{ site.links.docker-compose }} needs to know to
download, build and start the image, looking at the image's
[documentation](https://hub.docker.com/_/postgres/) reveals that we can start it
it with some _environment variables_ to automatically create a database if none
exists already. So let's add them do `docker-compose.yml`:

{% highlight yaml %}
db:
    image: "postgres"
    environment:
        - POSTGRES_USER=sinatra
        - POSTGRES_PASSWORD=sinatra
{% endhighlight %}

The variables `POSTGRES_USER` and `POSTGRES_PASSWORD` are **optional** and
specify a database and a user that get created during initialization. This is
done only **if no database exists**, so you don't have to worry about
overwriting your existing data.

To be able to access the database via the command line utility **psql** or with
other tools, it is useful to expose the database's port to the host. This is
done by simply adding a new `ports` directive to `docker-compose.yml`:

{% highlight yaml %}
db:
  image: "postgres"
  environment:
    - POSTGRES_USER=sinatra
    - POSTGRES_PASSWORD=sinatra
  ports:
    - "5432:5432"
{% endhighlight %}

When you run `docker-compose up` now, you should see a whole bunch of output
coming from both a `api_1` and a `db_1` service. Using **psql**, you can inspect
the database with the following command (change IP to `localhost` on **Linux**):

{% highlight bash %}
$ psql -h 192.168.99.100 -U sinatra sinatra
{% endhighlight %}

This connects to the database _sinatra_ (last argument) as the user _sinatra_ on
the host _192.168.99.100_ (aka _docker-machine_).

## Configuring Sinatra

The next step is to configure our application to make use of a database. I have
posted detailed instructions on how to combine {{ site.links.sinatra }} and
{{ site.links.active_record }}, which you can find here:

[How to set up Sinatra with ActiveRecord]({{ page.previous.url }})

Head over there and come back once you set your application up.

After preparing {{ site.links.sinatra }}, there is one thing we still want to
do. If you followed the instructions, this step is not required, but still a
good practice. In the file `config/database.yml`, we configured our app to make
use of some environment variables that are not available to the container yet.
So let's add some lines to `docker-compose.yml`:

{% highlight yaml %}
api:
  build: .
  ports:
    - "4567:80"
  links:
    - "db"
  environment:
    - DATABASE_HOST=db
    - DATABASE_NAME=sinatra
    - DATABASE_USER=sinatra
    - DATABASE_PASSWORD=sinatra
{% endhighlight %}

The `environment` section sets the **environment variables** our app uses to
discover the database and connect to it. Having these in place makes it easy to
make changes to the configuration, e.g. changing the database user.

## Linking app and database

The last step for us to do is to link our **application container** with the
**database container**. While you could use the port and the IP address of
the {{ site.links.docker-machine }}, it is better to use the dynamic linking
feature of {{ site.links.docker-compose }}. Just imagine that a colleague wants
to check out the project, but he's running **Linux**, while you are running
**OS X** and have configured to application to connect to **192.168.99.100**.
The application will break for your colleague, since {{ site.links.docker }}
runs natively on his machine and makes no use of {{ site.links.docker-machine}}.

To set up linking with {{ site.links.docker-compose }}, just open up its
configuration in `docker-compose.yml` and add the directive `links` to the app:

{% highlight yaml %}
api:
  build: .
  ports:
    - "4567:80"
  links:
    - "db"
{% endhighlight %}

That's it. Your app is now magically linked to your database. Ok, not magically,
but it's pretty close. {{ site.links.docker-compose }} takes care of that for
you.

Since our application now has a database, you can start playing around with it,
and create records either via the API or with **psql**.

## Persistent Data

While this was all pretty straightforward and easy, there is one thing to keep
in mind:

**If we were to rebuild the database container, all our data would be
gone**. It is only stored within the container.

For a **development environment**, data persistency is not that important, and
with our database setup, it is unlikely that we need to recreate the container
regularly or maybe even at all. So I decided to go with the much simpler
approach, and accept the limitations it has. But your mileage my vary.

Whenever you need persistency within a {{ site.links.docker }} container, you
need to add a **[volume](https://docs.docker.com/engine/userguide/dockervolumes/)**
to the container. This is outside the scope of this post, though, so I encourage
you to check out the documention on
**[volumes](https://docs.docker.com/engine/userguide/dockervolumes/)** if you
have the need for data persistency in your container.

## Wrapping up

In this post, we successfully **orchestrated** {{ site.links.docker }} with the
help of {{ site.links.docker-compose }}. Isn't it amazing how easy it was to
deploy a full-blown {{ site.links.postgres }} server?

You can take the same approach to add even more compenents to your app if you
need to. There are a bunch of images on {{ site.links.docker-hub }}, and a lot
of articles around on how to go totally crazy with
{{ site.links.docker-compose }}.

If you have any questions, please share them in the comments below. The same
goes for mistakes you find or if you know a better approach. You can find the
source code for this project on GitHub:
[jdno/docker-sinatra-api@0.4.0](https://github.com/jdno/docker-sinatra-api/tree/0.4.0)

See you in the next post!

_Jan David_
