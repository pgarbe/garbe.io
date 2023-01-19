---
layout: post
title: 'Hey CDK, how should I handle dependencies?'
date: 2023-01-19 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS, CDK]
keywords: 'AWS, CDK, AWS CDK, npm, peerDependencies, dependencies, dependency, bundledDependencies'
description: 'Hey CDK, how should I handle dependencies?'
cover: /assets/heycdk.png
---

At some point, every CDK developer has to deal with dependencies. While it's easy within CDK Applications, it is more complicated when writing CDK Libraries. In this article, I will show how to manage dependencies in CDK Applications and Libraries written in TypeScript.

> This is another part of my ['Hey CDK'](https://garbe.io/category/cdk/) series.

## CDK Applications
In a CDK Application, you don't have to care too much about dependencies. Put libraries you need to build/test your application as [devDependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#devdependencies) and libraries that are used to run the code as [dependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#dependencies). 

![Dependencies of a CDK Application](/assets/cdk-dependencies-app.png)

Within the *package.json* file use [semantic versioning (SemVer)](https://docs.npmjs.com/about-semantic-versioning#using-semantic-versioning-to-specify-update-types-your-package-can-accept) to specify the version you expect. For example, to specify acceptable version ranges up to 2.60.4, use the following syntax:

| Range          | Syntax                    | Example                   |
| -------------- | ------------------------- | ------------------------- |
| Patch releases | 2.60 or 2.60.x or ~2.60.4 | 2.60.1 ✅ 2.61.0 ❌ 3.0.0 ❌ |
| Minor releases | 2 or 2.x or ^2.60.4       | 2.60.1 ✅ 2.61.0 ✅ 3.0.0 ❌ |
| Major releases | * or x                    | 2.60.1 ✅ 2.61.0 ✅ 3.0.0 ✅ |

The *package.json* just defines the requirements you have on your dependency. Be as open as possible to not exclude newer versions.  

The actual version of your dependency is stored in the lock file (yarn.lock or package-lock.json). With your pipeline run `npm ci` or `yarn install --frozen-lockfile` to install exact the version as defined in the lock file. 

> Be aware that by running just `npm install` or `yarn install` existing packages in node_modules are not touched but missing packages are downloaded and the lock file could be updated. 


## CDK Libraries
In CDK Libraries it's more important how dependencies are defined as it affects all applications that are using this library. 

![Dependencies of a CDK Library](/assets/cdk-dependencies-lib.png)

In this picture, you see a CDK Library that depends on aws-cdk-lib. But also a CDK Application that depends on both, the CDK Library and the AWS CDK (aws-cdk-lib). Can you already guess the possible conflicts?

### Dev Dependencies
Like for a CDK Application, add libraries you need to build or test your library under [devDependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#devdependencies). These dependencies will be ignored by users of your library.

### Dependencies
All direct and indirect dependencies must be resolved within the CDK App. This works as long as there are no conflicts. But both, the App and the Library will likely depend on the AWS CDK. Let's say the *CDK Application* expects v2.45.0 and the *CDK Library* depends on v.2.60.0. Which version should be used? 

There are two ways how to solve it:

#### Bundled Dependencies
Bundled dependency means the package of the dependent library is packed as part of your npm package. This also includes dependent packages of your package. You will see a nested *node_modules* folder in your package content.

A disadvantage of this approach is that it increases the size of your package. And the user of your library must rely on you to bundle the latest version as they can't upgrade that dependency to a newer version (which is useful for security fixes).

> Use bundled dependencies **only** when you require a specific version.

In the *package.json* add the library as [dependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#dependencies) with a version (see SemVer above) and [bundledDependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#bundleddependencies) (just the name of the package without version information).

```json
{
  "name": "CDKLibrary",
  "dependencies": {
    "libB": "^1.0.5"
  },
  "bundledDependencies": [
    "libB"
  },
}
```

#### Peer Dependencies
The [peerDependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#peerdependencies) tell the application that the library depends on other libraries. It delegates the decision of which exact version should be used for the CDK Application. Here it is important to not be too restrictive to give the user of your library more choices if they want to use the same library in a different version.

In the *package.json* add the library as [devDependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#devdependencies) with the exact minimum version you support and as [peerDependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#peerdependencies) with a range of possible versions that might also work (provided that SemVer rules are respected).

```json
{
  "name": "CDKLibrary",
  "devDependencies": {
    "aws-cdk-lib": "2.45.0",
  },
  "peerDependencies": {
    "aws-cdk-lib": "^2.45.0",
  },
}
```


### Answer
Dependencies are handled a bit differently in CDK Applications and Libraries. 

For CDK Applications it's easy. Add dependencies you need to build or test your code as [devDependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#devdependencies) and dependencies you need to run your code as [dependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#dependencies).

In a Library, the same rules apply for [devDependencies](https://docs.npmjs.com/cli/v6/configuring-npm/package-json#devdependencies) but when it comes to dependencies you need to run your code you can either bundle them or define them as a peer dependencies. The correct versioning is important here.
