---
layout: post
title: Creating smart Data Collection Rules by parsing EventIDs from Analytic Rules
subtitle: Data Collection Rules allows us to create custom filters based on XPath-queries. If we do this based on active Analytic Rules, we can create DCRs that only ingest the data we actually have detection for. 
tags:
  - Microsoft Sentinel
  - Active Directory
  - Azure Monitor Agent
  - Azure Arc
  - Data Collection Rules
  - Windows Security Events
published: true
author: author_infernux
image: /img/sentinel.png
---

* Use windows event log events generated from enumeration to build data collection rules
* Deploy Azure Monitor Agent, create data collection rules and start gathering data
