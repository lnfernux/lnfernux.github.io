---
layout: post
title: Configure Windows Defender Firewall (act 2)
subtitle: Securing Windows Server - Chapter 3, Part 2
tags:
  - powershell
  - security
  - windows
  - infrastructure
  - networking
  - windows defender firewall
  - software defined networking
  - distributed firewall
  - group policies
  - connection security groups
  - securing windows server
published: false
image: /img/fw.png
---

A short post!

# Chapter 3, Part 2: Configure Windows Defender Firewall (act 2)

## Overview

![firewall](/img/fw.png "Windows Defender Firewall logo")

In this post we'll pick up where we left things last time. We're jumping straight into the world of network security rules and 

### Configure connection security rules

#### What are connection security rules?

While firewall rules provide security by checking traffic that are allowed to reach/leave the server/client, connection security rules are concerned with a different type of security. Everyone who's done a semester of CompSci have heard the "CIA" rule;

* Confidentiality - "data is encrypted such that only authorized parties can decrypt and examine it"
* Integrity - "data is guaranteed to be consistent from sender to reciever" 
* Authentication - "the identity of each communicating party is verified"

This is where IPSec comes into the picture. Often mistakenly refered to as a protocol, this industry standard and protocol suite provides various protections for IP traffic. 
The 744 exam ref summarizes this for us very well; "IPv4 traffic was never designed to provide CIA, IPv6 has IPSec built in, but how many businesses have shifted to IPv6 exclusively?"

So, let's get a move on. For today, the exam itself is concerned with only two protocols from IPSec, namely;

* Authentication Header (AH) - provides authentication, anti-replay and integrity, but no encryption.
* Encapsulating Security Payload (ESP) - provides data encryption on the IP packet's payload without offering authentication.

Now, for real life purposes, keep in mind that implementing IPSec protocols such as ESP adds a minimal overhead on your internal network. Is encryption on internal network traffic something you need? Depends. Enabling only AH for authentication, however, results in no overhead at all.

#### IPSec connection security type rules

Creating an IPSec rule first and foremost includes knowing of and understanding different rule types:

* Isolation - restricts connections based on auth criteria
* Auth exemption - blocks auth request from specified nodes
* Server-to-server - auths connections between two specified nodes
* Tunnel - auths connections between two VPN gateway nodes

For more information in depth, see [this](https://support.microsoft.com/en-us/help/942957/security-rules-for-windows-firewall-and-for-ipsec-based-connections-in) link. It might seem outdated, but was last updated (at the time of writing) April 17th, 2018.

#### Group Policy

* First, navigate to the following path in your desired GPO (recommended that you create a new one)

~~~CONSOLE
Computer Configuration\Policies\Windows Settings\Security Settings\Windows Defender Firewall with Advanced Security\Connection Security Rules
~~~

* Right-click the node and select New Rule.

#### GUI Console

#### Powershell

### Configure the Windows Defender Firewall for applications

### Configure authenticated firewall exceptions

### Importing and exporting settings

#### Standard disclaimer

The world of security is always changing and that's also the case for Microsoft. To follow all their updates, new products, what's retiring and namechanges please use the following link to [stay updated](https://blogs.technet.microsoft.com/secguide/) on all their blogs and updates. Here they discuss updated baselines and so much more.

Most of this writing is strongly influenced by the 70-744 Exam Reference - so there will be a lot of similarities. It's a great book, please check it out.