---
layout: post
title: Learning Docker with Sinatra
identifier: d2e28e01fb749d3253318bc2e22d80cab87ac932aae2fa89d0aed73959f40d45
---

In this series, I want to take a look at **[Docker](www.docker.com)** and how to
use it to deploy a JSON API powered by **[Sinatra](www.sinatrarb.com)**. In the
future, this container will be the foundation for a **microservice** in one of
my projects.

Since the API needs to get its data from somewhere, I will also create a
*database container* and orchestrate both containers with
**[docker-compose](https://docs.docker.com/compose/)**.

The series is split into multiple parts, each of them focusing on a single
step in the process. The series is layed out like this:

1. First, I will create an container for the **Sinatra** app, and verify that it
   works as expected on its own. I want to use this image for multiple projects,
   so I try to set it up as customizable as possible and share it on
   **[Docker Hub](https://hub.docker.com)**.

2. The second step is to start **Sinatra** with
   **[docker-compose](https://docs.docker.com/compose/)**. This is the first
   step on our way to orchestration.

3. To have something to orchestrate, the database will be the topic in the third
   part of this series. I will be looking at **database containers** and how to
   add **persistent storage** to them.

4. In the last part, I will be finalizing this setup by adding the database to
   the orchestration and **linking app and database**. After this, I expect to
   have a nice development environment set up and running.

But please know:

> I am by no means an expert in either **Docker** or **microservices**. Instead,
> I want to become an expert. And I use this blog to document my process and
> more importantly my learnings.

This means that you should take everything in this series with a grain of salt,
and if you know a better way to do things, please tell me. I am always eager to
learn.
