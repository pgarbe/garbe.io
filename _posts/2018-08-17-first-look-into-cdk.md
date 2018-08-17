---
layout: post
title: "First look into AWS Cloud Development Kit (CDK)"
date: 2018-08-17 09:30:29 +0200
author: Philipp Garbe
comments: true
published: true
# categories: [AWS]
keywords: "AWS, CloudFormation, CDK"
description: "First look into AWS Cloud Development Kit (CDK)"
---

I have been using CloudFormation since I started using AWS in 2014. It helped me to get an overview of all the different services on AWS. I also like the declarative style to define your infrastructure and let CloudFormation do all the work. Coming from a DataCenter where you had to write tickets to get changes done, this was pure magic! Now, AWS published the beta of an important addition to CloudFormation: the [Cloud Development Kit](https://github.com/awslabs/aws-cdk), aka CDK.

__What's the big deal about CDK?__

All the best practices and learnings how to write good CloudFormation templates can now easily be shared within your company or the community! At the same time, you also benefit from others. 

For example, think about DynamoDB. Should be easy to set it up in CloudFormation, right? Just some lines in your template. But wait. When you're in production, you realize that you've to setup auto-scaling, you need regular backups and most important, alarms for all relevant metrics. This can amount to several hundred lines. 

And think ahead: Maybe you've to create another application that also needs a DynamoDB. Do you copy&paste all the yaml code? What happens later, when you find some bug? Do you apply the fix to both code bases?

With CDK, you're able to write a "Construct" for your best practice DynamoDB and share it as npm package in your company or even the whole community!


## What is CDK?
Let's get one step back and see how CDK looks like. Compared to the declarative approach with YAML (or JSON), CDK allows you to declare your infrastructure imperatively. The main language is TypeScript, but several other languages will also be supported. 

> Be aware, that only TypeScript constructs can be used from any language. Non-TypeScript constructs can be used from within the same language, but not anywhere else.

This is how the hello world example from the [docs](https://awslabs.github.io/aws-cdk/versions/0.8.2/getting-started.html) looks like:

```typescript
import cdk = require('@aws-cdk/cdk');
import s3 = require('@aws-cdk/aws-s3');

class MyStack extends cdk.Stack {
    constructor(parent: cdk.App, id: string, props?: cdk.StackProps) {
        super(parent, id, props);

        new s3.Bucket(this, 'MyFirstBucket', {
            versioned: true
        });
    }
}

class MyApp extends cdk.App {
    constructor(argv: string[]) {
        super(argv);

        new MyStack(this, 'hello-cdk');
    }
}

process.stdout.write(new MyApp(process.argv).run());
```

CDK comes with two templates, an App and a Lib.  
__[Apps](https://awslabs.github.io/aws-cdk/versions/0.8.2/apps.html)__ are the root constructs and can be used directly by the CDK-Cli to render and deploy the CloudFormation template.  
Apps consist of one or more __[Stacks](https://awslabs.github.io/aws-cdk/versions/0.8.2/stacks.html)__ which are deployable units and contains information about the region and account. It's possible to have an App which deploys different Stacks to multiple regions at the same time.  
Part of Stacks are __[Constructs](https://awslabs.github.io/aws-cdk/versions/0.8.2/constructs.html)__ that are representations of AWS resources like a DynamoDB table or a Lambda function. 

![CDK Objects](/assets/cdk.png)

A __Lib__ is a Construct which typically encapsulates further Constructs. With that, higher class Constructs can be build and also reused. As the Construct is just TypeScript (or any other supported language), a package can be built and shared by any package manager.

### Constructs

As CDK is all about Constructs it's important to understand them. It's a hierarchal structure called construct tree. You can think of Constructs in three different level:

__Level 1: CloudFormation Resource__  
This is a one-to-one mapping of existing resources and will be auto-generated. It's the same as the resources you use currently in YAML. Ideally, you won't have to deal with these Constructs directly.

__Level 2: AWS Construct Library__  
These Constructs are on an AWS Service level. They're opinionated, well-architected and handwritten by AWS. They come with proper defaults and should make it easy to create AWS Services without worrying too much about the details. 

As an example, this is how to create a complete VPC with private and public subnets in all available AZs:

```typescript
import ec2 = require('@aws-cdk/aws-ec2');

const vpc = new ec2.VpcNetwork(this, 'VPC');
```

The AWS Construct Library has some nice concepts about least privilege IAM policies, event-driven API, security groups, and metrics. 

For example, IAM policies are automatically created based on your intent. When a Lambda subscribes to an SNS Topic, a policy is created which allows the topic to invoke the lambda function. AWS Services which offer CloudWatch metric have functions like `metricXxx()` and return metric objects that can easily be used to create alarms.

```typescript
new Alarm(this, 'Alarm', {
    metric: fn.metricErrors(),
    threshold: 100,
    evaluationPeriods: 2,
});
```

More information can be found [here](https://awslabs.github.io/aws-cdk/versions/0.8.2/aws-construct-lib.html)


__Level 3: Your awesome stuff__  
And here it becomes interesting. As mentioned above, Constructs are hierarchical and they can be higher level abstractions based on other Constructs. As an example, On Level 3 you can write your own ECS Cluster Construct which contains automatic node draining, auto-scaling, and all proper alarms. Or you write a Construct for all necessary alarms an RDS database should monitor.


## Conclusion
It's good that AWS went public in an early stage. The docs have already a good quality, but not everything is covered yet. Not all AWS services have already their Level 2 Construct Library but only the pure CloudFormation Constructs (Level 1).  
 It's also a bit weird (especially, when you're not used to TypeScript/JavaScript) to compile TypeScript into JavaScript and then use the cdk cli to deploy instead of the single statement `aws cloudformation deploy`. But I hope there will be some improvements as well.

In general, I think this is a huge step forward, as it allows us finally, to re-use the CloudFormation code and share it with others. This will allow people to work on awesome features and spend less time on writing "boring" CloudFormation code.
