---
layout: post
title: 'Bypass Docker Hub Rate Limits with ECR PullThrough Cache'
date: 2024-04-09 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK, ECR]
keywords: 'AWS, CDK, AWS CDK, ECR, Docker, PullThrough Cache'
description: 'Bypass Docker Hub Rate Limits with ECR PullThrough Cache'
---

Experiencing Docker Hub's rate limits? ECR PullThrough Cache seems like a lifesaver, but beware of potential pitfalls, especially with cross-account access. Learn how to navigate these challenges and optimize your container workflow effectively.

I have already described the reasons why you should not use DockerHub images in production [here](/blog/2020/04/22/cdk-ecr-sync/). In short, the lack of an SLA from DockerHub affects the availability of your own application. But costs can also play a role.

> This article focuses on Docker Hub as an upstream registry. In principle, however, it also applies to all other [supported registries](https://docs.aws.amazon.com/AmazonECR/latest/userguide/pull-through-cache-creating-rule.html), like Quay, GitHub, or Azure Container Registry.

## Overview
The basic idea is to use ECR as a PullThrough cache for any image that is hosted by Docker Hub, GitHub, K8s or Quay. This article explains what needs to be done in your AWS account 

![ECR PullThrough Cache](/assets/ecr-pull-through.png)

### Rules
Rules can be used to create repositories dynamically in ECR. A rule consists of a repository prefix and an upstream registry. If the repository does not yet exist during a *docker pull*, a repository is automatically created (provided that the prefix matches). The part after the prefix is used to pull the image from upstream, cache it in the ECR repository and return it to the user.

Further pulls end up directly in the ECR repository and are not forwarded to upstream. In addition, ECR regularly compares the images with upstream and updates them if necessary.

The *docker pull* command is divided as follows:
![ECR Pull Command](/assets/ecr-pull-command.png)


### Templates
Templates allow you to define defaults that are used for the newly created repositories. These include lifecycle rules, IAM policy, KMS encryption, and also tags.

### For Organizations
If the pull-through cache is also to be used by other accounts in the organization, a condition can be stored in the policy that allows access for all accounts in the organization.

There are two different policies that must be taken into account.

#### ECR Private Registry Policy

In ECR you can define your own policy for the "Private Registry" part of ECR. Here is an example with CDK that allows all roles in the accounts of the organization to dynamically create repositories for the path `/mydockerhubmirror/*` (assuming a suitable pull-through cache rule exists) and download images from the upstream registry.

> Strictly speaking, any repository can be created with this policy, but no images can be pushed or pulled.

```ts
new cdk.aws_ecr.CfnRegistryPolicy(this, 'RegistryPolicy', {
    policyText: {
        Version: '2012-10-17',
        Statement: [
            {
                Sid: 'AllowPullThrough',
                Effect: 'Allow',
                Principal: {
                    AWS: '*',
                },
                Action: [
                    'ecr:CreateRepository',
                    'ecr:BatchImportUpstreamImage',
                    'ecr:*', // For whatever reasons this is needed to pull with a cross-account role
                ],
                Resource: 'arn:aws:ecr:eu-central-1:123456789012:repository/mydockerhubmirror/*',
                Condition: {
                    StringEquals: {
                        'aws:PrincipalOrgID': 'o-xxx',
                    },
                },
            },
        ],
    },
});
```

> ⚠️⚠️ **ATTENTION** ⚠️⚠️  
> For some reason, `ecr:*` is required if you initially pull an image where no ECR repository yet exists and the request comes from a role of a different account. For this reason you might do the initial pull only with a role from the same account. I am still trying to find out which permission is missing or if it is a bug in ECR.

#### ECR Repository Policy
You also need a permission for the repository. In addition to the known permissions for pull, `ecr:BatchImportUpstreamImage` is required. The condition for the OrganizationId is also required here.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCrossAccountPull",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "ecr:BatchCheckLayerAvailability",
                "ecr:BatchGetImage",
                "ecr:GetDownloadUrlForLayer",
                "ecr:ListImages",
                "ecr:BatchImportUpstreamImage",
            ],
            "Condition": {
                "StringEquals": {
                    "aws:PrincipalOrgID": "o-xxx"
                }
            }
        }
    ]
}
```


## Pitfalls
While many error possibilities are caught in the AWS Console, there are several ways to run into confusing errors in CDK or CloudFormation.

### Naming Conventions for Secrets 
The credentials for the upstream registry are saved as a secret in SecretsManager. To do this, the name must begin with the prefix `ecr-pullthroughcache/...`. This unnecessary restriction makes it impossible to follow the best-practices for IaC to avoid hardcoded names. 

In such a case I use the StackName as part of the name (both in CDK and CloudFormation) to make the name unique.

### Upstream Registry needs Url
Although the [docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecr-pullthroughcacherule.html) say that *upstreamRegistry* and *upstreamRegistryUrl* are optional, both must be specified. The values for *upstreamRegistry* are fixed and documented, the *upstreamRegistryUrl* must be copied from [this page](https://docs.aws.amazon.com/AmazonECR/latest/userguide/pull-through-cache-creating-rule.html#pull-through-cache-creating-rule-cli). Without *upstreamRegistryUrl* only a "null" error is returned.

```sh
11:55:45 AM | CREATE_FAILED        | AWS::ECR::PullThroughCacheRule | DockerImagesMirror/DockerHub
Resource handler returned message: "null" (RequestToken: xxx, HandlerErrorCode: InternalFailure)
```


### Invalid Repository Prefix 
The *EcrRepositoryPrefix* has a [documented](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ecr-pullthroughcacherule.html#cfn-ecr-pullthroughcacherule-ecrrepositoryprefix) restriction on which characters may be used. In the case of invalid characters, however, the resource is apparently created but the delete on rollback fails because the resource does not exist. This takes about 5 minutes until the stack is completely rolled back.


```sh
12:32:08 PM | CREATE_FAILED        | AWS::ECR::PullThroughCacheRule | DockerImagesMirror/DockerHub
Resource handler returned message: "Invalid parameter at 'ecrRepositoryPrefix' failed to satisfy constraint: 'Member must satisfy regular expression pattern: (?:[a-z0-9]+(?:[._-][a-z
0-9]+)*/)*[a-z0-9]+(?:[._-][a-z0-9]+)*' (Service: Ecr, Status Code: 400, Request ID: xxx)" (RequestToken: xxx, HandlerErrorCode: InvalidRequest)
```

BUT Rollback now also fails (only 6min until it gave up)

```sh
12:32:14 PM | DELETE_FAILED        | AWS::ECR::PullThroughCacheRule | DockerImagesMirror/DockerHub
Resource handler returned message: "Invalid parameter at 'ecrRepositoryPrefix' failed to satisfy constraint: 'Member must satisfy regular expression pattern: (?:[a-z0-9]+(?:[._-][a-z
0-9]+)*/)*[a-z0-9]+(?:[._-][a-z0-9]+)*' (Service: Ecr, Status Code: 400, Request ID: xxx)" (RequestToken: xxx, HandlerErrorCode: GeneralServiceException)
```

### Creation Templates are still in Preview
As this feature is still in preview, there are some limitations. There is neither API nor CloudFormation support. And it is not possible to make changes to existing templates. To do this, the template must be deleted and recreated. Changes to templates do not affect repositories that have been created with this template, but only for future repositories.

> Tip: If the template is changed, simply delete the existing repositories. But be careful that you do not run into rate limits when filling the "cache" again.

### PullThroughCacheRule requires always a replacement
All properties of *PullThroughCacheRule* require a replacement. Changing the secret (means *CredentialArn*) is just not possible since CloudFormation creates the new *PullThroughCacheRule* (with a conflicting EcrRepositoryPrefix) first before it deletes the old one.


## Conclusion
Pulling images from ECR instead of DockerHub or other public registries improves your resilience and can save money. The ECR PullThrough cache works quite nice once it is set up. 

But getting it set up with CDK or CloudFormation is bumpy and could be improved. And be aware of the wildcard permissions that are needed for cross-account access! This gets hopefully fixed soon by AWS!
