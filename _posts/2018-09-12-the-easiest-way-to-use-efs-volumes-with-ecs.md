---
layout: post
title: "The easiest way to use EFS volumes with ECS"
date: 2018-09-12 09:30:29 +0200
author: Philipp Garbe
comments: true
published: true
# categories: [AWS]
keywords: "AWS, ECS, Volume, EFS"
description: "The easiest way to use EFS volumes with ECS"
---


Finally, AWS [released the ECS support for Docker volume drivers and volume plugins](https://aws.amazon.com/about-aws/whats-new/2018/08/amazon-ecs-now-supports-docker-volume-and-volume-plugins/). You can now use volume plugins like [Rexray](https://github.com/rexray/rexray) or [Portworx](https://docs.portworx.com/cloud/aws/ecs.html) to automatically mount EBS or EFS volumes. But if you just want to use an EFS volume, there's an easier way.

### Mount EFS volumes by using Docker's local volume driver

First, create the EFS volume and open the settings. It shows you a link called "Amazon EC2 mount instructions", which gives you all the information you need to mount the volume on an EC2 instance.

![EFS mount instructions](/assets/efs-mount-instructions.png)

In the TaskDefinition, you can now add the new [dockerVolumeConfiguration](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_DockerVolumeConfiguration.html) and use Docker's built-in local driver. This volume driver accepts options similar to the Linux mount command.

You've to add "type", "device" and "o" as driver options. The value of _type_ is just "nfs". The _device_ option gets the EFS mount point. And the _o_ option can be copied from the EFS instruction but you've to add "addr" with the DNS name of the EFS volume as well. 

Here is how it should look at the end: 

```json
{
  "volumes": [
    {
      "name": "jenkins_home",
      "host": null,
      "dockerVolumeConfiguration": {
        "autoprovision": null,
        "labels": null,
        "scope": "task",
        "driver": "local",
        "driverOpts": {
          "type": "nfs",
          "device": "fs-1234abcd.efs.eu-west-1.amazonaws.com:/",
          "o": "addr=fs-1234abcd.efs.eu-west-1.amazonaws.com,nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2"
        }
      }
    }
  ]
}
```

__Update:__ It's also supported in CloudFormation now. In this example, the ${FileSystem} points to an [EFS FileSystem](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-efs-filesystem.html). 

> Please note, that CloudFormation force you to define a `Label`, even while the docs says it's [not required](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ecs-taskdefinition-dockervolumeconfiguration.html#cfn-ecs-taskdefinition-dockervolumeconfiguration-labels).

```yaml
TaskDefinition:
  Type: AWS::ECS::TaskDefinition
  Properties:
    Volumes:
      - Name: jenkins_home
        DockerVolumeConfiguration:
          Driver: local
          DriverOpts:
            type: nfs
            device: !Sub "${FileSystem}.efs.${AWS::Region}.amazonaws.com:/"
            o: !Sub "addr=${FileSystem}.efs.${AWS::Region}.amazonaws.com,nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2"
          Labels:
            foo: bar
          Scope: task
```
