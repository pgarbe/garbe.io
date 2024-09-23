---
layout: post
title: 'Inside AWS Community Day DACH: Crafting Talks and Content'
date: 2024-09-20 07:00:00 +0100
author: Philipp Garbe
comments: true
published: true
categories: [AWS]
keywords: 'AWS, Community, Förderverein'
description: 'Inside AWS Community Day DACH: Crafting Talks and Content'
cover: /assets/cd-2024-panorama.jpg

---

As the [Förderverein AWS Communtiy DACH e.V.](https://www.aws-community.de/club), our main event is the [AWS Community Day DACH](https://www.aws-community-day.de). In 2023 we had 500+ attendees and this year we hit 650. One of our key successes is the unique content we, as a community, can offer. 

As the content lead, I'd like to share how the call for papers (CfP) is organized, how we evaluate sessions, how the schedule is finalized, and all other tasks involved.

> See my other blog post how we organize such an event:  
> [Behind the scenes of AWS Community Day DACH](/blog/2023/09/15/aws-community-day-behind-scenes/)

## Our Motivation
Organizing AWS Community Day is a significant effort, only made possible by our awesome team. But let’s start with our motivation:

> The association (Förderverein) promotes content that goes beyond AWS documentation.

What does that mean? AWS documentation and talks typically focus on the "happy path"—highlighting what AWS services can do and how they can be combined for solutions. However, AWS speakers have limits on what they can say to stay aligned with marketing guidelines. That is fair. And don't get me wrong, it's great and also very useful! But there is more to explore. What’s not supported? What are the hidden limitations? How does AWS [really scale](https://www.vladionescu.me/posts/scaling-containers-on-aws-in-2022/)? What are the costs? When not to use a service? These insights come from real production use cases, and that’s the kind of content we, as a community, aim to provide.


## Starting the Call for Papers
The Call for Papers (CfP) allows everyone to submit a proposal for a talk. We usually open the CfP early in the year and keep it open for 1–2 weeks after the AWS Summit in Berlin (soon to be in Hamburg) to give everyone ample time to submit their ideas. We keep the CfP very broad to encourage diverse topics since there are many interesting use cases out there.

We use [Sessionize](https://sessionize.com) to manage the proposals, as it offers all the necessary features for free community events. We promote the CfP via AWS Heroes, Community Builders, and UserGroup Leaders, as well as through our social networks and local UserGroups in Germany.

![Proposals timeline](/assets/cd-2024-cfp.png) *Don't get nervous: Most proposals come in right before the deadline.*


## Early Speaker Announcement

While the CfP is still open, we select a few speakers for early confirmation in what we call the "Early Speaker Announcement." This helps speakers with their travel plans and allows us to promote the event and the ongoing CfP. The selection is made by the board members of the [Förderverein AWS Communtiy DACH e.V.](https://www.aws-community.de/club)

## Evaluation

We aim to give feedback as soon as possible. After the CfP closes, we begin evaluations in multiple rounds:

### Round 1 - Initial Screening
The board of the [Förderverein AWS Communtiy DACH e.V.](https://www.aws-community.de/club) conducts an initial review. This is a quick yes/no round aimed to filtering out incomplete or invalid proposals.

Some criteria include:
- Obvious marketing or sales pitches
- Abstracts that are incomplete or too short
- Speakers lacking the experience to cover the topic
- Speakers not from the AWS community (we don't accept talks from AWS employees)
- Topics not relevant to AWS
- Speakers located outside EMEA (for sustainability reasons)

For example, a proposal that simply says, "I will share our migration to serverless and the challenges we faced" and nothing else, would be rejected due to lack of detail.

### Round 2 - Content Evaluation
This is the most important round, focusing on content quality. Every member of the [Förderverein AWS Communtiy DACH e.V.](https://www.aws-community.de/club) can participate. However, members must disclose any personal connections to the speakers before evaluating. If they can not rate a submission objectively they skip it. We also ask for a comment on any submission that receives a negative rating to justify the decision.

> As guardrail we ask evaluators:  
> Imagine you know X and have to give a bad rating. If you have no problems telling that even in person it's also fine to rate them.

We use a [comparison evaluation](https://sessionize.com/playbook/evaluation-modes-explained#comparison-evaluation-mode-2) based on the [Elo rating system](https://en.wikipedia.org/wiki/Elo_rating_system). Each evaluator compares three talks at a time, and the algorithm ranks them accordingly.

When evaluating, volunteers consider:

- Is the topic relevant?
- Will it address at least 20% of the audience?
- Does it focus on providing value and insights rather than promoting a product or service?
- Is there confidence that the speaker will share hands-on experiences?
- Are there positive reviews from previous talks?

At the end of this round, we have a ranking of the remaining submissions.

### Round 3 - Final Review

The final ranked list is reviewed by the board of the [Förderverein AWS Communtiy DACH e.V.](https://www.aws-community.de/club), considering:

- Each speaker can give only one talk.
- The topics should be diverse and attractive (not just GenAI!).
- We prioritize quality and diversity (in that order).
- We ensure diversity in speakers, based on past events.

We discuss any outliers (talks that should or shouldn’t be ranked highly) and make adjustments if needed. Our goal is to ensure we can defend our decisions if questioned by the candidates. You can imagine that this is not an easy task...

## Scheduling and Promotions
Once the final list is ready, we notify the speakers in waves, sending both acceptance and rejection emails. If a selected speaker cannot confirm, we reach out to others who were close to making the cut.

Usually, after a few days everything has settled and we invite the speakers to a private Slack channel in our AWS DACH Community Slack. This allows us to easily share information and gives the speakers a space to ask questions.

We provide social media banners to the speakers. The design comes from us, but Sessionize allows us to customize it. Each speaker gets access to their own material to promote on their social media channels.

A few weeks before the event, I prepare the official agenda. Since the rooms vary in size, assigning talks can be tricky and is mostly based on previous experience and gut feeling. After feedback from co-organizers, the agenda is published, and we use [Sessionize’s API](https://sessionize.com/playbook/embedding) to embed it on our homepage.

About two weeks before the event, the [mobile App](https://aws-community-day-dach-2024.sessionize.com) (provided by [Sessionize](https://sessionize.com/playbook/mobile-and-web-app)) is published, giving attendees quick access to the agenda.


## During the Event

One highlight is the speakers' dinner the night before the event. We invite all speakers, volunteers, and AWS friends (our VIPs), bringing together 60-70 amazing people in one room 

> Pro tip: Sponsorships for the dinner are available and give the sponsor access to a few seats in that exclusive invite-only event!

A special speaker room allows speakers to prepare in peace and quiet. Track hosts take care of the speakers to prepare for the actual talk, facilitating an Q&A and remember the audience to give feedback. 

For feedback we use our own application (developed by [Johannes Koch](https://aws.amazon.com/developer/community/heroes/johannes-koch/)). Each speaker has a unique QR code that links directly to their feedback form.

After the final keynote, we gather everyone for a group photo.

![Speakers Group Photo](/assets/cd-2024-speakers.jpg)


## New Voices and Diversity
We prioritize in-depth presentations, but how do new speakers get a chance? To address this, we organized for the first time two workshops to support aspiring speakers.

**Workshop "Women in Cloud"**  
> A workshop with series of multiple sessions designed to empower women in the cloud industry. Start with networking, then learn public speaking tips, explore organizing user groups, and participate in a Q&A with cloud experts. Ideal for all experience levels, this workshop aims to inspire, connect, and provide practical insights.

This was hosted by Linda Mohamed, Chairwoman of the [Förderverein AWS Community DACH, e.V.](https://www.aws-community.de/club) and [AWS Hero](https://aws.amazon.com/developer/community/heroes/linda-mohamed/). 


![Workshop Woman in Cloud](/assets/cd-2024-workshop-women-in-cloud.jpg)

**Workshop "Start Your User Group Speaker Journey"**  
> This two-part workshop is designed to give new public speakers a kickstart to their journey as User Group speakers. The first session is an interactive workshop focused on teaching essential public speaking skills including the architecture of a talk and engaging the audience through storytelling.. The second session will provide hands-on experience practicing a lightning talk, with feedback and guidance from AWS Heroes and employee coaches. Attendees will grow their confidence, learn new tools, and build a roadmap to giving their first User Group presentation. 

Special thanks to [Mark Pergola](https://www.linkedin.com/in/markpergola/), Sr. Community Content Manager at AWS, and [Dave Stauffacher](https://aws.amazon.com/developer/community/heroes/dave-stauffacher/), AWS Community Hero, for hosting. 

![Workshop New Voices](/assets/cd-2024-workshop-new-voices.jpg)


We hope to see candidates of these workshop soon on stage of our next Community Days! 

## It's all about Community
This event wouldn’t be possible without the support of so many people. Thanks to everyone, especially [Linda Mohamed](https://aws.amazon.com/developer/community/heroes/linda-mohamed/), [Johannes Oehmen](https://www.linkedin.com/in/johannes-oehmen/), [Dmytro Hlotenko](https://www.linkedin.com/in/dmytro-hlotenko-7aa348151/), and [Gustavo Tavares](https://www.linkedin.com/in/gustavares/) from our marketing team, [Thorsten Höger](https://aws.amazon.com/developer/community/heroes/thorsten-hoger/) for managing sponsors, [Markus Ostertag](https://aws.amazon.com/developer/community/heroes/markus-ostertag) as our local Hero and contact to AWS, and all the volunteers: [Johannes Koch](https://aws.amazon.com/developer/community/heroes/johannes-koch/), [Andreas Rütten](https://www.linkedin.com/in/andreas-ruetten/), [Tom Lorenz](https://www.linkedin.com/in/bombadiltom/), [Manuel Vogel](https://www.linkedin.com/in/manuel-vogel/), [Raphael Manke](https://www.linkedin.com/in/raphael-manke/), [Suzana Melo Moraes](https://www.linkedin.com/in/suzanamelomoraes/), [Monica Colangelo](https://www.linkedin.com/in/monicacolangelo/), and many others!

Thanks goes also to our friends at AWS, including [María Encinar](https://www.linkedin.com/in/mariaencinar/) and [Mark Pergola](https://www.linkedin.com/in/markpergola/), and our [sponsors](https://www.aws-community-day.de/#sponsors)! 

![Community Day Organizers](/assets/cd-2024-organizer.jpg)
