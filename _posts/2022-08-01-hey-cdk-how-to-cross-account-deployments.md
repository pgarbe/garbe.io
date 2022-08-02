---
layout: post
title: 'Hey CDK, how do cross-account deployments work?'
date: 2022-08-01 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: 'AWS, CDK, AWS CDK, Bootstrap, CDK Bootstrap, IAM, Deployment '
description: 'Hey CDK, how do cross-account deployments work?'
cover: /assets/heycdk.png
---

The AWS CDK makes it easy to deploy your application, regardless if it consists of multiple stacks that are deployed in multiple accounts and regions. Even [assets](https://docs.aws.amazon.com/cdk/v2/guide/assets.html), for example, the code of lambda functions or the container image of Fargate services, are not a problem. But in your pipeline, you might need some more control over what happens when. And you should understand what happens under the hood. Why? Because CI/CD systems need a lot of permissions in your AWS account and you should grant only as many permissions as needed. 

> This is another part of my ['Hey CDK'](https://garbe.io/category/cdk/) series.


## Bootstrap your AWS accounts
As soon as you start using assets in your CDK application, you need to bootstrap your AWS account. Assets could be files or container images that are part of your application, for example, the code of your lambda function or the image for your Fargate service. These things will be packed together with the stack template into a [Cloud Assembly](https://docs.aws.amazon.com/cdk/v2/guide/apps.html#apps_cloud_assembly), typically in the cdk.out folder.

![CDK Bootstrap stack visualized](/assets/cdk-multi-account-bootstrap.png)

The `cdk bootstrap` command creates a CloudFormation stack in your account which contains an S3 bucket for files and an ECR repository for container images. It also creates the following IAM roles:

**File Publishing Role**  
*Role name: cdk-hnb659fds-file-publishing-role-\${AWS::AccountId}-\${AWS::Region}*  
The file publishing role is used in `cdk deploy` and `cdk-assets` to publish file assets to S3. It allows to read and write to the *Staging Bucket*. As the S3 bucket is encrypted with a KMS key (also created by bootstrap) it grants additional access to the KMS key.

**Image Publishing Role**  
*Role name: cdk-hnb659fds-image-publishing-role-\${AWS::AccountId}-\${AWS::Region}*  
This role is used by `cdk deploy` and `cdk-assets` to publish container image assets to ECR. It allows to read and push container images. Note, that with template version 13 (introduced with CDK version 2.25.0), the ECR repository is immutable so that images can't be overwritten. 

**Lookup Role**  
*Role name: cdk-hnb659fds-lookup-role-\${AWS::AccountId}-\${AWS::Region}*  
The lookup role is used to read information about existing resources whenever you use `.fromLookup(...)` functions. This should be used during development as the necessary information is stored as [context](https://docs.aws.amazon.com/cdk/v2/guide/context.html) (typically `cdk.context.config` file). The role has *ReadOnlyAccess* permissions and an explicit deny on KMS so that it can't access encrypted data!

**CloudFormation Execution Role**  
*Role name: cdk-hnb659fds-cfn-exec-role-\${AWS::AccountId}-\${AWS::Region}*  
This role is used as a [service role](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-servicerole.html) that allows CloudFormation to make calls to resources in a stack on your behalf. Only CloudFormation can assume this role so it can't be used directly by you or your pipeline. 

Per default, it gets the managed *AdministratorAccess* policy attached, but this can be [customized](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html#bootstrapping-customizing).

**Deployment Action Role**  
*Role name: cdk-hnb659fds-deploy-role-\${AWS::AccountId}-\${AWS::Region}*  
The `cdk deploy` command uses this role to create, update or delete CloudFormation stacks. In addition, it has the permission to [pass](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html) the *CloudFormation Execution Role* so that CloudFormation can use it as a service role.


### Cross Account
Cross-account deployments mean you run `cdk deploy` on Account A and want to deploy the stack to Account B. In that case, Account B has to trust Account A so that the roles described above (except *CloudFormation Execution Role*) can be assumed from Account A.

To trust another account, the bootstrap command needs to know the account ids. Use the `--trust` and `--trust-for-lookup` options to provide those values. The roles are updated accordingly to allow to assume them from these trusted accounts.

```sh
cdk bootstrap --trust <accountIds> --trust-for-lookup <accountIds>
```

## How deployments work

Deploying a CDK application is about synthesizing (the cloud assembly), publishing (the assets), and deploying (the stacks). 

![Tools used for Build Publish and Deploy](/assets/cdk-multi-account-options.png)


### Option A

The CDK CLI comes with a `deploy` command which does all three steps at once. Just provide a stack name or `--all` and it will **synthesize** the application, **publish** the assets (to S3 and ECR), and **deploy** the stack. It automatically assumes the publishing and deploy roles for each target account and region. If needed, missing values are read from the target accounts by assuming the lookup role.

### Option B

Another approach is to split up build and deployment. Publishing assets and deployment is then one step. Use the `cdk synth` command to create the cloud assembly (typically in cdk.out folder). Running `cdk deploy --app path/to/cdk.out` publishes the assets to the target accounts and triggers deployment of the specified stacks. 

> **When are Container Images built?**  
> You would probably think that this is part of the *synth* phase. But it isn't. `cdk synth` copies all necessary files (think about the [Docker context](https://docs.docker.com/engine/context/working-with-contexts/)) into the cloud assembly (cdk.out folder). During the **publishing** of the assets, the Docker image is built! This is different to file assets which are just copied from the cloud assembly folder into the S3 bucket.


### Option C

This option splits it into three different parts. Again, `cdk synth` is used to create the cloud assembly. But the asset publishing happens by another tool called [cdk-assets](https://www.npmjs.com/package/cdk-assets). It requires the path to the *assets.json* file and publishes file assets to S3 and container assets to ECR. Also here, cdk-assets builts the container images (see "When are Container Images built?"). As each stack produces its assets.json this needs to be executed multiple times.

Finally, the CloudFormation Stacks can be created or updated by running `aws cloudformation deploy`. 


> **What is cdk-assets?**  
> [cdk-assets](https://www.npmjs.com/package/cdk-assets) is an experimental package of the CDK which can publish assets from the cloud assembly folder to all specified accounts and regions. It reads the necessary information from the assets.json file.

## When to use what option?

With all these options, which one should you use? As usual, it depends :) 

### Local Development
Option A is very handy for local development. You need at least permissions to assume the roles from CDK Bootstrap in the target account and that's it. 

### CDK Pipeline
If you use the [CDK Pipeline](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.pipelines-readme.html) constructs to deploy your application, the heavy lifting is already done for you. It creates the publish and deploy steps automatically, based on the Stages and Stacks that are defined in your application. 

It is based on Option C. The cloud assembly is created in the build step and forwarded as a build artifact to the next steps. It uses *cdk-assets* to create the container images and publish all assets to their target accounts at once. And CodeDeploy steps create or update the Cloudformation stacks.

This approach follows the least privilege principle. The build step does not require any permission at all (the lookup should be done on your local machine). The publish steps just need to assume the *File/Image Publishing Roles*. And the deployment step assumes the *Deployment Action Role* which can only create or update CloudFormation stacks. CloudFormation itself is using the *CloudFormation Execution Role* which normally has Administrator permissions. Of course, this does not 100% prevent any misuse but it reduces the attack surface. 


### Other CI/CD Systems
If you use other CI/CD systems like [GitHub Actions](https://github.com/features/actions), [GitLab CI](https://docs.gitlab.com/ee/ci/), [Travis](https://www.travis-ci.com/), or [CircleCI](https://circleci.com) you have to write your own pipeline. 

Although you can use any option from above, the first one should not be used as it requires access to your AWS account in your build step. This should be avoided to reduce the attack surface.

Keep in mind that *cdk deploy --app* and *cdk-assets* will also run docker build for container image assets. Separating publishing in multiple pipeline steps would cause multiple docker builds which can lead to different container images for different environments.

### Answer

Although AWS CDK deployments across accounts work out of the box, it's essential to understand how it works and which roles and permissions are involved. CDK Bootstrap comes with a set of predefined IAM roles that are used during build, publish, and deployment that follow the least privilege principle. The CdkPipeline construct set up everything automatically for you, but also in your own pipelines the same tools can be used.
