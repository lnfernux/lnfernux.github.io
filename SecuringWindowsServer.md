---
layout: page
title: Securing Windows Server 2016
subtitle: A summary of the most important aspects from the 70-744 exam
---

# Securing Windows Server

## About the guide

In this "guide" I'll write about the things I summarized for myself while studying for the "Securing Windows Server 2016" exam. I will present the technologies, concepts Microsoft use and outline basic commands and actions for implementing these. In other words, this is not a guide in the true sense of the word. This is my ramblings and experiences reading the study material, with additional research. Take it with a grain of salt.

I highly recommend checking out the 70-744 Exam Ref (it's )

Please note, however:

* The steps I go through in my guide are correct at the time of publishing, but technologies change. If you see an error, please contact me.
* This is NOT a how-to or step-by-step - I will however try to link to those types of guides in every chapter I can find one.

### Chapter 1 - Server Hardening

A server is a soft-target if operating system files installed from non trusted source, system is not current with system and security patches, administrators have weak passwords or if file systems don't use NTFS and are unencrypted. Chapter 1 will look at implementing solutions to deal with this.

[Part 1 - Disk and file encryption](https://www.infernux.no/2018-10-22-securingwindowsserver11/)

[Part 2 - Implement server patching, updating solutions and malware protection](https://www.infernux.no/2018-10-28-securingwindowsserver12/)

[Part 3 - Protect credentials and create security baselines](https://www.infernux.no/2018-11-01-securingwindowsserver13/)

### Chapter 2 - Secure a Virtualization Infrastructure

Here we take a look at implementing a Guarded Fabric solution, complete with HGS, shielded VMs and the likes.

[Part 1 - Implement a Guarded Fabric solution](https://www.infernux.no/2018-11-12-securingwindowsserver21/)

[Part 2 - Implement Shielded and encryption-supported VMs](https://www.infernux.no/2019-01-09-securingwindowsserver22/)

### Chapter 3 - Secure a network infrastructure

It's time for some networking! We will look at the Windows Firewall, setting up a software-defined Distributed Firewall and securing network traffic.

#### COMING SOON (most likely during february/march 2019)

[Part 1 - Configure Windows Firewall](https://www.infernux.no/SecuringWindowsServer/)

[Part 2 - Implement a software-defined Distributed Firewall](https://www.infernux.no/SecuringWindowsServer/)

[Part 3 - Securing network traffic](https://www.infernux.no/SecuringWindowsServer/)

### Chapter 4 - Manage privileged identities

TBA

### Chapter 5 - Implement threat detection solutions

TBA

### Chapter 6 - Implement workload-specific security

TBA

### Scripts and more

[Securing Windows Server (70-744) scripts](https://www.infernux.no/2018-09-18-powershell/)

### Bytesize-pieces 

I'm planning to do a more in depth guide on how to set up and test some of these technologies on your homelab if possible. 
I'll gladly take suggestions on what to do first, please contact me if you want to see something here.

## More resources

The official Microsoft exam page for 70-744: Securing Windows Server 2016 can be found [here.](https://www.microsoft.com/en-us/learning/exam-70-744.aspx)

[ADSecurity.org](https://adsecurity.org/) is a great resource for securing your Windows Servers and AD-environment.

Also check out the [/r/WindowsSecurity](https://www.reddit.com/r/WindowsSecurity) subreddit.

Last, but not least, my friend Chryzsh has a [great gitbook](https://hunter2.gitbook.io/darthsidious/) with both blue- and red-team resources and information.