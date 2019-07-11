---
layout: post
title: "The fear of new features"
date: 2014-01-16 10:51:51 +0100
author: Philipp Garbe
comments: true
# categories: [Continuous Delivery, Continuous Deployment, Feature Toggles, FeatureBee]
published: true
keywords: "FeatureBee, Continous Delivery, Continous Deployment, Canary Releases, Feature Toggles"
description: "The fear of new features, decouple feature releases from code releases, from continous delivery to continuous deployment, FeatureBee"
---

How do you feel when you think about your next release? Are you scared? Or is your operations team scared? But why are releases often so painful?

Most of the time releases are painful because they are not done regularly. There could be many reasons but in this post I want to talk about a not so obvious problem. The **fear of new Features**.


To be honest, developers are normally not so scared about releasing new features, because they built it. But operations people are. Why? Devs are paid to produce new features, but Ops are paid to host a stable platform. And every change (release) is a risk.

Continuous Delivery
----
To reduce the risk of a release we can simply reduce the number of changes by releasing more often. Sounds easy but it isn´t. It needs quite some effort to build an automatic release pipeline. Have a look on the next two pictures and you can see that the risk will be reduced the more often you release because the changes per release getting less.

![BigBang Releases](/assets/bigbangreleases.png)
![Continuous Delivery](/assets/continuousdelivery.png)

To be able to release that often you need [Feature Toggles](http://martinfowler.com/bliki/FeatureToggle.html) to work on new features without affecting the releases process. Now you can commit your changes and the feature is not visible as long as the toggle is switched off. Plus you can revert that feature without changing the code but only some config.

As long as new features are in development and toggled, a code release is not a big thing anymore. But what if you want to finally toggle on your feature? Then a new code release is needed with the feature toggled on. And again we have a risky release. Why? Because you enable a LOT of code with a single change.

![BigBang Releases with Feature Toggles](/assets/featuretogglerelease.png)


Patterns to the rescue
---

How do we handle the risk of a new feature?  

* **Single Responsibility**<br>
Shipping code to live servers and switching on new features are in my opinion two different concerns. The first one is very technical, the later business driven. What happens when you decouple the feature releases from your code release? Code releases becomes boring for you and also for operations. At some point you maybe release every commit automatically (and that´s the [difference](http://blog.crisp.se/2013/02/05/yassalsundman/continuous-delivery-vs-continuous-deployment) between Continuous Delivery and Continuous Deployment).

* **Make it easy to toggle on and off**<br>
It should be easy to toggle a feature on. Just a click. Everyone in your team should be able to do it. When the feature is accepted by your product guy he can immediately release it by himself.
And it should be easy to rollback a feature when it´s not (properly) working.

* **Canary Release**<br>
To get feedback if the feature works (technically and also from a business perspective) it should be rolled out step-by-step and monitored. You maybe start only with 5% of your visitors or you show it only to Chrome users (or some other condition) and when it works increase the number of users.


Summary
-----
Knowing how to deal with that issues is one thing, the tooling is another. Together with [Matthias](http://www.matthias-kainer.de) I started a new project called [FeatureBee](https://github.com/autoscout24/featurebee) where we want to make it easier to release new features. The idea behind FeatureBee is to make it for developers as easy as possible to toggle new features and enable Product Owners at the same time to release features when and how they want it. Without all the technical stuff like configuration files, VCS and build processes.

In my opinion this will change the way you and your team works. Imagine that you can release every green build automatically. You can switch features on and off whenever you want. You can test features on live environment without showing the feature to all other users. Think about it.


In my next blog posts I´ll show how to work with FeatureBee.

Happy Releasing!
