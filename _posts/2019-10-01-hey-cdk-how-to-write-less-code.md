---
layout: post
title: "Hey CDK, How can I write even less code?"
date: 2019-10-01 07:00:00 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: "AWS, CDK, CloudFormation, AWS CDK, CloudFormation to CDK, CDK GetAtt, CDK Inheritance, CDK Aspects, CDK Composition"
description: "Hey CDK, How can I write even less code?"
cover: /assets/heycdk.png
---

After seeing some demos or trying out the [AWS Cloud Development Kit (CDK)](https://aws.amazon.com/cdk/), many questions arise. How can I migrate? What's about my existing resources? What is the Context about? And so on.

This is the third part of a series 'Hey CDK'
- [How can I migrate my existing CloudFormation templates into CDK?](/blog/2019/09/11/hey-cdk-how-to-migrate/)
- [How can I reference existing resources?](/blog/2019/09/20/hey-cdk-how-to-use-existing-resources/)
- How can I adopt the same behavior for multiple constructs?

### How can I write even less code?
One benefit of the imperative approach of CDK is that the advantages of higher-level languages can now also be used for infrastructure code. Let's have a look at inheritance, composition, and aspects.

#### Inheritance
With an "Is a" relationship to an existing Construct, it makes sense to create a new Construct that inherits from it. Props of the existing Construct can and should continue to be re-used. Properties to be overwritten can be modified directly in the constructor.

```typescript
  // SuperSecureBucket...
export class SuperSecureBucket extends s3.Bucket {
  constructor(scope: cdk.Construct, id: string, props?: s3.BucketProps) {

    super(scope, id, { 
      ...props, 
      versioned: true,
      publicReadAccess: false,
      blockPublicAccess: BlockPublicAccess.BLOCK_ALL
    });
  }
}  
```

#### Composition
If it's more of a "has a" relationship then composition is the better choice. Composition means to build your own Construct on top of other, existing Constructs.

> __And that's what the CDK is all about: Write (your own) higher-level Constructs and share them!__

This example defines a Construct called SinglePageAppHosting. It contains everything that is needed to host a static webpage on s3. It even uses [s3-deployments](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-s3-deployment-readme.html) to upload files from a local folder to the newly created s3 bucket.

```typescript

export interface SinglePageAppHostingProps {
  /**
   * Custom properties
   */
}

export class SinglePageAppHosting extends Construct {

  public readonly webBucket : Bucket;
  public readonly distribution : CloudFrontWebDistribution;

  constructor(scope : Construct, id : string, props : SinglePageAppHostingProps) {
    super(scope, id);

    const domainName = props.redirectToApex ? props.zoneName : `www.${props.zoneName}`;
    const redirectDomainName = props.redirectToApex ? `www.${props.zoneName}` : props.zoneName;

    const zone = props.zoneId ? HostedZone.fromHostedZoneAttributes(this, 'HostedZone', {
      hostedZoneId: props.zoneId,
      zoneName: props.zoneName,
    }) : HostedZone.fromLookup(this, 'HostedZone', { domainName: props.zoneName });

    const certArn = props.certArn || new DnsValidatedCertificate(this, 'Certificate', {
      hostedZone: zone,
      domainName,
      region: 'us-east-1',
    }).certificateArn;

    const oai = new CfnCloudFrontOriginAccessIdentity(this, 'OAI', {
      cloudFrontOriginAccessIdentityConfig: { comment: Aws.STACK_NAME },
    });

    this.webBucket = new Bucket(this, 'WebBucket', { websiteIndexDocument: 'index.html' });
    this.webBucket.addToResourcePolicy(new PolicyStatement({
      actions: ['s3:GetObject'],
      resources: [this.webBucket.arnForObjects('*')],
      principals: [new CanonicalUserPrincipal(oai.attrS3CanonicalUserId)],
    }));

    this.distribution = new CloudFrontWebDistribution(this, 'Distribution', {
      originConfigs: [{
        behaviors: [{ isDefaultBehavior: true }],
        s3OriginSource: {
          s3BucketSource: this.webBucket,
          originAccessIdentityId: oai.ref,
        },
      }],
      aliasConfiguration: {
        acmCertRef: certArn,
        names: [domainName],
      },
      errorConfigurations: [{
        errorCode: 403,
        responseCode: 200,
        responsePagePath: '/index.html',
      }, {
        errorCode: 404,
        responseCode: 200,
        responsePagePath: '/index.html',
      }],
      comment: `${domainName} Website`,
      priceClass: PriceClass.PRICE_CLASS_ALL,
      viewerProtocolPolicy: ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
    });

    if (props.webFolder) {
      new BucketDeployment(this, 'DeployWebsite', {
        sources: [Source.asset(props.webFolder)],
        destinationBucket: this.webBucket,
        distribution: this.distribution,
      });
    }

    new ARecord(this, 'AliasRecord', {
      recordName: domainName,
      zone,
      target: RecordTarget.fromAlias(new CloudFrontTarget(this.distribution)),
    });

    new HttpsRedirect(this, 'Redirect', {
      zone,
      recordNames: [redirectDomainName],
      targetDomain: domainName,
    });
  }
}  
```
By courtesy of [https://github.com/taimos/cdk-constructs/](https://github.com/taimos/cdk-constructs/)


#### Aspects
Another approach are [Aspects](https://docs.aws.amazon.com/cdk/latest/guide/aspects.html). In CDK, aspects are an implementation of the [visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern) and used for [tags](https://docs.aws.amazon.com/cdk/latest/guide/tagging.html). Of course, it's also possible to write your own aspects by implementing the `IAspect` interface.

```typescript
interface IAspect {
   visit(node: IConstruct): void;
}
```

A good example what an aspect can do is [cdk-watchful](https://github.com/eladb/cdk-watchful). It detects all (supported) resources in a given scope that should be watched and adds typical alarms. A CloudWatch Dashboard is generated as well.

> Idea: The same could be done for backups as well. 

```typescript
const wf = new Watchful(this, 'watchful', {
  alarmEmail: 'your@email.com'
});
wf.watchScope(myStack);
```

Or think about compliance rules that need to be ensured. This (incomplete) example shows how to check for insecure policies and throws an exception during build time.

```typescript
export class AspectsStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ... some unsecure policies

    this.node.applyAspect(new PolicyChecker());
  }
}

export class PolicyChecker implements cdk.IAspect {

  public visit(node: cdk.IConstruct): void {

    if (node instanceof iam.CfnPolicy) {
      let statements: PolicyStatement[] = node.policyDocument.statements;

      statements.forEach(statement => {
        let statementJson = statement.toJSON();

        // Be aware that `Resource` could also be a single value
        statementJson.Resource.forEach((resource: string) => {
          if (resource === '*') {
            node.node.addError("Asteriks are not allowed");
          }
        });
      });
    }
  }
}
```


### Answer
With CDK there are three patterns which can be used to not repeat yourself. Of course, it depends which one should be used when :) 

1. __Inheritance__ is useful when some sane defaults of an existing Construct should be set. 
2. __Composition__ should be used when multiple other Constructs are put together to create something new.
3. __Aspects__ can provide an easy way to apply a behavior to all resources of a stack.

