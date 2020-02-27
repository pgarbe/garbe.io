---
layout: post
title: "CloudFormation Resource Providers - A Chicken and Egg Problem"
date: 2020-02-24 07:00:00 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS]
keywords: "AWS, CloudFormation, CloudFormation Resource Provider, Third party resource provider, Resource provider, Custom Resource"
description: "CloudFormation Resource Providers - A Chicken and Egg Problem"
cover: /assets/cfn-resource-provider-title.png
---

The first service I used on AWS was actually CloudFormation. It's a great way to handle your resources based on some configuration. Over time, the desire grew to configure all other SaaS providers in the same way. And with [CloudFormation Resource Providers](https://docs.aws.amazon.com/cloudformation-cli/latest/userguide/resource-types.html) it's finally possible. 

In the first part of this blog post, I explain how resource providers work and in the second part, I share my thoughts and opinion about it.


> __Disclaimer__:  
> Resource providers are a new feature and they follow AWS's strategy to start small and improve incrementally based on customer feedback. My feedback should not be perceived as grumbling but rather an encouragement. 

## How it works
With resource providers, everyone can define a CloudFormation resource like any other AWS team! It's basically some code (supported languages: Java and Go; Python in preview) that handles the create, update, and delete requests and a specification of how the resource type has to look like (The later one is not possible with plain custom resources). With the additionally published [CLI](https://github.com/aws-cloudformation/cloudformation-cli) vendors can package everything together and publish it. 

![CloudFormation Resource Providers](/assets/cfn-resource-provider.png)

Within an AWS account this package can be registered under a specific name (following the format _company_or_organization::service::type_). Details can be looked up in the AWS [docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/registry.html). After successful registration, this type can be used like any other CloudFormation resource. 

#### Versioning
Resource providers support versioning. Each registration is a new version and versions can be listed with [list-type-versions](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/list-type-versions.html) command. A specific version can (and must) be set by calling the [set-type-default-version](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/set-type-default-version.html) command. The version can't be defined manually but is defined by AWS and looks like a number that simply increments.

Also, references to execution role and logging configuration are versioned (the references, not the roles itself).

#### Permissions
Needed permissions are handled by the execution role and highly depends on the use case of the resource provider. If it's just calling some external API (e.g. DataDog) and doesn't change anything within the AWS account, no additional permissions are needed. But if the resource provider wants to change something in your account, it needs an execution role with that permissions.

#### Logging
Logging is useful to see what the resource provider is actually doing. It's optional and configured separately from the other permissions. The configuration is part of the _register-type_ command and needs an IAM role and an s3 bucket. 

> Ensure that the role trusts __resources.cloudformation.amazonaws.com__ and not just cloudformation.amazonaws.com as I did intentionally. You won't see any error but also no logs. The same also applies to the execution role and is actually well [explained](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/registry.html#registry-register-permissions).

#### Pricing
Although CloudFormation is for free it [charges](https://aws.amazon.com/cloudformation/pricing/) for third party resource providers (This applies not for "Custom::*" custom resources). The free tier includes 1,000 handler operations per month per account. Additional handler operations cost $0.0009 per handler operation. And if your operation takes more than 30 seconds every second above will be charged $0.00008. 

Example: 500 third party resources that are updated every day costs you $12.60 per month (including free tier).

#### Available Providers

So far I've seen resource providers from [Atlas MongoDB](https://github.com/mongodb/mongodbatlas-cloudformation-resources), [Atlassian OpsGenie](https://github.com/opsgenie/opsgenie-cloudformation-resources), [Datadog](https://github.com/DataDog/datadog-cloudformation-resources), [Densify](https://github.com/densify-dev/cloudformation-optimization-as-code), [Dynatrace](https://github.com/mnalezin/DynatraceInstallerAgent), [Fortinet](https://github.com/fortinet/aws-cloudformation-resource-provider), [New Relic](https://github.com/newrelic/cloudformation-partner-integration), and [Spotinst](https://github.com/spotinst/spotinst-aws-cloudformation-registry). 

Most of the repositories haven't been touched (or just slightly) after the re:Invent announcement last November and contain only a subset of the vendor's functionality. Based on the activity at the GitHub repos DataDog seems to be the most active one. They provide versioned artifacts and changelogs. 


## My take on it
These are the things that I noticed and they are of course very opinionated:

#### Versioning
It's common sense (means I couldn't find any evidence that there are written-down rules) that resources in CloudFormation have to be backward compatible. Breaking changes have to be a new resource type like [Elastic Load Balancing](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/AWS_ElasticLoadBalancing.html) and [ElasticLoadBalancingV2](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/AWS_ElasticLoadBalancingV2.html). 

It seems that the CloudFormation team expects the same behavior also from resource providers. That explains why a version can't be set and the internal version returned by CloudFormation is just an incremental number (Although it looks like a number, treat it as string).

If you look how the API calls handle the version it's a bit confusing: 
* The new version is not returned in _register-type_ call but it has to be looked up manually with _list-type-versions_ command (after the registration was successful).
* Calling _describe-type-registration_ returns "TypeVersionArn" (like "arn:aws:cloudformation:eu-west-1:123456789012:type/resource/Atlassian-Opsgenie-User/00000006") but not "VersionId" which is required for _set-type-default-version_.
* The ARN returned by _describe-type_ contains the current default version (which means it changes after each _set-type-default-version_ call)

I'm wondering why the version is not part of the resource provider package provided by the vendor. Now I've to deal with two different kinds of versioning. The one from the vendor and the one from CloudFormation.

That versioning confusion maybe also explains my next issue:

#### Missing CloudFormation Support
Normally, CloudFormation support for new features is rarely from Day 1 and it can take quite a while. You might think that the CloudFormation team is an exception and sets a good example but that's not the case. Not only was there no support from day 1, but it is still not possible to register 3rd party resource providers using CloudFormation. There has been quite a discussion in the raised [issue](https://github.com/aws-cloudformation/aws-cloudformation-resource-providers-cloudformation/issues/3) and [pull request](https://github.com/aws-cloudformation/aws-cloudformation-resource-providers-cloudformation/pull/4/) which also reveals how hard it is to make it easy within CloudFormation. 


#### Secrets Handling
I was positively surprised that resource providers, as opposed to custom resources, support [dynamic references](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html). To make calls to the vendors API the resource provider has to pass credentials to authenticate. These are normally part of the resource type itself. 

Here an example from OpsGenie:

```yaml
  UserA:
    Type: Atlassian::Opsgenie::User
    Properties:
      OpsgenieApiKey: {{ "%7B%7Bresolve:secretsmanager:MyOpsGenieSecret:SecretString:key%7D%7D" | url_decode }}
      OpsgenieApiEndpoint: !Ref OpsgenieApiEndpoint
      Username: 'opsgenie-user-one1@sample.com'
      FullName: 'user one1'
      Role: User
```
Unfortunately, secure string parameters can only be used for a [few supported properties](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html#template-parameters-dynamic-patterns-resources) and are not available in resource providers. 


#### Integration in Organizations
Naturally, the integration with Organizations comes at a later point in time. I'd like to see that I'm able to roll out a resource provider to my whole organization or a particular organization unit (OU). Once, CloudFormation support is available it should be easy to roll them out with StackSets. 

Another idea is to have secrets managed in a central place and allow only the execution role to decrypt it. From an IAM perspective, this should be possible but as far as I've seen it's not supported by CloudFormation's dynamic references to use secrets from another account. Let me know if I'm wrong.

#### Missing registry and trust
When a resource provider is registered, CloudFormation does not validate the type name as long as it is syntactically correct. That means I can register my own implementation of `Atlassian::Opsgenie::User` and it's not obvious that it's not the official implementation from Atlassian. Also, in each account, there could be a different implementation or version. 

In comparison to official resources from AWS, I can't trust custom ones. Why are type names (or at least parts of them) not unique and reserved for the vendor (like on GitHub, npm, ...)?

There's also no place to look up which vendors provide which types. A central registry or website where available resource providers are listed would be very useful. 

#### And what about CDK?
You might know that I'm a huge fan of [CDK](https://garbe.io/category/cdk/), so how does it work with CDK?

At the moment, there's no special treatment in CDK for types from resource providers. You can use the [CustomResource](https://docs.aws.amazon.com/cdk/api/latest/docs/custom-resources-readme.html) construct but there's no type safety. 

Over time I expect that it's possible to generate level 1 constructs out of their resource specification and have L2/L3 constructs based on them.

#### Community
There's not much to say as there's no community or I didn't find them. The repositories are all publicly available on GitHub and I got quick answers to my questions but that's it. I'm looking forward to seeing more blog posts, talks at conferences, and examples from developer advocates and others...


## Summary
CloudFormation resource provider is still in an early stage. I don't think that they are used much at the moment, at least not in production. To get better adoption vendors have to provide working examples, proper documentation, useful error messages, and versioned packages. But they will propably only act if they see some demand from engineers. A chicken and egg situation. 

AWS is the party in the middle. But they're also in charge. Please, provide a global registry to be able to find available resource providers, avoid confusion about type names (not everyone should be able to register every name), and finally support CloudFormation.

> "It does not exist until it's supported by CloudFormation." _(author unknonw)_

Thanks :) 
