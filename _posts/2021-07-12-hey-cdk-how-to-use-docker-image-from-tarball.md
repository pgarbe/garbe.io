---
layout: post
title: 'Hey CDK, how can I use a tarball as Docker image asset?'
date: 2021-07-12 16:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: 'AWS, CDK, AWS CDK, Docker, Image, Tarball'
description: 'Hey CDK, how can I use a tarball as Docker image asset?'
cover: /assets/heycdk.png
---

With [assets](https://docs.aws.amazon.com/cdk/latest/guide/assets.html), the AWS CDK provides an easy way to build Docker images based on a Dockerfile. But sometimes, Docker images are built natively by other tools (e.g. [jib](https://github.com/GoogleContainerTools/jib)). How can they be integrated as native container image assets into your stack?

Since version [1.113.0](https://github.com/aws/aws-cdk/releases/tag/v1.113.0) it's possible to [use tarballs](https://github.com/aws/aws-cdk/pull/15438) as source for CDK's container image assets.

> This is another part of my ['Hey CDK'](https://garbe.io/category/cdk/) series.

### How can I use a tarball as Docker image asset?

A typical way to build Docker images is to use a Dockerfile and the `docker build` command. In restrictive environments, however, using the Docker daemon is not possible. Tools such as [jib](https://github.com/GoogleContainerTools/jib) therefore build OCI-compatible images directly for your application. These can be saved as a tarball.

I'll skip the details and create a local tarball from the latest nginx image using the following command:

```sh
docker save public.ecr.aws/nginx/nginx:latest -o nginx.tar
```

In my CDK application, I can use the new command `ContainerImage.fromTarball` to add the locally stored tarball as a container image asset.

```ts
export class AppStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const cluster = new ecs.Cluster(this, 'Cluster', {});

    const taskDefinition = new ecs.FargateTaskDefinition(this, 'TaskDef');
    taskDefinition.addContainer('DefaultContainer', {
      image: ecs.ContainerImage.fromTarball('nginx.tar'),
      memoryLimitMiB: 512,
    });

    // Instantiate an Amazon ECS Service
    new ecs.FargateService(this, 'Service', {
      cluster,
      taskDefinition,
    });
  }
}
```

What happens behind the scene is that the tarball is added as a file asset and moved to the cloud assembly directory folder (normally, it's cdk.out).

![Tarball in cdk.out folder](/assets/tarball-cad.png)

The assets file contains now two assets: The tarball and the container image. The image asset contains only an executable that ensures that the image is available in Docker (using `docker load`) and returns the name of the image.

![Assets manifest](/assets/tarball-assets.png)

This works also in a [CdkPipeline](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_pipelines.CdkPipeline.html). Extend the [SynthAction](https://docs.aws.amazon.com/cdk/api/latest/docs/pipelines-readme.html#synths) to build the tarball and run `cdk synth` afterward.

```ts
const synthAction = pipelines.SimpleSynthAction.standardYarnSynth({
  environment: {
    buildImage: codebuild.LinuxBuildImage.STANDARD_5_0,
    privileged: true,
  },
  sourceArtifact: sourceArtifact,
  cloudAssemblyArtifact,
  buildCommand: './gradlew jibToTarball && yarn build',
});
```

### Answer

Use the `ContainerImage.fromTarball` method to load a Docker image from a tarball and add it as a container image asset to your stack. Everything else will be handled by the CDK.
