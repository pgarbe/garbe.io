---
layout: post
title: "ECS - Migrate from EC2 to Fargate"
date: 2019-10-31 07:30:29 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS, ECS]
keywords: "AWS, ECS, Fargate, ECS Migration, Migrate to Fargate"
description: "ECS - Migrate from EC2 to Fargate"
# cover: /assets/rapid-docker_buch01.jpg
---


Are you tired of maintaining your ECS cluster? Regular updates of the [AMI](https://ami-has-3-syllables.online/), challenges with [auto scaling](https://github.com/aws/containers-roadmap/issues/76), [automated draining](https://github.com/aws/containers-roadmap/issues/256), and so on... 

__It's time to migrate your ECS Services from EC2 to Fargate!__

This is a checklist of things you have to consider and change when you want to migrate from an EC2 based service to Fargate. It assumes that you're deploying your ECS services with CloudFormation. 

## Before the migration

You can make your ECS service compatible with Fargate without actual migrating it. This reduces the risk as the whole migration can be split into smaller steps.

#### 1) Have the CPU settings defined on the container and task level
Fargate requires that CPU and memory are defined on a task level. The values depend on each other, so check [this page](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#task_size) to see what combinations are supported.

If you're unsure how much CPU or memory you need, start with a bigger size. Watch your application in CloudWatch and adjust the size later on accordingly.

> Be aware that CPU setting on task-level is a hard limit and does not allow any bursting (even on EC2). A deep dive into that topic can be found on the [AWS Containers blog](https://aws.amazon.com/blogs/containers/how-amazon-ecs-manages-cpu-and-memory-resources/)

```yaml
TaskDefault:
  Type: 'AWS::ECS::TaskDefinition'
  Properties:
    Family: foobar
    Cpu: 1024
    Memory: 2048
    ContainerDefinitions:
      - Name: myApp
        Cpu: 1024
        Memory: 2048
```

#### 2) Change network to awsvpc
Fargate works only with awsvpc network. If you don't use it already, you've to change the NetworkMode to awsvpc in your task definition. 

```yaml
TaskDefault:
  Type: 'AWS::ECS::TaskDefinition'
  Properties:
    ... 
    ExecutionRoleArn: !GetAtt FargateExecutionRole.Arn
    NetworkMode: awsvpc
```
Your service needs to know which subnets can be used and if the service is public or not. One benefit of the `awsvpc` network is that it has it's own SecurityGroup and doesn't share it with the whole cluster. It will be created in the next step.

```yaml
Service:
  Type: 'AWS::ECS::Service'
  Properties:
    ...
    NetworkConfiguration:
      AwsvpcConfiguration:
        AssignPublicIp: DISABLED
        SecurityGroups:
          - !Ref EcsServiceSecurityGroup
        Subnets:
          - !ImportValue 'vpc-PrivateSubnetA'
          - !ImportValue 'vpc-PrivateSubnetB'
          - !ImportValue 'vpc-PrivateSubnetC'
```

#### 3) Allow traffic from your load balancer
Let's assume there's an Application LoadBalancer in front of our ECS Service. The SecurityGroup needs to allow ingres from the Application Load Balancer on the container port. 

And that's a bit different as before because with the default "bridge" network the SecurityGroup of the cluster instances had to allow the whole dynamic port range (usually 32768–61000).

```yaml
EcsServiceSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Allows ingress traffic from ALB
    VpcId: !ImportValue 'vpc-id'
    SecurityGroupIngress:
      - FromPort: !Ref ContainerPort
        ToPort: !Ref ContainerPort
        IpProtocol: tcp
        SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
```

#### 4) Change the load balancer's target type
The default target type of a TargetGroup is "instance". With awsvpc network, this has to be changed to `ip`. Verify that the port of the TargetGroup is now the same as your container port. 

```yaml
  ALBTargetGroup:
    Type: 'AWS::ElasticLoadBalancingV2::TargetGroup'
    Properties:
      Port: !Ref ContainerPort
      TargetType: ip
```

#### 5) Logging and Metrics
With the awsvpc network, your tasks are not able to talk to any other container or process on the same host. And if your service depends on any third party software to send logs or metrics (like DataDog or SumoLogic), you've to add these agents as so-called [sidecars](https://dzone.com/articles/sidecar-design-pattern-in-your-microservices-ecosy-1) to your task definition. 

```yaml
TaskDefault:
  Type: 'AWS::ECS::TaskDefinition'
  Properties:
    Family: foobar
    Cpu: 1024
    Memory: 2048
    ContainerDefinitions:
      - Name: myApp
        Cpu: 896
        Memory: 1792
        DependsOn:
          - Condition: HEALTHY
            ContainerName: mySidecar
      - Name: mySidecar
        Essential: true
        Cpu: 128
        Memory: 256
        PortMappings:
          - ContainerPort: 1234
            Protocol: udp
```

#### 6) Enforce compatibility
To verify that our changes are compatible with Fargate we can enforce it:

```yaml
  TaskDefault:
    Type: 'AWS::ECS::TaskDefinition'
    Properties:
      ... 
      RequiresCompatibilities:
        - EC2
        - FARGATE
```

#### 7) Clean up of unnecessary service role

You might run into this error: 
> "You cannot define an IAM role for services that require a service-linked role"

In that case, you still have an IAM Role attached to your ECS service. Just remove it as it isn't needed anymore. ECS uses a service-linked role that normally already exists in your account. It's called "AWSServiceRoleForECS".

If the role doesn't exist check [this page](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using-service-linked-roles.html) with further explanations.


#### 8) Deploy
Now it's time to deploy your changes. Your service still runs on EC2 and we just improved CPU/Memory settings and changed the network.


## Migration

Now it's time to migrate to Fargate. With all the preparations that we did the actual migration is quite easy. Just add the Fargate LaunchType and PlatformVersion to your ECS Service. 

If you have any Placement Strategies remove them as they're not needed in Fargate.

> During deployment CloudFormation creates a new service, waits until all desired tasks are running and delete the old service afterward. Ensure that your application can run in parallel for a few minutes.

```yaml
  Service:
    Type: 'AWS::ECS::Service'
    Properties:
      Cluster: 'default'
      LaunchType: FARGATE
      PlatformVersion: 1.3.0
```


## Conclusion
The actual migration to Fargate is pretty easy. More effort is needed to make your ECS service compatible with Fargate. I hope this guide helps you with your migration. Let me know if anything is missing.
