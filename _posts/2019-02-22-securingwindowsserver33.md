---
layout: post
title: Scure network traffic
subtitle: Securing Windows Server - Chapter 3, Part 3
tags:
  - powershell
  - security
  - windows
  - infrastructure
  - networking
  - windows firewall
  - group policies
  - connection security groups
  - securing windows server
published: false
image: /img/ws2.png
---

Talking about the guarded fabric and more!

### Chapter 2, Part 2: Implement a Guarded Fabric solution

The world of security is always changing and that's also the case for Microsoft. To follow all their updates, new products, what's retiring and namechanges please use the following link to [stay updated](https://blogs.technet.microsoft.com/secguide/) on all their blogs and updates. Here they discuss updated baselines and so much more.

In this part we will look at shielded and encryption supported VMs and how to create and deploy them, using PowerShell. You can also use [Virtual Machine Manager](https://docs.microsoft.com/en-us/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-tenant-deploys-shielded-vm-using-vmm) or [Windows Azure Pack](https://docs.microsoft.com/en-us/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-shielded-vm-windows-azure-pack), but we won't get into that here.

Running shielded VMs consists of two main components:

* The Host Guardian Service (HGS) provides attestation and key protection
* Approved and healthy Hyper-V hosts - also known as guarded hosts

![alt text 2](https://docs.microsoft.com/en-us/windows-server/security/media/guarded-fabric-shielded-vm/guarded-host-hgs-plus-host-diagram-basic.png "Basic Overview of HGS + Guarded Host + Shielded VM")


### Links