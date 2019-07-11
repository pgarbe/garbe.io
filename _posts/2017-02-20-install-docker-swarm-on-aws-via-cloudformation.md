---
layout: post
title: "Install Docker Swarm on AWS via CloudFormation"
date: 2017-02-20 06:00:29 +0100
author: Philipp Garbe
comments: true
published: true
# categories: [AWS, Docker, CloudFormation]
keywords: "AWS, Docker, Swarm, Container, CloudFormation"
description: "A sample CloudFormation template to setup Docker 1.13 Swarm mode cluster"
---

Have you ever tried to setup a Docker Swarm cluster on AWS by yourself? In this blog post, I'll provide the necessary [CloudFormation](https://aws.amazon.com/cloudformation/) templates that are needed to setup a Docker Swarm cluster from scratch.

> If you want to avoid setting up Docker Swarm manually, have a look at [Docker for AWS](https://www.docker.com/products/docker#/AWS) which is a native AWS application provided by Docker and easy-to-install.

This templates extends my Docker on AWS templates that I described in my 
[previous blog post]({{ site.baseurl }}{% post_url 2017-01-20-install-docker-1.13-on-aws-via-cloudformation %}). I also assume that the basic infrastructure (VPC, NAT, bastion host) already exists.


## Manager: Swarm Init
In order to create a swarm, we first have to initialize it. After that, we need `join-tokens` in order to add additional manager (and worker) nodes. I created two templates - one for the manager and one for the worker nodes. Because of the dependency to the join-tokens we've to do one manual step the first time when we create the swarm. All other steps will be fully automated, I promise.

> To run these examples, check out my [GitHub repository](https://github.com/pgarbe/containers_on_aws)

First, create the [manager stack](https://github.com/pgarbe/containers_on_aws/blob/master/swarm-mode/manager.yaml) with only one instance. This will be our first manager node.

```bash
$ aws cloudformation create-stack  \
  --template-body file://./swarm-mode/manager.yaml \
  --stack-name swarm-manager \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc \
               ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion \
               ParameterKey=KeyName,ParameterValue=pgarbe \
               ParameterKey=DesiredInstances,ParameterValue=1
```

When the stack has been launched we can ssh via the bastion host into that machine and grep the necessary join-tokens. Please note that there are different tokens for workers and managers.

```bash
# ssh into node via bastion host
$ ssh -A ec2-user@<Public IP of bastion host>

# ssh into manager node 
$ ssh ubuntu@<Private IP of manager node>

# Get the swarm join tokens and copy them
$ docker swarm join-token manager --quiet
$ docker swarm join-token worker --quiet
```

> To make it more secure, I recommend to encrypt the tokens with separate [KMS](https://aws.amazon.com/kms/) keys for manager and worker and allow the respective roles to encrypt that value. To make the sample not too complicated, I will skip that step. 

## Join more Manager Nodes
Once we have the tokens, the manager stack can be updated by providing the manager join-token. It also makes sense to increase the number of desired instances to 3 or 5 to make the swarm cluster highly available.

```bash
# Update stack to create more manager nodes
aws cloudformation update-stack  \
  --template-body file://./swarm-mode/manager.yaml \
  --stack-name swarm-manager \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc \
              ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion \
              ParameterKey=KeyName,ParameterValue=pgarbe \
              ParameterKey=DesiredInstances,ParameterValue=3 \
              ParameterKey=SwarmManagerJoinToken,ParameterValue=<Manager Join Token>
```

The magic happens now inside [cfn-init](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-init.html):

```yaml
  SwarmManagerLaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Metadata:
      AWS::CloudFormation::Init:
        configSets:
          default:
            !If
            - HasSwarmJoinToken
            - !If [HasIAMUserSSHAccess, [ssh-access, docker-ubuntu, swarm-join], [docker-ubuntu, swarm-join]]
            - !If [HasIAMUserSSHAccess, [ssh-access, docker-ubuntu, swarm-init], [docker-ubuntu, swarm-init]]

        ssh-access:
          ...
        docker-ubuntu:
          ...
        swarm-init:
          commands:
            'a_join_swarm':
              command: 'docker swarm init'

            'b_swarm_healthcheck':
              command: 'docker node ls'

        swarm-join:
          commands:
            'a_join_swarm':
              command: !Sub | 

                INSTANCE_ID="`wget -q -O - http://instance-data/latest/meta-data/instance-id`"
                # Get all instances of the swarm manager autoscaling group
                ASG_NAME=$(aws autoscaling describe-auto-scaling-instances --instance-ids $INSTANCE_ID --region eu-west-1 --query AutoScalingInstances[].AutoScalingGroupName --output text)

                # Iterate through the IPs of all manager nodes (some might not be part of the swarm cluster, so we maybe need several tries)
                for ID in $(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $ASG_NAME --region eu-west-1 --query AutoScalingGroups[].Instances[].InstanceId --output text);
                do
                  # Ignore "myself"
                  if [ "$ID" == "$INSTANCE_ID" ] ; then
                      continue;
                  fi

                  IP=$(aws ec2 describe-instances --instance-ids $ID --region eu-west-1 --query Reservations[].Instances[].PrivateIpAddress --output text)
                  if [ ! -z "$IP" ] ; then
                    echo "Try to join swarm with IP $IP"

                    # Join the swarm; if it fails try the next one
                    docker swarm join --token ${SwarmManagerJoinToken} $IP:2377 && break || continue
                  fi
                done

            'b_swarm_healthcheck':
              command: 'docker node ls'
```

When the `SwarmManagerJoinToken` parameter is empty, cfn-init executes swarm-init, otherwise swarm-join. While swarm-init is pretty simple, we need some more logic in swarm-join. We've to find out the IP addresses of existing swarm manager nodes in order to join the swarm cluster. Important is the last command which verifies that our node has successfully joined the swarm cluster. If not, it returns a failure signal back to the autoscaling group which results in a rollback.

> If you skip the last validation, it can happen that AutoScaling Group believes that this EC2 instance has been successfully launched and continuous the rolling update. But actually, it destroys the whole swarm cluster!

## Join Worker Nodes
As for the last step, let's add some worker nodes. The [worker template](https://github.com/pgarbe/containers_on_aws/blob/master/swarm-mode/worker.yaml) looks very similar to the manager template but has some small differences. For example, it needs a different join-token.

```bash
# Add some workers
aws cloudformation create-stack  \
  --template-body file://./swarm-mode/worker.yaml \
  --stack-name swarm-worker \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc \
               ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion \
               ParameterKey=KeyName,ParameterValue=pgarbe \
               ParameterKey=DesiredInstances,ParameterValue=3 \
               ParameterKey=ParentSwarmStack,ParameterValue=swarm-manager \
               ParameterKey=SwarmWorkerJoinToken,ParameterValue=<Worker Join Token>
```

Another important difference is related to the fact that worker nodes can join a Swarm cluster only by sending the join-token to a manager node. To get the private IPs of the manager nodes, the script to join the swarm is a bit different. It imports the value of the managers AutoScaling Group and based on that the corresponding EC2 instances can be determined.

{% raw %}

```yaml
swarm:
  commands:
    'a_join_swarm':
      command: !Sub 
      - | 
        for ID in $(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names ${ASG} --region eu-west-1 --query AutoScalingGroups[].Instances[].InstanceId --output text);
          do
            IP=$(aws ec2 describe-instances --instance-ids $ID --region eu-west-1 --query Reservations[].Instances[].PrivateIpAddress --output text)
            if [ ! -z "$IP" ] ; then
              echo "Try to join swarm with IP $IP"

              # Join the swarm; if it fails try the next one
              docker swarm join --token ${SwarmWorkerJoinToken} $IP:2377 && break || continue
            fi
          done
      - ASG: 
          'Fn::ImportValue': !Sub '${ParentSwarmStack}-SwarmManagerAutoScalingGroup'
    'b_swarm_healthcheck':
      command: '[ -n "$(docker info --format "{{.Swarm.NodeID}}")" ]'
```
{% endraw %}
Finally, another healtcheck test if the worker node has sucessfully joined our swarm cluster.

## Summary
This is one way how to setup a Docker Swarm cluster on AWS. If you're not used to CloudFormation the templates might be a bit scary. But believe me, it's getting better the more you work with CloudFormation. 

To make it more secure and production ready it needs some more steps. What would be interesting for you? Securing this setup with KMS? Auto Scaling of worker nodes? Or monitoring? Let me know in the comments!

