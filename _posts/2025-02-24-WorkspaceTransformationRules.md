---
layout: post
title: 'Two very useful workspace transformation rules'
subtitle: 'This post will show you two very useful workspace transformation rules that you can use to save money on your data ingestion in Microsoft Sentinel.'
tags:
  - Cyber Security
  - Microsoft Sentinel
  - Azure
  - Log Analytics
  - Workspace Transformation Rules
  - Data Collection Rules
published: true
author: author_infernux
image: /img/sentinel.png
---

Workspace transformation rules are defined in data collection rules and use kusto query language (KQL) in order to transform data in the cloud pipeline, before ingestion into Microsoft Sentinel.

![](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/media/data-collection-transformations/transformation-overview.png)

Doing it this way requires you to 
> "...create a new DCR using its JSON definition or add a transformation to an existing DCR."

Now, what about tables that don't use DCR for data ingestion? Well, we have something else we can use, namely the *workspace transformation data collection rules*, which is a real mouthful. 

> "The purpose of this DCR is to perform transformations on data that does not yet use a DCR for its data collection, and thus has no means to define a transformation." - [Microsoft Docs](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/data-collection-transformations#workspace-transformation-dcr//?wt.mc_id=SEC-MVP-5005030)

![](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/media/data-collection-transformations/workspace-transformation-dcr.png)

## Rule 1 - Filter `AADNonInteractiveUserSignInLogs`

This rule is useful if you have use `AADNonInteractiveUserSignInLogs` in your workspace. Each of these entries contain the `ConditionalAccessPolicies` field, which will always contain _all_ CA-policies, applied or not. This can be a lot of data, and if you're not using it, you can filter it out. 

To do that, simply head over to your log analytics workspace, find the **Tables** menu under **Settings** and locate `AADNonInteractiveUserSignInLogs`. 

From here, click the three dots and select **Create Transformation**. If you don't already have a data collection rule for workspace transforms, you'll need to create one. If you do, simply proceed with **Next**. 

In **Schema and transformation**, click **Transformation Editor** and paste the following KQL query:

```kql
source
| project-away ConditionalAccessPolicies
```

Hit **Apply**, **Next** and **Create** and you should be all done.

## Rule 2 - Filter `SecurityEvent`

If you're ingesting data from Active Directory, you'll have some `SecurityEvent`-logs in your workspace. These logs are quite verbose and contain a lot of data that you might not need.

The number one field we can filter out is `EventData`, which contains the entire event in it's raw xml-form. So in essence, you're storing the same data twice.

Follow the recipe from above, but use this query instead:

```kql
source
| project-away EventData
```

And that's it! You've now saved some money on your data ingestion in Microsoft Sentinel.

Got any good transformation rules? Feel free to reach out and share!