---
layout: post
title: "7 Awesome CloudFormation Hacks"
date: 2017-07-17 09:30:29 +0200
author: Philipp Garbe
comments: true
published: true
# categories: [AWS]
keywords: "AWS, CloudFormation"
description: "CloudFormation hacks beyond the typical hello world"
---

Since AWS introduced native YAML support, CloudFormation templates are much more readable than before. Also, the introduced [intrinsic functions](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html) help a lot to build awesome templates. But sometimes the solution to a problem is not that obvious and often not well documented. This blog post should remind me to some of my hacks, so that I can google them later on. Maybe it's helpful for you as well.


## Hack I: Combine two sequent intrinsic functions

__Problem:__ Due a restriction in YAML, it's not possible to use the shortcut syntax for two sequent intrinsic functions. This is useful when a value should be imported and the variable name should be subsituted with the stack name.


```yaml
Parameters:
  ClusterStack: 
    Type: String

Resources:
  Service:
    Type: AWS::ECS::Service
    Properties:
      Cluster: !ImportValue !Sub '${ClusterStack}-ClusterName'
```

__Solution:__ The solution is to use a combination of the standard and the tag syntax. The standard syntax has to be used as first and needs to be written in a new line.


```yaml
Parameters:
  ClusterStack: 
    Type: String

Resources:
  Service:
    Type: AWS::ECS::Service
    Properties:
      Cluster:
        Fn::ImportValue: !Sub '${ClusterStack}-ClusterName'
```


## Hack II: Use exported values from other stacks in !Sub

__Problem:__   
The `!Sub` function allows to replace a variable inside a string with the value of a stack parameter. But how can exports from other stacks be used?

__Solution:__  
Not only a CloudFormation parameter can be used in a `!Sub` function but also a custom parameter, which has to be defined as second argument to the `!Sub` function. The value can be hardcoded or another intrinsic function like `!ImportValue`.

```yaml
Pipeline:
  Type: AWS::CodePipeline::Pipeline
  DependsOn: CloudFormationExecutionRole
  Properties:
    Stages:
    - Name: DeployPipeline
      Actions:
      - Name: DeployPipelineAction
        Configuration:
          TemplatePath: 'Source::templates/pipeline.yaml'
          ParameterOverrides: !Sub 
              - |
                {
                  "VpcId": "${VpcId}"
                }
              - VpcId: 
                  'Fn::ImportValue': !Sub '${VpcStack}-VpcId'
```

Also multiple parameters can be defined:

```yaml
Pipeline:
  Type: AWS::CodePipeline::Pipeline
  DependsOn: CloudFormationExecutionRole
  Properties:
    Stages:
    - Name: DeployPipeline
      Actions:
      - Name: DeployPipelineAction
        Configuration:
          TemplatePath: 'Source::templates/pipeline.yaml'
          ParameterOverrides: !Sub 
              - |
                {
                  "VpcId": "${VpcId}",
                  "Subnets": "${PrivateSubnets}"
                }
              - |
                { 
                  PrivateSubnets: !ImportValue vpc-stack-PrivateSubnetIds, 
                  VpcId: !ImportValue vpc-stack-id 
                }
```

## Hack III: Changes in cfn-init don't trigger redeployment in AutoScaling Group

__Problem:__   
I often use the [cfn-init helper function](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-init.html) instead of scripting all the things in [UserData](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html#cfn-ec2-instance-userdata). Therefore my UserData is very small and normally doesn't change. Unfortunately, changes at the cfn-init configuration are not detected by the AutoScaling Group and does not trigger a replacement of the existing ec2 instances. In worst case, the CloudFormation update is successful but due a bug in the cfn-init script the next ec2 instance that gets started uses the new launch configuration and fails. 

__Solution:__  
To detect bugs in cfn-init during the deployment, the UserData script needs to be changed. The easiest way is to add the current build number. With every build the UserData gets changed and forces the AutoScaling Group to replace all ec2 instances with the newer launch configuration.

```yaml
Parameter:
  BuildNumber:
    Type: String

Resources:
  LaunchConfig:
    Type: AWS::AutoScaling::LaunchConfiguration
    Metadata:
      AWS::CloudFormation::Init:
        configSets:
          ...

    Properties:
      UserData: 
        "Fn::Base64": !Sub |      
          #!/bin/bash
          # This is needed for cfn-init to reinitialize the instances with the new version on updates
          BUILD_NUMBER="${BuildNumber}"

          /opt/aws/bin/cfn-init -v \
              --stack ${AWS::StackName} \
              --resource LaunchConfig \
              --region ${AWS::Region}

          /opt/aws/bin/cfn-signal -e $? \
              --stack ${AWS::StackName} \
              --region ${AWS::Region} \
              --resource AutoScalingGroup
```


## Hack IV: Get Stack name of sibling stack in nested stacks

__Problem:__  
The problem happens when you create nested stacks and one stack needs the stack name of a sibling stack as parameter. Because the name of the stack is generated you don't know that in advance. Unfortunately, `!Ref` returns only the complete ARN (like arn:aws:cloudformation:us-east-1:123456789012:stack/mystack-mynestedstack-sggfrhxhum7w/f449b250-b969-11e0-a185-5081d0136786) and not the stack name (like mystack-mynestedstack-sggfrhxhum7w)

> You can find many production-ready cloudformation templates at [https://github.com/widdix/aws-cf-templates](https://github.com/widdix/aws-cf-templates)

__Solution:__ 
If you are in control of the stack templates you can return the actual StackName as output parameter.

```yaml
# VPC-Stack:
Outputs:
  StackName:
    Value: !Ref AWS::StackName
```

Inside your parent stack you can now reference that output parameter:
```
# Parent Stack
  Vpc:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-3azs.yaml

  Bastion:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-ssh-bastion.yaml
      Parameters:
        ParentVPCStack: !GetAtt Vpc.Outputs.StackName
```

If you can't change the template, the stack name can be extracted from ARN with `!Select` and `!Split`.

```yaml
  Vpc:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-3azs.yaml

  Bastion:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/widdix-aws-cf-templates/vpc/vpc-ssh-bastion.yaml
      Parameters:
        ParentVPCStack: !Select [1, !Split ['/', !Ref Vpc]] # Workaround to get the stack name
```


## Hack V: AccountIds with leading zero

__Problem:__  If you want to create a resource based policy like [Lambda Permission](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-permission.html) the following error can bother you: `The provided principal was invalid. Please check the principal and try again.` 

__Solution:__ When you specifiy only the AccountId it's treated as an integer internally at AWS. Therefore the leading zero gets truncated. As Workaround a full ARN has to be provided.

```yaml
  # AccountIds with leading zero need a special handling
  LambdaInvokePermissionWithLeadingZero:
    Type: "AWS::Lambda::Permission"
    Properties:
      FunctionName: !Ref TriggerLambdaArn
      Action: "lambda:InvokeFunction"
      Principal: arn:aws:iam::012345678912:root

  # AccountIds without leading zero can be used directly
  LambdaInvokePermissionWithoutLeadingZero:
    Type: "AWS::Lambda::Permission"
    Properties:
      FunctionName: !Ref TriggerLambdaArn
      Action: "lambda:InvokeFunction"
      Principal: 123456789123 
```


## Hack VI: Use Dictionaries as Stack Parameter

__Problem:__  
Unfortunately, there is no support to define the type of CloudFormation parameters as key-value pairs or dictionaries. This could be useful for properties like `tags` or `environment`. 

__Solution:__ 
`!If` functions can be used to return not only a single value but a whole block. The idea is to create an optional stack parameter and a condition for each key-value pair. That implies that the length of the list needs to be restricted.

The parameters can be used in a nested `!If` construct which I can't really describe, so have a look at the code (and don't mess up the indention!).

```yaml
Parameters:
  Env1:
    Type: String
    Description: An item of possible environment variables
    Default: ''

  Env2:
    Type: String
    Description: An item of possible environment variables
    Default: ''

  Env3:
    Type: String
    Description: An item of possible environment variables
    Default: ''


Conditions:
  Env1Exist: !Not [ !Equals [!Ref Env1, '']]
  Env2Exist: !Not [ !Equals [!Ref Env2, '']]
  Env3Exist: !Not [ !Equals [!Ref Env3, '']]


  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      TaskRoleArn: !Ref TaskRole
      ContainerDefinitions:
      - Name: !Ref AWS::StackName
        Image: !Ref ImageName

        Environment:
          'Fn::If':
            - Env1Exist
            -
              - Name: !Select [0, !Split ["|", !Ref Env1]]
                Value: !Select [1, !Split ["|", !Ref Env1]]
              - 'Fn::If':
                - Env2Exist
                -
                  Name: !Select [0, !Split ["|", !Ref Env2]]
                  Value: !Select [1, !Split ["|", !Ref Env2]]
                - !Ref "AWS::NoValue"
              - 'Fn::If':
                - Env3Exist
                -
                  Name: !Select [0, !Split ["|", !Ref Env3]]
                  Value: !Select [1, !Split ["|", !Ref Env3]]
                - !Ref "AWS::NoValue"

            - !Ref "AWS::NoValue"

```

## Hack VII: DependsOn with condition 
__Problem:__  
It's not possible to add a `!If` function to the [DependsOn](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-dependson.html) attribute. This could be useful if the dependent resource itself has a `Condition`.

In this example the ECS Service has a dependency to the ALBListenerRule. As the ALBListenerRule has a Condition, also the DependsOn has to have that condition which is Unfortunately not supported.

```yaml
  Application:
    Type: AWS::ECS::Service
    DependsOn: !If [HasAlb, WaitCondition, !Ref AWS::NoValue] # Not supported!

  ALBListenerRule:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Condition: HasAlb
    Properties:
      ...

```

__Solution:__ 
The dependency between ECS Service and the ALBListenerRule can be detoured to a WaitCondition. The WaitCondition can now implement the `HasALB` condition at the [Handle](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-waitcondition.html#cfn-waitcondition-handle) attribute and use either `AlbWaitHandle` which depends on the ALBListenerRule or the `WaitHandle` which has no further dependencies.

```yaml
Parameters:
  ContainerPort:
    Type: String
    Default: ''
    Description: HTTP Port of the container

Conditions:
  HasAlb: !Not [ !Equals [ !Ref ContainerPort, ""]]

Resources:
  AlbWaitHandle: 
    Condition: HasAlb
    DependsOn: ALBListenerRule
    Type: "AWS::CloudFormation::WaitConditionHandle"

  WaitHandle: 
    Type: "AWS::CloudFormation::WaitConditionHandle"

  WaitCondition: 
    Type: "AWS::CloudFormation::WaitCondition"
    Properties: 
      Handle: !If [HasAlb, !Ref AlbWaitHandle, !Ref WaitHandle]
      Timeout: "1"
      Count: 0

  Application:
    Type: AWS::ECS::Service
    DependsOn: WaitCondition

  ALBListenerRule:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Condition: HasAlb
    Properties:
      ...

```
