---
layout: post
title: 'Hey CDK, what should I know before refactoring CDK applications?'
date: 2024-04-30 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: 'AWS, CDK, AWS CDK, npm, peerDependencies, dependencies, dependency, bundledDependencies'
description: 'Hey CDK, what should I know before refactoring CDK applications?'
cover: /assets/heycdk.png
---

For those who know me it's not a surprise that I'm a big fan of the CDK. I have worked many years with CloudFormation and that's why it's crystal clear to me what happens under the hood when I change something in my CDK code. But I see many people struggling when making changes to CDK code. Often they started directly with CDK without knowing what CloudFormation is and how it works. 


## Refactoring CDK Code
Refactoring is a fundamental aspect of software development. It's a practice that developers, with their innate drive for optimization and efficiency, are intimately familiar with. And just like any other codebase, Cloud Development Kit (CDK) applications are not exempt from the need for refactoring. 

But, as with any form of renovation, there's a cautionary tale to heed. Refactoring CDK applications isn't without its perils, and the danger lies in the potential recreation of resources. For stateful resources like S3 buckets or databases, this can quickly escalate from an inconvenience to a critical issue.

So, how do you navigate the treacherous waters of CDK application refactoring without inadvertently capsizing your stateful resources? In this blog post, I'll explore best practices, strategies, and safeguards to ensure that your CDK refactoring endeavors are not only smooth but also devoid of any surprises. 


## Back to CloudFormation

To delve into the realm of refactoring CDK applications effectively, we must first revisit the foundation upon which CDK is built — CloudFormation. Understanding how CloudFormation operates is essential because, despite CDK's abstraction layer, it remains the underlying engine. 

1. **Understanding CloudFormation's Core Concepts:** CloudFormation operates based on templates that describe the AWS resources and their dependencies. When you refactor a CDK application, you're essentially working with these templates. Therefore, comprehending CloudFormation's nuances is pivotal to successful CDK refactoring.

2. **Creation and Cleanup Phases:** CloudFormation follows a two-phase process: creation and cleanup. During the creation phase, resources are provisioned or updated as needed. However, it's only during the cleanup phase that orphaned resources are removed. This ensures that (stateless) resources that require a replacement are replaced without any interruption.

    *Example:*
    If you change a property like the path of an IAM role, CloudFormation gracefully creates a new role, attaches it to the relevant resource, and then removes the old one. 

    ![CloudFormation two phases](/assets/cfn-two-phases.jpg)

3. **Deletion Triggers:** Especially for stateful resources it is important to understand when CloudFormation tries to delete them. It can occur under specific circumstances:
    - **Removal from the Template:** If you remove a resource from the CloudFormation template altogether, it will be deleted during the next stack update.
    - **Logical ID Changes:** Changing the logical ID of a resource essentially tells CloudFormation that it's a different resource, triggering its deletion in cleanup phase.
    - **Property Changes Requiring Replacement:** Some property changes necessitate the replacement of a resource. When this happens, CloudFormation deletes the old resource after creating the new one.

In the next sections, we'll explore strategies to mitigate these risks and ensure a seamless refactoring experience.


## Tips & Tricks

When refactoring CDK applications, ensuring a seamless transition and avoiding unexpected hiccups is of paramount importance. Here are some tips and tricks to help you navigate this process with confidence:

### 1. Be Aware of Stateful and Stateless Resources

Understanding the nature of your AWS resources is crucial. Most resources can be replaced by CloudFormation with relative ease, thanks to its built-in orchestration capabilities. However, some resources, like S3 buckets, RDS databases, or DynamoDB tables, should be preserved unless replacement is your explicit intention. These stateful resources require special attention during refactoring to prevent unintended consequences.

### 2. Construct IDs and Hierachy Matter

CloudFormation identifies resources based on their Logical IDs. In CDK, you work with constructs, each of which is assigned an ID within the construct tree. These IDs are mapped to logical IDs, using a hash based on their position in the hierarchy. 

If you change a resource's logical ID, CloudFormation interprets it as a "new" resource, resulting in the deletion of the one with the old ID. To avoid this, plan a logical naming schema for your IDs from the outset. Consistency and forethought in naming, such as using PascalCase, can save you from potential refactoring woes.

<!-- TODO: Picture how a "Id" leads to LogicalId and therefor PhysicalId -->

In case you need to keep the former LogicalId, CDK allows to override it. But it works only for L1 constructs so you might need to figure them out first. Use `cdk diff` for each environment to see which resources will be replaced and to find out the former LogicalId.

> This might be tricky for RDS resources as they include a few L1 constructs that needs to be overriden.

```ts
const bucket = new cdk.aws_s3.Bucket(this, 'Bucket');

// Go to the underlying L1 CfnBucket construct 
// and override the logicalId to keep the former value
(bucket.node.defaultChild as cdk.aws_s3.CfnBucket).overrideLogicalId('FormerLogicalId');
```

### 3. Watch Your Resource Naming

While many AWS resources allow you to define a name (e.g. [FunctionName](TODO) in Lambda), take heed when considering resource names that could be replaced during refactoring. CloudFormation operates in two phases: it attempts to create a new resource with the same name before cleaning up the old one with that name. This can lead to errors, as resource names must be unique within a stack. To circumvent this, avoid defining resources names, especially when dealing with properties that may require replacement.

![CloudFormation conflicts with named resources](/assets/cfn-duplicate-name.jpg)

### 4. Properties Requiring Replacement

Certain properties of AWS resources necessitate a complete replacement when modified. Identifying these properties and their impact on your stack is crucial. Be sure to consult the AWS documentation for specifics on properties that trigger replacements, as understanding this aspect is key to successful refactoring.

<!-- TODO: Check them in the CloudFormation docs -->

### 5. Deletion Policies

Every AWS resource supports a deletion policy. By default, CloudFormation deletes resources if they are not part of the template, but you have the option to specify other behaviors, such as Retain or RetainExceptOnCreate. The Retain policy ensures that a resource is not physically deleted but is removed from the stack, allowing stack updates to proceed without failure. Additionally, some resources support Snapshot as a deletion policy, which can be especially useful for preserving data. For detailed information on deletion policies, refer to the AWS documentation [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html).

<!-- RDS comes with delete protection, S3 Buckets require to be empty -->
<!-- TODO: They are useful if you want to "import" existing resources -->

### 6. (Snapshot) Tests

For stateful resources, it's a wise practice to create tests that verify their integrity during refactoring. These tests help ensure that these resources are not inadvertently replaced. Snapshot tests can also be valuable for quickly identifying changes in the resource state. By employing a robust testing strategy, you can confidently refactor your CDK applications without compromising the stability of your stateful resources.


## Summary
Refactoring CDK applications is possible but needs some awareness and ideally some basic knowledge how CloudFormation operates.

Keep in mind:
* You mainly have to care about stateful resources (like databases, buckets)
* Stateless resources are gracefully replaced by CloudFormation's two phases
* Be aware what changes within CDK trigger replacements in CloudFormation
* Write (snapshot) tests to ensure that stateful resources are not deleted
* Replacement triggered by changed properties can still bite you
