---
layout: post
title: "cdk-ecr-sync: Sync Docker images to improve availability and save costs"
date: 2020-04-22 07:00:00 +0200
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: "AWS, ECS, ECR, DockerHub, EKS, Container, Docker, Images"
description: "cdk-ecr-sync: Sync Docker images to improve availability and save costs"
# cover: /assets/heycdk.png
---

In the last years, [Docker Hub](https://hub.docker.com/) became the the world's largest library and community for container images. But if you use it wrong it can have negative effects on your availability and cause some unnecessary costs. To address this, I created a high-level CDK Construct which can be used to sync Docker images from DockerHub to ECR.

### Problem 1: Availability
Images from Docker Hub are often used in production as either [parent images](https://docs.docker.com/glossary/#parent_image) or [sidecars](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar). While in the first case images are downloaded during build time, in the second case they are downloaded during runtime. Especially in the latter case, you make your availability dependend on the availabilty of Docker Hub. If you want to deploy a new version or just scale out, every time the sidecare image has to be downloaded. 

I'm not an legal expert but there're differences between the [SLAs of ECR](https://aws.amazon.com/ecr/sla/) (which is 99,9%) and the [SLAs of Docker Hub](https://www.docker.com/sites/default/files/d8/2019-04/docker-master-subscription-and-services-agreement-april-2019.pdf) which defines the availabiltiy as "AS-IS" and "IS AT CUSTOMERS SOLE RISK".

### Problem 2: Data Transfer Costs
While ECR images can be pulled without any charges within the same region, a pull from Docker Hub can lead to some costs. This applies to services that run in a private subnet as the NAT Gateway charges $0.045 per GB. This might not be much for a single service but it can add up, especially when your [container is constantly failing](https://dev.to/raphael_jambalos/secret-costs-of-ecs-fargate-4j3b).

Be aware that [ECR charges](https://aws.amazon.com/ecr/pricing/) $0.10 per GB-month. But data transfer is free within the same region.

### What the cdk-ecr-sync construct does

![cdk-ecr-sync construct](/assets/cdk-ecr-sync.png)

The [cdk-ecr-sync](https://github.com/pgarbe/cdk-ecr-sync) construct basically pulls images from DockerHub and pushes them to ECR. It expects a list of Docker images that are hosted on DockerHub and creates for each an ECR repository. 

A lambda checks daily (customizable) which images needs to be synced. It compares the image tags on both sides and writes a list of missing tags into an S3 bucket. A CodePipeline is triggered and a CodeBuild step pulls the missing images from DockerHub and pushes them to ECR. 

That's all! :)

Here an example with the datadog agent image. First, install the package

```
npm i @pgarbe/cdk-ecr-sync
```

and in your CDK App add the following snippet:

```javascript
new EcrSync(this, 'ecrSync', {
  dockerImages: [
    { imageName: 'datadog/agent' }
  ],
  lifcecyleRule: {...} // Optional lifecycle rule for ECR repos
});
```

Let me know how it works for you and open issues in my [GitHub repo](https://github.com/pgarbe/cdk-ecr-sync).
