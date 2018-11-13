---
layout: post
title: Securing Windows Server - Protect credentials and create security baselines
subtitle: 'Hiding credentials from baddies and baselining things'
tags:
  - powershell
  - security
  - windows
  - server
  - credentials
  - security baseline
  - securing windows server
published: true
---

Let's go on a ride together!

### Chapter 1, Part 3: Protect credentials and create security baselines

The world of security is always changing and that's also the case for Microsoft. To follow all their updates, new products, what's retiring and namechanges please use the following link to [stay updated](https://blogs.technet.microsoft.com/secguide/) on all their blogs and updates. Here they discuss updated baselines and so much more.

#### What is Credential Guard

Kerberos, NTLM, and Credential manager isolate secrets by using virtualization-based security. Previous versions of Windows stored secrets in the Local Security Authority (LSA). Prior to Windows 10, the LSA stored secrets used by the operating system in its process memory. With Windows Defender Credential Guard enabled, the LSA process in the operating system talks to a new component called the isolated LSA process that stores and protects those secrets. Data stored by the isolated LSA process is protected using virtualization-based security and is not accessible to the rest of the operating system. LSA uses remote procedure calls to communicate with the isolated LSA process.

![alt text](https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/images/credguard.png "High Level Overview")

For more information, read [this](https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard-how-it-works) great TechNet-article.

#### Determine Requirements for Credential Guard

Credential Guard uses Virtulization Based Security to store NTLM and Kerberos secrets in an isolated Local Security Authority process (LSA).

Requirements are as follows:

* 64-bit operating system
* UEFI firmware with v.2.3.1 or higher
* CPU virtulization extensions (intel VT-x or AMD-V and support of Second Level Address Translation SLAT as well)
* TPM v1.2 or 2.0

To verify credential guard, either look at system information if the Device Guard Virtualization Based Security shows as running and the Device Guard Security Services Running item lists Credential Guard it's enabled. You can also check for a virtual lsass process in the process tool.

#### Configure Credential Guard

We can use GPO, WMI (powershell) or CMD. First, check if your PC is ready for Credential Guard with [this](https://www.microsoft.com/en-us/download/details.aspx?id=53337) tool from Microsoft.

~~~powershell
DG_Readiness_Tool_v3.5.ps1 -Ready
~~~

The Device Guard Readiness tool is designed to check a number of requirements for creating a PC that supports a variety of security enhancement features. [Read more about it here.](https://docs.microsoft.com/en-us/windows-hardware/drivers/driversecurity/use-device-guard-readiness-tool)

##### Configure using GPO

From the Group Policy Management Console, go to

>Computer Configuration -> Administrative Templates -> System -> Device Guard.

* Double-click Turn On Virtualization Based Security, and then click the Enabled option.
* In the Select Platform Security Level box, choose Secure Boot or Secure Boot and DMA Protection.
* In the Credential Guard Configuration box, click Enabled with UEFI lock, and then click OK. If you want to be able to turn this off remotely, choose Enabled without lock.

This is all we need GPO-wise.

##### Configure using registry

Enable virtualization-based security:

First go to the following regkey

>HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\DeviceGuard.

Then, add a new DWORD value named EnableVirtualizationBasedSecurity. Set the value of this registry setting to 1 to enable virtualization-based security and set it to 0 to disable it. After that, add a new DWORD value named RequirePlatformSecurityFeatures. Set the value of this registry setting to 1 to use Secure Boot only or set it to 3 to use Secure Boot and DMA protection.

Enable Windows Defender Credential Guard:

First go to the following regkey

>HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\LSA.

Then, add a new DWORD value named LsaCfgFlags. Set the value of this registry setting to 1 to enable Windows Defender Credential Guard with UEFI lock, set it to 2 to enable Windows Defender Credential Guard without lock, and set it to 0 to disable it. After this we can close Registry Editor.

To verify credential guard, either look at system information if the Device Guard Virtualization Based Security shows as running and the Device Guard Security Services Running item lists Credential Guard it's enabled. You can also check for a virtual lsass process in the process tool.

Windows 10 and Windows Server 2016 have a WMI class for related properties and features: Win32_DeviceGuard. This class can be queried from an elevated Windows PowerShell session by using the following command:

~~~powershell
Get-CimInstance -NameSpace root/Microsoft/Windows/DeviceGuard -classname Win32_DeviceGuard

AvailableSecurityProperties                  : {1, 3, 5, 6}
CodeIntegrityPolicyEnforcementStatus         : 0
InstanceIdentifier                           : 4xx8042-1667-41b8-bdd1-e80fad1cce71
RequiredSecurityProperties                   : {0}
SecurityServicesConfigured                   : {0}
SecurityServicesRunning                      : {0}
UsermodeCodeIntegrityPolicyEnforcementStatus : 0
Version                                      : 1.0
VirtualizationBasedSecurityStatus            : 0
PSComputerName                               :
~~~

The output of this command provides details of the available hardware-based security features as well as those features that are currently enabled.

#### Weakness in Credential Guard

* Local accounts and microsoft accounts not protected (credguard works with ad credentials only)
* Third party cred mgmt software might not be compatible
* Key loggers
* Physical attacks
* Malware that uses currently logged on users creds
* Digest and CredSSP credentials

This is just a bullett list, but for more in-depth please check out [Windows Defender Credential Guard protection limits](https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard-not-protected-scenarios) and [considerations when using Windows Defender Credential Guard](https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard-considerations).

#### Implement NTLM blocking

NTLM blocking is just preventing NTLM being used for authentication in your domain.

You can use several GPOs:

* Network Security
    * Restrict NTLM
        * NTLM Authentication in this domain
		* Incoming NTLM traffic
		* Outgoing NTLM traffic to remote servers
		* Audit NTLM Authentication in this domain
		* Audit incoming NTLM traffic

The idea with NTLM  blocking is to audit first, then see what effects blocking brings to your system. [This article from 2009](https://blogs.technet.microsoft.com/askds/2009/10/08/ntlm-blocking-and-you-application-analysis-and-auditing-methodologies-in-windows-7/) goes over the concepts for how to go about this, but the core concept as mentioned is to audit for a long time before making any changes and implementing policies.

#### Install and configure Security Compliance Manager

Important to note, from 2017 and onwards [Microsoft retired SCM](https://blogs.technet.microsoft.com/secguide/2017/06/15/security-compliance-manager-scm-retired-new-tools-and-procedures/).
This WILL impact the exam. The reference material I worked with referenced SCM a bunch, but there "new kid on the block" is the [Security Compliance Toolkit](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-compliance-toolkit-10), or SCT for short.

You need to download SCM v4.0 to configure WS16 and W10
You don't have to install on a server, you can perform all baseline conf work from your windows 10 administrative workstation. The SCM.exe is around 130mb.

Installing SCM v.4.0:

1. Install Microsoft Visual C+ 2010 x86 redist (SCM does this on it's own)
2. .NET Framework 3.5 needs to be installed (SCM does not do this on it's own)
3. On the installer wlecome page make sure to leave the option always check for scm and baseline updates checked, scm team updates baselines regularly
4. Accept license agreement terms
5. SCM tool is backed by a SQL server database, you can either point the installer to an existing local or remote SQL instance, or let the installer deploy a SQL Server 2008 express  edition on the local computer.
6. SCM MCC should now show up

To update baselines, click file and "check for updates"
In the templates pane, you have three columns, for each GPO , three states exists:

1. Default - ship with the server 2016 by default
2. Microsoft - microsoft recommended security baseline
3. Custom - settings contained in your customized version of the template, if you have one

This material is outdated at the time of writing, please refer to the next chapter on SCT.

#### Security Compliance Toolkit

What is it? According to the aforementioned TechNet-article it is 
>a set of tools that allows enterprise security administrators to download, analyze, test, edit, and store Microsoft-recommended security configuration baselines for Windows and other Microsoft products.

To download the SCT please use [this official download link](https://www.microsoft.com/en-us/download/details.aspx?id=55319).

#### Create, view and import security baselines

In order to create your own, you need to duplicate one of Microsoft Read-Only baselines. A quick way to do this is to expose a security setting and click the "Customize this setting by duplicating the baseline" hyperlink. This baseline will now show up under custom baselines.

#### Import and compare security baselines.

We'll export a GPO of a DC, import it into SCM and then compare the deployed GPOs security settings with that of a stored baseline.

1. Open an elevated powershell console and run the powershell command
    * Backup-Gpo -Guid XXXXXXXXXXXXXXXXXX -Path C:\backups
    * To find GUID, run Get-GPO -All -Domain  domain.com
2. Action pane in SCM , click GPO Backupr form the import section
3. In the Browse for folder dialog, select the backed up DC gpo 
4. You'll see a new section in the baseline library tree called GPO import
5. Comparison - go to  the action pane, find the baseline section, click compare/merge
6. In the comparison dialog, chose a builtin or custom sec baseline and click ok
7. If you have excel you can export the results to a spreadsheet for offline analysis

For more information on security baselines in general, check out [this article](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-security-baselines)

#### Deploy configurations to domain and non-domain-joined servers

The most common method to deploy a custom sec baseline is to export that baseline as a GPO backup folder, you can then import that to a standalone server using PowerShell or to a domain-joined server using Group Policy mgmt console.

LocalGpo.wsf is a legacy Windows Script File that is a useful Swiss Army Knife for deploying SCM-created sec baselines to non-domain-joined servers. It's now known as LGPO.exe, for more information take a look at [this](https://blogs.technet.microsoft.com/secguide/2016/01/21/lgpo-exe-local-group-policy-object-utility-v1-0/) TechNet-article.

~~~powershell
.\LGPO.exe /g 'GPO_Backup_Folder_Path'
~~~

### Links

[Enable virtualization-based protection of code integrity](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-exploit-guard/enable-virtualization-based-protection-of-code-integrity)

[Manage Windows Defender Credential Guard](https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard-manage)