---
layout: post
title: "Rapid Docker on AWS"
date: 2019-10-07 20:30:29 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS, ECS]
keywords: "AWS, ECS, Fargate"
description: "Rapid Docker on AWS"
cover: /assets/rapid-docker_buch01.jpg
---

What's the best way to run containers on AWS? "Rapid Docker on AWS", the [latest book](https://cloudonaut.io/rapid-docker-on-aws/#buy) from [Andreas and Michael Wittig](https://widdix.net) gives an opinionated answer. I had the honour to write the foreword and now I want to share it with you:


### Foreword

When Solomon Hykes presented Docker for the first time in 2013, its success could not yet be foreseen. However, developers immediately loved Docker’s simplicity, and its success was unstoppable. Today, nearly every developer works with containers in some way or another.

It took some time until it was possible to deploy Docker containers efficiently in production. Things changed when the first orchestration tools became available. Docker built Swarm, Google open sourced Kubernetes, and AWS released the Elastic Container Service (ECS).

Nowadays, there are many options for deploying container workloads. This book does not blindly follow any technology hype, but gives you an opinionated blueprint for running container-based applications on AWS in a highly available, cost-effective and scalable manner with as little operational effort as possible. It covers not only Fargate but also other AWS services like Application Load Balancer (ALB), Relational Database Service (RDS), and CloudWatch to demonstrate what is needed for production readiness.

What I personally like very much is that the whole infrastructure is defined as code and comes with the book. It’s also great to see that version control and continuous deployment are covered in their own chapter.

The two authors of the book, Andreas and Michael, have been working with AWS for many years and have gained a lot of experience. They are active in the community, sharing many of their experiences for free on their blog as well as in their production-ready CloudFormation templates. I had the opportunity to work with them for several months and much of what I know about AWS today I owe to them!

Now, get your hands dirty and deploy your containers rapidly on AWS!



[![CDK Constuct Library](/assets/rapid-docker_buch02.jpg)](https://cloudonaut.io/rapid-docker-on-aws/#buy)

Get your copy at [cloudonaut.io: Rapid Docker on AWS](https://cloudonaut.io/rapid-docker-on-aws/#buy)
