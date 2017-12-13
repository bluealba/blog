---
layout: post
title: From Mayhem to Docker
subtitle: Our journey to get a complex environment up and running without effort
headerImage: images/docker.png
author: 
  name: Claudio Fernandez
  email: claudio.fernandez@bluealba.com
  avatar: images/claudio.jpg
---

The idea of this post is to show our experience in the use of docker as a tool that enables us to easily provision and setup a local development environment in a constantly changing ecosystem with one main goal in mind: allow developers to focus in developing stories, not in maintaining their local environments.  

## Overview
It all began with a fairly simple app. We have our frontend that consumed a single backing service that interacts with a fairly regular database. Nothing too fancy. Setting up a local environment was a little bit cumbersome but fairly simple:

- Install the database.
- Seed the database with some SQL scripts.
- Clone the backend project.
- Clone the fronten project.
- Start everything up.

As the application growth we ended up not with a single backend but with a pletora of microservices. We gain complexity in a way that we can no longer describe our ecosystem without at least a fairly simple hand draw, but it payed off drastically: Each microsevice has its own set of responsibilities, it's tied up to their own testing and release cycle (developing in some of them is very active, others are fairly stable) and expose a clear REST API that helps us to encapsulate the complexity of its internals. 

All our microservices are fronted with a reverse proxy that abstracts clients from knowning exactly which will serve a request. For the UI there's still only one backend exposing a big REST API. Routing, among other things like authentication and authorization, is handled by this [API gateway](http://microservices.io/patterns/apigateway.html). 

![Our environment](https://www.lucidchart.com/publicSegments/view/be7b8a35-a178-4f53-af36-ebea74276431/image.png)

The actual implementation of the API Gateway is an inhouse product but you can think of it like a regular [nginx](https://www.nginx.com/) or [traefik](https://traefik.io/).

## The local environment hell
Here is when everything got really messy. For a developer starting up a local environment to test her changes was kind of a nightmare. It's worth to mention that development usually involves changing a single microservice, but testing it an more or less real enviroment require starting the whole thing up! Of course, we could try to mock-everything-up but we decided against it mainly because:

- mocking everything and keeping mocks in sync with real stuff requires an extra effort.
- we wanted to keep the whole thing as real as we can get. 

We did mock _some_ things, but only those services that were sitting in the boundaries between our application and external ones. The rest we wanted to keep it real. 

So at this point setting up a single environment required installing two different persistent storages (elastic search and postgres), seeding them with initial data, cloning several git repos, setting up a bunch of enviroment variables for each microservice and starting all of them. 

Just to test a one-line change somewhere. 

Not good.

## The VM solution
Of course we saw this issue and we took action. Our first approach was to distribute a prebuilt vm image with all the microservices installed there. We liked this approach at first because the contents of the image itself were pretty similar to what the real environment in production looked like. 

![Our big VM](https://www.lucidchart.com/publicSegments/view/eed5002b-dd7c-468b-bef8-15bb57c02cb0/image.png)

However we quickly run in some caveats:

- the VM image was HEAVY since it carries the whole operating system.
- the VM image was preinstalled with a baseline of microservices. Such baseline quickly run out of date as newer versions were released. The act of creating a newer versions of it required a series of manual, error-prone, steps. And downloading such newer versions was painful too! Even if the actual diff from the previous vm to the newer one was a couple of bytes we ended up downloading several gigas just for it!

There was also an issue with landing with a simple way to actually develop locally. While some members of the team choose to develop inside the vm others used different approaches. We try not to impose a way to do anything, but not reaching a consensus in something like this crimple the work sinergy between our team members (I cannot help you with that problem because I don't run the app that way)

## CI / CD - the light at the end of the tunnel.
While we were struggling to keep our local environments up to date and running something completely different happened on our development environment. We always had the latest development version deployed in DEV and we didn't even have to worry about it. 

The reason behind it was our CI server that always deployed the latest commit on `development` branch into DEV. Why can't we have something like that locally? Of course, we didn't want Bamboo to eagerly deploy into our local boxes but at least it should leave something there ready for us to 

