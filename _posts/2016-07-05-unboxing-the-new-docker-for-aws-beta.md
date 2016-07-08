---
layout: post
title: "Unboxing the new Docker for AWS beta"
date: 2016-07-05 09:15:29 +0200
author: Philipp Garbe
comments: true
published: true
# categories: [Docker, AWS]
keywords: "Docker, Docker for Aws, Beta, AWS"
description: "Unboxing the new Docker for AWS beta"
---
One of the advantages of being a [Docker Captain](https://www.docker.com/community/docker-captains) is the early access to new products. Recently, I got my invitation email for the [Docker for AWS beta](https://beta.docker.com/docs/) and I want to share my experience with you.

## Setup - What do you get?
I was curious - what exactly is Docker for AWS? The invitation email already revealed that it's a CloudFormation stack and it will create a Docker swarm cluster with the latest version (currently docker 1.12.0-rc3).

The setup itself is very easy and can either be done in the AWS Console or on the command line. It's like every other CloudFormation stack and the only thing you've to specify are a few parameters which let you define the number of Manager and Worker instances and their instance types.

```
aws cloudformation create-stack \
  --stack-name teststack \
  --template-url <templateurl> \
  --parameters ParameterKey=KeyName,ParameterValue=<keyname> \
               ParameterKey=InstanceType,ParameterValue=t2.micro \
               ParameterKey=ManagerInstanceType,ParameterValue=t2.micro \
               ParameterKey=ClusterSize,ParameterValue=1 \
  --capabilities CAPABILITY_IAM`
```

The output of the stack returns "DefaultDNSTarget" which gives you the URL to access the running services and  "SSH" containing the command to log into Docker console.

As you can see in the diagram the stack creates a VPC and two Subnets. For Worker and Manager, an independent AutoScaling Group will be created. In addition, the stack also defines two SQS queues and a DynamoDB table. The network communication is secured by a couple of SecurityGroups and two ELBs define the entry-points to the cluster (one for ssh and the other for the Docker services). Docker for AWS requires also some IAM policies.

![Dashboard showing outdated deployments](/assets/dockerforaws_stack.png)
*Overview of the whole Docker for AWS stack*

## Deployment
Thanks to the new [service definition](https://docs.docker.com/engine/swarm/swarm-tutorial/deploy-service/) also the deployment part is easy. First of all, you have to ssh to your Manager node. Just copy the "SSH" output parameter of the CloudFormation stack.

```
ssh docker@docker-swarm-ELB-SSH-1234567890.eu-west-1.elb.amazonaws.com
```

You can get an overview of the nodes and get a similar output when you run `docker node ls`.

```
ID                           HOSTNAME                                      MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
12345xr50f1l0nhvfc1ugufhh *  ip-192-168-34-248.eu-west-1.compute.internal  Accepted    Ready   Active        Leader
12345pdzggn704l9d3cdznpos    ip-192-168-33-6.eu-west-1.compute.internal    Accepted    Ready   Active        
12345i3uh89pm5fgp856pgsoj    ip-192-168-34-191.eu-west-1.compute.internal  Accepted    Ready   Active        
```

Now we want to run nginx which listens on port 80 and we want to have two instances for high availability.

```
$ docker service create --name helloswarm --replicas 2 --publish 80:80 nginx
$ docker service ls
ID            NAME        REPLICAS  IMAGE  COMMAND
1234518h5mk6  helloswarm  2/2       nginx  
```

Now you can try it out, use the "DefaultDNSTarget" output parameter of the CloudFormation stack and you should see the default nginx welcome page in your browser.

What happens in the background is that Docker for AWS listens to the events of the swarm. And if you define a new service with a public port it automatically adopts the changes and configures the ELB for you.


## First impression
The setup was straightforward and after 10 minutes I got a Docker swarm cluster up and running on AWS. By getting the stack definition as JSON file it allows everyone to adjust the stack manually and have control over it.

But how will updates of the stack itself look like? Is an upgrade to a production environment possible without risks?

Scaling the swarm cluster has to be done manually right now by updating the parameters of the CloudFormation stack. Unfortunately the parameter sets the DesiredCapacity of the AutoScaling Group. This prevents users from defining their own Scaling Policies as the DesiredCapacity will be overwritten on every stack update.

Currently, there is a limitation as the [documentation says](https://beta.docker.com/docs/aws/) that changing the numbers of Managers is not possible yet. Also interesting to me is what actually happens to my running containers when the number of Worker nodes gets increased or decreased? Or if I change the instance types? Will Docker for AWS also support multiple instance types in the future?

Many questions yet, but I'll try to find the answers :)
