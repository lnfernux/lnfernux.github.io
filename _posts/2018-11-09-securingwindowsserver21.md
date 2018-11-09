---
layout: post
title: Securing Windows Server - Implement a Guarded Fabric solution
subtitle: 'Chapter 2, Part 1'
tags:
  - powershell
  - security
  - windows
  - hyper-v
  - guarded fabric
  - hgs
  - securing windows server
published: false
---

### Chapter 2, Part 1: Implement a Guarded Fabric solution

The world of security is always changing and that's also the case for Microsoft. To follow all their updates, new products, what's retiring and namechanges please use the following link to [stay updated](https://blogs.technet.microsoft.com/secguide/) on all their blogs and updates. Here they discuss updated baselines and so much more.

#### Install and configure the Host Guardian service

Building a trusted fabric for Hyper-V involves deploying a Host Guardian Service (HGS) cluster. HGS is a new server role in WS16. 
HGS cluster exists in a seperate forest called a safe harbor forest. This creates a strong security and isolation boundary between the HGS cluster and your production forest. 
You need to manually create a one way external trust between the HGS and the prod forest. 

#### Preparing your HGS nodes

The HGS server role provides two services that enable guarded hosts, also known as HGS clients. For now, consider a shielded VM to be a protected VM. These two services are:

    * Attestation, HGS unlocks a shielded VM only if the identity and integrity of the VM has been verified
    * Key protection, these are the encryption keys that enable the shielded VM to transition between the encrypted and unencrypted states

It's recommended to deploy at least 3 virtual / physical HGS servers to provide high availability. The reason is simple, you won't be able to work with a shielded VM unless the HGS cluster is available. Lets install HGS, first make sure you are logged in on the HGS server with the local administrator and that you have an elevated powershell session.

#### Installing the HGS role

~~~powershell
Install-WindowsFeature -Name HostGuardianServiceRole -IncludeManagementTools -restart
~~~

#### Configuring the HGS server

Confirm the new HGS node isn't already a member of an AD domain, you can set up the safe harbor forest.

~~~powershell
$pw = ConvertTo-SecureString -String 'Password99' -AsPlainText
Install-HgsServer -HgsDomainName 'safe.local' -SafeModeAdministratorPassword $pw -Restart
~~~

When the server starts after it's second boot you'll have a brand new dc in a  brand new forest. Make sure you login to the HGS server with the new credentials, being safe\Administrator and the password you provided $pw.

#### Configure Admin and TPM-trusted attestation

Let's imagine that a workload admin attempts to access a shielded VM, by using RDP.
Because the VM is shielded and  resides on a guarded host the Hyper-V host requests two things:

* Host attestation 
* Decryption keys from the HGS cluster

Depending on how attestation is confed, the host is either allowed or not allowed to unlock the shielded VM
If the shielded VM is approved for unlock, the HGS transmits the decryption keys.
The workload adm continues his work.

Important to note that fabric admins is limited to only turning on and off the shielded vms.

#### Choosing an attestation method

We can chose between:
Admin-trusted - simple config - ensures that only Hyper-V hosts that are designated as guarded hosts can start shielded VMs - guarded hosts are placed in a special AD-group
TPM - hosts hw must have tpm v.2.0 and uefi 2.3.1 with sec boot enabled - difficult config - ensures that only guarded hyper-v hosts can start shielded vms - also ensures that guarded hosts can run only trusted code as defined in CI policies 

Initializing the HGS server
Start with creating a one way external trust between our safe.local HGS forest and our domain.com production forest. We can do this with netdom:

~~~powershell
Netdom trust safe.local /domain:domain.com /userD:domain\Administrator /passwordD:Password99 /add
~~~

We also need to ensure DNS resolution between the forests:

~~~powershell
Add-DnsServerConditionalForwarderZone -Name 'safe.local' -ReplicationScope 'Forest' -MasterServers 10.0.0.2

$certPW = -ConverTo-SecureString -AsPlainText "password99" -force

$signcert = New-SelfSignedCertificate -DnsName 'signing.safe.local' 
Export-PfxCertificate -Cert $signcert -Password $certpw -FilePath 'C:\signingcert.pfx'

$enccert =New-SelfSignedCertificate -DnsName 'encryption.safe.local'
Export-PfxCertificate -Cert $enccert -Password $certpw -FilePath 'C:\enccert.pfx'
~~~

Now let's initialize the HGS server:

~~~powershell
Initialize-HgsServer -LogDirectory C:\ -HgsServiceName 'HGS' -http -TrustActiveDirectory -SigningCertificatePath 'C:\SigningCert.pfx' -SigningCertificatePassword $certpw -EncryptionCertificatePath 'C:\enccert.pfx' -EncryptionCertificatePassword $certpw
~~~

##### Main takeaways

* HgsServiceName is the host name of the HGS cluster, therefore the safe.local dns includes and entry for hgs.safe.local that refers to the cluster itself
* TrustActiveDirectory signifies admintrusted attestation, for tpm use the -TrustTPM switch
* Pfx files contain the private and public keys for the certificates the cluster uses to encrypt and decrypt shielded vms

Since we're using admin trusted attestation, we need to make sure we create a global security group in our production forest called GuardedHostGroup that contains the hostname of our hyperv1.domain.local Hyper-V server.
In the HGS forest we run a powershell command to include the GuardedHostGroup to the HGS clusters attestation group. That means only hosts inside this group are allowed to work with shielded vms.

The command:

~~~powershell
Add-HGsAttestationHostGroup -Name 'GuardedHostGroup' -Identifier 'SID' 
~~~

You can get the SID from running Get-ADGroup on one of the DCs in the prod forest. 

To make sure everything is ok, run:

~~~powershell
Get-HgsTrace -RunDiagnostics 
~~~

#### Configure Key Protection Service using HGS

The Key Protection Services KPS is installed automatically when you install HGS. KPS stores and transmits encryption keys for use with shielded VMs.

Hardware security module HSM. To migrate existing certs to the HSM after installation, you can use:

~~~powershell
Add-HgsKeyProtectionCertificate
~~~

To make your HSM-backend keys the default ones to be used by each node in the cluster use:

~~~powershell
Set-HgsKeyProtectionCertificate
~~~

#### Configuring a guarded host.

Install Hyper-V server and Host Guardian Cluent server roles

~~~powershell
Install-WindowsFeature -Name Hyper-V, Host-Guardian -Includemanagementtools -restart
~~~

On HGS cluster node you can run Get-HgsServer to retrieve HGS cluster attestation and key protection URLs. Then, run the following command on the guarded host:

~~~powershell
Set-HgsClientConfiguration -AttestationServerUrl 'http://hgs.safe.local/Attestation' -KeyProtectionServerUrl 'http://hgs.safe.local/KeyProtection' 
~~~

#### Migrate Shielded VMs to other guarded hosts

High level procedure (unshielded generation 2 VM running on a Hyper-V host that is presently not a guarded host, and we want it moved to a guarded host):

Retrieve the HGS guardian metadata from the HGS server, the output, which allows us to create a key protector for the VM is an extensible markup language (XML) file that needs to be copied to the non-guarded hyper-v host

~~~powershell
Invoke-WebRequest http://hgs.safe.local/KeyProtection/service/metadata/2014-07/metadata.xml -OutFile C:\HGSGuardian.xml'
~~~

The VM we want to shield is named vs1. This powershell command performs the shielding operation on the VM. The key protector that is created contains one owner guardian, and one or more HGS guardians.

~~~powershell
$vm = 'vs1.contoso.local'
Stop-VM -VMName $vm
$Owner = New-HGSGuardian -name 'Owner -GenerateCertificates 
$Guardian = Import-HgsGuardian -Path 'C:\HgsGuardian.xml' -Name 'TestFabric' -AllowUntrustedRoot
$KP =New-HgsKeyProtector -Owner $Owner -Guardian $Guardian -AllowUntrustedRoot
Set-VMKeyprotector -VMName $VM -KeyProtector $KP.RawData
Set-VMSecurityPolicy -VMName $VM -Shielded $true
Enable-VMTPM -VMName $VM
~~~

Because you won't have console access to the vm it's important you prepare beforehand. This involves setting the firewall, setting up Wsman remote mgmt and enabling bitlocker drive encryption on the vms virtual hd.
Export the VM from the tenant host and import on a guarded host.

As long as the shielded VM is powered off, the VHDX is unlocked, and you can attach it to your host system. If this is not bitlocker encrypted, yo ucan access it freely.

Migrating a shielded VM to another guarded hosts - you can use standard methods as long as it's inside the HGS cluster, live migration, hyper-v replica, vm-checkpoints and hyper-v export/import.

#### Troubleshoot guarded hosts

Determine if a VM is shielded

~~~powershell
Get-VmSecurity -VmName 'vs1.domain.local' 
~~~

Can also verify the VMs status in the properties in Hyper-V manager.

If a client fails RDP and bitlocker is enabled, you can do the following:

Export the shielded VM from the guarded host and import it on a host along with the owners guardian key, then run this powershell command to disable shielding:

~~~powershell
Set-VMSecurityPolicy -Vmname 'vs1.domain.local' -Shielded $false
~~~

### Links

[Enable virtualization-based protection of code integrity](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-exploit-guard/enable-virtualization-based-protection-of-code-integrity)

[Manage Windows Defender Credential Guard](https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard-manage)