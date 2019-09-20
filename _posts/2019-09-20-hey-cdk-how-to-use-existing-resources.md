---
layout: post
title: "Hey CDK, how can I reference existing resources?"
date: 2019-09-20 07:30:29 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: "AWS, CDK, CloudFormation, Construct"
description: "How Can I reference existing resources"
cover: /assets/heycdk.png
---

After seeing some demos or trying out the [AWS Cloud Development Kit (CDK)](https://aws.amazon.com/cdk/), many questions arise. How can I migrate? What's about my existing resources? What is the Context about? And so on.

This is the second part of a series 'Hey CDK'
- [How can I migrate my existing CloudFormation templates into CDK?](https://garbe.io/blog/2019/09/11/hey-cdk-how-to-migrate/)
- How can I reference existing resources in CDK?


### How can I reference existing resources in CDK?

Most of the time it's necessary to reference existing resources which have already been created in your AWS account. Either by another CloudFormation Stack or by another CDK App.

To do that, many of the CDK Constructs support `fromXXX()` methods. Let's have a look at two typical examples:


#### Example VPC
Often, CDK examples start with a line that creates a whole VPC. Although it's quite impressive it's not what you would use in your daily business. Normally, VPCs have already been created for you (e.g. from [Landing Zone](https://aws.amazon.com/solutions/aws-landing-zone/))

Now you have two options:

__fromLookup__  
The fromLookup() method is called during `cdk synth` and checks a file called 'cdk.context.json' if there are matching VPC information based on the criteria you have defined. If the file does not exist or there were no hits, CDK scans your AWS account and writes the necessary information about the VPC into that file.

> CDK needs to know in which AWS Account the lookup should happen. That's why you have to define [env](https://docs.aws.amazon.com/cdk/latest/guide/environments.html) for your stack. How this works will be covered in another blog post.

To use the default VPC is as simple as creating a new one:

```typescript
const vpc = Vpc.fromLookup(this, 'MyExistingVPC', { isDefault: true });
new cdk.CfnOutput(this, "MyVpc", {value: vpc.vpcId });
```

> But: VPC.fromLookup fails with asymmetric subnets (like 3 public and 6 private subnets). See the [GitHub issue](https://github.com/aws/aws-cdk/issues/3407)

Be aware that the looked-up value is rendered into your template and that this template is not environment-agnostic.

```yaml
Outputs:
  vpcoutput:
    Value: vpc-12ab123d
```

__fromVpcAttributes__   
This method does not need any access to your AWS Account. Instead, you have to set all the properties manually. If the needed values are available as [CloudFormation exports](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html), it's easy to build the VPC Construct:

```typescript
const vpc = Vpc.fromVpcAttributes(this, 'VPC', {
  vpcId: Fn.importValue('VPC'),
  availabilityZones: ['eu-west-1a', 'eu-west-1b', 'eu-west-1c'],
  publicSubnetIds: [
    Fn.importValue('Public-SubnetA'),
    Fn.importValue('Public-SubnetB'),
    Fn.importValue('Public-SubnetC'),
  ],
});
```

As CDK does not know about the actual values, it renders an ImportValue function which makes the template environment-agnostic.

```yaml
Outputs:
  vpcoutput:
    Value:
      Fn::ImportValue: VPC
```

Both methods, `fromLookup()` and `fromVpcAttributes()`, look very similar, but under the hood they are quite different. Be aware of it.

## Example ALB

Besides VPC, the Application Load Balancer is also a good example how existing resources can be referenced inside CDK.

To get the Construct of an existing ALB, just call the `fromApplicationLoadBalancerAttributes()` method which takes the ARN of the load balancer and the Id of the Security Group. 

In this example, the values are hardcoded, but you can also use the `Fn.importValue('xxx')` method.

```typescript
// ALB from existing arns
const existingAlb = elb.ApplicationLoadBalancer.fromApplicationLoadBalancerAttributes(this, "ImportedALB", {
  loadBalancerArn: "arn:aws:elasticloadbalancing:eu-west-1:123456789012...",
  securityGroupId: "MyAlbSecurityGroupA12345AB"
});
```

In case you want to share the ALB with many different services, you need the Listener Construct. Once you have it, a new TargetGroup with some rule can be added as in this example:

```typescript
const listenerArn = "arn:aws:elasticloadbalancing:eu-west-1:123456789012:listener/app/..."
const securityGroup = SecurityGroup.fromSecurityGroupId(this, "MyAlbSecGroup", "MyAlbSecurityGroupC60015CB:")

// Listener from existing arns
const existingListener = elb.ApplicationListener.fromApplicationListenerAttributes(this, "SharedListener", {
  listenerArn,
  securityGroup
});


// Add new TargetGroup for your own service
const targetGroup = new elb.ApplicationTargetGroup(this, "myService", {
  vpc: vpc,
  port: 80,
  // targets: []
});

existingListener.addTargetGroups("MyService", {
  hostHeader: "myservice.local",
  priority: 1,
  targetGroups: [targetGroup]
});
```

### Answer
Existing resources can be referenced in CDK by calling the Construct's `fromXXX()` method. They often need ARNs or Ids which can be imported from existing CloudFormation Stacks.

Just `Vpc.fromLookup()` is a special case as it reads the values from your AWS account during `cdk synth` and stores them in 'cdk.context.json'.
