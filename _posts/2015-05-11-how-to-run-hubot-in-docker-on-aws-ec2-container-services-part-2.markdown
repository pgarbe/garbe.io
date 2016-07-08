---
layout: post
title: "How to run HuBot in Docker on AWS EC2 Container Services - Part 2"
date: 2015-05-11 19:53:16 +0200
author: Philipp Garbe
comments: true
published: true
# categories: [Docker, AWS, ECS, HuBot]
keywords: "Docker, AWS ECS, EC2 Container Service, HuBot, CircleCI"
description: "Continuous delivery and deployment of a docker container to AWS EC2 Container Services ECS"

---
In [part 1](http://garbe.io/blog/2015/03/24/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-1/) I explained how to run HuBot inside a Docker container. Now I'll show you how to set up a continuous deployment pipeline in order to deploy docker images automatically to AWS EC2 Container Services (ECS).


A couple of days after I wrote the previous post ECS became [public available](https://aws.amazon.com/blogs/aws/ec2-container-service-ready-for-production-use/). AWS also introduced support for long-running applications. Even while I don't need the automatic load balancing and scaling to run HuBot the update management of the introduced service scheduler makes deployment very easy.

Before I start, I want to let you know that all the scripts can also be found on my GitHub [repository](https://github.com/pgarbe/tatsu-hubot). I also want to mention [this](
https://blogs.aws.amazon.com/application-management/post/Tx32RHFZHXY6ME1/Set-up-a-build-pipeline-with-Jenkins-and-Amazon-ECS) blog post which was a good template and inspiration for my solution.

Ok, but now let's go!

## Build and Test
I'm used to that every time I make some changes on a repository a build gets triggered and my changes are immediately built and tested. That's what I also expect from a docker based solution. One of the few managed CI services that currently supports generating docker images is [CircleCI](https://circleci.com/).

After signing up all I had to do was to select my GitHub repository that should be watched and CircleCI automatically added a webhook on my [repository](https://circleci.com/gh/pgarbe/tatsu-hubot).

To tell CircleCI what to do with my code it needs a configuration file called `circle.yml`. The setup for my container is pretty simple as you can see here.

{% gist 5f478284fa5de86fe885 %}

You might wonder why I use the build number to tag the docker image. For now it enables me to identify the matching docker image for each build. We'll see later that it's also important for deployment.

The test step is for now just a simple verification if the container can be started. For web applications this might be the right moment to send some http requests to test the application inside the docker container.

At the end the generated container will be pushed to [docker hub](https://registry.hub.docker.com/u/pgarbe/tatsu-hubot/). But also a private docker registry can be used in this step. The rake task at the very end starts the deployment to ECS.

## Continuous Deployment to ECS
In order to run docker containers on AWS ECS you have to setup at least one EC2 machine that runs the ECS agent. The AWS documentation is much better than I could ever explain so please read it [here](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_GetStarted.html) and follow the instructions.

Next is to define a task definition file which can be a set of docker images that needs to run together.

{% gist 1f695b7167451ceefb5f %}

Note that there are no portMappings in the moment. It seems that there is a bug in the ruby sdk. When the portMappings are defined the register method throws an `ArgumentError: unexpected value at params[:container_definitions][0]["portMappings"]` error. Luckily I don't need the mappings right now for HuBot. For obvious reasons I did also not test how to setup ELB and get it working together with the docker container.


The rake script which I use for deployment registers the task definition and creates a service. When the service already exists it updates the service with the new task definition file and the desired count of instances.

In this script I also use the build number. The placeholder in the task definition file will be replaced by the actual number so that ECS takes the corresponding image from the Docker Hub. So it's ensured that when a newer image exists on Docker Hub and ECS creates a new task instance (for example the container crashed) the same image is used and not a newer one.

{% gist ff6f9748758185afcfd1 %}

All you have to do now is to push your changes to GitHub and wait a couple of minutes until the new container is deployed.


## Conclusion
With the support of long-running applications ECS feels now mature enough to have a closer look on it. It still feels not complete (compared to other AWS products). For example not all error messages are clear and you have to understand the terminology of tasks, task definitions, services, container instances, schedulers and so on. It's also difficult to find answers to problems on google because except the offical AWS documentation there is not much to find.

On the other side it makes a lot of fun to see how fast your code changes can go live and how easy it is to create a new docker-based application and deploy it (automatically).

See [part 3](http://garbe.io/blog/2015/07/10/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-3/) how to deal with secrets.
