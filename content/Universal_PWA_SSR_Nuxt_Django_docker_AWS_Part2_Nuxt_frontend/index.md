---
title: Production ready Universal PWA with SSR using Nuxt, Django, Docker and AWS (Part 2)
description: Create an universal PWA with SSR and deploy it on AWS usign Nuxt, Django and Docker. Part 2, Nuxt frontend
writtenBy: "Florian Bigot"
---

In this article, we will setup a production ready universal PWA using Nuxt, Django, Docker and AWS.

This is the part 2 of the tutorial where we will setup the universal PWA frontend using Nuxt

Part 1 here: https://blog.florianbgt.com/http://localhost:3000/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part1_Django_API

Part 3 here: https://blog.florianbgt.com/http://localhost:3000/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part3_AWS_Deployment

You can find the source code on my github: https://github.com/florianbgt/Universal_PWA_SSR_Nuxt_Django_Docker_AWS

## 1) Setting up our project

To setup our project, use the following command:

```bash
npm create nuxt-app pwa
```

<div><blog-img src="nuxt_create.PNG" alt="Create nuxt app" width="100%" height="auto" class="shadow mb-3"/></div>

You then can delete the node_modules directory. We will not need it as we are going to dockerize the app.

```bash
rm -r pwa/node_modules
```

we now can dockerize our app.

We start by creating the following file:

```bash
touch pwa/Dockerfile
```

```yaml
### pwa/Dockerfile
FROM node:16.6
WORKDIR /code
COPY package*.json /code/
RUN npm install
COPY . /code/
```

```yaml
### docker-compose.yml
version: "3.9"

services:
  db:
    image: postgres
    volumes:
      - ./db:/var/lib/postgresql/data
    env_file:
      - .env
  api:
    restart: always
    build:
      context: api
      dockerfile: Dockerfile
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./api:/code
      - ./media:/media
    ports:
      - "8000:8000"
    depends_on:
      - db
  pwa:
    restart: always
    build:
      context: pwa
      dockerfile: Dockerfile
    command: npm run dev
    volumes:
      - ./pwa:/code
      - /code/node_modules
      - /code/.nuxt
    ports:
      - "3000:3000"
    environment:
      HOST: 0.0.0.0
```

We can now spin up our containers using `docker-compose`:

```
docker-compose up
```

If we go to http://localhost:3000, we now see our nuxt app up and running

<div><blog-img src="nuxt_landing.PNG" alt="Nuxt landing page" width="100%" height="auto" class="shadow mb-3"/></div>

## 2) Create layouts

## 3) Authentication

## 4) Receipes page

## Conclusion
