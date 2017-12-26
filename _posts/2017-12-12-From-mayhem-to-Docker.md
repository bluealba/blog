---
layout: post
title: From Mayhem to Docker
subtitle: Our journey to get a complex environment up and running without effort
headerImage: /images/docker.png
author: 
  name: Claudio Fernandez
  email: claudio.fernandez@bluealba.com
  avatar: /images/claudio.jpg
---

The intent is to show our experience with docker as a tool that enables us to easily provision and setup a local development environment in a constantly changing ecosystem with one main goal in mind: allow developers to focus in developing stories, not in maintaining their local environments.  

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

The reason behind it was our CI server that always deploys the latest commit of the `development` branch into DEV servers. Can we have something like that but for our local environment? Of course, we didn't want Bamboo to eagerly go and deploy into our boxes but at least it should leave something ready for us to pull and start from.  Yes, we could have gone with a simple `npm install` but we wanted something simpler.

## Enter Docker
Our solution was to modify our build server to create docker images for each microservice. A new task was added to the CI build so a brand new image is created every time. Images are automatically tagged with both the actual version and the branch name and they are pushed into our docker repository.

CI builds images from a fairly simple `Dockerfile`. 

```
FROM docker.bluealba.com/centos7-node:6.10.0
LABEL maintainer "claudio.fernandez@bluealba.com"

EXPOSE 8080
VOLUME ["/var/logs/data-aggregation-service"]

COPY package.json /home/svc_spinnaker_dev/package.json
COPY yarn.lock /home/svc_spinnaker_dev/yarn.lock
RUN yarn install --non-interactive --pure-lockfile

RUN yarn add data-aggregation-service@${version} --non-interactive --exact

CMD ["node", "node_modules/data-aggregation-service/app.js"]
```

That enables us to get the latest version of any image and run it:

```
docker pull docker.bluealba.com/data-aggregation-service:develop 
docker run -d -p 8080:8080 -e ELASTIC_HOST=localhost docker.bluealba.com/data-aggregation-service:develop
```

However even if it looks fancier we are still a few steps away of actually solving our initial problem. Running the whole environment require a bunch of manual command lines

```
docker run -d -p 5432:5432 postgres:9.5.4
docker run -d -p 9200:9200 elasticsearch:2.3
docker run -d -p 8080:8080 --net=host docker.bluealba.com/api-gateway:develop
docker run -d -p 8002:8080 -e POSTGRES_URL=localhost:5432 --net=host docker.bluealba.com/master-data-service:develop
docker run -d -p 8002:8080 -e POSTGRES_URL=localhost:5432 --net=host  docker.bluealba.com/user-setttings-service:develop
docker run -d -p 8001:8080 -e ELASTIC_URL=localhost:9200 -e MASTER_URL=localhost:8002/master -e EXTERNAL_URL=localhost:8004/external --net=host docker.bluealba.com/data-aggregation-service:develop
docker run -d -p 8003:8080 docker.bluealba.com/user-tracking-service:develop
docker run -d -p 8004:8080 docker.bluealba.com/external-data-service-mock:develop
```

With a lot of enviroment configuration variables that need to be properly setup! And we're simply doing a walkaround the container networking issue by running everything in the container host (--net=host). This solution is error prone, scales poorly and it doesn't work on mac.

## Docker-compose to the rescue

After dockerinzing all of our applications we went full steam ahead into using `docker-compose` to orchestate the whole thing. 

```
version: '2.1'
services:

  api-gateway:
    image: "docker.bluealba.com/api-gateway:develop"
    ports:
      - "8080:8080"
    volumes:
      - ./config/api-gateway:/var/config/api-gateway

  master-data-service:
    image: "docker.bluealba.com/master-data-service:develop"
    environment:
      - POSTGRES_URL: postgres:5432
    volumes:
      - ./logs:/var/logs/master-data-service

  user-settings-service:
    image: "docker.bluealba.com/user-settings-service:develop"
    environment:
      - POSTGRES_URL: postgres:5432
    volumes:
      - ./logs:/var/logs/user-settings-service      

  data-aggregation-service:
    image: "docker.bluealba.com/data-aggregation-service:develop"
    environment:
      - ELASTIC_URL: elastic:9200
      - MASTER_URL: api-gateway:8080/master
      - EXTERNAL_URL: api-gateway:8080/external
    volumes:
      - ./logs:/var/logs/data-aggregation-service

  user-tracking-service:
    image: "docker.bluealba.com/user-tracking-service:develop"
    volumes:
      - ./logs:/var/logs/user-tracking-service
  
  external-data-service:
    image: "docker.bluealba.com/external-data-service-mock:develop"

  postgres:
    image: "postgres:9.5.4"
    ports:
      - 5432:5432
    volumes:
      - ./data/postgres:/var/lib/postgresql/data

  elastic:
    image: "elasticsearch:2.3"
    ports:
      - 9200:9200
    volumes:
      - ./data/elastic:/usr/share/elasticsearch/data
      - ./config/elastic:/usr/share/elasticsearch/config
```

This is very cool. Docker compose does the networking for us, making every container visible from every other by service name. No need to expose any port to the host container but the one from the our `api-gateway` service. The API gateway is then configured to redirect all the incoming requests to the appropiate service by a simple configuration which may look like this:

```
{
  e"applications": [
    {
      "baseURL": "/data/master",
      "host": "master-data-service",
      "port": 8080
    },
    {
      "baseURL": "/data/aggregated",
      "host": "data-aggregation-service",
      "port": 8080
    },
    {
      "baseURL": "/data/external",
      "host": "external-data-service",
      "port": 8080
    },
    {
      "baseURL": "/user/settings",
      "host": "user-settings-service",
      "port": 8080
    },
    {
      "baseURL": "/user/tracking",
      "host": "user-tracking-service",
      "port": 8080
    }
  ]
}
```





