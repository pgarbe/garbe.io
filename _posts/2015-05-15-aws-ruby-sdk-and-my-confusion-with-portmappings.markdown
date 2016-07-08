---
layout: post
title: "AWS Ruby SDK and my confusion with portMappings"
date: 2015-05-15 12:25:25 +0200
author: Philipp Garbe
comments: true
published: true
# categories: [AWS, ECS]
keywords: "AWS ECS, EC2 Container Service, Ruby, SDK"
description: "AWS Ruby SDK snake_case portMappings"

---
When you use the [AWS Ruby SDK](http://aws.amazon.com/sdk-for-ruby/) to register your ECS task definition files you maybe struggle with the same error I struggled around.

    ArgumentError: unexpected value at params[:container_definitions][0]["portMappings"]

The same error message was also returned when I tried to define mountPoints.


The short answer is that the AWS Ruby SDK use also Rubyish snake_case instead of camelBack JSON-style. After changing "portMappings" into "port_mappings" everything worked (same applies to containerPort and hostPort).


If I had started with Ruby directly I would maybe never run into that situation because it's well [documented](http://docs.aws.amazon.com/sdkforruby/api/Aws/ECS/Client.html#register_task_definition-instance_method). But I started with the AWS Command Line Interface (CLI) to play around with ECS and I also had my task definition in a separate file.

    aws ecs register-task-definition --cli-input-json file://<path_to_json_file>/hubot_task_definiton.json

Based on that I [started](http://garbe.io/blog/2015/05/11/how-to-run-hubot-in-docker-on-aws-ec2-container-services-part-2/) with the automation and reused my existing task definiton file.

    ecs.register_task_definition(family: "HuBot", container_definitions: JSON.parse(taskDefintion))

After that confusion I can now also define port mappings as well as mount points. The later I need for my idea to have secret keys as environment variables inside a docker container instead of defining them in the task definition file. Of course, if it works I'll post my solution.
