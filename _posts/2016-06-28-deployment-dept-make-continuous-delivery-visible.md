---
layout: post
title: "Deployment Debt - Make Continuous Delivery visible"
date: 2016-06-28 09:15:29 +0200
author: Philipp Garbe
comments: true
published: true
categories: [Continuous-Delivery]
keywords: "Continuous Delivery, Continuous Deployment, Deployment Debt, Technical Debt"
description: "Definition of Deployment Debt and what it means to Continuous Delivery"
---

You probably have heard about [technical debt](http://martinfowler.com/bliki/TechnicalDebt.html) but maybe never about "Deployment Debt". What is it and why should you be interested?

## What is Deployment Debt?
A debt is normally considered as something negative. Let's imagine the bank gives you some money. At some point they want it back again and not only the money but also interest. As long as you pay your rates there is no problem. But whenever you're not able to pay the troubles starts. So in real life it's important to know how much debt you have and if you're able to handle it.

That's also the idea behind Deployment Debt. It should make your continuous delivery process visible and warn you before you're getting into troubles.

There is neither an exact definition nor exists standard tools to measure it. Consider the following as a suggestion of metrics where I think they're important.

![Dashboard showing outdated deployments](/assets/deploymentdebt.png)
*Dashboard showing outdated deployments*

## What should be measured?
What should be measured depends, of course, on your type of application. For example, it does not make sense to compare the deployment of a web application to a mobile phone app. The first one can easily be deployed by yourself, but the latter one needs to get through an approval process of the corresponding app store. Mobile phone users also wouldn't be happy if they would get every day dozens of updates of your app whereas dozens of updates of a web applications wouldn't be a problem and also not noticed.

#### Commits
The most important number to measure is which commits have **not** been deployed to production. If this number increase more than usual it tells you that there is maybe a deeper problem which needs to be investigated. This could be the fear of a Friday-evening release or code changes which have not been toggled. Maybe there is an external restriction from your marketing department. Whatever it is this number makes it visible to you and draws your attention to the real problem behind.

The same is true for commits in feature branches, especially when you do the [GitHub flow](https://guides.github.com/introduction/flow/). Also pull requests that you get from contributors should be handled in a timely manner.

#### Feature Toggles
If you use [Feature Toggles](http://martinfowler.com/articles/feature-toggles.html) track the number of concurrent toggles that you have in your code base. Introducing a new feature toggle is easy but developers tend to forget to remove them. This makes your code base unreliable and harder to refactor.

Also the age of a toggle is an interesting metric and should be watched. Depending on the type of your toggle you should also define a maximum age (find here an overview of the [different types of toggles](http://martinfowler.com/articles/feature-toggles.html#CategoriesOfToggles)). An experiment toggle should exist only for a couple of weeks. When it exists for months the team has probably forgotten to remove it.


#### No deployments
Something not so obvious is to track the number of days since the last deployment. Even if you don't have any code changes it can make sense to do a regular deployment. Why? It takes the fear from you to trigger a deployment when you have a real change. When it worked yesterday it's likeley that it works today as well. But what if the last deployment happend 2 or 3 months ago?


## Why is it useful?
What I've learned over the last years is that the faster you can deploy the better. It's a huge benefit for your business and an important competitive advantage.

Although there is no single definition of Deployment Debt it despite helps you to keep focus on your continuos delivery process. As every application is different, it's important that you create your own definition. Setup a monitoring and tracks these numbers. It helps detect subtle changes.

When your team is currently adopting continuous delivery it helps you to track your progress. Also mature teams who are doing continuous delivery for some time can benefit as it helps them to keep their high standards.


What do you think? Does it make sense to you? What would you measure?
