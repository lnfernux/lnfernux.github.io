---
layout: post
title: Hardening Azure Active Directory Part 2
subtitle: Continuing on from where we left in part 1, we look at using Conditional Access policies to limit enumration - and some other attack/pivot paths. 
tags:
  - Microsoft Sentinel
  - Azure Active Directory
  - Hardening
  - Logging
  - App registration
  - Enterprise applications
  - Consent
published: true
author: author_infernux
image: /img/azad.png
---

## Introduction

This is part 2 of a series on hardening Azure AD. In [the first post](https://www.infernux.no/AzureActiveDirectory-GeneralHardening/) we talked about some common attack paths that can be partially mitigated using free tols in Azure AD, namely how to avoid external and internal application consent attacks with tools like [o365-attack-toolkit](https://github.com/mdsecactivebreach/o365-attack-toolkit) and limiting portal access to Azure AD.

This time we'll continue where we left off - blocking enumeration using conditional access policies, before looking into some other common attack or pivot paths we can close down using CA policies.

## Blocking access to Microsoft Azure Management


https://docs.microsoft.com/nb-no/azure/active-directory/conditional-access/concept-conditional-access-cloud-apps#microsoft-azure-management