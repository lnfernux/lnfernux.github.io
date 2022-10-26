---
layout: post
title: IP Whitelisting in Conditional Access
subtitle: Quick introduction to IP Whitelisting in Azure AD Conditional Access and some thoughts around how to (not) implement it.
tags:
  - Cloud Security
  - Microsoft Sentinel
  - Playbooks
  - Storage account
  - Azure DevOps
  - Conditional Access
  - Azure AD
published: true
author: author_infernux
image: /img/azad.png
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

### Conditional access named location policy for accessing certain applications

option 1, whitelist country (vulnerable to that country, vpn)
option 2, whitelist physical locations, this provides a decent option but guest networks etc might be an issue and this does not support work from home
option 3, per person whitelisting, either manual (from admin/engineering team) or secure solution where users can add their own ip to the whitelist. This is the most restrictive and can be a bigger workload if no self service solution, if self service then obviously that needs to be well protected (also catch 22 if the self service solution requries azure ad logon, but is part of conditional access policy - so make sure that access to that application is not behind the ca policy with named locations)

## Summary

These are just my thoughts on the matter of IP whitelisting in certain scenarios in the Azure/Azure DevOps stack. Any errors or better ideas are welcomed. I do also recognize that not all these solutions scale very well to large businesses without proper self service solutions (and even then), but I feel like certain applications or access need to be behind a collection of restrictive policies, IP whitelisting being one of them.