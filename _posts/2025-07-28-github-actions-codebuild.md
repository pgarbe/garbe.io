---
layout: post
title: 'GitHub Actions and AWS CodeBuild - The Ultimate Guide for Container Nerds'
date: 2025-07-28 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS]
keywords: 'AWS, CodeBuild, GitHub'
description: 'GitHub Actions and AWS CodeBuild - The Ultimate Guide for Container Nerds'
cover: /assets/github-actions-codebuild-cover-4.png

---

Switching between GitHub-managed runners and AWS CodeBuild sounds easy — and it mostly is — but there are important caveats to consider.

> For simplicity, this blog post focuses on Linux workloads and ignores CodeBuild’s Lambda runtime option.

## GitHub Managed Runners

![GitHub Managed Runners](/assets/github-runner.png)

First, let’s look at how GitHub-managed runners work. A common option is to use `ubuntu-latest` runner image in the `runs-on` field. This provides a virtual machine with the GitHub Actions runner installed. If no additional container image is defined in your job, it executes directly on that machine. Because this environment has a [lot of tools and language frameworks pre-installed](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md), it's sufficient in most cases.

---

## About CodeBuild

AWS CodeBuild provides a [managed way](https://docs.aws.amazon.com/codebuild/latest/userguide/action-runner-overview.html) to provide so-called “self-hosted” runners for your GitHub Actions workflows.

CodeBuild does not offer a long-running runner, but instead provisions a new instance for every job. It uses [CodeConnections](https://docs.aws.amazon.com/dtconsole/latest/userguide/welcome-connections.html) to install a webhook at the repo or org level. This webhook subscribes to multiple events and ensures that a CodeBuild instance starts if a job begins with a matching `runs-on`.

What’s not always obvious (and often confusing):

- CodeBuild provides EC2, Container, and Lambda runtimes, which can be defined per project (this is simplified, the actual configuration options are very confusing).

- In GitHub Actions, `runs-on` can [include a container image](https://docs.aws.amazon.com/codebuild/latest/userguide/sample-github-action-runners-update-labels.html). This overrides the CodeBuild project configuration (and cannot be restricted).

- The size of the runner can also be [configured](https://docs.aws.amazon.com/codebuild/latest/userguide/sample-github-action-runners-update-labels.html) in `runs-on`, again overriding the CodeBuild project (no restriction possible here either).

To understand these nuances, let’s look at a few scenarios. In all cases, assume a CodeBuild project is configured to use EC2 as the compute option.

---

## Option 1: GitHub Runner on EC2 (No Container)

![GitHub Runners on CodeBuild - Option 1](/assets/github-codebuild-option1.png)

If the job only defines a `runs-on` value pointing to CodeBuild, the GitHub Actions runner is automatically installed during the `BUILD` phase of CodeBuild. The job then runs directly on the EC2 instance.

While simple, this approach has drawbacks:

- GitHub Actions based on TypeScript/JS may fail because Node.js is not installed.
- Composite actions might fail as not all required build tools are available.

---

## Option 2: GitHub Runner with Job Container Image

![GitHub Runners on CodeBuild - Option 2](/assets/github-codebuild-option2.png)

In this case, the job defines a custom container image in addition to `runs-on`. The GitHub Actions runner pulls the image, starts a container, and runs the job inside it.

This setup also works with GitHub-managed runners, since it relies on the container’s tooling rather than the VM’s.

Good to know:

- If the container image requires authentication (e.g., for ECR), you have two options:
  - Configure the job [*with credentials*](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idcontainercredentials)
  - Use a [*buildspec override*](https://docs.aws.amazon.com/codebuild/latest/userguide/action-runner.html#sample-github-action-runners-update-yaml) to run `docker login`  
  (more details on a follow-up blog post)

- GitHub Actions mounts several folders and [overrides](https://github.com/actions/runner/issues/863) the `HOME` environment variable. This breaks tools expecting config files in `~` (e.g., [amazon-ecr-credential-helper](https://github.com/awslabs/amazon-ecr-credential-helper)).
  - **Workaround**: reset `HOME` as the first job step (like `echo "HOME=/home/username" >> $GITHUB_ENV`)

- The Docker socket is automatically mounted into the container.

---

## Option 3: GitHub Runner and Job in same Container

![GitHub Runners on CodeBuild - Option 3](/assets/github-codebuild-option3.png)

Specific to CodeBuild is the option to define a container image in the `runs-on` field. Since `runs-on` accepts only strings (or string arrays), the syntax is special and strict.

> This override is equivalent to configuring a CodeBuild project to use a container image.

It comes with a few restrictions:

- The image must have the GitHub Actions runner installed or be compatible (Alpine-based images don't work).
- The Docker socket is available only if *privileged mode* is enabled in the CodeBuild project.
- This configuration overrides the compute type and architecture, even if the CodeBuild project is set to EC2.
- **There is no IAM permission to prevent this override.**
- Format: `image:custom-<arch>-<container-image>`, where `<arch>` is `arm` or `linux`.

Similarly, the runner size can be [overridden](https://docs.aws.amazon.com/codebuild/latest/userguide/sample-github-action-runners-update-labels.html) here — meaning any repository can potentially trigger a `72xlarge` instance!

---

## Option 4: Separate Runner and Job Containers

![GitHub Runners on CodeBuild - Option 4](/assets/github-codebuild-option4.png)

This final option combines aspects of Option 2 and 3: the runner and job are each in separate containers.

I haven’t seen much advantage in this setup, as it primarily adds complexity without significant benefit.

Important:

- *Privileged mode* must be enabled in the CodeBuild project so the job container (e.g., `FooBar`) can be started.

---

## Conclusion

This article covered only a small portion of the topic — there's much more to explore. For me, understanding how a job runs on CodeBuild was key to getting the right configuration. I hope this post helps others as well (and maybe feeds some AI too).
