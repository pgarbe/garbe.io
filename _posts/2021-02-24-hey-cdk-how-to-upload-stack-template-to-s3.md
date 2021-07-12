---
layout: post
title: 'Hey CDK, how can I upload a stack template to S3?'
date: 2021-02-24 16:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: 'AWS, CDK, CloudFormation, AWS CDK, StackSet, ServiceCatalog, S3'
description: 'Hey CDK, how can I upload a stack template to S3?'
cover: /assets/heycdk.png
---

Most of the time, CDK hides it very well that behind the scenes a CloudFormation template is synthesized and used for deployment. But sometimes it's necessary to have a CloudFormation template on S3. Think about [StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudformation-stackset.html#cfn-cloudformation-stackset-templateurl) or [ServiceCatalog products](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-servicecatalog-cloudformationproduct-provisioningartifactproperties.html#cfn-servicecatalog-cloudformationproduct-provisioningartifactproperties-info). As CDK user, you might want to use [BucketDeployment](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-s3-deployment-readme.html) and think it's fine but it isn't. During synth, the template file that should be uploaded has not been written... How to solve that?

> This is another part of my ['Hey CDK'](https://garbe.io/category/cdk/) series.

### How can upload a stack template to S3?

Given there's a CDK stack that defines a ServiceCatalog product or StackSet stack.

```ts
export class MyServiceCatalogProduct extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // doing aweseome stuff here
  }
}
```

Another stack is required that creates the StackSet or ServiceCatalog product and therefor needs the CloudFormation template on S3. The synth of the `MyServiceCatalogProduct` stack has to be enforced before the BucketDeployment can use that file.

Create a dummy stage and add the that MyServiceCatalogProduct stack as part of this stage. The stage object has a `synth()` function that does what it name says and returns a CloudAssembly. Afterwards, the BucketDeployment can be used to upload that file.

```ts
export class DeploymentStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Create a stage to synthesize our stack
    const stage = new cdk.Stage(this, 'DummyStage');

    // The stack needs to be created as part of the dummy stage!
    new MyServiceCatalogProduct(stage, 'stack', {
      synthesizer: new cdk.BootstraplessSynthesizer({}), // Do not synth the bootstrap check
    });

    // Enforce the synth
    const assembly = stage.synth();

    // Upload the synthesized template to S3
    const templateFullPath = assembly.stacks[0].templateFullPath;
    const deployStack = new s3deploy.BucketDeployment(this, 'DeployStack', {
      sources: [s3deploy.Source.asset(path.dirname(templateFullPath))],
      destinationBucket: this.bucket,
    });

    // Get the S3 object url to use it in StackSets, Service Catalog or other stuff
    const templateFileName = assembly.stacks[0].templateFile;
    const s3Url = this.bucket.s3UrlForObject(templateFileName);
  }
}
```

> Thanks to [redbaron](https://github.com/aws/aws-cdk-rfcs/issues/66#issuecomment-754599325) for the initial hint.

Normally, CDK includes a check to ensure the account has been [bootstrapped](https://docs.aws.amazon.com/cdk/latest/guide/bootstrapping.html). In scenarios with StackSets or ServiceCatalog products it's not always possible to ensure that cdk bootstrap has run on the target accounts and it's often not needed. Replace the default synthesizer with `BootstraplessSynthesizer` so that the template does not contain this check.

Don't forget to add an dependency to `deployStack` in your StackSet or ServiceCatalog product to be sure the stack template exists on S3 before it is actually used.

```ts
scProduct.node.addDependency(deployStack);
```

### Answer

To write the template file to file system before it is used in a `BucketDeployment`, put the stack into a dummy stage and run a `synth()` on that stage (a synth-in-synth).
