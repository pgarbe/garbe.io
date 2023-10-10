---
layout: post
title: 'Deep Dive on StackSets with CDK'
date: 2023-10-10 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: 'AWS, CDK, AWS CDK, StackSet, SelfManaged, ServiceManaged'
description: 'Deep Dive on StackSets with CDK'
# cover: /assets/cdk-stackset-6.png
---

Can StackSets be fully managed with CDK? In this deep dive you will learn how to synthesize CDK Stacks into StackSet templates and how to deal with Assets when you need them across accounts and regions.

> This is the written form of my talk at the [CDK Day](https://cdkday.com) 2023. You can watch it here: 

<iframe width="560" height="315" src="https://www.youtube.com/embed/qlUR5jVBC6c?si=k0LuPs8Mqb71hgqi&amp;start=1240" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## Why StackSets?

StackSets are a powerful feature in AWS CloudFormation that allow you to deploy resources across multiple AWS accounts and regions simultaneously. They are particularly useful for rolling out infrastructure consistently across an organization's accounts. However, working with StackSets in CDK can be challenging due to some unique considerations.


## StackSets in CloudFormation

Let's start with a basic understanding of StackSets in CloudFormation. In CloudFormation, you use templates (typically in YAML or JSON format) to define and provision your AWS resources. These templates are used to create CloudFormation stacks, which represent sets of resources.

![A simple CloudFormation stack](/assets/cdk-stackset-1.png)

But what if you need to deploy the same template across multiple AWS accounts? This is where StackSets come into play. A StackSet is essentially an orchestration mechanism that takes a CloudFormation template and deploys it into multiple AWS accounts, ensuring consistency across the organization.

![Use StackSet to deploy to multiple accounts](/assets/cdk-stackset-3.png)

Of course, a CloudFormation Stack is used to create the StackSet. You have to deal with two templates now. The Stack Template is responsible to create the StackSet resource (like any other AWS resources).

*Stack Template:*  
```yaml
Resources:
  MyStackSet:
    Type: "AWS::CloudFormation::StackSet"
    Properties:
      // other properties...
     TemplateURL: "https://s3.amazonaws.com/.../stackset-template.yaml"
```

The StackSet Template is like any other CloudFormation template. It is referenced in the Stack Template and used to be rolled out in the target accounts as CloudFormation Stack.

*StackSet Template:*  
```yaml
Resources:
  MyLambdaToRunInEveryAccount:
    Type: "AWS::Lambda::Function"
    Properties:
      Runtime: nodejs18.x
      Handler: index.handler
      Code: ...
```

#### Self-Managed vs. Service-Managed StackSets
There are two types of StackSets: self-managed and service-managed. Here's a brief overview of the differences:

**Self-Managed StackSets:**   
With self-managed StackSets, you have more control but also more responsibility. You decide which accounts to add or remove from your StackSet (that's called *stack instances*). However, you need to handle the Identity and Access Management (IAM) roles and permissions for the operations between the management account and target accounts.

**Service-Managed StackSets:**   
AWS takes care of adding or removing accounts for you based on organizational units (OUs). Whenever an account is added or removed from an OU, it's automatically included in or removed from the StackSet. AWS also manages the IAM roles and permissions, reducing your administrative overhead.


## CDK and StackSets

Now, let's bring CDK into the equation. CDK is commonly used to synthesize and deploy AWS CloudFormation stacks. 

![StackSets created with CDK](/assets/cdk-stackset-4.png)

When working with StackSets, CDK has to perform two key tasks:  
1. Synthesize a stack template and deploy it as a CloudFormation stack (business as usual).
2. Synthesize the StackSet template and upload it to an S3 bucket, which is used as a template URL for the StackSet.

The second case is not (yet) supported by AWS CDK, but the team provides an experimental L2 construct ([cdklabs/cdk-stacksets](https://github.com/cdklabs/cdk-stacksets/)) which I use in the following snippets. 

> ⚠️ The projects in cdklabs are experimental and under active development. They are subject to non-backward compatible changes or removal in any future version. 

It introduces a new *StackSetTemplateStack* class to handle StackSet templates in CDK. By extending from StackSetTemplateStack, it is not listed as deployable stack but rather synthesized as JSON template and added as an Asset to the parent stack.

```ts
class MyStackSet extends StackSetTemplateStack {

  constructor(scope: Construct, id: string, props: StackSetTemplateStackProps) {
    super(scope, id, props);

    new.Function(this, 'Lambda', {
      // ...
    });
  }
}
```

In the application create a new instance of *MyStackSet* class and reference it as template for the StackSet.

```ts
const app = new cdk.App
const stack = new Stack(app, ‘MyStack’);

// Create the StackSetTemplateStack
const stackSetStack = new MyStackSet(stack, 'MyStackSet');

new StackSet(stack, 'StackSet', {
  target: StackSetTarget.fromAccounts{…}),
  template: StackSetTemplate.fromStackSetStack(stackSetStack),
});
```


## Handling Assets in StackSets

Assets, such as Lambda function code, are a common part of CloudFormation templates. However, working with assets in StackSets can be tricky. By default, CDK uploads assets to a private bucket (provided by CDK Bootstrap), which can't be accessed from other accounts.

![StackSets created with CDK](/assets/cdk-stackset-5.png)

To address this, *StackSetTemplateStack* comes with a synthesizer that ensures assets are copied to a public bucket, making them accessible across accounts. This feature simplifies asset management when working with StackSets.

![StackSets created with CDK](/assets/cdk-stackset-6.png)

You have to provide that public bucket when instantiating the StackSetTemplateStack. The bucket should [grant access to specific accounts or your organization only](https://aws.amazon.com/blogs/security/control-access-to-aws-resources-by-using-the-aws-organization-of-iam-principals/). 

The synthesizer 
* adds the asset to the parent stack
* creates an S3 Deployment to copy from the bootstrap bucket to the provided public bucket and
* ensures that the synthesized StackSetTemplateStack points to the assets in the public bucket.


## Cross-Region Deployment

One of the challenges with StackSets is handling cross-region deployments. When adding an account with a different region to a StackSet, deployment can fail because assets must reside in the same region where they are used. 

![StackSets created with CDK](/assets/cdk-stackset-7.png)

This is currently not supported by [cdklabs/cdk-stacksets](https://github.com/cdklabs/cdk-stacksets/) but discussed in [this issue](https://github.com/cdklabs/cdk-stacksets/issues/159)

As a workaround, you can deploy the StackSet in each region separately and then roll it out to the target accounts in the same region. While this adds some overhead, it is a functional approach.

> 💡 Personally I have created a different approach and use a "StackSet Bootstrap" to create S3 Buckets with a specific naming convention in each region. The Synthesizer uses that naming convention and copies the assets to all specified regions.  
> See [pgarbe/cdk-stackset](https://github.com/pgarbe/cdk-stackset)

## Conclusion

In this deep dive into StackSets with AWS CDK, I've explored why StackSets are valuable for managing infrastructure across multiple accounts and regions. We've also seen how *StackSetTemplateStack* class from [cdklabs/cdk-stacksets](https://github.com/cdklabs/cdk-stacksets/) simplifies the handling of StackSet templates and assets, making it easier to work with this powerful AWS feature.

I hope you found this blog post informative and that it helps you leverage StackSets effectively in your AWS CDK projects. If you have any questions or need further clarification, feel free to reach out to me on [CDK Slack](https://cdk.dev) or [any other channel](/about). 
