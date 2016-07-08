---
layout: post
title: "How to run HuBot in Docker on AWS EC2 Container Services - Part 1"
date: 2015-03-24 08:28:58 +0100
author: Philipp Garbe
comments: true
published: true
# categories: [Docker, AWS, HuBot]
keywords: "Docker, AWS ECS, EC2 Container Service, HuBot, CircleCI"
description: "The fear of new features, decouple feature releases from code releases, from continous delivery to continuous deployment, FeatureBee"

---
It all began when I was too lazy to check the website of our local [canteen](http://www.leonardi-kg.de/) to find out todays lunch menu. Ok, it's only a couple of clicks and not actual "work" but hey...

I have already played around with [HuBot](https://github.com/github/hubot) in the past and I'm also very excited about [Docker](https://www.docker.com/) so the idea raised to combine these things. And because [we are in the cloud now](http://inside.autoscout24.com/2015/01/04/autoscout24-changes-technology-aws-linux-jvm) it has to be hosted on AWS as well. Meanwhile AWS published the new [EC2 Container Service (ECS)](http://aws.amazon.com/ecs/) in Ireland so I had everything I needed. Let me share my experiences (and some code) with you.


### Big Picture
The idea is to have all relevant sources in one GitHub repository. With each commit a build should be triggered and when the tests are green it should automatically deployed to AWS EC2 Container Service (ECS).

    GitHub -> CircleCI -> Docker Hub -> AWS ECS

Sources are available on my [GitHub repository](https://github.com/pgarbe/tatsu-hubot).

### Run HuBot in Docker container
Installing HuBot is quite easy. You can find a good documentation [here](https://hubot.github.com/docs/). With Docker it's also very easy. First of all we need to define all requirements:

    FROM ubuntu

    RUN apt-get update
    RUN apt-get -y install expect redis-server nodejs npm
    RUN ln -s /usr/bin/nodejs /usr/bin/node

    RUN npm install -g coffee-script
    RUN npm install -g yo generator-hubot


#### Create your own user
Normally you would run "yo hubot" to create a new instance but that leads to the exception  *Error: EACCES, permission denied '/root/.config/configstore/insight-yo.yml'*

The reason is that [yo doesn't run as root user](https://github.com/yeoman/yo/issues/101). So we have to create our own user and switch to it.

    # Create hubot user
    RUN	useradd -d /hubot -m -s /bin/bash -U hubot

    # Log in as hubot user and change directory
    USER	hubot
    WORKDIR /hubot

    # Install hubot
    RUN yo hubot --owner="You" --name="HuBot" --description="HuBot on Docker" --defaults


#### Adding custom scripts
The next step is to load some scripts. HuBot supports three different ways to install custom scripts.

1. __Built-in scripts__: HuBot is shipped with some built-in scripts which are disabled by default. All you need is a [hubot-scripts.json](https://github.com/pgarbe/tatsu-hubot/blob/master/hubot-scripts.json) file where you define all the scripts which should be enabled. This file has to be added to your container

        ADD hubot-scripts.json /hubot/

2. __Community scripts__: Scripts from the community can be installed by npm. In addition these scripts have also be enabled in a [external-scripts.json](https://github.com/pgarbe/tatsu-hubot/blob/master/external-scripts.json) file which needs also be added to the container.

        RUN npm install hubot-standup-alarm --save && npm install
        ADD external-scripts.json /hubot/

3. __Custom scripts:__ That's the easiest and fastest way to add your self written scripts without publishing them to npm. Just add the [(coffee) script file](https://github.com/pgarbe/tatsu-hubot/blob/master/hubot-leitwerk.coffee) to your container.

        ADD hubot-leitwerk.coffee /hubot/scripts/

#### Integrate with Slack
We use [Slack](https://slack.com) as chat tool. Luckily there is an adapter and the installation is also easy. At the end of our [Dockerfile](https://github.com/pgarbe/tatsu-hubot/blob/master/Dockerfile) when we start HuBot we have to define the desired adapter.
    RUN npm install hubot-slack --save && npm install
    CMD bin/hubot -a slack

#### Let it run

To run HuBot first build the container and start it. In addition you have to give the slack adapter a security token. As I don't want to share that token with all of you it's not part of the Dockerfile but I  pass it as environment variable when I start the container instance.

    # Build the container
    docker build -t hubot .

    # And run it
    docker run -e HUBOT_SLACK_TOKEN=xxx -d tatsu

### To be continued
Running HuBot inside a Docker container was just the beginning for me. It's amazing how fast I could start and run it w/out any experience with Docker or nodejs.

The [next blog post](http://garbe.io/blog/2015/05/11/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-2/) will show how I deploy the container to AWS.
