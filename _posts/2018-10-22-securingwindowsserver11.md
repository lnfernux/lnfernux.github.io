---
layout: post
title: Disk and file encryption
subtitle: Securing Windows Server - Chapter 1, Part 1
tags:
  - powershell
  - security
  - windows
  - server
  - uefi
published: true
image: /img/ws2.png
---

This time we're looking at disk and file encryption!

# Chapter 1, Part 1: Disk and file encryption

In this series of blogposts I'll try to summarize briefly the most important takeaways from exploring security on the Windows Server product line. We're starting out with server hardening, this includes disk and file encryption, boot-protection, patching and upgrading, malware protection, credential protection and working with security baselines.

I might glance over some things - this is not a technical "how to do it" approach, I'll mostly be presenting the concepts and then resources like TechNet will be able to fill in the blanks (there will be many!).

This first part is all about disk and file encryption - if there's any feedback or something you feel that's important (and missing), please hit me up on [twitter](https://twitter.com/infernuxmonster)

### Overview

![windows boot process](https://docs.microsoft.com/nb-no/windows/security/information-protection/images/dn168167.boot_process%28en-us%2cmsdn.10%29.png "windows boot process")

The image above describes the boot process. We will get into some of the parts described in the diagram, but focusing mostly on the server side. For more on securing the Windows 10 boot process, check [this out.](https://docs.microsoft.com/nb-no/windows/security/information-protection/secure-the-windows-10-boot-process)

#### UEFI

What does UEFI provide us with?

* Ability to support Windows 10 security features like Secure Boot, Windows Defender Device Guard, Windows Defender Credential Guard, and Windows Defender Exploit Guard. All require UEFI firmware.
* Faster boot and resume times.
* Ability to more easily support large hard drives (more than 2 terabytes) and drives with more than four partitions.
* Support for multicast deployment, which allows PC manufacturers to broadcast a PC image that can be received by multiple PCs without overwhelming the network or image server.
* Support for UEFI firmware drivers, applications, and option ROMs.

#### Enabling UEFI

Enabling UEFI is done by hitting a key-stroke (OEM dependant) on boot and [enabling it](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/boot-to-uefi-mode-or-legacy-bios-mode). Newer hardware is UEFI by default.

#### Secure Boot

Is an UEFI feature that protects a servers startup enviroment. UEFI firmware stores a database of trusted hardware, drivers, operating systems and option ROMs. This DB is structured by the server's OEM. This way the server only starts if its OS boot loader files and device drivers are digitally signed and trusted by the Secure Boot database.

Secure Boot can be turned off with physical access. Also smart to have a UEFI admin password enabled.

#### TPM

Stands for Trusted Platform Module, which is a microchip that's installed on current gen servers and desktop-class mobos. Main function is protecting security related data, particulary encryption and decryption keys. Server 2016 supports TPM v1.2 and 1.0.

#### TPM vs Secure Boot

Technically TPM can provide the same type of boot-time protection that UEFI Secure Boot can. However, the two systems are seperate and rely upon seperate trust stores.

To read up further on TPM, UEFI and Secure Boot in a security perspective check out [this excellent article on infosecinstitute](https://resources.infosecinstitute.com/uefi-and-tpm/).

### BitLocker

![alt](https://upload.wikimedia.org/wikipedia/en/d/de/BitLocker_icon.png "bitlocker")

BitLocker is a full disk encryption feature included in Windows. It is designed to protect data by providing encryption for entire volumes.

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

That should be all for BitLocker installation.

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

You can also use manage-bde command line executable.

#### Implement BitLocker on Hyper-V VM

Hyper-V in WS16 allows both Secure Boot and virtualized TPM (vTPM) for VM guests. This is a part of each machine's properties - meaning the setup for this is the same as a physical machine/server.

#### Implement BitLocker on Cluster Shared Volumes (CSV) and Storage Area Networks (SAN)

Was implemented in WS12, you can encrypt volumes before or after you add them to a cluster. Use either Windows Powershell or managezbde.exe to perform the task.

#### Configure Network Unlock

WS16 supports the BitLocker Network Unlock feature. It allows automatic access to BitLocker decryption keys, which means that you can start, restart or remotely manage your windows servers without having to manually input a PIN. The requirements, in addition to UEFI firmware and TPM chips are the following:

1. UEFI DHCP - previously known as PXE (preboot execution enviroment)
2. No CSM (i.e Compability Support Modules must be disabled)
3. Separate WDS and DHCP servers. (i.e these roles can't be on the same server)
4. PKI - this is in order to generate the X.509 digital certificates that's required for Network Unlock. AD Certificate Services (AD CS) works fine.
5. Must also configure the previously mentioned GPO settings for BitLocker to specify pin+TPM
6. Network Unlock Group Policy Settings (upload the .cer file to this location)

>Computer Configuration\Policies\Windows Settings\Security Settings\Public Key Policies\BitLocker Drive Encryption Network Certificate

#### The Network Unlock sequence

![alt text3](https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/images/bitlockernetworkunlocksequence.png "The Network Unlock Sequence")

1. The Windows boot manager detects that a Network Unlock protector exists in the BitLocker configuration.
2. The client computer uses its DHCP driver in the UEFI to obtain a valid IPv4 IP address.
3. The client computer broadcasts a vendor-specific DHCP request that contains the Network Key (a 256-bit intermediate key) and an AES-256 session key for the reply. Both of these keys are encrypted using the 2048-bit RSA Public Key of the Network Unlock certificate from the WDS server.
4. The Network Unlock provider on the WDS server recognizes the vendor-specific request.
5. The provider decrypts it with the WDS server’s BitLocker Network Unlock certificate RSA private key.
6. The WDS provider then returns the network key encrypted with the session key using its own vendor-specific DHCP reply to the client computer. This forms an intermediate key.
7. The returned intermediate key is then combined with another local 256-bit intermediate key that can only be decrypted by the TPM.
8. This combined key is used to create an AES-256 key that unlocks the volume. Windows continues the boot sequence.

I rewrote my sequence (mine was 5 steps) to match that of the TechNet article detailing the Network Unlock feature (see link below.)

#### Enable Network Unlock

First, check that WDS is running:

~~~console
Get-Service WDSServer
~~~

Then, install the Network Unlock feature

~~~console
Install-WindowsFeature BitLocker-NetworkUnlock
~~~

Then we create a certificate, in this example a self signed one (please see the link furthest down in this part for more in-depth information here):

~~~console
New-SelfSignedCertificate -CertStoreLocation Cert:\LocalMachine\My -Subject "CN=BitLocker Network Unlock certificate" -Provider "Microsoft Software Key Storage Provider" -KeyUsage KeyEncipherment -KeyUsageProperty Decrypt,Sign -KeyLength 2048 -HashAlgorithm sha512 -TextExtension @("1.3.6.1.4.1.311.21.10={text}OID=1.3.6.1.4.1.311.67.1.1","2.5.29.37={text}1.3.6.1.4.1.311.67.1.1")
~~~

After that we deploy the private key to our WDS server and configure Group Policy settings for Network Unlock.

For more in depth information, particularly creating the certificate template for network unlock, check out [this article!](https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-how-to-enable-network-unlock)

#### Implement the BitLocker Recovery Process

Easiest way is the recovery password that we generated and saved to a file/usb or printed out when we enabled bitlocker - a 48-digit unlock key. Saving this file to an offline, removable media that's stored securely should be the one of the main ways to handle BitLocker recovery passwords.

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

#### Accessing the recovery password

Locate the target server in AD Users and Computers, open it's properties sheet and navigate to the bitlocker recovery tab. You'll see the recovery password here. Keep in mind that anyone with access to this object in AD will be able to see it, so this protection is only as good as your AD-security itself.

#### Self-service recovery

You can also use Microsoft BitLocker Administration and Monitoring toolset, or MBAM for short. 

It has a pretty complex installation that we won't get into, full-fledged multi-tier application that can be deployed stand-alone or through SCCM. Provides end-to-end automation for BitLocker, including self-service key retrieval, agent-based user guidance.

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
   *. Computer Configuration\Windows Settings\Security Settings\Public Key Policies
   *. You'll see two subfolders, EFS and BitLocker
   *. You can set DRAs for both
4. Right click the EFS policy folder and select Add Data Recovery Agent from the context menu.
    a. Here you can either Browser Directory - locate the user by searching AD, if you use this option the certs must be published to AD
    b. Browse Folders, locate the .cer exported EFS recovery agent cert in a local or remote file system

Refresh GPO (Invoke-GPUpdate or gpupdate from cmd) and your new DRAs have privilege to decrypt all domain users EFS-encrypted files. Comes in handy during emergiencies, but if someone gain access to DA it will become a problem. For more, [see this article from Microsoft](https://docs.microsoft.com/en-us/windows/security/information-protection/windows-information-protection/create-and-verify-an-efs-dra-certificate).

### Links

[TechNet-article on server hardening](https://social.technet.microsoft.com/wiki/contents/articles/18931.security-hardening-tips-and-recommendations.aspx)

[Network Unlock article](https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-how-to-enable-network-unlock)