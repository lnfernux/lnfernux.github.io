---
layout: post
title: 'Security Monitoring Antipatterns'
subtitle: 'A little bit of a deconstruction of some antipatterns in Security Operations'
tags:
  - Cyber Security
  - Security Architecture
  - Antipatterns
  - Security Monitoring
published: true
author: author_infernux
image: /img/lock.png
---

Welcome back to another post - this time we are talking about antipatterns. If you don't know what an antipattern is, we can describe it as almost the polar opposite of best practice, put into a pattern.  

> **It's basically a varying degree of non-optimal to bad practice as a common response to a problem.**

## Table of contents

- [Introduction to antipatterns](#introduction-to-antipatterns)
  - [Collection is not Detection](#collection-is-not-detection)
  - [Network is the only source of truth](#network-is-the-only-source-of-truth)
  - [Not invented here](#not-invented-here)
  - [Shiny object syndrome](#shiny-object-syndrome)
  - [One tool to rule them all](#one-tool-to-rule-them-all)
  - [Toolapalooza](#toolapalooza)
  - [Summary](#summary)

# Introduction to antipatterns

Now, even if you have never heard of antipatterns as a concept, I can almost guarantee you've heard of some - if you are a reader here you will have seen me mention multiple.

The idea for this post is pretty simple - Mark Simos and friends over at Microsoft have created the Microsoft Cybersecurity Reference Architecture or MCRA for short. Here we can find a lot of good examples of antipatterns, which I'm going to *borrow/steal*. To be more specific, I'm going to take one of their slides on antipatterns in security monitoring and deconstruct it, to see if it makes sense.

!["Antipatterns"](/img/antipatterns.png)

Starting from the top, let's get started with the first antipattern:

## Collection is not Detection

I've talked about this phenomenon multiple times already, maybe most at length in my post called [Field notes for Security Strategy](https://www.infernux.no/SecurityStrategy/). It's basically a case of **assuming that if you have all the data, you can detect all the things**. In this case we also have the added dimension of collecting data, or security monitoring, in stark contrast with hunting and closing down path.  

!["Collection is not Detection"](/img/allthethings.jpg)

Security monitoring is a dicipline within security, but basic IT hygiene and securing your systems should always come before monitoring. The analogy that comes to mind is adding floors on top of building with poorly designed infrastructure - if the fundamentals are not strong, it will crumble to dust.  

In my mind, you should always have an idea of what you are protecting - most likely what makes the business the most money. From there, identify the best security features you can implement to protect it. This usually includes a combination of:

- Network segmentation
- Asset management
- Patching
- Backup and disaster recovery (also testing that this works)
- EDR
- MFA
- Least privilege IAM

From here we can graduate to email security, maybe upgrade our EDR to XDR and then we start thinking about logging and security monitoring. If we do too much too quickly we often end up with a solid case of **alert fatigue**, where we have a lot of data and alerts, but no real way to act on them. **Alerts must be actionable, this also means we have to have the capability to act on them.**  

## Network is the only source of truth

This is a funny one - I'm almost certain Cisco and Palo Alto have a similar one where they say *"identity is not the only perimeter"* or something to the same effect. Knowing who's saying what and why is important. **Every supplier out there is wrong and right at the same time** - we don't need only their thing, we need end-to-end security. That is what zero trust is all about.

Focusing on this specific antipattern, it's also not very aligned with zero trust. A bias towards netflow and firewall is apparent in when there's a heavy background from heavy on-premise environments, but there are a few things that don't make sense in that context:

1. **The perimeter has shifted heavily to identity** for a lot of companies
2. **More and more traffic is encrypted**, so you are mostly stuck working with metadata or you're decrypting data on your exit/entry nodes
3. **Network data in itself is less useful for detection**

Third point is also important - network segmentation is very important, but unless we have utter control of what network data is supposed to travel between zones, using network data for detection is usually low value for the cost you pay. Logs are usually ingested for three reasons:

1. Detection
2. Incident Response
3. Compliance

Points two and three doesn't need to live in a security monitoring platform - **as long as the data can be queried in due time when it's needed**.

## Not invented here

Usually everyone believes their company to be *"too special"* to use off the shelf solutions. Truth is, **most of the companies in the world could buy off the shelf solutions and save themselves a lot of time**. Will it be as good as a purpose built solution in some areas? Maybe not, but it will most likely be better developed, be updated and upgraded more steadily, and using it will also result in less overhead on maintaining and updating internal tools.

There's also the matter of risk versus level of security. **People who have a technical security background tend to be very binary - it's secure or it's not**. Governance, risk and compliance are usually not something that's considered and the business people can go to hell. Buuuut, **no business no job, no job no security**. So it's important to build security in the context of the system, what information it serves and who are using it.

Unless whatever you are monitoring is one of a kind, there's usually a solution to help you.

## Shiny object syndrome

No network segmentation internally, no asset management, patching happens when we remember to... but **we need Copilot and a purple-team test**. Does that make sense? **Not at all.**  

Obviously we want to have fun at work, so we want to work with the new, cool things. This often results in bad decisions, like buying a shiny new cool tool that you don't really need. Does it help? Maybe, but we can now say we have that thing and that's the most important???

!["Shiny object syndrome"](/img/shiny.png)

I think I mentioned adding floors to a building with poor infrastructure above, but this is just that all over again. Good IT security is good basic IT hygiene. Some of it might be boring in concept, but having close to total control of your environment is more important than putting some fancy security solution on top of it.  

**People, process, tools - in that order**. If you reach a point of maturity where the next step is a tool, good for you. **But never start just outright buying tools to "complete the puzzle"** (we're missing X type of tool) or because it's what Gartner told you is the new hype.

## One tool to rule them all

Let me just get rid of all these tools we've selected to do their job and fits our use cases perfectly and have been adapted with years of work - **because I just got myself an XDR!**

**Toss that IDS out - we're moving to SIEM!** (/s)

This topic is the same as the previous and the next - it's based around **misunderstanding of what tools can and should do**. It's also rooted in the misconception people have that the **tools you buy give you inherently more security, which is mostly wrong**. If you integrate XDR into a unsecure environment, it might stop some things, but it will still be bypassed. One tool does not excuse you implementing other controls. **It's like buying zero trust networking equipment for the company wifi and then turning of MFA on that same wifi** (if you don't understand why that's not a good idea, read a little about zero trust).  

Out of the box tools also don't do as much as people expect them too - SIEMs need tuning and configuration (and an infrastructure for log collection), XDRs need integrations and policies, updated settings - you get the gist.  

## Toolapalooza

Time adds up, quickly. Time spend opening new tabs, logging into new tools - it all takes time. Some of it might be needed for the purpose of least privilege and just in time access - that's fine.

!["Toolapalooza"](/img/toolapalooza.gif)

**Let's look at an example - we're buying a new tool because we've identified a specific requirement that we can't achieve with the stack we currently have**. One of the tools we are looking into is a best of breed security product, the other one is considered "mid" (I might be young enough to pull that off). If the best of breed tool is not easy to integrate and automate with, you might choose the "mid" tool if it fits into a certain ecosystem.  

The consideration boils down to - **how much security do you gain vs how much time do you lose because of the lack of integrations?**  

## Summary

I love antipatterns, if you couldn't tell. They help me identify when I might be walking into making mistakes, but of course I need to consider who the antipattern is created by and what they are selling. It's like consuming news - you need to know the bias of the source.

**Antipatterns applied in the correct context can help us avoid our own bias, or our own wrong understanding of how we thought things work.**  
