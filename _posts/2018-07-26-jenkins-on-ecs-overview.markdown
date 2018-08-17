---
layout: post
title: "Jenkins on ECS: An Overview"
date: 2018-07-26 07:30:29 +0200
author: Philipp Garbe
comments: true
published: true
# categories: [AWS]
keywords: "AWS, ECS"
description: "Run Jenkins on Amazon ECS using Docker containers"
---

With the new pipelines architecture, Jenkins re-invented itself and is still one of the most popular CI/CD tools. Let me show, how you can set up a fully containerized Jenkins running on top of Amazon ECS. This blog post gives you a rough overview of the involved components. In future posts, I'll go into more details and share some learnings.


## Proper Pipeline as Code
There're several good reasons why to pick Jenkins. The most important one for me is that it offers a proper [pipeline as code](https://www.thoughtworks.com/radar/techniques/pipelines-as-code). 

Pipeline as code in general means that the definition of the pipeline is stored in a markup file in a version control system. There are many tools out there to support this. But, a __proper__ pipeline as code additionally has one important property: the pipeline is defined in the same revision as the code itself so that you can change and roll-back your pipeline in the same commit. Jenkins is one of the tools which supports it. 

## ECS Plugin
The second reason is that the [ECS plugin](https://wiki.jenkins.io/display/JENKINS/Amazon+EC2+Container+Service+Plugin) enables you to run as many agents as you need. 

Previously, a typical Jenkins setup contained the master instance and one or multiple agents. An agent runs the whole time and was scheduled to execute multiple build jobs. In that case, there were either too many agents (which costs you some money) or too fewer agents (which forces you to wait for a free agent). Also, these agents often had (over time) a lot of different tools installed, and were not reliable anymore.

With containers, we can avoid these snowflaky agents, as a new agent is started for each run. And, as long as the underlying ECS cluster [scales](https://garbe.io/blog/2017/04/12/a-better-solution-to-ecs-autoscaling/), we can start as many agents as we need.

## Plugin-Hell
When you work with Jenkins be careful about the plugins. It's easy to install a plugin for every use case, but this can be dangerous. Not all Jenkins plugins have the same quality standard. Some of them rely on other plugins which you have to install. 

Your build and deploy scripts will be coupled to these plugins which means, it's not possible to run (and test) them locally. Jenkins (or any other CI/CD tool) should just be the trigger and orchestrator, but the logic should be part of your scripts.

For me, it's similar to the [12-factor apps](https://12factor.net/). The plugins should allow you to define the workflows (like parallel or sequential stages) and the environment (e.g. which AWS credentials). The plugins should not prevent you to run your build and deployment locally.


## An example setup
Let me show how Jenkins on ECS could look like: 

![Jenkins on ECS](/assets/jenkins_ecs.png)

Jenkins runs as ECS Service behind an Application Load Balancer. Optional, a nice domain can be set up through Route53. To make it secure, Jenkins supports multiple ways for authentication and authorization like SAML integration and the [matrix authorization plugin](https://plugins.jenkins.io/matrix-auth). 

The agents run also as containers on the same ECS cluster and communicate back to the master through a Network Load Balancer. As the master runs as container, it's likely the IP and port changes after a restart. Normally, that breaks the communication between agent and master, as the agent still tries to connect to the former IP/Port. With the NLB, we can provide a static address to the agents (needs to be configured) that doesn't change. But, the Jenkins master needs to be registered not only to the ALB but also to the NLB. (more details in a follow-up post; hint: use CloudWatch Events)

As Jenkins stores the configuration of the jobs and their runs on a file system, it needs some storage as well. Unfortunately, neither EBS nor EFS is supported by ECS. There are several ways how to do that by your own and EFS is in that case much easier to handle as it can be mounted to all container instances in the cluster and it works across Availability Zones. 


## What's next?
This was just a rough overview to see what are the good parts of Jenkins and where should you be careful. I also showed an example setup of Jenkins on ECS and what resources are involved. The next blog post will cover how to set up a Jenkins master and let it run on ECS (with some code).
