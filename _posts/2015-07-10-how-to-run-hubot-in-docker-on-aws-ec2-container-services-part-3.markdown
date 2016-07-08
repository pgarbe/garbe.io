---
layout: post
title: "How to run HuBot in Docker on AWS EC2 Container Services - Part 3"
date: 2015-07-10 07:17:05 +0200
author: Philipp Garbe
comments: true
published: true
# categories: [Docker, AWS, ECS, HuBot]
keywords: "Docker, AWS ECS, EC2 Container Service, HuBot"
description: "How to store secrets like apikeys, username and passwords of a docker container while using AWS EC2 Container Services ECS"
---

In [part 1](http://garbe.io/blog/2015/03/24/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-1/) I explained how to run HuBot inside a Docker container. [Part 2](http://garbe.io/blog/2015/05/11/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-2/) was about setting up a continuous deployment pipeline. This part shows you how you could handle your secrets like API Keys, usernames and passwords.


## Hide your secrets
HuBot uses environment variables to configure scripts. For example the variable ```HUBOT_SLACK_TOKEN``` is used to store the API Key of the Slack adapter. With ECS you can have the variables inside the docker file or as part of the task-definition file. But storing such sensitive data inside a repository is not a good idea. Even if its a private repository it can be viewed by more people than necessary. ~~Security by obscurity~~ (Edited) The principle of least privilege is also in this case the preferable way.d

An advice you often hear is to keep secrets in a s3 bucket and define a restrictive policy to access it. But how do I get it inside my container? I got another helpful idea from [Michael Wittig](https://michaelwittig.info/). He suggested to store all my secrets in an ```env.sh``` file like this and put it manually on s3.

    export HUBOT_SLACK_TOKEN=xxx
    export HUBOT_AUTH_ADMIN=xxx
    export HUBOT_SLACK_ADMIN=xxx
    export HUBOT_GITHUB_KEY=xxx
    export HUBOT_GOCD_USERNAME=xxx
    export HUBOT_GOCD_PASSWORD=xxx

Inside my dockerfile I should load that file and execute it immediately just before I launch my application.

    CMD ["/bin/sh", "-c", "aws s3 cp --region eu-west-1 s3://your-bucket/env.sh .; . ./env.sh; bin/hubot --adapter slack"]

This command starts a shell and downloads the ```env.sh``` file from my secured s3 bucket. Next it executes that file so that the environment variables are set and finally starts HuBot.

That's it. With this little trick I can keep my secrets in my secured s3 bucket and I don't have to commit them to my repository. You can find all the sources on my [GitHub repo](https://github.com/pgarbe/tatsu-hubot).

In my [next blog post](http://garbe.io/blog/2015/08/24/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-4/) I will change my previous implementation of continuous deployment and use CloudFormation to deploy the whole stack. [Subscribe](http://feeds.feedburner.com/pgarbe) to my blog so you won't miss it.
