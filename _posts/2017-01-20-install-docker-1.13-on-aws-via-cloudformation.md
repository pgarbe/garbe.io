---
layout: post
title: "Install Docker 1.13 on AWS via CloudFormation"
date: 2017-01-20 07:00:29 +0100
author: Philipp Garbe
comments: true
published: true
# categories: [AWS, Docker, CloudFormation]
keywords: "AWS, Docker, Container, CloudFormation"
description: "A sample CloudFormation template to setup an Ubuntu instance with Docker 1.13 installed"
---

Docker 1.13 has recently been [released](https://blog.docker.com/2017/01/whats-new-in-docker-1-13/) and what I realized is that there are no good CloudFormation templates available. So I created my own CloudFormation template which creates an EC2 instance based on Ubuntu AMI and installs Docker.

> If you just want to play around with Docker (without any installation) have a look at [this awesome project](https://play-with-docker.com) by my fellow [Docker Captain](https://www.docker.com/community/docker-captains) [Marcos Nils](https://twitter.com/marcosnils).

## Basic VPC setup
First, we create a [VPC](https://aws.amazon.com/vpc/) which is a virtual network inside AWS. This helps us to isolate our EC2 instances and give them private IP addresses. Part of the VPC are subnets where each one is bound to an [Availability Zone](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html). Subnets can either be public or private. Public subnets route direct to the Internet and wherever private subnets can't route to the Internet. To give instances in private subnets access to the Internet a NAT gateway which does the network translation needs to be created.

I highly recommend the CloudFormation templates that have been built by [Andreas and Michael Wittig](https://cloudonout.io). These templates are available on  [GitHub](https://github.com/widdix/aws-cf-templates/tree/master/vpc) and you can choose from several of templates. I picked one that supports three different Availability Zones.

```bash
# Create VPC in 3 availability zones
aws cloudformation create-stack \
  --stack-name vpc \
  --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-3azs.yaml
```

### SSH via Bastion Host
Next, I'd like to create a bastion host to reduce the attack surface. The advantage is that the ssh port does not have to be open to the public on all our instances but only on the bastion host. From there you can then jump to all other instances. 

There are two ways how you can ssh into the machines. The first option is to [upload an existing key pair](http://docs.aws.amazon.com/cli/latest/reference/ec2/import-key-pair.html) and reference this name in the `KeyName` parameter while creating the next stacks.

```bash
# Upload your public key
aws ec2 import-key-pair --key-name my-key --public-key-material MIIB...G55tyuMbLD40QEXAMPLE

# Append this parameter when creating the bastion host stack
--parameters ParameterKey=KeyName,ParameterValue=my-key
```

The second option is to [add your public key to your IAM user](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html). If this is done, the `IAMUserSSHAccess` parameter needs to be set to true while creating the next stacks.

```bash
# Append this parameter when creating the bastion host stack
--parameters ParameterKey=IAMUserSSHAccess,ParameterValue=true
```

The bastion host stack needs an addition parameter `ParentVPCStack` to retrieve some output parameters from the VPC stack.

```bash
# Create bastion host for ssh access
aws cloudformation create-stack \
  --stack-name vpc-ssh-bastion \
  --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-ssh-bastion.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc 
  # Append here the parameters for ssh access 
```

If you later want to ssh into your instance, first ssh into the bastion host and forward our key. From the bastion host, you can ssh into the Docker instance that we create later. (The IPs can be found on the [AWS Console](https://console.aws.amazon.com))

```bash
# ssh into bastion host
local> ssh -A ec2-user@<Public IP of bastion host>

# ssh into our Docker instance
bastion> ssh ubuntu@<Private IP of our Docker instance>
```

### NAT gateway
As the last step in our VPC setup, we've to create a NAT gateway in order to route traffic from instances in a private subnet to the internet. 

```bash
# Create NAT gateway
aws cloudformation create-stack \
  --stack-name vpc-nat-instance \
  --template-body https://s3-eu-west-1.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-nat-instance.yaml
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc \
               ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion 
               # Append here the parameters for ssh access 
```

Now we have our basic setup and we can proceed with installing Docker.

## EC2 Setup
The whole stack template can be downloaded from [https://github.com/pgarbe/containers_on_aws/blob/master/ubuntu/stack.yaml](https://github.com/pgarbe/containers_on_aws/blob/master/ubuntu/stack.yaml). It needs the parameters `ParentVPCStack` and `ParentSSHBastionStack`. In addition, the parameter for your chosen ssh access should be provided.

```bash
aws cloudformation create-stack \
  --template-body file://stack.yaml \
  --stack-name docker \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=ParentVPCStack,ParameterValue=vpc \
               ParameterKey=ParentSSHBastionStack,ParameterValue=vpc-ssh-bastion 
               # Append here the parameters for ssh access 
```

There are some additional parameters with default values which can also be overwritten:

|Parameter           |Description                                                                                           |Default    |
|--                  |--                                                                                                    |--         |
|InstanceType        |The instance type for the EC2 instance                                                                | t2.micro   |
|DesiredInstances    |The number of EC2 instances                                                                           | 1          |
|SubnetsReach        |Should the instances have direct access to the Internet or do you prefer private subnets with NAT?    | Public     |
|DockerVersion       |Specifies the version of the Docker engine                                                            | 1.13.0     |
|DockerPreRelease    |Specifies if an experimental version of Docker Engine should be used                                  | false      |


Containers can be started by adding another [Command](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-init.html#aws-resource-init-commands) in the template which runs the `docker run` command.

```yaml
LaunchConfiguration:
  Type: AWS::AutoScaling::LaunchConfiguration
  Metadata:
    AWS::CloudFormation::Init:
      configSets:
        default:
          !If [HasIAMUserSSHAccess, [ssh-access, docker], [docker]]

      docker:
        commands:
          'a_get_certificates':
            command: 'sudo apt-get install apt-transport-https ca-certificates'
          'b_set_gpg_key':
            command: 'sudo apt-key adv --keyserver hkp://ha.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D'
          'c_add_docker_repo':
            command: !If [UsePreRelease, 'echo "deb https://apt.dockerproject.org/repo ubuntu-xenial testing" | sudo tee /etc/apt/sources.list.d/docker.list', 'echo "deb https://apt.dockerproject.org/repo ubuntu-xenial main" | sudo tee /etc/apt/sources.list.d/docker.list']
          'd_update_aptget':
            command: 'sudo apt-get update'
          'e_install_docker':
            command: !Sub 'sudo apt-get install -y docker-engine=${DockerVersion}-0~ubuntu-xenial'
          'f_create_service':
            command: 'sudo service docker start'
          'g_add_ubuntu_user_to_docker_group':
            command: 'sudo usermod -aG docker ubuntu'
          'h_verify_installation':
            command: 'docker run hello-world'
          # Optional
          # 'i_run_your_container':
          #   command: 'docker run -d -p 80:80 --name nginx nginx'
```


## Summary
This example shows how to install Docker on AWS. It is secure, immutable and provides all the configuration as code. 

