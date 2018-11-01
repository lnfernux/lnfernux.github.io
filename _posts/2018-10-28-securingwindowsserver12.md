---
layout: post
title: Securing Windows Server - Chapter 1 (Server Hardening)
subtitle: 'Part 2 - Implement server patching, updating solutions and malware protection'
tags:
  - powershell
  - security
  - windows
  - server
  - uefi
  - windows update
  - WSUS
  - securing windows server
published: true
---

### Chapter 1, Part 2: Implement server patching, updating solutions and malware protection



#### Install and configure WSUS

Windows Server Update Services can be deployed many ways. Also included as a part of SCCM.
Either as a single server standalone, or a replicated server farm. Secondary (downstream) servers pull their updates from upstream (master) WSUS server; master downloads updates of Microsoft Update over the internet.

Computer groups will help deploying and testing patches and hotfixes easier. You can have a group a "test servers" as a part of the production enviroment, but with nothing important on to see what implications a patch bring along. Summing up the benefits of WSUS:

* Saving bandwidth because local servers and clients download updates at LAN speed from downstream servers
* Improve stability by first testing, approving and/or blacklisting patches before the computers you support receive them
* Control how and when approved updates are installed in your enviroment
  


#### Installing WSUS

Follow these steps:

1. Install the WSUS role on a server
    * You can either use Windows Interal database (WID) or Microsoft SQL server (on server 2016)
    * Use the following cmdlet:
    ~~~powershell
    Install-WindowsFeature -Name UpdateServices, UpdateServices-WiDB, UpdateServices-Services, UpdateServices-API, UpdatesServices-UI
    ~~~
2. After installation, open the Windows Server Update Services console from the server manager, this starts the Complete WSUS Installation Wizard
    * You'll be asked for a update storage location, specify your desired path and type run
    * Post installation tasks take a few minutes, after which you're taken into a second wizard
        * Before you begin - verification step that asks you if WSUS servers firewall rules are configured and you're logged in with proper credentials
        * Microsoft Update Improvement Program - simple opt-in or out
        * Choose upstream server - sync with Microsoft update directly or via a upstream server already configured in the domain
        * Specify proxy server - if you're using one
        * Choose languages - select only the languages you support
        * Choose products - choose to download updates only for the operating systems and products you support
        * Choose classifications - Choose what to download, by default this is critical updates, windows defender malware definition updates and security updates. 
        * Configure sync schedule - manually specify sync with upstream partner. Choose when to perform the initial sync.
3. After initial sync completes, you're ready to define computer groups, apply approval policies and configure automatic update. All of this can be done via the Update Services MMC console
  
    
#### Create computer groups and configure Automatic Update

By default WSUS creates (but doesn't populate) a single computer group called Unassigned Computers. Let's create a new group for our infrastructure servers:

1. In the Update Services console, right click Computers, All computers node, select add computers group
2. Give the new group a descriptive name and click add

Adding computers to the group is a bit hard, we must use a GPO to point our client servers and desktop computers to a given WSUS server. Once that has been done, however, you can reassign the host by clicking the host and selecting change membership.

To point clients and servers to the right WSUS server, do the following in an AD-GPO:

1. Go to Computer Config\Policies\Administrative Templates\Windows Components\Windows Update
    * Open the Specify Intranet Microsoft Update Service Location and provide the following URLs
        * Intranet update service, HTTP(S) address of WSUS server, check IIS to see which port WSUS uses.
        * Intranet statistics server, same as above in our case.
2. In the same GPO path, open the Configure Automatic Updates policy. Here you control how often the targeted hosts query the WSUS server.

  
#### Implement Windows Defender

In Windows Server 16 defender behavior is configurable from the Update and Security pane in Settings. 

You can control real-time protection (runs defender in the background constantly), cloud-based protection (sends results to windows to help make defender better and faster at detecting), automatic sample submission (submits samples of detected malware to microsoft), exclusions (don't scan certain files / folders if you're sure they're safe), windows defender offline (you can scan the system from an alternative startup volume, but you have to install windows defender offline to do this, version info (how recent is the defitions and signature files.

  
#### Running scans from PowerShell

~~~powershell
Start-MpScan #to start a normal scan
Start-MpWDOScan #to start offline scan (if you've created an offline boot media).
~~~

  
#### Integrate Windows Defender with WSUS

Configure WSUS to automatically approve windows defender updates automatically.

1. Click options node and select Products and Classifications
2. Switch to the classifications tab and make sure Definition Updates is selected
3. Close dialog and open Automatic Approvals
4. Select Default Automatic Approval Rule (or new rule)
5. Edit Rule, for step 1, make sure the "When An Update Is In A Specific Classification" is select, in step 2, click "When An Update Is In: hyper-link" and Select "Definition Updates"

Your server should now automatically approve and download Windows Defender definition updates.
You can control defender from GPO:
>Comp Conf\Policies\Adm Temp\Windows Components\Windows Defender.

Here you can do stuff like enable headless ui mode for users, allow users to pause scans and set time of day to run scans.

  
#### Implement AppLocker Rules

AppLocker is basically whitelisting. Make sure that AppIDSvc is running on all protected servers, you can use GPOs to force-enable this service.
You can have five types of rules:

1. Executable rules (.exe and .com type files)
2. Windows Installer Rules (.msi and .msp packages)
3. Script rules (.ps1, .bat, .cmd, .vbs and .js)
4. Package app rules (universal windows apps)
5. DLL (dynamic link library binaries - not recommend to use this)

For each rule we have a choice of three conditions:

1. Publisher - can match on publisher, product name, file name or file version
2. Path - block or allow based on path they're run out of
3. File hash - if the  file recieves an update, the rule is useless, but still works for hardened offline systems

  
#### Implementing an AppLocker policy

We will try to create an automatically generated rule to whitelist the standard applications and block the firefox browser executable.

1. Make sure AppIDSvc is running
2. In GPO editor, navigate to Comp Conf\Policies\Windows Settings\Security Settings\Application Control Policies\AppLocker.
3. Right click the Executable Rules node and select Automatically Generate Rules. This allows the following:
    * Everyone to run all files located in the Program Files folder
    * Everyone to run all files located in the  Windows folder
4. Right click executeable rules and click Create New Rule and change the User or Group scope to Authenticated User
5. Permissions page, select Deny action
6. Conditions, select file hash
7. In the file hash page, navigate to C:\Windows\System32\calc.exe
8. Create to finish making the rule
9. User powershell to force domain wide gpo-update (invoke-gpupdate -computer $_.name -Force)

Log on to a computer and try to run calc.exe, doesn't work. You can also view this in  eventviewer, you have 4 applocker logs:

1. EXE and DLL
2. Msi and Script
3. Package App Deployment
4. Package App Execution

  
#### Implement Control Flow Guard

Control Flow Guard is a developer focused feature.
To enable it the creators of .NET software need to enable it in Visual Studio and recompile the program.

  
#### Implement Device Guard policies

Device Guard isn't a single product, it's a collection of security-related hardware and software features that fully protects the servers executable environment.
The exam reference guide is really vague on this, but the article on demystifying [Device Guard and Credential Guard on TechNet explains a lot!](https://blogs.technet.microsoft.com/ash/2016/03/02/windows-10-device-guard-and-credential-guard-demystified/)

You can manage device guard with:

1. Group Policy
2. System Center Configuration Manager SCCM
3. Microsoft Intune
4. Windows Powershell (ConfigCI-module)

  
#### Creating Code integrity policy rules

Define the actual application whitelist that is enforced on the target node. 
Create a "golden" server and workstation images that contain all the signed software and components that are allowed by the policy.

Each server/computer can only have one CI-policy at the time.
It's located at:

>C:\Windows\System32\CodeIntegrity\SIPolicy.p7b

Code integrity file rules specify the level at which applications are ided and trusted by Device Guard. Run levels are almost the same as in AppLocker, being Hash, FileName, SignedVersion, Publisher, FilePublisher, WHQL (used to trust windows kernel bins) and WHQLFilePublisher (specifies that bin code must be certifiedby Windows Hardware Quality Labs). 

High level steps to deploy a new CI-policy:

1. Prepare Golden computer
2. From an elevated powershell session, run the New-CIPolicy cmdlet
3. Use the ConvertFromCIPolicy cmdlet to convert the ci policy from plaintext xml to binary format
4. Audit the policy before applying it in production
5. To deploy, enable the Deploy Code Intergrity Policy Group Policy from Comp Conf\Pol\Adm Temp\System\DeviceGuard - this requires you to enter the path to your CI pol file.
6. Restart target system and check event log for results. Check the app and servcies\microsoft\windows\codeintegrity\operational log

  
#### Catalog files

You can store your application whitelisting exceptions in catalog files and device guard whitelists the catalog entries. This can be done with the PackageInspector.exe command line tool.

  
#### Enable Device Guard (high level)

1. Make sure all systems meet all DG requirements
    * System needs to be able to perform virtualization based security (VBS)
2. Group devices by their needed degree of control
    * Create apropriate Ous in GPO
3. Inventory and review all hardware and software 
    * If they aren't signed by a known CA you trust, they won't work with DG
4. Digitally sign all internal applications
  
  
### Links

[TechNet-article on server hardening](https://social.technet.microsoft.com/wiki/contents/articles/18931.security-hardening-tips-and-recommendations.aspx)
