---
layout: post
title: Field notes on security strategy
subtitle: Some thoughts and notes around implementing security features and what it is that we keep doing the wrong way.
tags:
  - Cloud Security
  - Microsoft Sentinel
  - Data connectors
  - Log ingestion
  - Log strategy
published: true
author: author_infernux
image: /img/sentinel.png
---

# A quick note on security mindset

THis just might be an assumption made due to my experience, but I find a lot of people in the security business think in a binary way, you're either **secure** or you're **not**. Which is a fair statement for the current status of your outfit, if you're a fan of dealing in absolutes. 

![](/img/obi.gif)

The point I'm trying to make is, security strategy is not a binary thing. Security posture is not a binary status.

# The percieved issue 

The pitfall I often observe is that of **"everything, everywhere, all at once"**, often coupled with **zero trust tool X** which will finally make you the most secure outfit on the planet. Let's attack the first thing, which is the problem of the **"enable everything"**.

## Enablers abound

Buying a new product will (almost) never solve your security issues. Yeah, this cool new tool just came out, but what does it actually help you with? Maybe you're actually already partly covered, if you'd only dive deep enough in the settings of the tools you currently have and configured them properly? 

Let's put it like this; every time we enable a tool, be it for security or whatnot, it needs to have purpose. It must be part of the strategy and, maybe most importantly, someone needs to manage it. If it's an EDR that generates alerts, someone needs to actually handle those alerts. Most EDR-solutions are arguably secure-ish out of the box, but you need to tweak and tune the settings in order to get the most out of them.

## Alert fatigue drowns defenders

![](/img/hand.jpg)


If you're working in an MSSP or SOC you'll know of alert fatigue. Even so, when going around different Azure setups and looking through Sentinel I keep seeing the same thing. Tons of data connectors enabled overflowing with logs, template analytic rules all with default settings and so many incidents I couldn't close them all in my lifetime even if I wanted. Who set this up? 

I also keep seeing people asking question about enabling all the template rules in Microsoft Sentinel. **Please, stop.** Why do you want that? 

Let me explain:

### Threat landscapes and templates

Just because an alert is generated does not mean you're doing your job as a Security operations Center or MSSP. An alert (or incident) should be actionable. It should be indicative of something bad that's happening. If 99% of all your alerts are false positives, what good does it do? Alerts needs to be enabled with purpose, tuned to specific environments and set at the right level of severity - and again, it must be actionable.

Log ingestion and security monitoring works the same as with security tooling and features. It's very similar. There needs to be a strategy, and it needs to be aligned with the business.

Every outfit, company and organization have one thing in common - they're all different. They all have different setups in terms of tools they use for general productivity, how on-prem heavy they are, how risk-tolerant or averse they are. They also have very different usage patterns. Maybe one company will RDP from one server to another, while others will always only RDP from their workstation directly into servers.

Going back to the scheduled queries, analytic rules, alert rules (or whatever you call them), hopefully it's clear that this needs to be aligned with the threat landscape and usage patterns. Enabling a rule for finding nested RDP (which is sometimes indicative of a threat actor moving laterally) could also for some companies be legitimate usage. 

When we buy a new tool, or set up ingestion of a new log source for detection we need to align it with our strategy.

## Logs are not all the same value

Depending on how your outfit is set up, be it cloud only or on-prem heavy, the logs you ingest for security monitoring will have different value. I will generally make the argument that netflow and firewall-logs are useless for most companies. Why? Well, first of all the security maturity is usually not at a point where they can actually gain anything from the logs, as it's mostly used post-breach and for enrichment purposes. 

If we could ingest all the logs in world without worrying about cost we wouldn't need to talk about this, but we can't. Speaking from a business perspective - how can you justify paying X amount for a log source you gain next to nothing from until you're breached? 

To put it plainly - if you needed to prioritize log sources we should prioritze highly actionable logs first, and then as we mature add more capabilities. I'm not saying never ingest firewall or netflow logs, but know how to use it well enough to actually gain value from them. Again, same with alerts, just because it's there doesn't mean you're doing your job - you need to do something with the logs first.

## One tool to rule them all

There's no "one tool" out there. If you don't have the process, people and knowledge to handle security alerts all a new tool will bring you are more alerts you can't do anything about.

# Security strategy notes

Enough ranting about what's wrong - what can we do better?

First of all, we need a plan. That plan should be to secure ourselves on a high level, but it should also have some stages of maturity. Our maturity in this case just means that as we grow more familiar with our tools and improve our processes and automation we can expand our toolbox and handle more complex security tooling and security monitoring use cases.

So where do we begin? Well, that will vary. Yes, vague, I know. My personal strategy is working on a perimeter basis. Identify your outer perimeter and work on securing that. 
**What is a perimeter?** Well, physical locations is one for sure. Networks, no doubt. Lately the most popular outer perimeter is **identity**.

## So lets ask ourselves some questions

> What do we want to protect? What is most important to us?

Maybe the answer is identity? Ok, so we go through Azure AD and work on some settings, maybe we enable conditional access policies, Azure AD Identity Protection. Maybe we try to secure the workstations we use our identities with EDR. Our attack vectors are usually mail and websites, so maybe we implement spam filters. The list goes on, but the general idea is that we have a plan.

> What is next? What do we do now?

Identify our next perimeter. Maybe we have business critical applications and we want to see how the access to those work, and make sure we have the correct settings and logs for that. 

## The general idea

In order to form a plan we need to know things. The list of things we *should* know is almost impossible to type out, but I tried to create a short list:

1. Know your own infrastructure and usage patterns.
2. Identify your threat landscape (what is our most valuable assets in our stack, who would want to try and steal it).
3. Find your current security posture (what tools do we have, what processes).

![](/img/know.gif)

Then, after we know things, we can plan to improve our security posture. One of the errors of security teams is often implementing things as a project. 

> Now we are here, in 6 months we will be there.

If it floats the boat, the boat floats. But, in this day and age technology moves so quickly it's hardly ever a good idea to implement security as a big project. It should probably be done agile (I don't know what that word really means, hopefully it makes some sense) with short feedback loops in order to be able to adjust course quickly:

1. Identify what you want to protect (let's say identity).
2. Gain a clear image of what your perimeter is (for identity this will be where it's being used, for what, etc).
3. Implement security controls, feature, logs or tools (keep it simple, work in batches).
4. Monitor the impact of your added security controls and tune them (this also goes for security monitoring and analytic rules).
5. Finish implementing.
6. See 1.

As you can probably tell, this is a cyclical approach. I can't overstate this enough, don't try to protect your users identities 100% during your first cycle. Work in batches. 

# TLDR 

* Process, people and knowledge before tooling
* Have a high level plan for improving security posture
* Identify your perimeter and then try to implement features to protect it 
* Work in small batches
* Configure and tune everything to match your usage patterns, threat landscape and risk appetite

# Disclaimers disclaimers

These are some of my thoughts, based on my experience and observation. If this does not apply to you, good, but hopefully it's something to think about.

It's also just what I think about the topic - you might have a different idea, which is not only fine but a necesarry piece of the puzzle. If we don't disagree and explore other solutions we will **never create anything better than what we currently have**.

Also, please disagree. Reach out and tell me why I'm wrong, because I'd love to be and learn something new.