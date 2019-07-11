---
layout: post
title: "My Road to Microservices"
date: 2014-10-09 13:00:00 +0100
author: Philipp Garbe
comments: true
# categories: autoscout24 microservices monolith
---

At the moment there is a big hype about microservices. Especially since Martin Fowler wrote an [article](http://martinfowler.com/articles/microservices.html) about it. Also for [AutoScout24](http://about.autoscout24.com/) it´s a big topic. Not because of the hype but our history.

Let me tell you my experience over the last couple of years…


###Monolith

When I started at AutoScout24 in 2010 all teams worked together on one big monolithic web application. We had releases every 4 weeks and it took us 2 weeks to ship it. Every team had to send one member to a virtual release team where they worked together with our release manager in order to ship the code. This was painful for several reasons:

- The people were partially blocked from working on their normal sprint.
- The information how to release was partially lost because we rotated the person who attended the virtual team. Another reason was because releases happend so rarely (~ 4 weeks).
- In the meantime the teams made so many changes that each release was always a big risk (and that was the reason why it took us 2 weeks to ship it).

But we learned our lessons and improved so we could release every 2 weeks, once a week and finally several times a week (a dedicated team and automation was the key). But this was still not good enough. Sometimes when one team had to release a feature on a specific day we had to manage dependencies to the other teams so that the release was not blocked because of some (risky) code changes. Our teams were normally required to work independent and take ownership about their portfolio. So our pain now was the dependency between the teams because we worked on the same repository and had to release together.

So we started with Swimlanes.

###Swimlanes

About two years ago we had first discussions about killing splitting up the monolith. But how? We defined a swimlane as an independent releasable unit. In about 6 months we were able to make the whole dealer area independent from the monolith. We created a new repository and a separate build and delivery process. Now only two teams worked on that swimlane and we were happy to release independent from all other teams. At that time we were able to release several times per day. One release took about 1 hour (from code commit to live) and was fully automated. Yeah!

For some time we were happy (again). Also our product guys were happy because we could ship new features much much faster than before. But after a while we wanted more. We saw the drawbacks of so much independency because other teams also started their swimlane. Problems that were already solved in the monolith suddenly popped up in the swimlanes. Also starting a new swimlane was a lot of manual effort and it looked every time a bit different. It was difficult to share experiences and learnings from one team to the other teams. And we still had the feeling that the swimlanes were too big. We couldn´t name it until Martin Fowler wrote his article about microservices.

###Next exit: Microservices?

What I told you before happend already in the past. At the moment we are trying to reflect what happend and how we can learn from our failures. Are microservices the answer to our problems? We hope but actually don´t know! We invested already some time to learn how an architecture with microservices could look like. But we still have questions like: How do you split your services? How can you define the macro architecture? How much freedom do you give the teams in their micro architecture?

###Purpose of this blog post

Beginning with this post I want to start a series of posts about our journey to microservices. My intention is to write them mostly for myself so that it helps me to order my ideas, thoughts and learnings. I also want to document decisions we made and the reasons for that. Maybe this becomes useful on some day.

If you are also interested in our journey just [follow me](http://feeds.feedburner.com/pgarbe). And give me feedback!

PS: If you are from Munich join our [Microservices Meetup](http://www.meetup.com/Microservices-Meetup-Munich/)
