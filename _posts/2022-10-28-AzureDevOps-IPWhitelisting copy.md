---
layout: post
title: IP Whitelisting in Azure DevOps pipelines
subtitle: Quick introduction to IP Whitelisting in Azure DevOps and some thoughts around how to (not) implement it.
tags:
  - Cloud Security
  - Storage account
  - Azure DevOps
published: true
author: author_infernux
image: /img/devops.png
---

## Introduction

Many of the components we use in Azure (and other cloud providers) allow us to whitelist IP adresses that are allowed to communicate and use the service. This is useful for normal access in order to enforce access to sensitive data only from approved locations, but can also be used as an added layer of security. 

Originally this post was going to be about adding IP adresses to Microsoft Sentinel playbooks using webhook-triggers to avoid calls from unauthorized locations, but in writing I found out some misconceptions I had and this post is the result of that.

The general idea is that in order to access resource X you will need to fullfill certain conditions Y. In this scenario a IP in the whitelist will be the condition:

```mermaid
graph LR
    R[Resource X]
    C[Conditions Y]
    U[User] -->|Fullfills| C --> |Access| R
```

## Scenarios

To make this as practical as possible I've created 3 scenarios from real life where can look at how to implement IP whitelisting and hopefully avoid making any errors or wrong assumptions while doing so. 


### 2. Sending data to an Azure storage account from an Azure DevOps pipeline task

hydra-diagram of solution

option 1, whitelist azure build agents (this leaves us vulnerable for anyone using azure build agents to try to abuse the service)
option 2, give the service connection enough access to whitelist ips and add two new tasks, one for adding the build agents current ip and one for removing it - please do note that splitting pipelines into multiple stages will cause it to be run on two agents, so these steps must be defined in each stage.


## Summary

These are just my thoughts on the matter of IP whitelisting in certain scenarios in the Azure/Azure DevOps stack. Any errors or better ideas are welcomed. I do also recognize that not all these solutions scale very well to large businesses without proper self service solutions (and even then), but I feel like certain applications or access need to be behind a collection of restrictive policies, IP whitelisting being one of them.