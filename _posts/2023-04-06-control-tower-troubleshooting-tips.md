---
layout: post
title: 'AWS ControlTower Troubleshooting Tips'
date: 2023-04-11 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS]
keywords: 'AWS, ControlTower, SCP, Service Control Policies, StackSet, Service Catalog'
description: 'AWS ControlTower Troubleshooting Tips'
---


AWS ControlTower is a powerful tool for centrally managing multiple AWS accounts. However, troubleshooting ControlTower issues can be challenging due to the complex architecture it relies on. This article provides some tips for resolving common ControlTower problems.

Firstly, it's important to stay calm if something goes wrong. ControlTower errors usually do not affect production systems, and resources and settings are typically still deployed in member accounts despite any errors.

> This article is not the ultimate source of truth, but reflects my current understanding. I'll update it accordingly.


### What you need to understand
To understand what ControlTower is, check out the official [product page](https://aws.amazon.com/controltower/features/). 

Essentially, ControlTower is built on top of CloudFormation Stacks, StackSets, and ServiceCatalog Products. The Account Factory in ControlTower is just an interface for ServiceCatalog. When the ServiceCatalog Product "AWS ControlTower Account Factory" is created, a member account is launched, and the account is added to the StackSets deployed by ControlTower.

![ControlTower and dependend services](/assets/control-tower.png)
(Note, that this diagram focus only on a part of ControlTower)

### General Troubleshooting
When troubleshooting ControlTower issues, look for more detailed error messages in the following places:

- **Service Catalog Product**  
Check the Provisioned Products for error details.  
- **StackSets**  
Check the status of StackSet stack instances for any problems. This helps also to identify which account makes the trouble.  
- **AWS (Member) Account**  
For issues with specific accounts, check CloudFormation Stacks within the account (usually starting with "StackSet-AWSControlTowerBP-BASELINE"). Change the filter also to *Deleted* to see if the creation of the Stack has failed.  

> Changes to an organizational unit (OU), like re-registering, will affect all accounts and if one fails all accounts will be shown as tainted.


### Protect CloudFormation Stacks
Although ControlTower protects its resources by Service Control Policies (SCP), it does not protect CloudFormation stacks. You can delete them, but as the resources can't be deleted it ends up in a *DELETE_FAILED* state which makes it impossible for ControlTower to fix it.

To protect CloudFormation Stacks add the following SCP to your OUs:

```json
{
    "sid": "DenyModifyControlTowerStacks",
    "effect": "DENY",
    "actions": ["cloudformation:Delete*", "cloudformation:Update*", "cloudformation:Create*"],
    "resources": ["arn:aws:cloudformation:*:*:stack/StackSet-AWSControlTower*"],
    "conditions": {
        "ArnNotLike": {
            "aws:PrincipalARN": "arn:aws:iam::*:role/AWSControlTowerExecution",
        }
    }
}
```

### Fix Tainted Accounts
If the account enrollment failed, you might need to clean up the resources of the StackSet stacks manually to be able to delete the stack completely and let ControlTower re-create it. As the resources are protected you need to assume the *AWSControlTowerExecution* role from a role (or user) in the main account. Use the following command to assume the role:

```sh
# To assume the AWSControlTowerExecution role you need to log in to your main account first
export $(printf "AWS_ACCESS_KEY_ID=%s AWS_SECRET_ACCESS_KEY=%s AWS_SESSION_TOKEN=%s" \
    $(aws sts assume-role \
        --role-arn arn:aws:iam::123456789012:role/AWSControlTowerExecution \
        --role-session-name MySessionName \
        --query "Credentials.[AccessKeyId,SecretAccessKey,SessionToken]" \
        --output text))
```


Here are a few scripts that can help:

#### StackSet-AWSControlTowerBP-BASELINE-ROLES

```sh
aws iam delete-role --role-name aws-controltower-AdministratorExecutionRole
aws iam delete-role --role-name aws-controltower-ReadOnlyExecutionRole
```

#### StackSet-AWSControlTowerBP-BASELINE-SERVICE-ROLES

```sh
aws iam delete-role --role-name aws-controltower-ConfigRecorderRole
aws iam delete-role --role-name aws-controltower-ForwardSnsNotificationRole
```

#### StackSet-AWSControlTowerBP-BASELINE-CONFIG

```sh
aws configservice delete-configuration-recorder --configuration-recorder-name aws-controltower-BaselineConfigRecorder
aws configservice delete-delivery-channel --delivery-channel-name aws-controltower-BaselineConfigDeliveryChannel
```

#### StackSet-AWSControlTowerBP-BASELINE-CLOUDWATCH

```sh
aws events remove-targets --rule aws-controltower-ConfigComplianceChangeEventRule --ids Compliance-Change-Topic
aws events delete-rule --name aws-controltower-ConfigComplianceChangeEventRule

aws sns delete-topic --topic-arn arn:aws:sns:eu-central-1:123456789012:aws-controltower-SecurityNotifications

aws lambda delete-function --function-name aws-controltower-NotificationForwarder
aws logs delete-log-group --log-group-name "/aws/lambda/aws-controltower-NotificationForwarder"
```



<!-- 
Error: Account enrollment failed.

Error
AWS ControlTower failed to set up your landing zone completely: The AWS Config aggregation authorization for audit account 123456789012 and home region eu-central-1 was not found under the target account 123456789012 and target region eu-central-1. It may have been deleted outside of AWS ControlTower. To continue, delete the AWSControlTowerBP-BASELINE-CONFIG stack set instance for target account 123456789012 and target region eu-central-1. Then, try again. Learn more

Account enrollment failed.
AWS ControlTower could not enroll your account for the following reason: AWS STS isn't activated in some regions for the following accounts: 513899454231. Ask your account administrator to activate AWS STS from the IAM console in each account for the following regions: eu-west-1, eu-central-1, us-east-1. After STS is enabled in those regions, delete the failed stack instances in those accounts: null.

Error: Registration failed for <OU>
AWS ControlTower detects issues that prevent registering this organizational unit. Try again, or contact AWS Support. 
-->
