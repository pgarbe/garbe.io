---
layout: post
title: "How to run HuBot in Docker on AWS EC2 Container Services - Part 4"
date: 2015-08-24 14:04:25 +0200
author: Philipp Garbe
comments: true
published: true
# categories: [Docker, AWS, ECS, HuBot]
keywords: "Docker, AWS ECS, EC2 Container Service, AutoStacker, AutoStacke24, CloudFormation, HuBot"
description: "How to deploy docker containers to AWS EC2 Container Services ECS with CloudFormation and AutoStacker24"
---


You might say that I showed already in the first three parts (see [here](http://garbe.io/blog/2015/03/24/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-1/), [here](http://garbe.io/blog/2015/05/11/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-2/) and [here](http://garbe.io/blog/2015/07/10/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-3/)) how to run HuBot on AWS EC2 Container Services (ECS). That's true but since AWS CloudFormation also supports ECS I wanted to deploy HuBot as a CloudFormation stack.


If you feel confused with all the names (container, cluster, service and so on) have a look to the [ECS component description](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html). I tried to stick on these terms.

## Setting up ECS with CloudFormation
AWS provides a nice [CloudFormation template](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/quickref-ecs.html) which gives you an example how you can create an EC2 Cluster and an ECS service.
That's a good start but what I want is to have two stacks:

**1. Stack "Cluster":** Creates the ECS cluster and EC2 cluster instance. AutoScaling of the cluster is also defined here. ([source code](https://github.com/pgarbe/tatsu-hubot/blob/master/ecs-cluster-stack.json))

**2. Stack "Service":** Registers task definition file and service. Creates ELB for the container. ([source code](https://github.com/pgarbe/tatsu-hubot/blob/master/tatsu-hubot-stack.json))

Separating the stacks helps me to get the benefits of ECS. One stack is responsible for the ECS cluster and how to scale it. It gives me a cluster where I can start my container instances.

The service stack on the other side only takes care about the docker-based ECS service. I can have multiple of these stacks (with different applications) and run the instances on the ECS cluster of the first stack.

![Two CloudFormation stacks for ECS cluster and ECS service](/assets/hubot_ecs.png)

#### Stacking of stacks
When you're familiar with CloudFormation you know that names of resources gets a generated name by CloudFormation. Also the name of my ECS Cluster. In my Cluster-Stack we have the ECSCluster name as output parameter and I need this value as input parameter of our Service-Stack.

Luckily my colleagues [Christian Rodemeyer](http://atombrenner.blogspot.de/) and Johannes Müller wrote already a ruby module to manage AWS CloudFormation stacks a little bit better than the AWS SDK and published it [here](https://github.com/autoscout24/autostacker24).

One feature of AutoStacker24 is to read output parameters of an existing stack and merge the values to the given parameters of the current stack file. Therefore the new stack can easily reference resources (like ECSCluster name) from a parent stack.

![Stacking stacks with AutoStacker24](/assets/hubot_autostacker24.png)

#### Security drawbacks
Unfortunately ECS does not support IAM policies for containers. All policies have to be defined for ECS container instances and so all services (in the same ECS cluster) have the same rights.

In my example I wanted to setup one ECS cluster for multiple different applications (like HuBot) so that I don't waste too many resources and to make it easy to deploy a new application.

But from a security point of view this is dangerous. It would be better if the policies could be defined for each container. I hope that the ECS team will build this feature in the near future. In the meantime I'd suggest for security critical applications to setup an own ECS cluster for each service.


## Summary
This is the last post of this little series. You can find the whole source code in my GitHub [repository](https://github.com/pgarbe/tatsu-hubot).

I love feedback so feel to contact me on [twitter](https://twitter.com/pgarbe) or leave some comments here.
