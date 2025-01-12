---
layout: post
title: 'On the use of Threat Intelligence in Detection'
subtitle: If applied correctly, Threat Intelligence can be a useful tool in your belt. Mostly, however, it might be barking up the wrong tree depending on your maturity level. Let's explore that!
tags:
  - Cyber Security
  - Security Monitoring
  - Threat Intelligence
  - MISP
  - Cyber Threat Intelligence
  - Detection
published: true
author: author_infernux
image: /img/azure.png
---

First things first, for those who might be wondering - what is Cyber Threat Intelligence (CTI)? Well, for starters, **it's data**. Data that is **collected** **processed**, and **analyzed** to **understand the threat landscape**. This data can be **used to make informed decisions** and take actions to **protect your organization**.

Obviously, there's a whole more to it. Let me start by just underlining the fact that **I'm not a CTI-guy**, I'm a bolts and spanners kind of guy. I design, test, and implement. For this I need a good understanding of what I'm setting up and what it should accomplish. 

*So do take **everything** I say with a grain of salt here.*

## Maturity

First, we need some context. Or rather, you do. You need to understand where you are in terms of maturity. For this, I've drawn a very nice picture (below) that outlines a sort of maturity ladder. On this ladder, I've tried to make a **very simple outline of IT security maturity**. This is **not a well-researched model**, it's just a way to illustrate the point I'm trying to make.

![](/img/OnTI/MaturityLadder.png)

The point is that on level 0, you start out in the void. Basically, you have no structure at this point and everything is ad-hoc. From here, we evolve:

* Level 1 - this is where we apply basic IT security principles, like EDR, firewalls, patching, networking zone models and so on.
* Level 2 - here we start to mature our IT security by making our systems more resilient by adding stricter configurations both on devices, servers and networks. We might also start to implement some sort of SIEM or log management solution.
* Level 3 - usually at this point we start having a really mature IT security layer, we know what traffic is supposed to go where in our networks, what systems are supposed to talk to each other and so on. For the detection piece, we might have a SOC function at this point. Detection is, based on our knowledge of the environment, more purpose built and this also allows for threat hunting. Automation is starting to pick up steam.
* Level 4 - and so on, and so forth.

Again, this is not a model or a framework, it's just a way to illustrate the point I'm trying to make:

1. **You usually don't need a SOC function before you can keep your boat from sinking**. (Having someone tell you there's a bunch of holes leaking water in everywhere while the boat is 90% underwater is dumb).

This leads me straight to the core of this post, which is the use of Threat Intelligence in Detection. When do we need it? When is it useful? When is it not? At what maturity level should we start looking at it?

## Threat Intelligence

Threat Intelligence is a very broad term. It can be anything from a list of IP addresses that are known to be malicious, to a full-blown report on a specific threat actor. The point is that it's **information** that can be used to **make decisions** or **take actions**.

Let's look at an example of TI that is very common in my line of work; indicators of compromise (IOC). These are usually IP addresses, domains, hashes, and so on that are known to be malicious. Usually, they might follow a flow like the one outline in the picture below, where we collect an IOC that is known to be used by an attacker, put it into a list/feed that we consume using a threat intelligence platform (like MISP), and then pass that on to our SIEM (like Microsoft Sentinel) for detection.

![](/img/OnTI/IOC%20Example.png)

I've talked about this before, where this use case is a very common one. It's also a very low hanging fruit, and it's usually where most organizations start when they start to look at Threat Intelligence. There are a lot of problems with this, first of which is that not all data is good data. 

![](/img/onTi/Data.png)

I've heard some people argue that **data is not the issue**. It's usually "our ability to handle the data" that is the issue. I agree to an extent, but when we are looking at external data sources, especially when it comes to TI, we need to also keep in mind the **data quality**.

- Does it contain false positives? **We've seen multiple cloud providers IP ranges included in threat feeds**. Imagine now, if you had automatic blocking based on this feed. This might also be a false negative, where the attacker uses a cloud provider to host their C2 infrastructure.
- When was the data collected? **Is it still relevant?** Keep in mind that different types of IOCs have different lifetimes. An IP address might be used for a short period of time, while a domain might be used for a longer period of time.
- How is the data collected? **Is it reliable?** If the data is collected by a single source, how can we be sure that it's not manipulated?
- Can we trust the data and the source? **Is the data source reliable?** Is it a reputable source? Is it a source that is known to be manipulated by threat actors?

There are a lot of things to keep in mind, considering the fact that I see a lot of people asking for "a list of free threat feeds" on BlueSky or Reddit. This is usually where I start to get a bit worried, because **free is not always good**. Now it obviously depends on what you do with the data, but if you're going to use it for blocking, you need to be sure that the data is good. For detection, as long as we can tune and also have a feedback loop that allows us to remove feeds that are not good, it's usually not a problem. **Maybe the best place to start is to use IOCs simply for enrichment, not detection and blocking.**

## The soft side

Now, this is where it usually stops for people like me. We like our bits and bytes - throw up a MISP server, shove some feeds in and connect that to Microsoft Sentinel and Defender XDR - done. I can understand if people working in CTI are a bit annoyed by this, because there's a whole lot more we can use threat intelligence for.

**If we take a step back at the maturity ladder, consider for a moment why we choose to implement a function when we do.** Are we doing it because it's what everyone else is doing? What we were taught? What we read on LinkedIn? **What is driving these decisions?**

This can also be said for when we do anything - creating detections for our SIEM, buying new tools, deciding to threat hunt, and so on. **What is driving these decisions?** 

If the answer does not contain a part of "threat intelligence", "threat modelling", "threat landscape" or something similar you might want to reconsider a tiny bit. I'm not saying TI/CTI should be the primary driving factor, but it should **heavily influence your investments in the cyber security space**.

I will say, at this point, for small and medium businesses, **you might not need a CTI function, or have the budget for one**. Even so, you should still consider the threat landscape when making decisions. It's not rocket science to threat model your environment and consider what threats are relevant to you. 

**Let's consider again the maturity ladder**:

* If you're at level 1, you might manually upload new indicators you find to your AV/EDR/XDR solution and firewalls to block them.
* When you're somewhere around level 2, you might start feeding these indicators into your SIEM for detection.

These are both examples of using TI as a tangible data source. I'd advise that taking CTI into account when making technical decisions is a good idea. Threat model your environment, consider what threats are relevant to you, and use that information to help you make decisions going forward. The **more mature** you become as an organization, the more you can use CTI to **help you make decisions in smaller parts of the security work**.

### Detection

When it comes to detection, let's zoom all the way out. We have two paths we usually take when we're looking at detection:

1. **IOC detection**
2. **Threat-based detection**

Let's explore these two paths:

#### IOC detection

![](/img/OnTI/IOCDetection.png)

IOC detection is the most common path - it simple requires feeding indicators into tables that can be queried by the detection component of the SIEM. Usually this is done by creating a rule that **triggers on a match**. There are ways to make this more accurate, like making sure the sources we use are high fidelity and has a low frequency of false positives. If it's supported, we can also use the tags and confidence levels that are provided by the TI source to make the detection more accurate.

#### Threat-based detection

![](/img/OnTI/ThreatBased.png)

Here we use our understanding of our orginization and sectors threat landscape, usually an artifact from a threat modelling excercise, to inform our detection engineering work. This is a bit more complex, but it allows us to work on creating detections that cover the things we as an organization are most likely to be targeted by. 

## What's the point?

(Cyber) threat intelligence is a **massive field inside security that can applied in a lot of different ways**. Assuming that **starts and ends with ingesting some free threat feeds** into your SIEM is, ah, **not optimal**. 

Maybe the most important function of threat intelligence is to **inform you** to make decisions and in your actions. This can be as menial as **blocking an IP address**, to **creating a detection rule** all the way to **influencing security strategy** on an organizational level.

Want to learn more about this? Read my post on [threat modelling and data sources](https://www.infernux.no/SecurityMonitoring-DataSources/). In the future I will explore creating detections from threat modelling a fictional organization to show some examples of how this can be done.
