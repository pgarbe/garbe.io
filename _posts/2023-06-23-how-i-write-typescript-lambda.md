---
layout: post
title: 'How I write TypeScript lambdas in CDK'
date: 2023-06-23 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS]
keywords: 'AWS, CDK, TypeScript, Lambda, Native Cloud'
description: 'How I write TypeScript lambdas in CDK'
---

In this blog post, I'll share my personal approach to developing TypeScript lambdas using CDK. Join me as I walk you through the tools and techniques I use to make my lambda functions production-ready.

TL;DR:

* I utilize [Projen](https://github.com/projen/projen) for project setup.
* I rely on [NodeJsFunction](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda_nodejs.NodejsFunction.html) for building TypeScript lambda functions in CDK.
* I write unit tests using [Jest](https://jestjs.io).
* For debugging, I make use of [HotSwap](https://aws.amazon.com/blogs/developer/increasing-development-speed-with-cdk-watch/) and PersonalStacks.
* [CDK Integ Tests](https://docs.aws.amazon.com/cdk/api/v2/docs/integ-tests-alpha-readme.html) are performed using CDK Integ Tests.
* I ensure safe rollouts with the help of [Triggers](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.triggers-readme.html) and [FeatureFlags](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.FeatureFlags.html).

Now let's dive into an example to see these concepts in action.


## Project setup with Projen
I usually start by leveraging the power of [Projen](https://github.com/projen/projen) to create a basic project structure. To generate a project with Projen, run the following command:

```sh
npx projen new awscdk-app-ts --projenrcts true
```

By default, Projen sets up the project to generate an `awscdk.LambdaFunction` for each *.lambda.ts file in the repository. While this can be advantageous for defining common properties for all functions (e.g., Runtime), I personally prefer to disable it in my .projenrc.ts file to have more control over individual functions:

```ts
const project = new pj.awscdk.AwsCdkTypeScriptApp({
    ....
    lambdaAutoDiscover: false,
}
```

## Infrastructure in CDK

When it comes to building the infrastructure in CDK, the NodeJsFunction construct is often sufficient for my needs. It utilizes [esbuild](https://esbuild.github.io) to package the lambda code into a zip file, which is then added as a [CDK Asset](https://docs.aws.amazon.com/cdk/v2/guide/assets.html). 

Here's an example of a stack that creates a lambda function responsible for writing events into an S3 Bucket:

```ts
export class MyStack extends cdk.Stack {
    // used later for integration tests
    readonly lambdaFunction: cdk.aws_lambda.IFunction;
    readonly bucket: cdk.aws_s3.IBucket;

    constructor(scope: Construct, id: string, props: cdk.StackProps = {}) {
        super(scope, id, props);

        this.bucket = new cdk.aws_s3.Bucket(this, 'MyBucket', {
            removalPolicy: cdk.RemovalPolicy.DESTROY,
            autoDeleteObjects: true,
        });

        // define resources here...
        this.lambdaFunction = new cdk.aws_lambda_nodejs.NodejsFunction(this, 'MyFunction', {
            environment: {
                BUCKET_NAME: this.bucket.bucketName,
            },
        });

        // Grant the lambda access to the bucket
        this.bucket.grantWrite(this.lambdaFunction);
    }
}
```

By default, the NodejsFunction looks for a file named *mystack.myfunction.ts* (assuming MyStack lives in mystack.ts). However, you can easily configure it to point to another file or add additional runtime and environment variables as needed.

## Development and Unit Testing 

I find it valuable to treat a lambda function as a simple **input -> process -> output** model, making it easier to write testable code. By utilizing the appropriate type definitions (such as @types/aws-lambda), I can create well-typed unit tests for my lambdas. 

Add the following dependency in your projenrc.ts:

```ts
const project = new pj.awscdk.AwsCdkTypeScriptApp({
    ....
    deps: ['@types/aws-lambda'],
    ...
}
```

Now I can define the handler. For the sake of a better example I decided for a CloudFormation custom resource implementation that writes the event as JSON into a given S3 Bucket.

```ts
export async function handler(
    event: CloudFormationCustomResourceEvent,
    context: Context,
): Promise<CloudFormationCustomResourceResponse> {

    // Run some initialization code
    const bucketName = process.env.BUCKET_NAME!;

    return await handleEvent(event, bucketName);
}

// Have a separate function to test without handling dependencies
export async function handleEvent(
    event: CloudFormationCustomResourceEvent,
    context: Context,
): Promise<CloudFormationCustomResourceResponse> {

export async handleEvent(event: CloudFormationCustomResourceEvent): Promise<CloudFormationCustomResourceResponse> {

    // My first, very simple, implementation
    return {
        Status: 'SUCCESS',
        PhysicalResourceId: 'MyUniqueId',
        StackId: event.StackId,
        RequestId: event.RequestId,
        LogicalResourceId: event.LogicalResourceId,
    };
}
```

You might have noticed that the actual handling of the event happens in a separate function called `handleEvent(...)`. With that, I can separate some initialization logic, like reading from ENV, from my actual business logic and make the latter more testable.

The corresponding unit test can be created first or at the same time. I am not that ideomatic. Usually, I have a split view in my IDE with the implementation on the left and tests on the right side. 

> Pro tip: Start with (all kind of) tests as early as possible!

Writing test as early as possible has two advantages:
1. It helps to create more tests (just copy & ~~paste~~ adopt)
2. It prevents your code to become untestable which makes it frustrating when you have to add tests later

```ts
import { CloudFormationCustomResourceCreateEvent } from 'aws-lambda';
import { handleEvent } from '../src/main.myfunction';

describe('MyFunction Lambda', () => {
  it('will sends a success response', async () => {
    const event: CloudFormationCustomResourceCreateEvent = {
      RequestType: 'Create',
      ServiceToken: 'string',
      ResponseURL: 'string',
      StackId: 'string',
      RequestId: 'string',
      LogicalResourceId: 'string',
      ResourceType: 'string',
      ResourceProperties: {
        ServiceToken: 'string',
      },
    };
    const response = await handleEvent(event, 'MyBucketName');

    expect(response).toStrictEqual({
      LogicalResourceId: 'string',
      PhysicalResourceId: 'MyUniqueId',
      RequestId: 'string',
      StackId: 'string',
      Status: 'SUCCESS',
    });
  });
});

```

Now comes the interesting part: The integration with AWS. I don't want to deal with AWS (or any other external dependency) in my Unittest. That's why I mock the calls to the AWS API. 

Create a new file ./src/aws-adapter.ts with the follwing implementation:

```ts
export async function putObject(obj: any, bucketName: string) {
    throw new Error('not implemented');
}
```

The test looks like this:

```ts
import * as adapter from '../src/aws-adapter';

    // ...

    it('will write the event to S3 bucket', async () => {
        const event: CloudFormationCustomResourceCreateEvent = { ... };

        // Create a mock for the function in the adapter 
        const putObjMock = jest.spyOn(adapter, 'putObject').mockImplementation();

        await handleEvent(event, 'MyBucketName');

        // Now I can check if the function has been called as expected
        expect(putObjMock).toBeCalledTimes(1);
        expect(putObjMock).toBeCalledWith(event);
    });

```

Now add `await putObject(event, bucketName);` to the implementation and that's it. The tests are green 🎉

But what about the `putObject()` implementation? This can be done with some integration tests as not only the implementation needs to be tested but also other runtime dependencies and configuration like IAM permissions.

The [Adapter-Pattern](https://refactoring.guru/design-patterns/adapter) I used here has the advantage that it
1. keeps the code clean and moves all dependency-related stuff outside
2. makes actual business logic testable


## Watch & HotSwap with PersonalStacks
To streamline local development and testing, I rely on the *PersonalStack* approach. This allows me to deploy a personalized stack within the same AWS account/region without interfering with other stages of development. 

The cdk watch command, combined with the --hotswap option, automatically triggers redeployment whenever a file changes, providing rapid feedback during the development process. It's important to note that this option should not be used in a production pipeline.


> My application runs in the Cloud, so why should I mock it locally? That's why I have a PersonalStack (and not LocalStack 😉) 

Create a new personalized stack with a unique stack name. This avoids any conflict with stacks for your dev and prod stages.

```ts
new MyStack(app, 'MyStackPersonal', {
  stackName: `MyStackPersonal${Case.pascal(process.env.USER!)}`,
  env: devEnv,
  /* other stack configuration */
});
```

Use the cdk watch command `npx cdk deploy --watch --hotswap-fallback MyStackPersonal` to trigger an initial deployment. It watches your local files for changes and starts a new deployment automatically. With the [--hotswap](https://aws.amazon.com/blogs/developer/increasing-development-speed-with-cdk-watch/) option it replaces the resources directly without triggering a full CloudFormation deployment. This speeds up the feedback loop, with the cost of a stack drift. It works for resources like Lambda, StepFunctions, or ECS Services.

Lets finish the implementation of the aws-adapter:

```ts
export async function putObject(obj: any, bucketName: string) {
  const s3Client = new s3.S3Client({});

  await s3Client.send(new s3.PutObjectCommand({
    Bucket: bucketName,
    Key: 'events.json',
    Body: JSON.stringify(obj),
  }));
}
```

Try it out and test your lambda function in the AWS Console. You will notice that logs are also shown in your CLI.

## Integration Tests

While unit tests are crucial for verifying the correctness of individual functions, integration tests validate the functionality of the entire lambda function within an AWS environment. To perform integration testing, I make use of the [integration tests](https://docs.aws.amazon.com/cdk/api/v2/docs/integ-tests-alpha-readme.html) package. 

It requires some changes in projenrc.ts file:

```ts
const project = new pj.awscdk.AwsCdkTypeScriptApp({
    ...
});

// Configuration for integration tests
project.addDevDeps('@aws-cdk/integ-tests-alpha', '@aws-cdk/integ-runner');
project.gitignore.addPatterns('cdk-integ.out.*.snapshot');

project.addTask('integ:update', { exec: 'npx integ-runner --parallel-regions eu-central-1 --directory ./test --language typescript --update-on-failed' });
const integ = project.addTask('integ', { exec: 'npx integ-runner --parallel-regions eu-central-1 --directory ./test --language typescript' });
project.testTask.spawn(integ);

project.synth();
```

Explaining integration tests is beyond the scope of this article but the [CDK docs](https://docs.aws.amazon.com/cdk/api/v2/docs/integ-tests-alpha-readme.html) are a good starting point. 


## Deployment

To ensure safe deployments, I leverage [CDK Triggers](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.triggers-readme.html) and [Feature Flags](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.FeatureFlags.html). 

Triggers can call a lambda function as part of your CloudFormation deployment. For lambdas without side effects I trigger them directly to see if they work. But it's also possible to create a separate lambda function that does some smoke tests to verify your application. The good part here is that if it fails, the stack gets automatically rolled back and you're back to a safe state of your application. 

I use the built-in Feature Flags in CDK to toggle a new implementation. This is very handy when writing CDK Libs to roll out a new feature or improvement and let users opt-in by enabling the feature flag. Later you can invert it and allow users to opt-out before you completely remove the flag.


## Conclusion
Often product mangers and other stake holders try to talk us into the way we program. "Testing is not necessary", "we need to deliver the feature ASAP", "can't we do it faster?" But would an electrician actually refrain from installing fuses just because the customer wants it? The same is true for us software developers. It should be a matter of course to deliver code only with appropriate tests and a continuous deployment capable pipeline.

[Let me know](http://twitter.com/pgarbe) how you are doing it!
