---
title: Nuxt Django app from development to AWS - Part 2
description: Deploy a DJango Nuxt application to AWS using RDS, S3 and ECS services. Part 2, deploy to the cloud using AWS ECS
image: "aws.png"
writtenBy: "Florian Bigot"
---

In this tutorial, we will deploy a production ready app to the cloud using AWS ECS

This is the part 2 of the tutorial where we deploy the app to AWS using ECS service.

[Part 1 here](https://blog.florianbgt.com/Nuxt_Django_from_dev_to_AWS_Part1)

You can find the source code of this article on [my github](https://github.com/florianbgt/Nuxt-Django-ECS)

## 1) Prepare our app for ECS

On ECS, we are going to deploy out `app` and `api` services on 2 different environement so they can scale up and down independently.

Because of this, we need to make some ajustement to our `nuxt.config.js` file.

```javascript
### app/nuxt.config.js
privateRuntimeConfig: {
    axios: {
        baseURL: Boolean(parseInt(process.env.USE_ECS)) ? publicRuntimeConfig.axios.browserBaseURL : 'http://api:8000/api',
    },
},
```

We also need to modify our `.env` file to include this `USE_ECS` parameter.

```txt
### .env
USE_RDS=1
RDS_NAME=prod_db
RDS_USER=postgres
RDS_PASSWORD=<your password>
RDS_HOST=<db endpoint>
RDS_PORT=5432

USE_S3=1
AWS_ACCESS_ID=<your access id>
AWS_ACCESS_SECRET=<your access secret>
AWS_S3_BUCKET_NAME=awesome-recipe-bucket-1
AWS_S3_BUCKET_REGION=us-east-2

USE_ECS=1

DOMAIN=app.localhost
DEBUG=1
```

## 2) Push our images to Dockerhub

To use AWS ECS< our images must be accessible over the internet. The easiest way to do this is to use dockerhub!

We just need to create 2 repository, one for our `app` and another for ours `api`.

Then, we will modify our `Dockerfile.prod` for both `app` and `api`.

```yaml
### app/Dockerfile.prod
FROM node:16.6
WORKDIR /code
COPY package*.json /code/
RUN npm install
ENV NUXT_HOST 0.0.0.0
ENV NUXT_PORT 80
COPY . .
RUN npm run build
EXPOSE 80
CMD ["npm", "run", "start"]
```

```yaml
### api/Dockerifle.prod
FROM python:3.8
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED=1
WORKDIR /code
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 80
CMD ["gunicorn", "--bind=0.0.0.0:80", "--workers=2", "--threads=4", "--reload", "_project.wsgi"]
```

Once done, we can build our images and push them to docker hub using the following commands.

```bash
docker build api -f api/Dockerfile.prod -t florianbgt/nuxt-django-aws-tutorial-api
docker push florianbgt/nuxt-django-aws-tutorial-api
docker build api -f api/Dockerfile.prod -t florianbgt/nuxt-django-aws-tutorial-app
docker push florianbgt/nuxt-django-aws-tutorial-app
```

And that is it, our images are now accessible from Dockerhub!

## 3) Create our ECS cluster

In this section, we will create a ECS cluster.

For this, on the AWS console, go to: `services > Containers > Elastic Container Service > Cluster > Create Cluster`.

Then, follow the steps as below.

<div><blog-img src="cluster_1.png" alt="Create ECS cluster 1/2" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="cluster_2.png" alt="Create ECS cluster 1/2" width="100%" height="auto" class="shadow mb-3"/></div>

The creation of the cluster will take a few minutes.

## 4) Create task definition

In this section, we will create 2 task definitions (one for `app` and another for `api`).

To create a task definition, go to: `Services > Containers > Elastic Container Service > Task Definitions > Create a new Task Definition`.

Then, use the same parameters as below to create the task definition for our `app`.

<div><blog-img src="taskDefinition_1.png" alt="Create ECS task definition 1/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_2.png" alt="Create ECS task definition 2/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_3.png" alt="Create ECS task definition 3/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_3_1.png" alt="Create ECS task definition 3_1/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_3_2.png" alt="Create ECS task definition 3_1/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_3_3.png" alt="Create ECS task definition 3_1/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_3_4.png" alt="Create ECS task definition 3_1/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_3_5.png" alt="Create ECS task definition 3_1/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_4.png" alt="Create ECS task definition 4/7" width="100%" height="auto" class="shadow mb-3"/></div>

Then, we create the task definition for our `api` service. Use the same parameter as above except for the one below.

<div><blog-img src="taskDefinition_5.png" alt="Create ECS task definition 5/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_6.png" alt="Create ECS task definition 6/7" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="taskDefinition_7.png" alt="Create ECS task definition 7/7" width="100%" height="auto" class="shadow mb-3"/></div>


## 5) Setup a SSL certificate

Our app will be serve from 2 different url. `domain.com` for our `app` and `api.domain.com` for our `api`.

I am going to create a 2 subdomain of my personal domain to do this.

Our `app` will be serve from `awesomerecipes.florianbgt.com` and our `api` from `api.awesomerecipes.florianbgt.com`.

The first thing we need to do is to get a SSL certificate for these 2 domains. We are going to use AWS certificate manager.

Sign into the AWS console and go to `services > Security, Identidy, & Compliance > Certificate Manager`.

Then click on `Get started` under `Provision certificates`.

Select `Request a public certificate` and click on `Request a certificate`.

Then use the following settings (**replace my domain by yours**).

<div><blog-img src="SSL_1.png" alt="Generate SSL certificate 1/6" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="SSL_2.png" alt="Generate SSL certificate 2/6" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="SSL_3.png" alt="Generate SSL certificate 3/6" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="SSL_4.png" alt="Generate SSL certificate 4/6" width="100%" height="auto" class="shadow mb-3"/></div>

Both of the domain should then be shown `Pending validation`. You then can edit your DNS reccord according to what is ask (see below).

<div><blog-img src="SSL_5.png" alt="Generate SSL certificate 5/6" width="100%" height="auto" class="shadow mb-3"/></div>

Once done, and sfter a bit, you should see the certificates issued as below.

<div><blog-img src="SSL_6.png" alt="Generate SSL certificate 6/6" width="100%" height="auto" class="shadow mb-3"/></div>

Congratulation, we now have a SSL certificate for our domain!

## 6) Setup our target groups for our load balencers

In the next section we will setup 2 load balencers. But before, we need to setup target groups.

From the AWS console go to `services > Compute >  EC2 > Load Balancing > Target Groups > Create target group`.

Then, use the following settings to create the target group for our `app`.

<div><blog-img src="targetGroup_1.png" alt="Create target group 1/5" width="100%" height="auto" class="shadow mb-3"/></div>

Above, it is really important to select the same VPC our cluster is in.

<div><blog-img src="targetGroup_2.png" alt="Create target group 2/5" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="targetGroup_3.png" alt="Create target group 3/5" width="100%" height="auto" class="shadow mb-3"/></div>

For our `api`, repeat the same opperation as above except for below.

<div><blog-img src="targetGroup_4.png" alt="Create target group 4/5" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="targetGroup_5.png" alt="Create target group 5/5" width="100%" height="auto" class="shadow mb-3"/></div>


## 7) Setup our balencers

To setup our load balencers, go to `services > Compute >  EC2 > Load Balancing > Load Balancers > Create Load Balencer > Application Load Balancer`.

Then, use the following settings to create the target group for our `app`.

<div><blog-img src="loadBalancer_1.png" alt="Create load balancer 1/6" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="loadBalancer_2.png" alt="Create load balancer 2/6" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="loadBalancer_3.png" alt="Create load balancer 3/6" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="loadBalancer_4.png" alt="Create load balancer 4/6" width="100%" height="auto" class="shadow mb-3"/></div>

For our `api` repead the above operation but the following.

<div><blog-img src="loadBalancer_5.png" alt="Create load balancer 5/6" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="loadBalancer_6.png" alt="Create load balancer 6/6" width="100%" height="auto" class="shadow mb-3"/></div>

## 8) Create ou services

We now can come back to our cluster and create our 2 services.

For this go to `services > Containers >  Elastic Container Service > Clusters > awesome-recipe-cluster > Services > Create`.

<div><blog-img src="service_1.png" alt="Create service 1/11" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="service_2.png" alt="Create service 2/11" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="service_3.png" alt="Create service 3/11" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="service_4.png" alt="Create service 4/11" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="service_5.png" alt="Create service 5/11" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="service_6.png" alt="Create service 6/11" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="service_7.png" alt="Create service 7/11" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="service_8.png" alt="Create service 8/11" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="service_9.png" alt="Create service 9/11" width="100%" height="auto" class="shadow mb-3"/></div>

Once the service created, go back to the load balencer `awesome-recipe-lb-app` and add the security group created during our service creation

<div><blog-img src="lb_add_security_group.png" alt="Add security group to load balancer" width="100%" height="auto" class="shadow mb-3"/></div>

Four our `api`, repeat the same above steps except the following.

<div><blog-img src="service_10.png" alt="Create service 10/11" width="100%" height="auto" class="shadow mb-3"/></div>
<div><blog-img src="service_11.png" alt="Create service 11/11" width="100%" height="auto" class="shadow mb-3"/></div>

Here as well, once our service created, we need to add the security group created to our `awesome-recipe-lb-api` load balancer.

<div><blog-img src="lb_add_security_group_2.png" alt="Add security group to load balancer" width="100%" height="auto" class="shadow mb-3"/></div>

## 8) Give access to the database

Our `api` is for now not able to access our database.

Let's fix this!

For this, we form the AWS console, we go to `Services > Database > RDS > Databases`.

Then, click on your database and identify security group associated with the database.

After that, go to `Services > Compute > EC2 > Security Groups`.

Finally click on the security group associated with the database and edit the inbound rules as below.

<div><blog-img src="RDS_inbound_rules.png" alt="Edit RDS inbound rules" width="100%" height="auto" class="shadow mb-3"/></div>

## Conclusion

Our application has now been successfuly deployed on AWS!

This setup allow us to scale the application up and down very easily (or even automatically). Thus, it is perfect for accessibility.

You can find the source code of this article on [my github](https://github.com/florianbgt/Nuxt-Django-ECS)

If you have any question or just want to chat, feel free to email me florian.bigot321@gmail.com
