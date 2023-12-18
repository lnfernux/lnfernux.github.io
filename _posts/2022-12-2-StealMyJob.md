---
layout: post
title: I want you to steal my job
subtitle: I'm a Security Engineer (whatever that means) and maybe you want to be to? Hopefully this helps a little towards that.
tags:
  - Cloud Security
  - Azure DevOps
  - Powershell
  - Microsoft Sentinel
  - Azure
  - AWS
  - GCP
published: true
author: author_infernux
image: /img/avatar-icon.png
---

# Introduction

Hello and welcome. Today I'm trying something different - I'm going to teach you how to take my job. 
Now, I'm pretty impatient when it comes to reading stuff and I like learning by doing, so I'll try to keep it short and sweet.

**me**: I'm pretty impatient

**me**: I'll try to keep it short and practical 

**narrator**: he did not, in fact, keep it short and practical

## What to expect

This post is intended to be a guide to working in a Security Engineering team in a Security Operations Center. I can only speak to my experiences and what I've done, so this will be advice from me to "me". It's free advice, so keep in mind it might not be very good.

## My experience

In order to understand why I do what I do and why I enjoy it, you need to know two things:
1. What I enjoy doing
2. What I've tried and learned from trying

### What I enjoy doing

I'm pretty interested in learning, but I'll never be the guy to do research and publish new findings. I don't need to understand things 100% to make it work, some of the aspects of the things I do can remain "automagical" to me, that's fine. In my eyes this is both a blessing and a curse; I can get stuff done quickly, but sometimes I'll misunderstand and something will stop working.

I like to build stuff, and I'd rather read a compact blogpost with 10 commands and some pictures than a long winded post with detailed explaination. I'll try to fill in the blanks myself. 

### What I've tried

* System administration
* Network administration
* Windows developer/automation engineer
* Penetration tester
* Security Engineering

### Technologies I've worked with

* Windows Server (2003, 2008, 2012, 2012R2, 2016)
* Active Directory
* Cisco routers and switches
* Palo Alto firewalls
* Powershell
* C, Assembly (don't remember anything)
* Python
* Azure
   * Azure Lighthouse
   * Microsoft Sentinel
* AWS (a little)
* Azure DevOps
   * Pipelines
* Virtualization
   * VMWare vCenter/ESXi
   * VirtualBox
   * Hyper-V
* Windows (XP, 7, 10, 11)
* Linux (Ubuntu, Kali)
* Probably a lot more that I don't remember, but these are the main things

### What did I learn?

1. I like building stuff
2. I'm not very organized and easily jump from technology to technology
3. I like trying new things
4. Because I'm a bit of a untidy person, I need something static (usually my team)

This resulted in a job in Security Engineering - it's not really defined, I do development, maintenance, I create tooling to support analysts, CTI, management and my own team, I get to design solutions etc. I think it's very much *"what you make it"*. I can do a lot of different things while staying in the same team and with the same people.

---

# Basics

To quote from [Teri Radichel's post "I Want to Be A Cloud Security Engineer"](https://medium.com/cloud-security/i-want-to-be-a-cloud-security-engineer-f2a04041d45c):

> Way too many people try to go straight into penetration testing. In my opinion, you’ll be better off if you first get a deep understanding of networking fundamentals, software development, identity and access management, operating systems, and how encryption works prior to diving in and testing for security bugs.

I agree with this sentiment in the general sense - you should learn your basics before working in Cyber Security. I believe any Cyber Security function or role can be enhanced by having some prior experience in that same field. I'm not saying this is the rule and everyone should abide by it, but if we take penetration testing as an example;

If you know how to configure Active Directory, how trusts work, how the Windows operating system work, have some basic knowledge of Powershell and system administration in general - you'll write a much better report for the blue team and chances are you'll also be better at finding weaknesses and misconfigurations, compared to someone running a NESSUS scan. 

Someone new might not know that Microsoft often bundles updates into cumulative updates, so for certain windows of time you might have different scan results in NESSUS showing missing critical patches until you updated NESSUS.

---

## Learn operating systems

You should know basic usage of Linux and Windows, including their respective terminals.

## Learn scripting

Learn a scripting language (I prefer Powershell, but Python also works very well) that you can use for both writing simple scripts and automation.

My best tip here is to have a project, something you do daily that you'd like to automate - then use google and make it in your scripting language of choice.

### Resources

* [Getting started with Powershell](https://learn.microsoft.com/en-us/powershell/scripting/learn/ps101/01-getting-started?view=powershell-7.3/?wt.mc_id=SEC-MVP-5005030)
* [Getting started with Python](https://www.python.org/about/gettingstarted/)

## Learn a hypervisor

Know how to set up and configure a hypervisor like VMWare, Hyper-V or VirtualBox (this is free) in order to spin up Virtual Machines to test and develop stuff.

### Resources

* [Getting started with VirtualBox](https://www.virtualbox.org/manual/ch01.html)

## Learn basic networking

Know how you interconnect computers, virtual and physical. Also learn what components usually exist in a network, like routers, switches and firewalls.

### Resources

* [Networking 101](https://medium.com/@softwaresisterz/networking-101-the-basics-of-computer-networks-and-the-internet-8cf82011c656)

## Learn directory structures

You should know your way around [Active Directory](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc759186(v=ws.10)) and know how it allows you to manage users, groups, computers and policies. Learn the basics about how it's built up and how you interconnect it. You should also probably know how it connects to Azure AD.

You should also learn Azure AD or similar tooling from another provider like AWS IAM.

My advice here is to look at the diagrams of the architecture and try to understand how it works. You should probably also spin up a lab using your hypervisor of choice.

### Resources

* [AWS IAM](https://aws.amazon.com/iam/?nc=bc&pg=ft)
* [Azure AD Fundamentals](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals//?wt.mc_id=SEC-MVP-5005030)
* [Building a small lab using VirtualBox by chryzsh](https://hunter2.gitbook.io/darthsidious/building-a-lab/building-a-small-lab)
* [Building a lab with ESXi and Vagrant by chryzsh](https://hunter2.gitbook.io/darthsidious/building-a-lab/building-a-lab-with-esxi-and-vagrant)

## Learn a cloud

Get into a cloud provider like Azure, AWS or GCP. I recommend trying out Azure and looking at the [AZ-900](https://learn.microsoft.com/en-us/certifications/exams/az-900/?wt.mc_id=SEC-MVP-5005030) certification and learning path:

1. [Microsoft Azure Fundamentals: Describe cloud concepts](https://learn.microsoft.com/en-us/training/paths/microsoft-azure-fundamentals-describe-cloud-concepts/?wt.mc_id=SEC-MVP-5005030)
2. [Azure Fundamentals: Describe Azure architecture and services](https://learn.microsoft.com/en-us/training/paths/azure-fundamentals-describe-azure-architecture-services/?wt.mc_id=SEC-MVP-5005030)
3. [Azure Fundamentals: Describe Azure management and governance](https://learn.microsoft.com/en-us/training/paths/describe-azure-management-governance/?wt.mc_id=SEC-MVP-5005030)

Try creating some simple services and understanding how it all works together and translate the things you've looked at earlier to the Cloud, for example networking - how does CloudX implement networks and subnets, firewalls? 

### Resources

* [Azure 30 day trial](https://azure.microsoft.com/en-gb/free/)
* [Azure for Students](https://azure.microsoft.com/en-gb/free/students/)
* [Enterprise Mobility + Security E5 Trial](https://signup.microsoft.com/get-started/signup?OfferId=87dd2714-d452-48a0-a809-d2f58c4f68b7&ali=1&products=87dd2714-d452-48a0-a809-d2f58c4f68b7)

---

# Next step

Moving on from the basics, we need to implement some security into all this.

## Learn basic cyber security principles

There's a lot to be said about what's important in Cyber Security and there's a lot of it, but know the following:

> The purpose of the cyber security principles is to provide strategic guidance on how an organisation can protect their systems and data from cyber threats. These cyber security principles are grouped into four key activities: govern, protect, detect and respond.

* **Govern**: Identifying and managing security risks.
* **Protect**: Implementing controls to reduce security risks.
* **Detect**: Detecting and understanding cyber security events to identify cyber security incidents.
* **Respond**: Responding to and recovering from cyber security incidents.

For more detailed information, look at [the cyber security principles](https://www.cyber.gov.au/acsc/view-all-content/advice/cyber-security-principles) by Australian Cyber Security Centre (ACSC). 

You should also know the [CIA](https://www.f5.com/labs/learning-center/what-is-the-cia-triad) principle:

* Confidentiality - *refers to an organization’s efforts to keep their data private or secret. In practice, it’s about controlling access to data to prevent unauthorized disclosure.*
* Integrity - *in everyday usage, integrity refers to the quality of something being whole or complete. In InfoSec, integrity is about ensuring that data has not been tampered with and, therefore, can be trusted. It is correct, authentic, and reliable.*
* Availability - *systems, applications, and data are of little value to an organization and its customers if they are not accessible when authorized users need them. Quite simply, availability means that networks, systems, and applications are up and running. It ensures that authorized users have timely, reliable access to resources when they are needed.*

Or as I drew it for another blogpost (I really am an artist):

![](/img/cia.png)

## Hack your stuff

I think that in order to better understand how things work and how to protect them, we should know in some way how to hack them. I helped write a little book on this topic on [how to hack](https://hunter2.gitbook.io/practical-hacking/part-1-how-to-hack#how-to-hack) and the general steps are as follows:

1. Enumeration
2. Vulnerability analysis
3. Exploitation
4. Privilege escalation

You can read more about it, but the idea is to use **tools or commands** to enumerate what you **currently have access to**. This can be on a computer, inside the network or an external web server. From here we analyze what we find, perform a vulnerability analysis which **usually consists of googling a lot** to see if we can spot any unpatched vulnerable software or misconfigurations. 

After this we can exploit what we find and **gain access**, or **higher privileges** as mentioned in the last step. Then we loop around and enumerate with our new access or privileges until we find what we want.

I really recommend building your own labs, or trying out a commercial solution. There's multiple Github-projects for this, and sites like [TryHackMe](https://tryhackme.com/) and [HackTheBox](https://www.hackthebox.com/) that allows you to try hacking single machines/webservers or big environments.

### Resources

* [Active Directory Hunting Lab in Azure](https://github.com/christophetd/Adaz)
* [vulnerable-AD](https://github.com/WazeHell/vulnerable-AD)
* [HackTheBox hacking guide by cryzsh and me](https://hunter2.gitbook.io/practical-hacking/part-1-how-to-hack)

## Automate everything

Take your scripting language and automate everything. Have something you do every single day? Why? **Friends don't let friends right-click publish.** 

Put your stuff into a pipeline, automate testing as far as possible, automate scheduled tasks. Automate everything. 

I wrote about [templating analytic rules](https://www.infernux.no/MicrosoftSentinel-TemplateAnalyticRules/) earlier and included some resources to get started with [a free Azure DevOps organization](https://azure.microsoft.com/en-us/products/devops/) and how to get [1800 minutes free runtime](https://aka.ms/azpipelines-parallelism-request). You can also use Github and [actions](https://github.com/features/actions).

My best advice is to identify a process you manually perform quite often and look into automating it:

* Does it have an API?
* How can I call the API using my scripting language of choice?
* Implement the script in a pipeline with secure variables (we don't hardcode tokens here)

### Resources

* [Create your first pipeline in Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/pipelines/create-first-pipeline?view=azure-devops&tabs=java%2Ctfs-2018-2%2Cbrowser/?wt.mc_id=SEC-MVP-5005030)

## Fail a bunch

Not mentioned above; I've tried a lot of stuff and most of it probably failed. Part of the reason I'm where I am today isn't because I'm very intelligent, quite the contrary. My lack of second thought often let's me click "deploy" or run a script I didn't quite understand, creating some sort of unforeseen circumstance. A simple translation might be **"I've fucked up a lot"** or something like that. The end result is that I am quite knowledgeable (at least on what **not** to do). 

Embrace errors, embrace failures. Find yourself a boss, team and company that will allow you to fail a lot and who doesn't point fingers. That's how you learn.

**Important note; always be honest.** If you don't know, you don't know. If you're unsure about something that means you're sure you're unsure, if that makes any sense?

---

# Most importantly

I don't really subscribe to the whole "pull yourself up by the bootstraps" or any sort of grindset mentality. You don't need to work 16 hours a day to make it.

That being said, you need to be curious and hungry to learn new things. If you're already working then you might need to spend some free time learning, but if you really enjoy what you're working on the you might not even feel like it's work. 

## Closing thoughts

Cyber Security is a vast field. I work in a small subsection doing things I enjoy. Try a lot of things, fail a lot and try to find out what makes you tick.

Also - if you have any questions, feedback or want to discuss something relating to this post (or anything else related to security, really) - feel free to reach out. There's an email (which I check rarely) supplied in the [about me](https://www.infernux.no/aboutme/) section, or you can find me on LinkedIn, Discord etc.
