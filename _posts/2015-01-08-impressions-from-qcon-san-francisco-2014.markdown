---
layout: post
title: "Impressions from QCon San Francisco 2014"
date: 2014-11-14 09:00:00 +0100
author: Philipp Garbe
comments: true
#categories: [conference,qcon,san francisco]
---

This year [AutoScout24](http://about.autoscout24.com/) was generous again to let me go to [QCon Conference in San Francisco](https://www.qconsf.com/). The overall topic was “Software is changing the world”. Let me share my impressions with you.


![Obligatory picture from Golden Gate Bridge](/assets/qcon2014_goldengate.jpg)

##Location

After 2012 this is the second time I went to QCon in San Francisco. The conference is organized by [InfoQ](http://www.infoq.com/) and took place again at the [Hyatt Regency Hotel](https://goo.gl/maps/LCoCB). It was sold out and the rooms were almost too crowded.

It seems that people in San Francisco are very healthy. There were no softdrinks, just water, tea and coffee. Also the food was more for vegeterians. I missed some good old American steaks :)

On the other site it´s a nice hotel and easy to reach. The BART station as well as the Cable Car turntable is next to the hotel. Market street and Union Square are in a walking distance.

##Talks

If I had to describe the conference this year in three words it would be: __reactive__, __functional__ & __microservices__. Obviously that´s my opinion. There were also a lot of other talks about engineering culture, continuous deployment and architectures and and and. Unfortunatley I couldn´t attend them all.

It´s hard to summarize what I´ve learned. There were so many little but interesting details. Reactive Extensions (Rx) is definitely a technology I have to evaluate further. Netflix showed how they use Rx to solve problems with race conditions, uncaught exceptions from async calls and concurrency.


> Concurrency and async are non-trivial. Rx doesn’t trivialize it. Rx is powerful and rewards those who go through the learning curve.   

> Ben Christansen - *Netflix*



It was also impressive how many developers work at companies like Netflix (~900) or LinkedIn (~1800). At LinkedIn they had a single code repository some time ago. You can imagine how long it took until you were able to commit your code.


> If you don´t end up regretting your early technology decisions, you probably over-engineered

> Randy Shoup - *[http://www.slideshare.net/RandyShoup/...](http://www.slideshare.net/RandyShoup/concurrency-at-scale)*



Microservices were everywhere. Even startups showed how they use a microservices based architecture. To ensure the code quality you often heard about dependency tests which are part of the build pipeline. If you break these tests you cannot deploy your service.

When you have microservices you also need a strong ownership. Teams own few services and are completely responsible for them. This also includes that devs are on call rotation.


> You think of code in a completely different way when you are up in the middle of the night dealing with an outage.

> Melody Meckfessel - *Google*


Maybe because of that code reviews are mandatory for them. Your commit is not merged into the main branch until it was reviewed (e.g. with [Reviewboard](https://reviewboard.org/)). One company told that they have also ownership for every file.

This is a list of my favorite talks. The slides are already available [here](https://qconsf.com/schedule). The videos should be public in the next weeks / months.

- [Asynchronous Programming at Netflix](https://qconsf.com/presentation/asynchronous-programming-netflix-0)
- [Functional Systems @Twitter](https://qconsf.com/presentation/functional-systems-twitter)
- [Reactive Programming with Rx](https://qconsf.com/presentation/reactive-programming-rx)
- [Building and Deploying Microservices with Event Sourcing, CQRS and Docker](https://qconsf.com/presentation/building-and-deploying-microservices-event-sourcing-cqrs-and-docke)
- [Scalable Microservices at Netflix](https://qconsf.com/presentation/scalable-microservices-netflix-challenges-and-tools-trade)
- [Concurrency At Large-Scale: The Evolution To Reactive Microservices](https://qconsf.com/presentation/concurrency-large-scale-evolution-reactive-microservices)

If you know other good talks write them in the comments.

##Networking

One thing I absolutely like at the QCon in San Francisco is that you´re surrounded by all the big web companies. Developers from [Netflix](http://techblog.netflix.com/), [LinkedIn](https://engineering.linkedin.com/), [Facebook](https://code.facebook.com/), Google and Amazon where there as well as guys from startups like [500px](https://500px.com/about) and many others.

To enforce communication the organisators of the QCon came up with nice ideas. One idea was a new designed badge. Sounds not like a big thing, but somehow the big first name forced you to look on the badge. I came in contact with many guys just because I or he looked on it.

![First name as eye catcher](/assets/qcon2014_badge.jpg)

Second idea was topic tables. During lunch they prepared some smaller tables (max 4 persons) with a topic (e.g. Big Data, Culture, Functional). So you had the chance to come in contact with others because you were interested in the same topic.

![Topic tables](/assets/qcon2014_topictables.jpg)

Anbd there was also the traditional conference party at [Thirsty Bear](http://thirstybear.com/) on Monday evening. The whole bar (2 floors) was crowded by nerds and beer and food was for free.

##Conclusion

> I have to go to #qconsf again to become gold alumni and get the fancy q-tshirt

> me - *[https://twitter.com/pgarbe/status/...](https://twitter.com/pgarbe/status/529317839560138752)*


Of course that´s not the main reason. When you work as web developer this conference is a must. Not only they provide trending topics and interesting insights but you have also the chance to extend your network with smart people.

I hope we´ll see us (again) at QCon SF 2015!
