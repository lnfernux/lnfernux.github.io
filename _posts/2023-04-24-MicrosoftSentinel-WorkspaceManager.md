---
layout: post
title: Microsoft Sentinel Workspace Manager
subtitle: Short introduction to the new preview, what it does and what I think of it currently.
tags:
  - Cloud Security
  - Microsoft Sentinel
  - Workspace Manager
  - MSSP
published: true
author: author_infernux
image: /img/sentinel.png
---

# Introduction

**Workspace Manager** is a new cool feature that's just reached public preview. In this short post I'll be describing quickly what it allows us to do, how it works and what I think of the feature currently.

# What can Workspace Manager do?

Currently there are some limitations as to what it can do, but it can manage the following components:

* Analytic rules
* Automation rules
* Workbooks
* Kusto functions (hunting queries, parsers etc)

Obviously playbooks missing is a big deal, and workspace manager can't delete content. It's all about part of the [known expectations](https://learn.microsoft.com/en-us/azure/sentinel/workspace-manager#known-limitations) however, so expect this to probably be remedied in the future.

---

# Architecture

Workspace Manager utilizes Azure Lighthouse. It requires the person configuring it to have Microsoft Sentinel Contributor roles on all workspaces you want to manage, and the workspace you manage from.

In the [official documentation](https://learn.microsoft.com/en-us/azure/sentinel/workspace-manager) Microsoft presents the following three architectures:

![](/img/MicrosoftSentinel/architectures.png)

For an MSSP this shows that we can:
* Simply manage a few workspaces
* Co-manage workspaces
* Create nested configurations for complex customer environments

---

# Thoughts and usage

## First impressions

Please note, I'm comparing this to a solution built on the original [Sentinel as Code](https://github.com/javiersoriano/sentinelascode) concept, which allows for push/pull/delete of all components in Sentinel, as well as other Azure-components. 

### Pro's

* The configuration of it is fairly simple.
* You can easily configure groups to manage content a bit more granularly. 
* Pushing content was a breeze and I had no errors on this part. 
* Nesting workspaces using N-Tier was also nice.
* You can set up scheduled updates.



### Con's 

* Missing Playbooks.
   * You can push them, but only if not attributed or attached to analytics and automation rules. This is _all_ of our playbooks, at least.
* No option to manage content created in the managed workspaces currently.
* Pushing using this feature requires constant access to customers workspaces as Microsoft Sentinel Contributor, so this does not support a limited access PIM system for automation.
   * Small note on this: I guess you can set this up with a SPN isntead of a user account, which would work fine and actually would be preferable. 
* No option to delete in the managed tenants from the central workspace.



### Needs more testing

* Last time I tested this I observed content being overwritten on a schedule from the central workspace - this would mean that **local tuning for specific customers would not be possible**. We would need to host multiple versions of the same rule locally and deploy using different groups.

---

## Thoughts

I can see this being a very useful feature. Having the ability to manage some content from a central workspace and have it being constantly pushed and overwritten on a schedule in order to make sure there's **no configuration drift** (or people making changes to the wrong thing, etc) would be nice. Unfortunately, this mostly goes for playbooks which _isn't fully supported_.

Compared to a Sentinel as Code solution **it's probably not strong enough yet to take over fully for most MSSP providers**, but if you are just getting into managing Sentinel as Code then this feature combined with the repository feature on a central workspace could be a good way to get started managing multiple workspaces. 

My wishlist would obviously include **full playbook support** and the option to **allow for local changes on certain content**, as well as the ability to delete things. If we look into other aspects of Sentinel that it might help manage, a central **health view** of analytic rules, automation and data ingestion would be very high on the wishlist!

![](/img/me3.gif)


