---
title:  "Rails app with docker for development"
date:   2019-09-16
<!-- tags: elasticsearch -->
excerpt: Lets see one way to use docker while developing one rails application with redis and mysql. Let's try make the development process just like we did before containers, using commands that we know like, rails server, rails console, rake db:migrate and not docker run something.
comments: true
---

Lets see one way to use docker while developing one rails application with **redis** and **mysql**. Let's try make the development process just like we did before containers, using commands that we know like, `rails server`, `rails console`, `rake db:migrate` and not `docker run something`.

## Docker compose

```
version: '3'

services:
  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
    volumes:
      - mysql_data:/var/lib/data
  redis:
    image: redis:5.0.5
    ports:
      - "6379:6379"
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - bundler:/usr/local/bundle/
    ports:
      - "3000:3000"
    links:
      - redis
      - mysql

volumes:
  mysql_data:
  bundler:
```

Our app uses **mysql** and **redis** as dependency, so we are creating two services for each one. For mysql, we created a volume (**- mysql_data:/var/lib/data**), so we won't lose data when the container restarts.

The app is build as **web** and there are two volumes:
- **- .:/app**: This is the volume of our code, we can change it from the host (outside container) and the changes will be known inside the container;
- **- bundler:/usr/local/bundle/**: This is the volume for the gems installed, with that, we wont reinstall all gems when one is added/removed/upgraded.

## Dockerfile.dev

```
FROM ruby:2.6.3

WORKDIR app

RUN gem install bundler:2.0.2

CMD tail -f /dev/null
```

This dockerfile is very simple, because it is just an example app, probably yours has a bit more commands, but there are two important things that I want to mention:

- no bundle install;
- `CMD tail -f /dev/null`

**Why not run bundle install in Dockerfile?**

There are two reasons, one of them I already told (not reinstalling all gems if some of them changes). It is different than using the docker caching mechanism. If we use the cache, for every change (a new gem for example), we would have to reinstall all gems. Using volumes will make we just install the new gem.
The other is to make the image tinier. We will install the gems direct from container. We will se how soon.

**Why run tail -f /dev/null and not rails server?**

I choose this approach because I want to attach my terminal in the container and run the commands by myself, like I always did when I used to install everything on my machine. With that, if we change some config file that force us to restart the server, we just stop and start the server, without killing the container. It is possible because what makes the container stays up is the last command that wont break (`tail -f /dev/null`).

With that config, just run the following commands to setup your environment and start programing:

```
# run compose
docker-compose up -d

# connect to rails app container
docker-compose exec web bash

# now is nothing but what you always did

# run bundle install
bundle install

# setup database
rake db:setup

# run server
rails s -b 0.0.0.0

# open other terminal and test
curl localhost:3000/beers | json_pp

# you can also attach another terminal to container and open the console
docker-compose exec web bash
rails c
```


## Let's see it in practice

![alt]({{site.url}}{{site.baseurl}}/images/docker_on_rails/docker_on_rails.gif)

This example app can be found here: [https://github.com/brunozrk/docker_on_rails_dev](https://github.com/brunozrk/docker_on_rails_dev)
