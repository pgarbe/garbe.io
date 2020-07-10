---
layout: post
title: "ECS Deployment Options - From rolling updates to blue green and canary"
date: 2020-07-14 07:00:00 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS, ECS]
keywords: "AWS, ECS, Fargate, Blue Green, Canary, Deployment"
description: "ECS Deployment Options - From rolling updates to blue green and canary"
---

"How often do you deploy to production?" - This is an important question as the best application is useless if you can't deploy it. And being able to deploy regularly and automated is [quite important](TODO Accelerate). 
The Elastic Container Service (ECS) is used by many AWS customers and offers different ways to deploy your containers. In this article, I'll explain the different deployment options of ECS, show how they work, and when to use them.

The different options can be set in the [DeploymentController](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_DeploymentController.html) property of an ECS service. Let's have a look:

### ECS
The first option is _ECS_ itself. The ECS service scheduler is responsible for performing a rolling update of newer versions. Current version of running tasks are replaced by newer versions. The optional [deploymentConfiguration](TODO) parameter defines how many tasks should run during the deployment. _minimumHealthyPercent_ indicates the percentage of tasks that must remain in RUNNING status and _maximumHealthyPercent_ parameter represents an upper limit on the number of your service’s tasks that are allowed in the RUNNING or PENDING state during a deployment. Both values define a percentage of the current desiredCount.

The configuration is easy and straight forward but also limits the possibilities. It's not possible to cancel or rollbck a deployment. And due the limited configuration it does not allow to do a blue/green or canary deployment. 

Another issue comes in combination with CloudFormation (see [Deep dive on load balanced ECS Service deployments with CloudFormation](TODO) for more details):
  * A CloudFormation stack creation will always complete successfully even with failing health checks and independent of any deploymentConfiguration
  * A CloudFormation stack update will fail only if minimumHealthyPercent is 100%, and the container health check is unhealthy (no other combination). And it takes CloudFormation 3 hours before it triggers the rollback.
  * CloudFormation rollback means that it triggers a new ECS deployment with the former taskdefinition (which could lead to some troubles as well)


### Code Deploy
The next option is *CODE_DEPLOY* where the deployment of an ECS service is orchestrated by CodeDeploy. During a deployment it creates tasks of the new version in parallel to existing tasks and then shifts the traffic over. The traffic can be [shifted](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html#deployment-configuration-ecs) _all-at-once_, _linear_ or as _canary_ (small percentage at the beginning and then the rest). 

[Hooks](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html#appspec-hooks-ecs) (basically lambda functions) that are executed before and after the traffic is shifted can ensure that the application works as expected. Failures in the hooks leads to automated rollbacks. In addition, CloudWatch alarms can be set that are watched during the deployment and if they go on a rollback is triggered. The deployment is also "baked" for some time to allow a fast rollback in case something wrent wrong after the new version has been rolled out completely.

> See also [Clare Liguori](https://twitter.com/clare_liguori)'s twitter [thread](https://twitter.com/clare_liguori/status/1225566939306590208) where she explains it in detail. 

In order to set up ECS deployments with CodeDeploy a certain order must be followed:
1. Create your infrastructure (like iam roles, security groups and load balancer
2. Set up an ECS Task Definition and Service (and optional a Cluster)
3. Create a CodeDeploy Application and DeploymentGroup (referencing ECS service and cluster and two target groups)
4. Set up a CodePipeline which builds the image, creates a taskdefinition.json and appsepc.yaml and calls CodeDeploy

Unfortunately, not everything can be configured in CloudFormation, as DeploymentGroup does not support ECS blue/green deployments. 

Another issue is that the CodeDeploy action needs not only an _appspec.yaml_ but also a _taskdefinition.json_ that contains our task definition (Although, AWS states in their docs that TaskDefinition is optional for an ECS Service, I couldn’t deploy a service without a referenced task definition). With those dependencies it’s not possible to create a pipeline first and deploy your application afterwards. Actually, it must be the other way around. First create your infrastructure and then your pipeline. Further updates to your task definition are deployed with CodeDeploy and not CloudFormation.

> One advantage of CDK is that it can fill the gaps of CloudFormation and abstract complicated things away. Although, blue/green or canary deployments for ECS in CodeDeploy are not part of the official CDK library, there is a community project ([cloudcomponents/cdk-constructs](https://github.com/cloudcomponents/cdk-constructs/tree/master/packages/cdk-blue-green-container-deployment) which does the heavy lifting for you.


### External Deployments
The last option is _EXTERNAL_. External deployments allow anyone to perform deployments by using TaskSets. A TaskSet is a set of ECS tasks that are part of a deployment. They are also used internally by CodeDeploy. 

There is an example for [Jenkins](https://aws.amazon.com/blogs/containers/blue-green-deployments-with-the-ecs-external-deployment-controller/) that performs basically the following steps: 

1. Create green task set with new task definition
2. Shift test traffic to green task set
3. Shift an increment of prod traffic to green task set
4. Shift all prod traffic to green task set
5. Remove blue task set with old task definition


But also CloudFormation [supports blue/green deployments for Amazon ECS](https://aws.amazon.com/about-aws/whats-new/2020/05/aws-cloudformation-now-supports-blue-green-deployments-for-amazon-ecs/) which is good as you can define your whole infrastructure as code. But how is it different to CodeDeploy and how does that work? 

CloudFormation also uses the external deployment option. In order to perform the steps mentioned above, CloudFormation introduced a new transformation ("AWS::CodeDeployBlueGreen") and a new “hooks” section (and a "AWS::CodeDeploy::BlueGreen" hook). Some resources needs to be created for blue and green in CloudFormation (e.g. load balancer target group) and for some resources only the “blue” part needs to be created (e.g. taskdefinition or taskset). During a stack creation or update, CloudFormation does multiple transformations of the template to perform the steps above.

> I can't link to any docs as they don't exist yet. Just have a look at the [example](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/blue-green.html). Also, look at [Clare Liguori](https://twitter.com/clare_liguori)'s twitter [thread](https://twitter.com/clare_liguori/status/1265645392038776834) which explains it in more detail.


But the CloudFormation solution comes with a lot of [limitations](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/blue-green.html#blue-green-considerations). Just a few:

* SSM parameters can't be resolved
* Importing existing resources is not supported
* Doesn't work in combination with nested stacks
* No visibility or management of in-progress deployments like in CodeDeploy.


### Conclusion
The one who has the choice is in agony. Every option has it's advantages but also disadvantes. What you should use depends heavily on your use-case. I hope I could shed some light on this with this article and help you with your decision.

| __Criteria__ 	              | __ECS__ | __CodeDeploy__ | __External (CloudFormation)__
| _Simple configuration_      | ✅      | ⚠️              | ❌
| _CloudFormation support_ 	  | ✅      | ❌             | ⚠️
| _Blue/Green deployments_ 	  | ❌      | ✅             | ✅
| _Canary deployments_        | ❌      | ✅             | ✅
| _Automatic rollback_        | ❌      | ✅             | ⚠️
| _Transparency_              | ✅      | ✅             | ❌

Nevertheless, I still feel that this is not what customers want. Too many cooks spoil the broth: ECS, CodeDeploy and CloudFormation offer you different ways to deploy an ECS service. But what I'd like to have is the simplicity of ECS rolling updates, the powerful configuration of CodeDeploy and the infrastructure as code support of the CloudFormation solution. 

PS: And a nice integration in CDK :) 
