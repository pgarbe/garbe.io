---
layout: post
title: 'Use Slack to handle AWS Support Cases'
date: 2023-12-14 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: 'AWS, CDK, AWS CDK, Slack, Support'
description: 'Use Slack to handle AWS Support Cases'
---

In this blog post, I aim to shed light on a relatively unknown feature of AWS: Slack integration for AWS Support.

> AWS provides different levels of support. Read more on their [Premium Support site](https://aws.amazon.com/premiumsupport/).

Typically, users resort to the AWS Console (Web) to open and respond to Support Cases. However, this approach has its drawbacks:

* Frustration with expired credentials leading to the loss of information before submission
* Inconvenient email notifications lacking readability and context
* Manual navigation to support cases due to the absence of deep links
* Chats in a separate browser window make messages easy to overlook, with limited sharing capability within a team

For those accustomed to using Slack for communication you'll immediately recognize the benefits:

* Each support case becomes its own Slack thread
* Follow other cases using Slack's "get notified about new messages" feature
* Create private channels for AWS support chats, facilitating team collaboration
* Leverage Slack notifications to stay informed about new responses

## Slack Support for AWS Support 
The setup process is surprisingly straightforward.

> Thanks to my colleague [Alina Domanov](https://www.linkedin.com/in/alina-domanov/) as she has done most of the work and implemented it for our teams at [Personio](https://www.personio.com/about-personio/careers/product-engineering/).


![AWS Slack App for Support in Management Account](/assets/aws-slack-support-management.png)

In the management account, create a Slack [SlackWorkspaceConfiguration](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-supportapp-slackworkspaceconfiguration.html) and [register it](https://docs.aws.amazon.com/supportapp/latest/APIReference/API_RegisterSlackWorkspaceForOrganization.html) for the entire organization. Additionally, add the [AWS Support App](https://slack.com/apps/A010CU512EQ-aws-support) to your Slack Workspace. This enables any member account to replicate the Slack Workspace without extensive configuration.

![AWS Slack App for Support in Member Account](/assets/aws-slack-support-member.png)

In any member account within your organization, create a SlackWorkspaceConfiguration and a [SlackChannelConfiguration](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-supportapp-slackchannelconfiguration.html). It's also beneficial to create an [AccountAlias](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-supportapp-accountalias.html) for a human-readable name instead of the Account Id.

Within Slack use `/awssuport create-case` command. It will open a pop-up window that guides you through the whole process. 

![Command to create new AWS Support Case](/assets/aws-slack-support-command.png)

![Wizard that guides you through the creation process](/assets/aws-slack-support-wizard.png)


## Example with CDK
Of course, you don't want to click around but set up everything with Infrastructure as Code (IaC). I hope this example with CDK helps you to get started.

### Organization Setup
The first stack is meant for the management account. It creates the SlackWorkspaceConfiguration and calls the *registerSlackWorkspaceForOrganization* API as there is no CloudFormation support there. A de-register on delete is not needed as it will be done as part of the deletion of the SlackWorkspaceConfiguration.

```ts
import { aws_iam, aws_supportapp as supportapp, custom_resources as cr, Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';


export interface AwsSupportAppSlackRegistrationProps extends StackProps {
    /**
     * The Slack Workspace Id
     */
    readonly teamId: string;
}


export class AwsSupportAppSlackRegistrationStack extends Stack {
    constructor(scope: Construct, id: string, props: AwsSupportAppSlackRegistrationProps) {
        super(scope, id, props);

        // specify your Slack workspace configuration
        const config = new supportapp.CfnSlackWorkspaceConfiguration(this, 'MyCfnSlackWorkspaceConfiguration', {
            teamId: props.teamId,
        });

        // Custom resource for RegisterSlackWorkspaceForOrganization
        const registerSlackWorkspace = new cr.AwsCustomResource(this, 'RegisterSlackWorkspace', {
            onCreate: {
                service: 'SupportApp',
                action: 'registerSlackWorkspaceForOrganization',
                parameters: {
                    teamId: config.ref,
                },
                physicalResourceId: cr.PhysicalResourceId.of(Date.now().toString()),
            },
            policy: cr.AwsCustomResourcePolicy.fromStatements([
                new aws_iam.PolicyStatement({
                    effect: aws_iam.Effect.ALLOW,
                    actions: ['supportapp:RegisterSlackWorkspaceForOrganization'],
                    resources: ['*'],
                }),
            ]),
        });

        // Add a dependency on the Custom Resource for Slack workspace registration
        registerSlackWorkspace.node.addDependency(config);
    }
}

```

The next Stack is for the member accounts. It comes with two parameters: The *channelId* to define in which Slack Channel the Bot should be added and *alias* to give the accounts a human readable name (otherwise it will show only the account ids).

> Be aware that the Account Alias is limited in size and supports only 30 characters!


```ts
export interface AwsSupportAppSlackIntegrationProps {
    /**
     * Slack Channel Id.
     */
    readonly channelId: string;
    /**
     * Human readable account alias (max 30 characters)
     */
    readonly alias: string;
}

export class AwsSupportAppSlackIntegrationStack extends Stack {
    constructor(scope: Construct, id: string, props: AwsSupportAppSlackIntegrationProps) {
        super(scope, id, props);

        // Create IAM role for the channelRoleArn with required permissions
        const channelRole = new aws_iam.Role(this, 'ChannelRole', {
            assumedBy: new aws_iam.ServicePrincipal('supportapp.amazonaws.com'),
        });

        // Attach policies for the required permissions
        channelRole.addToPolicy(
            new aws_iam.PolicyStatement({
                effect: aws_iam.Effect.ALLOW,
                actions: [
                    'servicequotas:GetRequestedServiceQuotaChange',
                    'servicequotas:GetServiceQuota',
                    'servicequotas:RequestServiceQuotaIncrease',
                    'support:AddAttachmentsToSet',
                    'support:AddCommunicationToCase',
                    'support:CreateCase',
                    'support:DescribeCases',
                    'support:DescribeCommunications',
                    'support:DescribeSeverityLevels',
                    'support:InitiateChatForCase',
                    'support:ResolveCase',
                ],
                resources: ['*'],
            }),
        );
        channelRole.addToPolicy(
            new aws_iam.PolicyStatement({
                effect: aws_iam.Effect.ALLOW,
                actions: ['iam:CreateServiceLinkedRole'],
                resources: ['*'],
                conditions: {
                    StringEquals: {
                        'iam:AWSServiceName': 'servicequotas.amazonaws.com',
                    },
                },
            }),
        );

        // Enable Slack integration for all organizations
        const slackChannelConfig = new supportapp.CfnSlackChannelConfiguration(this, 'SlackChannelConfiguration', {
            channelId: slackChannel.valueAsString,
            channelRoleArn: channelRole.roleArn,
            notifyOnCaseSeverity: 'all',
            teamId: props.teamId,
            notifyOnAddCorrespondenceToCase: true,
            notifyOnCreateOrReopenCase: true,
            notifyOnResolveCase: true,
        });

        new supportapp.CfnAccountAlias(this, 'CfnAccountAlias', {
            accountAlias: props.alias,
        });

        // specify your Slack workspace configuration
        const workspaceConfig = new supportapp.CfnSlackWorkspaceConfiguration(this, 'SlackWorkspaceConfiguration', {
            teamId: props.teamId,
        });

        slackChannelConfig.node.addDependency(workspaceConfig);
    }
}
```

This is a good candidate to roll out [with StackSets](/blog/2023/10/10/deep-dive-on-stacksets-with-cdk). I skipped that as the parameters (channelId and alias) need to be customized for each account and it depends on the actual use-case how to do that.


## Conclusion
In conclusion, integrating Slack with AWS Support offers a streamlined approach to handling support cases. By leveraging Slack's familiar interface, users can enhance collaboration, receive timely notifications, and efficiently manage their AWS support interactions. The seamless setup outlined in this post, coupled with AWS CDK examples, empowers organizations to harness the power of Slack in optimizing their AWS support workflows. 


