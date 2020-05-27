---
layout: post
title: "Hey CDK, How can I use tags in my custom constructs?"
date: 2020-01-21 07:00:00 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: "AWS, CDK, CloudFormation, AWS CDK, CloudFormation to CDK, CDK GetAtt, CDK Inheritance, CDK Aspects, CDK Composition, Tags, Tagging"
description: "Hey CDK, How can I use Tags in my custom Constructs?"
cover: /assets/heycdk.png
---

Tags are quite useful to assign metadata to your resources like service name, owning team, or criticality (Strategies are explained [here](https://aws.amazon.com/answers/account-management/aws-tagging-strategies/)). More and more AWS services are supporting them and also CloudFormation support is [becoming better](https://github.com/aws-cloudformation/aws-cloudformation-coverage-roadmap/issues/228). Another use case is [attribute-based access permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_tags.html).

In CDK tags are a special kind of aspects which I explained in my previous [blog post](/blog/2019/10/01/hey-cdk-how-to-write-less-code/). They are defined once and applied to every resource that supports tagging. But this blog post covers are more advanced use cases. How can you use tags in your constructs?


This is the fourth part of a series 'Hey CDK'
- [How can I migrate my existing CloudFormation templates into CDK?](/blog/2019/09/11/hey-cdk-how-to-migrate/)
- [How can I reference existing resources?](/blog/2019/09/20/hey-cdk-how-to-use-existing-resources/)
- [How can I write even less code?](/blog/2019/10/01/hey-cdk-how-to-write-less-code/)
- How can I use tags in my custom constructs?
- [How can I secure my Fargate Service with ALB authentication?](/blog/2020/05/27/hey-cdk-how-to-oidc-alb-fargate/)


### How can I use tags in my custom constructs?
My use case is to write a CDK construct which adds a [DataDog sidecar](https://www.datadoghq.com/blog/monitor-aws-fargate/) to an existing TaskDefinition. Basically, it just adds another [Container Definition](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-ecs.ContainerDefinition.html) as a dependency to an existing application container definition. This container runs the [DataDog Agent](https://github.com/DataDog/datadog-agent) which is used by the application to send metrics and logs to DataDog. One of the supported environment variables is DD_DOGSTATSD_TAGS which expects a list of tags that are attached to the metrics. As value, I want to re-use existing tags of the stack. 

To support tags a construct has to implement the `ITaggable` interface. It enforces you to create a property called `tags` of type `TagManager`. The TagManager itself is called during the prepare phase.

Now comes the twist: Normally, you'd create all your resources within the constructor of your construct. But in this case, it's not possible because `tags` is not filled at this time. It happens later when all constructs have been created. Also, the `ContainerDefinition` construct can't be changed after it's created. That's why we have to postpone the creation of the sidecar until the `prepare` phase. CDK calls [prepare()](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_core.Construct.html#protected-prepare) method and here we can now create our resources and use the tags.

The code looks like this:

```typescript
export class DatadogFargateIntegration extends cdk.Construct implements cdk.ITaggable {

  tags: cdk.TagManager; 
  taskDefinition: ecs.TaskDefinition;

  constructor(scope: cdk.Construct, id: string, taskDefinition: ecs.TaskDefinition) {
    super(scope, id);

    this.tags = new cdk.TagManager(cdk.TagType.KEY_VALUE, 'DatadogFargateIntegration');
    this.taskDefinition = taskDefinition;
  }

  prepare() {

    // Format the tags as expected by DataDog
    let datadogTags = this.tags.renderTags().map((tag: any)  => {
      return '"' + tag.Key + ":" + tag.Value + '"'
    });

    let env = {
      DD_DOGSTATSD_TAGS: "[" + datadogTags.toString() + "]",
      // other varaibles
    };

    const datadog = this.taskDefinition.addContainer('dd-agent', {
      image: ecs.ContainerImage.fromRegistry('datadog/docker-dd-agent'),
      cpu: 64,
      memoryLimitMiB: 128,
      environment: env,
      essential: false,
    });
  }
}

```

In the end, it renders our Container Definition which contains an environment variable DD_DOGSTATSD_TAGS containing all the tags that have been defined. 

```yaml

Resources:
  # ...
  task117DF50A:
    Type: AWS::ECS::TaskDefinition
    Properties:
      ContainerDefinitions:
        - Cpu: 64
          Environment:
            - Name: DD_DOGSTATSD_TAGS
              Value: '["Service:DataDogSidecar"]'
          Essential: false
          Image: datadog/docker-dd-agent
          Memory: 128
          Name: dd-agent
        # app container definition...
      Family: AppStacktask41611C1A
      NetworkMode: bridge
      Tags:
        - Key: Service
          Value: DataDogSidecar

```

You can find a working example in my GitHub repo [cdk-datadog](https://github.com/pgarbe/cdk-datadog).


> __Play with CDK__  
> If you want to play around, have a look at [play-with-cdk.com](https://play-with-cdk.com). It allows you to write constructs and render them as CloudFormation Yaml (with some restrictions).  
>   
> This example is available here: [https://play-with-cdk.com/?s=ed63e7f28e4101404cd917241159b8d5](https://play-with-cdk.com/?s=ed63e7f28e4101404cd917241159b8d5)


### Answer
With CDK you can use tags in your custom constructs by implementing `ITaggable`. The appending of the tags to your resources has to be done in the `prepare()` method. You might need to create your resources within that method as well.
