---
layout: post
title: '5 Years On - The Microsoft Sentinel Experience'
subtitle: 'Around 5 years ago, Microsoft announced the general availability of Azure Sentinel. This post aims to assess how far we along we have come - the good, the bad and the ugly.'
tags:
  - Cyber Security
  - Security Monitoring
  - Microsoft Sentinel
  - Microsoft Security
published: true
author: author_infernux
image: /img/sentinel.png
---

To briefly prepare you for what you are about to read and the context in which it is written; I work for an MSSP and have been working primarily with Microsoft Sentinel and the Microsoft stack since the release of Sentinel. Some of the things I will talk about and deem to be of a certain value, hit or miss, might be totally different for you and your team. I might forget to write about something, but I've tried to include everything that is top of mind for me. Other than that, I don't think Microsoft Sentinel needs much introduction, if you're here and you're reading this you probably know what it is and what it does.

***I will put a little disclaimer here:***

**This is my personal experience and opinion.** I'm writing this because I really like the product and I really like the community around it. However, as most things, it's not perfect. Some of the features that have been developed are a bit of a miss from my perspective (my word is not the truth, feel free to disagree) and I think that by voicing my opinion I can help provide important feedback for developing the product. **I'm very much looking forward to the future of Microsoft Sentinel and the Microsoft Security stack, and I'm excited to see where it goes.** 

During this post I'll also touch on some products that are not directly related to Sentinel, but are part of the Azure stack and work closely with Sentinel and MSP/MSSPs. This will be tools like Azure Lighthouse and Azure Monitoring Agent, the Defender suite and privileged identity management, amongst others.

![](/img/gbu.jpg)

## Table of Contents

1. [The Good](#the-good)
2. [The Bad](#the-bad)
3. [The Ugly](#the-ugly)


# The Good

![](/img/thegood.png)

## Everything as Code

**It all started with** [**this blogpost**](https://techcommunity.microsoft.com/t5/microsoft-sentinel-blog/deploying-and-managing-microsoft-sentinel-as-code/ba-p/1131928) from Microsoft Sentinel master [Javier Soriano](https://techcommunity.microsoft.com/t5/user/viewprofilepage/user-id/66621). Coming from a click-ops SIEM to a tool like Microsoft Sentinel where mostly everything was available through the API was a big upgrade. 

This allows you to automate the deployment progress and speed it up significantly, being able to [push](https://www.infernux.no/MicrosoftSentinel-AutomationRules/) and pull configuration from workspaces you have access to.

I've also had some input from my friend [Sami Lamppu](https://samilamppu.com/author/samilamppu/) regarding this topic, pointing out the [Microsoft Sentinel All-in-One solution](https://techcommunity.microsoft.com/t5/microsoft-sentinel-blog/announcing-microsoft-sentinel-all-in-one-v2/ba-p/3800037) as something that also helps speed up deployment and configuration of Sentinel.

## The Community

Microsoft Sentinel has one major advantage over something like Chronicle - it's the community. It's not unheard of to have big community around tools (looking at you, Splunk), but actually having one is a big boon. **As a normal user you can easily create a free Azure account and get access to a log analytics workspace to install Sentinel by yourself.** You can play around with ingesting logs, creating analytics rules and workbooks and implement whatever you'd like to try. 

Now combine this with a very active community, both in terms of creating ready to deploy content over at the [Azure Sentinel](https://github.com/Azure/Azure-Sentinel) repository and generally creating content around everything from playbooks to KQL in the form of blogs, tools and videos - **it's easy to get started and get help if you need it.**

Lastly - we have to mention the Security Customer Connection Program. This is a program where you can sign up to get early access to new features and provide feedback to the product group. This is a great way to get a sneak peak at what's coming and to have a say in the direction of the product. If you end up joining, say hello to me in the community teams and I'll see you there!

## The Microsoft Stack

No surprises here - but **integrating with the Microsoft stack is a breeze**. If you're already using Microsoft 365, Defender for X (not sure how many there actually are, at this point I'm too afraid to ask) and other Microsoft products, you're in for a treat. 

## Parsing

One of my team members came over from Splunk and he was amazed at how **easy connecting logs and hitting the ground running was in Sentinel**. Someone has had to do some work for it to be this easy, but as a user you usually don't have to worry to much about it. For Cloud based SIEMs I think this might be the new normal, but it was a big step up from what we were used to.

## Data Connectors in my heart

This brings me to data connectors. 1st party connectors being mostly click to connect and most third parties being well documented and already parsed means that deploying new log sources is a breeze (unless there's not data connector).

## Azure Lighthouse 

Azure Lighthouse has me in two minds. First of all, it's just great. It's great for us as an MSSP to be able to **define templates that dictate our level of access to a customers tenant**. We can keep it least privilege, and we **can't escalate beyond that without the customer's consent** (by adding a new template), so it's a great way to keep the customer in control of their own environment.

We can also automate the process of creating these configuration files for Lighthouse, since we can define templates for services. 

Now, for the bad part - you can [jump here](#azure-lighthouse-part-2) to read more on that.

To read more about Azure Lighthouse:

- [Azure Lighthouse access design considerations](https://www.infernux.no/AzureLighthouse-DesignConsiderations/)
- [Azure Lighthouse 101](https://www.infernux.no/AzureLighthouse-101/)

Also check out [this blogpost](https://techcommunity.microsoft.com/t5/itops-talk-blog/manage-multiple-azure-tenancies-with-azure-lighthouse/ba-p/833928) by Microsoft explaining more of the use cases.

# The Bad

![](/img/thebad.png)

## Cost

Not unique to Microsoft Sentinel, but the cost for ingesting logs can be **quite high**. It's not a problem if you're a small company, but if you're a large company you can easily rack up a big bill. You have to make informed decisions about what logs to ingest. I've written [on the topic of estimating costs](https://www.infernux.no/MicrosoftSentinel-CostEstimation/) before, and it's a good place to start if you're looking to get an idea of what you're looking at.

There are options for cheaper storage tables, but on the **configuration side there's something to be desired**. This is heavily linked with the next couple of points, which is related to export, cold storage and the Azure Monitoring Agent.

I've also written a bit about [security strategy](https://www.infernux.no/SecurityStrategy/) and how to approach security monitoring in a way that makes sense for your organization, long term. It's a bit more high level, but it also touches on the cost aspect of security monitoring in relation to Microsoft Sentinel.

## Export and cold storage

The ability to **store logs in archive natively in Sentinel is great**, but search is very slow and the restore function is quite pricey. Exporting to storage accounts works well, but actually using the data you output easily is a bit of a hassle. You can't easily search through the data, and you can't easily restore it back to Sentinel. Same goes for exporting to ADX - it's a bit of a hassle to set up and maintain. 

**A tool like Cribl can send data to cold storage and rehydrate it back to Sentinel.** You also have the option of configuring sources and destinations to easily send data wherever it needs to be; more on that in the next point.

## Azure Monitoring Agent

Don't get it twisted, **the Azure Monitoring Agent is a great tool**. It's easy to deploy (for the most part), but where the issue arises is the management. Ingesting logs via AMA is alright if done via the correct data connector, but there is something lacking in the ability to **define a source (the agent) and then visualize the data in a way that makes sense before choosing a target destination**. A lot of data engineering tools have given us this option in user friendly interfaces, and it's something that I think is missing in Sentinel and AMA.

This also goes double for the **workspace transformations - there are some great tools and workbooks out there, but the fact that this isn't implemented in a UI that allows you to see what workspace transforms have been applied and what the data looks like before and after is a missed opportunity.**

## Workbook API

What can you say - **it's not great**. You are **able to push workbooks as code**, but you **can't pull them down the same way you do other components** (last time I checked - if anyone has a workaround for this, hit me up). There's also the fact that you can't download PDF or PNG versions of workbooks, which is a bit of a bummer since it can only be done manually through the UI. Some improvements on the API side would be great, and a fever dream of mine is to be able to have **some sort of integration with PowerBI for more advanced reporting.**

## Azure Lighthouse Part 2

I mean, it's good for what it is, but I can't help but feel like **development has stalled a bit** on Azure Lighthouse. It's a great tool, but it's not without its issues. Some of the more annoying ones are:

- No custom roles
- No Entra ID roles
- Integration with some APIs are not possible (like the Upload Indicators API)

I'm sure there are more, but these are the ones that come to mind. Hopefully we can see some improvements in the future.

This also goes for the marketplace - creating offers there programatically can't be done (last time I checked, please correct me if I'm wrong). So you're stuck manually creating assignments. There are also some limitations on the number of offers you can create under a certain offer type (again, last time I checked). You have the option to create private offerings, but at that point you might as well just have your own ready to go templates and a simple parser script. 

## Defender Integration and Automation Rules

If you **ingest a large number of incidents from Defender your automation rules will start failing with internal errors**. This is mostly a load issue, and we've seen it across multiple environments. It's still not fixed as of 1st of July, but I know the product group is working on it. The reason I'm mentioning this is that **some people might not be aware that it's a thing, but you can use this query to see if you're affected**:

```kql
SentinelHealth
| where Status == "Failure"
| where Description contains "An internal server error occurred. Please contact support for assistance."
| project TimeGenerated, ExtendedProperties["IncidentName"], Description
```

Basically the cause is if there are too many incidents and changes being made to incidents (as they are updated in Defender), the automation rules will start failing. This could mean, in the extreme, that some playbooks that forward incidents to your ITSM tool will stop working. That's pretty bad.

## Workspace Manager

In theory this is a pretty cool tool - I had high hopes that I would be able to see ingestion errors, health data and other useful information in one place. **Turns out it was mostly a content management solution** without the ability to support all the content in Microsoft Sentinel. Even now, **most MSSPs will have trouble utilizing the tool because their own backend probably have more features and better support for their own content.**

I think it's a great idea, but not quite there yet. One to watch for the future.

# The Ugly

![](/img/theugly.png)

## Unified Portal Issues

The new unified portal (Defender XDR + Sentinel) has been a **bit of a mixed bag**. It's great to have everything in one place and searching across the tables from both Defender and Sentinel is great without having to pay the extra cost for the streaming connector. There's a bunch of other cool features as well, but the issue is grouping of incidents. 

I'll try to explain the context as best I can, but **as an MSSP you usually have some sort of third party ITSM tool to manage incidents.** Depending on how you are set up, usually you will have some level of severity that gets forwarded to that system, and maybe a naming scheme for controlling which rules are deployed by you and which are deployed by the customer. **To control the state of an incident, you will usually use tags.**

For tools like Defender, we can usually say that bar tuning, we will forward the incident as it arrives from Defender (given severity above a certain treshold) and that's that. 

Noooow, **the issue comes when you enable the Unified Portal**. It will **start to combine Microsoft Sentinel incidents with Defender incidents.** This means that you can get the following scenarios:

1. Incident created in Defender with low severity is updated with a high severity incident (created by the customer, so not managed by you). **This causes the incident to be updated and now forwarded, even if you normally would not look at this item.**
2. Incident created in Defender with high severity is updated with a new high severity incident from Defender. Depending on logic, a new incident will be created. **We've observed that sometimes this means that the new incident will lose all it's original tags, meaning we have no way of syncing the status of the incident with our ITSM tool.**

Other combinations of this issue will also arise, but these are the most common ones. If you're not aware of it, it can cause a lot of confusion and a lot of manual work to keep the incidents in sync. You can work around it using the Sentinel Health table, or creating external tables connecting the incidents to their ITSM ticket numbers, but that's a lot of work for something that should be handled by the tool.

### A little note on the `mto.security.microsoft.com`-portal

Another of Sami's points regarding Defender and Sentinel is the MTO-portal. **It's a great step in the right direction for "Sentinel-esque" multi-workspace management**, but it's reliant on identity governance being set up via access packages. This means there's a lot more configuration to do in terms of setting up access packages and roles, and it needs to be followed up in the governance of identities. Also, it requires the customers to trust your MFA (if they force MFA; which they should), **if not you're stuck activating roles all day to actually get any benefit from the MTO-portal**. That's a little side branch, but it's worth mentioning.

## Privileged Identity Management

Now, this isn't directly related to Microsoft Sentinel, but we use it a lot in our day to day operations. It can either be directly via Azure Lighthouse to access customers, or to access our own environment. **Now, the UI for PIM is just not great.** **It's slow, it's clunky** and it's not very intuitive. Every single activation takes forever, it's not optimized, and I'm not allowed to select and activate multiple roles and groups at once?

The same goes for the API - I get that the naming scheme is there to allow for activating roles on schedules, but it's just confusing. I have to lean heavily on resources like [this one on PIM for Groups by Marteen Rosier](https://dikkekip.github.io/posts/pim-for-groups-1/) which is a wizarding blogpost using multiple APIs to create PIM activated groups and assigning people to them.

Don't get me started on activating roles from an access package in a customers tenant.

If anyone has any good resources on this topic I'm open to learn, but there's a lot of room for improvement here (which is also very good - the products themselves are great in my opinion).

# Summary

**We've come a long way in 5 years.** ~~Azure~~ Microsoft Sentinel has been a constant through my couple of years working in the security field and I'm certainly privileged in a sense to work with a tool that is so easy to start with and that is getting a ton of development and attention from the product group. Combine that with a big community and it's no wonder I've stuck with it for so long.

We still have some ways to go, I think, mostly in terms of usability and user experience - however this is not exclusive to Sentinel and most of it comes from adjacent products that are indirectly connected that I utilize in my day to day work.

If I could have a wishlist going forward, it would be something like this:

1. **Azure Lighthouse improvements** - add support for Entra ID roles, support for more API integrations and possible integration with Unified RBAC in Defender XDR?
2. **Better data engineering options in Sentinel** - better support for AMA and workspace transformations. Mostly UX improvements, but also better support for managing data sources and destinations easily.
3. **Fix the unified portal incident merging** - allow us to opt out as a short term fix, and then work on a better solution for merging incidents from Defender and Sentinel, at least so we can persist certain tags on the original and new incidents.
4. **Update the PIM activation user experience** - make it faster, more reliable, and allow us to activate multiple roles or groups at a time. Also, add API support for activating eligible roles in other tenants please.

**That's it for now** - hopefully this has been a good read. I remembered a lot of stuff while doing my research for this post, and I also reached out to several community members to get their input on this post, so thanks to everyone who contributed some time towards this. 

**Have a nice summer vacation (for those of you who take those) and I'll see you in the next post!**

![](/img/goodriddance.jpg)