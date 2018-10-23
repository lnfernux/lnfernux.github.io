---
layout: post
title: Securing Windows Server summary - part 1
subtitle: 
tags:
  - powershell
  - security
  - windows
  - server
  - uefi
published: true
---

### Part 1: Disk and file encryption

In this series of blogposts I'll try to summarize briefly the most important takeaways from doing my 70-744 exam. We're starting out with server hardening, this includes disk and file encryption, boot-protection, patching and upgrading, malware protection, credential protection and working with security baselines.

I might glance over some things - this is not a technical "how to do it" approach, I'll mostly be presenting the concepts and then resources like TechNet will be able to fill in the blanks (there will be many!).

This first part is all about disk and file encryption - if there's any feedback or something you feel that's important (and missing), please hit me up on [twitter](https://twitter.com/infernuxmonster)

#### UEFI

Enabling UEFI is done by hitting a key-stroke (OEM dependant) on boot and enabling it. Newer hardware is UEFI by default.

#### Secure Boot

UEFI feature that protects a servers startup enviroment. UEFI firmware stores a database of trusted hardware, drivers, operating systems and option ROMs. This DB is structured by the server's OEM. This way the server only starts if its OS boot loader files and device drivers are digitally signed and trusted by the Secure Boot database.

Secure Boot can be turned off with physical access. Also smart to have a UEFI admin password enabled.

#### TPM

Stands for Trusted Platform Module, which is a microchip that's installed on current gen servers and desktop-class mobos. Main function is protecting security related data, particulary encryption and decryption keys. Server 2016 supports TPM v1.2 and 1.0.

#### TPM vs Secure Boot

Technically TPM can provide the same type of boot-time protection that UEFI Secure Boot can. However, the two systems are seperate and rely upon seperate trust stores.

#### Enable BitLocker to use Secure Boot and BCD integrity verification

Bitlocker Drive Encryption is Microsofts native disk encryption solution for operating systems and data drives.
Boot Configuration Database (BCD) is a firmware-independent database that stores Windows startup configuration data. In Server 2016 this is a unlettered 500 MB System Reserved partition on your startup disk.

In order to prepare BitLocker to use Secure Boot for vplatform and BCD database integrity validation we need to enable the Allow Secure Boot for Integrity alidation policy found in the GPO path:
> Computer Configuration\Policies\Administrative Templates\Windows Components\BitLocker Drive Encryption\Operating System Drives

#### Deploy BitLocker drive encryption

The first step is to install the BitLocker Drive Encryption Feature, this can be done through powershell (administrative):

~~~powershell
Install-WindowsFeature -Name BitLocker -IncludeAllSubFeature -IncludeManagementTools -Restart
~~~

We can also use Server Manager or, if you'd like, DISM through the EnableWindowsOptionalFeature-wrapper:

~~~powershell
Enable-WindowsOptionalFeature -Online -FeatureName Bitlocker,BitLocker-Utilities -All
~~~

That's all folks!

#### Configure BitLocker with or without TPM

BitLocker Drive Encryption can be configured to use a number of authentication methods called protectors.

Protector configuration | Startup behavior
--- | ---
No TPM | Requires a BitLocker password or a startup key on a USB drive
TPM + startup pin | Requires the presence of a TPM chip and a PIN
TPM + startup key | Requires a TPM chip and USB drive based startup key
TPM + startup PIN + startup key | Requires TPM, a pin and a startup key

Configuring the BitLocker Drive Encryption policy is done through GPO and the policy named Required Additional Authentication At Startup. It's located in the GPO path:
>Computer Configuration\Policies\Administrative Templates\Windows Components\BitLocker Drive Encryption\Operating System Drives

You can use TPM without any other protector, I.E just TPM and no PIN/KEY, it's better that nothing. You can also use BitLocker with no TPM (not advisable) by selecting the Allow BitLocker Without A Compatible TPM (Requires A Password Or A Startup Key On A USB Flash Drive) GPO setting.

After the GPO settings have taken effect, you need to actually encrypt your OS volume:

1. Open Control Panel and start the BitLocker Drive Encryption item
2. In the BitLocker control panel interface beneath the Operating System Drive, click "Turn On BitLocker"
3. Depending on how you have configured BitLocker in the previous steps your options may vary, but this is where you either enter a PIN, password or the USB startup key.
4. Create a strong password and choose to back it up in 1/3 locations (USB/File/Print)
5. Chose how much of your OS drive to encrypt (used disk space or entire drive) - for an existing server, choose the latter always.
    a. You should also encrypt data drives
6. Choose encryption algorythm 
    a. AES-128 (default algo+cipher length)
    b. AES-256
    c. XTS-AES-128 (incompatible with previous versions of win server)
    d. XTS-AES-256
7. Ensure that Run BitLocker system check option is selected and press continue to proceed
8. After a quick restart bitlocker will start to encrypt the OS volume

You can also use the BitLocker PowerShell cmdlets. Let's encrypt the C:\ drive using TPM and PIN protectors:

~~~powershell
PS> $SecureString = ConvertTo-SecureString '$trongPa$$w0rd1337' -AsPlainText -Force 
PS> Enable-BitLocker -MountPoint 'C:' -EncryptionMethod Aes256 -UsedSpaceOnly -Pin $SecureString -TPMandPinProtector
~~~

Ooooor if your legacy as fuck, you can use manage-bde command line executable.

#### Implement BitLocker on Hyper-V VM

Hyper-V in WS16 allows both Secure Boot and virtualized TPM (vTPM) for VM guests. This is a part of each machine's properties - meaning the setup for this is the same as a physical machine/server.

#### Implement BitLocker on Cluster Shared Volumes (CSV) and Storage Area Networks (SAN)

Was implemented in WS12, you can encrypt volumes before or after you add them to a cluster. Use either Windows Powershell or managezbde.exe to perform the task.

#### Configure Network Unlock

WS16 supports the BitLocker Network Unlock feature. It allows automatic access to BitLocker decryption keys, which means that you can start, restart aor remotely manage your windows serers without having to manually input a PIN. The requirements, in addition to UEFI firmware and TPM chips are the following:

1. UEFI DHCP - previously known as PXE (preboot execution enviroment)
2. No CSM (i.e Compability Support Modules must be disabled)
3. Separate WDS and DHCP servers. (i.e these roles can't be on the same server)
4. PKI - this is in order to generate the X.509 digital certificates that's required for Network Unlock. AD Certificate Services (AD CS) works fine.
5. Must also configure the previously mentioned GPO settings for BitLocker to specify pin+TPM
6. Network Unlock Group Policy Settings (upload the .cer file to this location)
>Computer Configuration\Policies\Windows Settings\Security Settings\Public Key Policies\BitLocker Drive Encryption Network Certificate

#### The Network Unlock sequence

1. Server starts, boot manager detects the presence of a Network Unlock protector
    a. Protector is realized by the Allow Network Unlock At Startup GPO
2. Server uses UEFI DHCP driver to obtain a valid Ipv4 adress
3. Server broadcasts a vendor-specific DHCP request that's encrypted with the WDS server's Network Unlock cert (which our server has thanks to the cert-GPO we confed)
4. WDS provider processess the rquest and produces and AES-256 key that unlocks the local servers OS volume
5. Server continues to boot

#### Implement the BitLocker Recovery Process

Easiest way is the recovery password that we generated and saved to a file/usb or printed out when we enabled bitlocker - a 48-digit unlock key.

#### Recovery password retrieval from AD DS

We can also back up BitLocker recovery keys to AD. The configuration setting to this lies in the following GPO path:
>Computer Configuration\Policies\Administrative Templates\Windows Components\BitLocker Drive Encryption\

#### The specific GPO is name Store BitLocker recovery information in Active Directory Domain Services

This policy gives you the choice of storing only BitLocker recovery passwords in AD DS, or both the passwords as well as the underlying encryption keys.
You'll also need to enable the policy Choose How BitLocker-Protected Operating System Drives Can Be Recovered from the Operating System Drives subfolder in the GPO path. Specifically you need to make sure the option Save BitcLocker Recovery Information To AD DS is enabled for operating system drives.

Next, we need to run the Invoke-GPUpdate cmdlet against relevant servers (imagine I have a file of all relevant servers in a txt file):

~~~powershell
Invoke-GPUpdate -Computer (Get-Content -Path .\servers.txt) -Force
~~~

From now on any server where you enable BitLocker stores its recovery password and also its encryption keys in Active Directory. This doesn't affect servers where BitLocker is already enabled, however. On these machines, run the following command to obtain the systems numerical password ID:

~~~powershell
Manage-bde -protectors -get c:
~~~

And then run this command to force the key/pass archival:

~~~powershell
Manage-bde -protectors -adbackup c: -id {password-id}
~~~

Accessing the recovery password
Locate the target server in AD Users and Computers, open it's properties sheet and navigate to the bitlocker recovery tab. You'll see the recovery password here.

#### Self-service recovery

You can also use Microsoft BitLocker Administration and Monitoring toolset (MBAM. Complex installation, full-fledged multi-tier application that can be deployed stand-alone or through SCCM. Provides end-to-end automation for BitLocker, including self-service key retrieval, agent-based user guidance.

#### Manage Encrypting File System (EFS)

BitLocker functions at the volume level. You can use it to encrypt removable media, but for most production servers you'll be encrypting entire fixed hard disk volumes. We can use BitLocker to create encrypted container files, but these too are treated by WS16 as VHD images.
Encrypting File System (EFS) is a more granular solution that can be leveraged to protect individual folders and files.

#### Data recovery agents

EFS generates self-signed certs and stores them in each user or administrators profile folder by default. This is a bad idea in production, because the keys can stolen/damaged and there's not trust chain with self-signing.
If you plan to implement EFS, you should have a true blue PKI established, like with AD CS, so you can fully manage EFS certs. AD CS includes basic EFS and EFS recovery agent templates out of box.

The data recovery agent (DRA) is a privileged user account who can decrypt other domain users EFS certificates. By default the domain admin account is the de facto DRA, but we can include others.

Steps to define the current administrator a new EFS DRA in WS16 AD domain that has an online enterprise root cert auth:

1. Request an EFS Recovery Agent cert from AD CS authority. From the Certificates MMC snap-in, do this by right clicking personal certificate store and clicking all tasks, then selecting "request new certificate"
2. From Cert snap-in, back up EFS, BitLocker or any other digital cert by right clicking all tasks and then export. We can then do the same procedure for import.
3. To assign DRAs at the domain level, open a GPO and navigate to:
>Computer Configuration\Windows Settings\Security Settings\Public Key Policies
        * You'll see two subfolders, EFS and BitLocker
        * You can set DRAs for both
4. Right click the EFS policy folder and select Add Data Recovery Agent from the context menu.
    a. Here you can either Browser Directory - locate the user by searching AD, if you use this option the certs must be published to AD
    b. Browse Folders, locate the .cer exported EFS recovery agent cert in a local or remote file system

Refresh GPO (Invoke-GPUpdate or gpupdate from cmd) and your new DRAs have privilege to decrypt all domain users EFS-encrypted files. Comes in handy during emergiencies, but if someone gain access to DA it will become a problem.

### Links

[TechNet-article on server hardening](https://social.technet.microsoft.com/wiki/contents/articles/18931.security-hardening-tips-and-recommendations.aspx)
