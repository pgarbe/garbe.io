---
layout: post
title: "Hey CDK, how can I migrate my existing CloudFormation templates?"
date: 2019-09-11 07:30:29 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: "AWS, CDK, CloudFormation, Construct"
description: "How Can I Migrate My Existing CloudFormation Templates"
cover: /assets/heycdk.png
---

After seeing some demos or trying out the [AWS Cloud Development Kit (CDK)](https://aws.amazon.com/cdk/), many questions arise. How can I migrate? What's about my existing resources? What is the Context about? And so on. 

Together with my fellow AWS Hero [Thorsten Hoeger](https://twitter.com/hoegertn), I gave a talk on CDK Deep Dive on the German [AWS Community Day](https://www.aws-community-day.de) where we tried to answer those questions. But I think it also makes sense to write them down. That's why I started a little series about typical questions around the CDK. 

This is the first part of a series 'Hey CDK'
- How can I migrate my existing CloudFormation templates?
- [How can I reference existing resources?](/blog/2019/09/20/hey-cdk-how-to-use-existing-resources/)
- [How can I write even less code?](/blog/2019/10/01/hey-cdk-how-to-write-less-code/)

### How can I migrate my existing CloudFormation templates?

There are multiple options that we will go through. Let's imagine we have the following (simple) CloudFormation template:

```yaml
AWSTemplateFormatVersion: '2010-09-09'

Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      VersioningConfiguration:
        Status: Enabled
```

#### A) Rewrite your template in CDK
To be honest, rewriting an existing template in CDK takes some effort. The bigger the template the more effort it is. 

CDK treats the CloudFormation resources as lower-level Constructs. If you want to write it in pure CDK, you normally use higher-level Constructs. The problem here is now to get the right higher-level Constructs and find out which CloudFormation resources will be replaced by them. For example, the higher-level Lambda `Function` Construct generates not only the CloudFormation resource of the function itself but also an IAM Role. 

The challenge in this process is to keep the generated template as similar as possible to your existing template. That's not as easy as it sounds because CDK generates LogicalIds by concatenating the elements of the path and adding an 8-digit hash. So, CloudFormation will always show that it replaces your resources. 

Let's use the CloudFormation template from above. In pure CDK we would probably write something like this:

```typescript
new s3.Bucket(this, "MyS3Bucket", { 
  versioned: true, 
  removalPolicy: cdk.RemovalPolicy.DESTROY 
});
```

> Note: The removalPolicy is needed as the default in CDK ("Retain") is different than the one in CloudFormation ("Destroy").

The `cdk diff` command is very helpful, as it shows you the changes which will be applied if you deploy it. 

```bash
$ cdk diff MigrationStack
Stack MigrationStack

Resources
[-] AWS::S3::Bucket MyS3Bucket destroy
[+] AWS::S3::Bucket MyS3Bucket MyS3Bucket4646DF6F 
```

Ups, it destroys the existing bucket and creates a new one. That's not what we want. Luckily, we didn't deploy it so far.  

> For most resources it's not a problem to be replaced. But be careful with your stateful resources (like S3 or RDS)!
>   
> __Tip:__ Add a DeletionPolicy to your important resources before you start the migration.

As we want to keep that Bucket, we have to ensure that the LogicalId doesn't change. Fortunately, it can be overridden:

```typescript
const bucket = new s3.Bucket(this, "MyS3Bucket", { 
  versioned: true, 
  removalPolicy: cdk.RemovalPolicy.DESTROY 
});

const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;
cfnBucket.overrideLogicalId("MyS3Bucket");
```

The diff looks much better now:

```bash
$ cdk diff MigrationStack
Stack MigrationStack

Resources
[~] AWS::S3::Bucket MyS3Bucket MyS3Bucket 
 ├─ [+] DeletionPolicy
 │   └─ Delete
 ├─ [+] Metadata
 │   └─ {"aws:cdk:path":"MigrationStack/MyS3Bucket/Resource"}
 └─ [+] UpdateReplacePolicy
     └─ Delete
```
Don't be confused. It adds the UpdateReplacePolicy and DeletionPolicy explicitly (to avoid any confusion with the defaults) and also some Metadata. But, it keeps the S3 bucket (with all your important data).


#### B) Include the CloudFormation template in your CDK App

The third option is to create an empty CDK App and include your existing CloudFormation template by calling `CfnInclude`. If we synth that app, it renders the same CloudFormation template.

```typescript
const include = new cdk.CfnInclude(this, "ExistingInfrastructure", {
  template: yaml.safeLoad(fs.readFileSync("./my-bucket.yaml").toString())
});
```

It's also possible to access attributes, like the Bucket ARN:

```typescript
const bucketArn = cdk.Fn.getAtt("MyS3Bucket", "Arn");
new cdk.CfnOutput(this, 'BucketArn', { value: bucketArn.toString() });
```

The synthesized template looks like this:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Resources:
  MyS3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      VersioningConfiguration:
        Status: Enabled
Outputs:
  BucketArn:
    Value:
      Fn::GetAtt:
        - MyS3Bucket
        - Arn
```

With this approach, you can keep your existing template but add new resources with CDK. It also helps if you want to rewrite your template into CDK step by step.

#### C) Convert CloudFormation template into CDK

Another option is to generate the CDK code automatically based on your CloudFormation template. By using the [CDK CloudFormation Disassembler](https://github.com/aws/aws-cdk/tree/master/packages/cdk-dasm) we can generate the following code from the template:

```bash
$ cdk-dasm < my-bucket.yaml

import { Stack, StackProps, Construct, Fn } from '@aws-cdk/core';
import s3 = require('@aws-cdk/aws-s3');

export class MyStack extends Stack {
    constructor(scope: Construct, id: string, props: StackProps = {}) {
        super(scope, id, props);
        new s3.CfnBucket(this, 'MyS3Bucket', {
            versioningConfiguration: {
              "status": "Enabled"
            },
        });
    }
}
```

Be aware that this tool has some (critical) limitations as explained in the [Readme](https://github.com/aws/aws-cdk/tree/master/packages/cdk-dasm#wip---this-module-is-still-not-fully-functional).

> __Warning:__
>   
>  This option is just for completeness. I don't recommend to use it! Mainly because it generates only low-level resources (like s3.CfnBucket) and you don't benefit from the higher-level resources (like s3.Bucket).

### Answer
To get the most out of CDK, it makes sense to completely rewrite your infrastructure in CDK. This can be very time-consuming. With the help of `Cfn.Include`, we can do the migration step by step.