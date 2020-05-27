---
layout: post
title: "Hey CDK, how can I secure my Fargate Service with ALB authentication?"
date: 2020-05-27 19:00:00 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: "AWS, CDK, CloudFormation, AWS CDK, ALB, OIDC, Load Balancer, Authentication, Fargate"
description: "Hey CDK, how can I secure my Fargate Service with ALB authentication?"
cover: /assets/heycdk.png
---

There are many use cases where you want to allow only authenticated users on your website. For example internal CI/CD tools, monitoring tools, or documentation sites. Application Load Balancer (ALB) provides a [managed way](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html) to authenticate users either by Cognito or OIDC.  

Unfortunately, this topic does not get many attentions and examples are rare. In this blog post, I'm going to show how you how it works with AWS CDK. How can the ALB be set up with OIDC (and Azure AD) and what else has to be configured for a Fargate Service?

> I want to shout out to [http://rayterrill.com/2019/09/20/AzureAD-Authentication-with-AWS-Application-Load-Balancer.html](http://rayterrill.com/2019/09/20/AzureAD-Authentication-with-AWS-Application-Load-Balancer.html) and [https://cloudonaut.io/how-to-secure-your-devops-tools-with-alb-authentication/?ck_subscriber_id=640789667](https://cloudonaut.io/how-to-secure-your-devops-tools-with-alb-authentication/?ck_subscriber_id=640789667) because their blog posts helped me a lot. 


This is the fifth part of a series 'Hey CDK'
- [How can I migrate my existing CloudFormation templates into CDK?](/blog/2019/09/11/hey-cdk-how-to-migrate/)
- [How can I reference existing resources?](/blog/2019/09/20/hey-cdk-how-to-use-existing-resources/)
- [How can I write even less code?](/blog/2019/10/01/hey-cdk-how-to-write-less-code/)
- [How can I use tags in my custom constructs?](/blog/2020/01/21/hey-cdk-how-to-use-tags-in-custom-constructs/)
- How can I secure my Fargate Service with ALB authentication?


### How can I use OIDC in ALB?
The OIDC authentication is added as an action to a listener. First create a target group, an ALB, and a listener on port 443. It's important to provide a certificate as well. 

```typescript
  // Create a target group which is used later in Fargate
  const targetGroup = new elbv2.ApplicationTargetGroup(this, 'TargetGroup', { 
    vpc, 
    port: 80, 
    targetType: elbv2.TargetType.IP 
  });

  // Create public load balancer
  const alb = new elbv2.ApplicationLoadBalancer(this, 'loadbalancer', { 
    vpc, 
    internetFacing: true, 
  });

  // and a listener (with certificate!) on port 443
  const listener = alb.addListener('Listener', { 
    port: 443, 
    certificates: [certificate] 
  });
```

Next, we need some information from our OIDC provider. See [here](http://rayterrill.com/2019/09/20/AzureAD-Authentication-with-AWS-Application-Load-Balancer.html) how to set up a new application in Azure AD.  

Replace _{tenantId}_ in the endpoint URLs with your actual tenant id and save clientId and clientSecret in SecretsManager. 

In the _authenticateOidc_ ListenerAction is defined what follows the authentication. In our case, we forward the traffic to the target group that we have defined earlier. 

An important step is to allow the ALB to talk to the configured Azure AD endpoints to verify the tokens. Therefore, add a rule for outbound traffic on port 443.

```typescript
  const clientId = cdk.SecretValue.secretsManager('clientId', { });
  const clientSecret = cdk.SecretValue.secretsManager('clientSecret', { });

  listener.addAction('DefaultAction', {
    action: elbv2.ListenerAction.authenticateOidc({
      authorizationEndpoint: "https://login.microsoftonline.com/{tenentId}/oauth2/v2.0/authorize",
      clientId: clientId,
      clientSecret: clientSecret,
      scope: "openid",
      issuer: "https://login.microsoftonline.com/{tenentId}/v2.0",
      tokenEndpoint: "https://login.microsoftonline.com/{tenentId}/oauth2/v2.0/token",
      userInfoEndpoint: "https://graph.microsoft.com/oidc/userinfo",
      onUnauthenticatedRequest: elbv2.UnauthenticatedAction.AUTHENTICATE,
      next: elbv2.ListenerAction.forward([targetGroup]),
    }),
  });

  // Important so that the ALB can talk to Azure to verify tokens
  alb.connections.allowToAnyIpv4(new ec2.Port({ 
    fromPort: 443, 
    toPort: 443, 
    protocol: ec2.Protocol.TCP, 
    stringRepresentation: 'Allow ALB to verify token'
  }));
```

Now, anonymous users get redirected to Azure. After successful authentication in Azure, they are redirected back and forwarded to the target group.

### How can I use OIDC in Fargate?
As we have already an load balancer we can't use one of the higher level ECS constructs as they create a new target group and forward action automatically. For the same reason, _listener.addTargets()_ can't be used.  

Create a FargateService and add a manual dependency to the listener to ensure that it is created before the Fargate service (it could run into a race condition where the listener is created in parallel to the ECS Service and ECS doesn't like ALBs without a listener...).

Attach the target group to the ECS Service and allow traffic from the load balancer.

```typescript
  const fargateService = new ecs.FargateService(...)

  // Add dependency manual
  fargateService.node.addDependency(listener);

  // Add the target group to the ECS Service
  fargateService.attachToApplicationTargetGroup(targetGroup)

  // Allow the service to get traffic from the load balancer 
  fargateService.connections.allowFrom(alb.connections, { 
    fromPort: 80, 
    toPort: 80, 
    protocol: ec2.Protocol.TCP, 
    stringRepresentation: '' 
  });
```

### Answer
Setting up OIDC authentication with ALB needs some additional tweaks:

At the ALB, an "authenticateOidc" ListenerAction needs to be configured before the traffic is forwarded to the target group. Also, allow the ALB to talk to the OIDC Provider to verify the token. 

On the Fargate side, the service needs to be attached manually to the ALB and the security group configured accordingly.
