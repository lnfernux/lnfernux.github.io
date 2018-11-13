---
layout: post
title: Securing Windows Server - Implement a Guarded Fabric solution
subtitle: 'Trying to unravel the mystery of the Host Guardian Service and more'
tags:
  - powershell
  - security
  - windows
  - hyper-v
  - guarded fabric
  - hgs
  - securing windows server
published: true
image: /img/shielded-vm.png
---

### Chapter 2, Part 1: Implement a Guarded Fabric solution

The world of security is always changing and that's also the case for Microsoft. To follow all their updates, new products, what's retiring and namechanges please use the following link to [stay updated](https://blogs.technet.microsoft.com/secguide/) on all their blogs and updates. Here they discuss updated baselines and so much more.

#### So what is a guarded fabric

A guarded fabric is a Windows Server 2016 Hyper-V fabric capable of protecting tenant workloads against inspection, theft, and tampering from malware running on the host, as well as from system administrators. These virtualized tenant workloads—protected both at rest and in-flight are called shielded VMs.

![alt text3](https://docs.microsoft.com/en-us/windows-server/security/media/guarded-fabric-shielded-vm/guarded-host-overview-diagram.png "Guarded Host Overview Diagram")

When a tenant creates shielded VMs that run on a guarded fabric, the Hyper-V hosts and the shielded VMs themselves are protected by the Host Guardian Service (HGS). The HGS provides two distinct services: attestation and key protection. The Attestation service ensures only trusted Hyper-V hosts can run shielded VMs while the Key Protection Service provides the keys necessary to power them on and to live migrate them to other guarded hosts.

#### What is the Host Guardian Service

The Host Guardian Service (HGS) is the centerpiece of the guarded fabric solution. It is responsible for ensuring that Hyper-V hosts in the fabric are known to the hoster or enterprise and running trusted software and for managing the keys used to start up shielded VMs. When a tenant decides to trust you to host their shielded VMs, they are placing their trust in your configuration and management of the Host Guardian Service. Therefore, it is very important to follow best practices when managing the Host Guardian Service to ensure the security, availability and reliability of your guarded fabric. The guidance in the following sections addresses the most common operational issues facing administrators of HGS.

More information can be found on [Microsoft's pages](https://docs.microsoft.com/en-us/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-manage-hgs)

#### Install and configure the Host Guardian service

Imagine your normal Hyper-V fabric, like the image below.

![alt text2](https://docs.microsoft.com/en-us/windows-server/security/media/guarded-fabric-shielded-vm/guarded-fabric-existing-hyper-v.png "Standard Hyper-V Fabric")

Building a trusted fabric for Hyper-V involves deploying a Host Guardian Service (HGS) cluster. HGS is a new server role in WS16.
HGS cluster exists in a seperate forest called a safe harbor forest. This creates a strong security and isolation boundary between the HGS cluster and your production forest.
You need to manually create a one way external trust between the HGS and the prod forest.

The image below shows your normal forest and a three-node HGS cluster, relecloud.com

![alt text](https://docs.microsoft.com/en-us/windows-server/security/media/guarded-fabric-shielded-vm/guarded-fabric-deployment-step-two-deploy-hgs.png "HGS Cluster")

If you wish to learn more about the steps in a high level overview, [this guide shows you each step with links to step-by-step guides for reach ... well, step](https://docs.microsoft.com/en-us/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-deployment-overview)

#### Preparing your HGS nodes

The HGS server role provides two services that enable guarded hosts, also known as HGS clients. For now, consider a shielded VM to be a protected VM. These two services are:

* Attestation, HGS unlocks a shielded VM only if the identity and integrity of the VM has been verified
* Key protection, these are the encryption keys that enable the shielded VM to transition between the encrypted and unencrypted states

It's recommended to deploy at least 3 virtual / physical HGS servers to provide high availability. The reason is simple, you won't be able to work with a shielded VM unless the HGS cluster is available. Lets install HGS, first make sure you are logged in on the HGS server with the local administrator and that you have an elevated powershell session.

#### Installing the HGS role

~~~console
Install-WindowsFeature -Name HostGuardianServiceRole -IncludeManagementTools -restart
~~~

Obviosuly you don't have to have -restart, but you need to restart soooo, yeah.

#### Configuring the HGS server

Confirm the new HGS node isn't already a member of an AD domain before you start setting up a safe harbor forest.

~~~console
$pw = ConvertTo-SecureString -String 'SuperSecurePassword1234' -AsPlainText
Install-HgsServer -HgsDomainName 'safe.local' -SafeModeAdministratorPassword $pw -Restart
~~~

When the server starts after it's second boot you'll have a brand new dc in a brand new forest. Make sure you login to the HGS server with the new credentials, that being safe\Administrator and the password you provided $pw.

#### Configure Admin and TPM-trusted attestation

Let's imagine that a workload admin attempts to access a shielded VM, by using RDP.
Because the VM is shielded and resides on a guarded host the Hyper-V host requests two things:

* Host attestation
* Decryption keys from the HGS cluster

Depending on how attestation is configured, the host is either allowed or not allowed to unlock the shielded VM.
If the shielded VM is approved for unlock, the HGS transmits the decryption keys.
The workload administrator continues his work.

Important to note that fabric admins is limited to only turning on and off the shielded vms.

The distinction that Microsoft attempts to make here is that workload admins, the ones that request a VM and use them for development, production or whatever have a need-to-use for these machines and are for that reason a workload admin. In short, anyone who works on one or more VM's is a workload admin. For huge fabrics this will obviously be several groups, but for now, a workload admin is someone who uses the virtual machine for their day-to-day job.

A fabric admin, on the other hand is the administrator of the fabric. It's the guy who makes sure there's no disk errors, storage problems and provisions new VM's for the workload administrators to use. Usually, these employees are trusted to not snoop around, steal or break applications and information found on the VM's they manage, but there's nothing short of trust and monitoring stopping them from doing so. In this scenario the fabric administrators would not be able to use the VM console or RDP into the VM's because only users in the workload-admin group would be able to.

#### Choosing an attestation method

We can chose between:

| Type | Difficulty rating | Who can start shielded VM's? |
| --- | --- | --- | --- |
| Admin-trusted | Simple config | Ensures that only Hyper-V hosts that are designated as guarded hosts can start shielded VMs, while guarded hosts are placed in a special AD-group |
| TPM | Difficult config | Ensures that only guarded hyper-v hosts can start shielded vms, also ensures that guarded hosts can run only trusted code as defined in CI policies |
| Host key | Simple config | Works almost the same as Admin-trust, see link below for more information | 

Keep in mind that we need to have TPM 2.20 and UEFI 2.3.1 with SecureBoot enabled for TPM-attestation to work.

*Admin-trusted is becoming deprecated as of Server 2019*

[Planning to deploy a guarded fabric? Look no further!](https://docs.microsoft.com/en-us/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-planning-for-hosters)

#### Initializing the HGS server

First we need to ensure DNS resolution between the forests:

~~~console
Add-DnsServerConditionalForwarderZone -Name 'safe.local' -ReplicationScope 'Forest' -MasterServers 10.0.0.2
~~~

Then we create a one way external trust between our safe.local HGS forest and our domain.com production forest. We can do this with netdom:

~~~console
netdom trust safe.local /domain:domain.com /userD:domain\Administrator /passwordD:SuperSecurePassword1234 /add
~~~

Microsoft is very vague when they define which direction the trust is, i.e who should trust who. [Examining this](https://social.technet.microsoft.com/Forums/en-US/19c517b6-724c-48eb-9ab1-bcbf194345d4/hgs-requirements?forum=winserversecurity) forum post on TechNet and [this](https://docs.microsoft.com/nb-no/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-configure-dns-forwarding-and-trust) DNS-Trust guide on Guarded Fabric and HGS we can conclude that official documentation's syntax suggests that our safe HGS-domain should trust the fabric domain, as the syntax is the following:

~~~console
netdom trust trusting_domain_name /domain:name_of_trusted_domain.com /UserD:account_used_to_make_connection /passwordD:password_of_account /add
~~~

I'd like input on this, however, as people on TechNet are saying MS is being overly vague and that the exam questions actually say that the fabric domain should trust HGS (which is the oposite of what I've concluded here)

#### Setting up admin-trusted attestation

In this expample we will create our own self-signed certificates, but for more information about obtaining certs for your HGS-server, [lookie here!](https://docs.microsoft.com/nb-no/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-obtain-certs)

~~~console
$certPW = -ConverTo-SecureString -AsPlainText "SuperSecurePassword1234" -force

$signcert = New-SelfSignedCertificate -DnsName 'signing.safe.local'
Export-PfxCertificate -Cert $signcert -Password $certpw -FilePath 'C:\signingcert.pfx'

$enccert =New-SelfSignedCertificate -DnsName 'encryption.safe.local'
Export-PfxCertificate -Cert $enccert -Password $certpw -FilePath 'C:\enccert.pfx'
~~~

Now let's *actually* initialize the HGS server:

~~~console
Initialize-HgsServer -LogDirectory C:\ -HgsServiceName 'HGS' -http -TrustActiveDirectory -SigningCertificatePath 'C:\SigningCert.pfx' -SigningCertificatePassword $certpw -EncryptionCertificatePath 'C:\enccert.pfx' -EncryptionCertificatePassword $certpw
~~~

It's a long command, but it's not actually that scary when you look at it.

#### Main takeaways thus far

* HgsServiceName is the host name of the HGS cluster, therefore the safe.local dns includes an entry for hgs.safe.local that refers to the cluster itself
* TrustActiveDirectory signifies admintrusted attestation, for tpm use the -TrustTPM switch
* Pfx files contain the private and public keys for the certificates the cluster uses to encrypt and decrypt shielded vms

Since we're using admin trusted attestation, we need to make sure we create a global security group in our production forest called GuardedHostGroup that contains the hostname of our hyperv1.domain.local Hyper-V server.
In the HGS forest we run a powershell command to include the GuardedHostGroup to the HGS clusters attestation group. That means only hosts inside this group are allowed to work with shielded vms.

~~~console
Add-HGsAttestationHostGroup -Name 'GuardedHostGroup' -Identifier 'SID'
~~~

You can get the SID from running Get-ADGroup on one of the DCs in the production forest.

#### Verify the HGS installation

Because the HGS configuration does not yet contain information about the hosts that will be in the guarded fabric, the diagnostics will indicate that no hosts will be able to attest successfully yet. Ignore this result, and review the other information provided by the diagnostics.

We can use powershell and this relatively simple cmdlet to run our diagnostics:

~~~console
Get-HgsTrace -RunDiagnostics
~~~

When running the Guarded Fabric diagnostics tool, incorrect status may be returned claiming that the HTTPS configuration is broken when it is, in fact, not broken or not being used. This error can be returned regardless of HGS’ attestation mode. The possible root-causes are as follows:

* HTTPS is indeed improperly configured/broken
* You’re using admin-trusted attestation and the trust relationship is broken
  * This is irrespective of whether HTTPS is configured properly, improperly, or not in use at all.

Note that the diagnostics will only return this incorrect status when targeting a Hyper-V host. If the diagnostics are targeting the Host Guardian Service, the status returned will be correct.

#### Configure Key Protection Service

The Key Protection Services (KPS) is installed automatically when you install HGS. KPS stores and transmits encryption keys for use with shielded VMs.

There's also something called hardware security module (HSM). To migrate existing certificates to the HSM after installation, you can use:

~~~console
Add-HgsKeyProtectionCertificate
~~~

To make your HSM-backend keys the default ones to be used by each node in the cluster we can do:

~~~console
Set-HgsKeyProtectionCertificate
~~~

And that's it. For more information and "Bring Your Own Key (BYOY)", look [here](https://blogs.technet.microsoft.com/datacentersecurity/2016/03/28/configuring-key-protection-service-for-host-guardian-service-in-windows-server-2016/)

#### Configuring a guarded host

Install Hyper-V server and Host Guardian Client server roles

~~~console
Install-WindowsFeature -Name Hyper-V, Host-Guardian -Includemanagementtools -restart
~~~

On a HGS cluster node you can run Get-HgsServer to retrieve HGS cluster attestation and key protection URLs. Then, run the following command on the host you wish to guard:

~~~console
Set-HgsClientConfiguration -AttestationServerUrl 'http://hgs.safe.local/Attestation' -KeyProtectionServerUrl 'http://hgs.safe.local/KeyProtection'
~~~

That should be it, please refer to [this](https://blogs.technet.microsoft.com/datacentersecurity/2016/03/16/windows-server-2016-and-host-guardian-service-for-shielded-vms/) low-level guide if there's anything that's unclear.

#### Migrate Shielded VMs to other guarded hosts

High level procedure (unshielded generation 2 VM running on a Hyper-V host that is presently not a guarded host, and we want it moved to a guarded host).

Retrieve the HGS guardian metadata from the HGS server (the output) which allows us to create a key protector for the VM. The key protector is an extensible markup language (XML) file that needs to be copied to the non-guarded hyper-v host

~~~console
Invoke-WebRequest http://hgs.safe.local/KeyProtection/service/metadata/2014-07/metadata.xml -OutFile C:\HGSGuardian.xml'
~~~

The VM we want to shield is named vs1. This powershell command performs the shielding operation on the VM. The key protector that is created contains one owner guardian, and one or more HGS guardians.

~~~console
$vm = 'vs1.contoso.local'
Stop-VM -VMName $vm
$Owner = New-HGSGuardian -name 'Owner' -GenerateCertificates 
$Guardian = Import-HgsGuardian -Path 'C:\HgsGuardian.xml' -Name 'TestFabric' -AllowUntrustedRoot
$KP = New-HgsKeyProtector -Owner $Owner -Guardian $Guardian -AllowUntrustedRoot
Set-VMKeyprotector -VMName $VM -KeyProtector $KP.RawData
Set-VMSecurityPolicy -VMName $VM -Shielded $true
Enable-VMTPM -VMName $VM
~~~

Because you won't have console access to the vm it's important you prepare for remote access. This involves setting the firewall, setting up WsMan remote management and enabling bitlocker drive encryption on the vms virtual harddrive.

#### Export the VM from the tenant host and import on a guarded host

As long as the shielded VM is powered off, the VHDX is unlocked you can attach it to your host system. If this is not bitlocker encrypted, you can access it freely.
The routine is basic:

* Shut down the VM
* Export the VM from the tenant host
* Import the VM on the guarded host
* Start the VM

That should be it. For more info, please see [this](https://blogs.technet.microsoft.com/datacentersecurity/2016/06/06/step-by-step-creating-shielded-vms-without-vmm/) article, which also covers creating fresh, shielded VMs instead of shielding already existing VMs.

#### Migrating a shielded VM to another guarded hosts

You can use standard methods as long as it's inside the HGS cluster, live migration, hyper-v replica, vm-checkpoints and hyper-v export/import are all options that will work inside the same HGS-cluster. If not, you have to use the method mentioned above.

#### Troubleshoot guarded hosts

Determine if a VM is shielded by running a simple cmdlet.

~~~console
Get-VmSecurity -VmName 'vs1.domain.local'
~~~

We can also verify the VMs status in the properties in Hyper-V manager.

If a client fails RDP and bitlocker is enabled, you can export the shielded VM from the guarded host and import it on a host along with the owners guardian key, then run this powershell command to disable shielding:

~~~console
Set-VMSecurityPolicy -Vmname 'vs1.domain.local' -Shielded $false
~~~

### Links

[Guarded fabric and shielded VMs overview](https://docs.microsoft.com/en-us/windows-server/security/guarded-fabric-shielded-vm/guarded-fabric-and-shielded-vms)

[Quick overview from Windows on YouTube](https://www.youtube.com/watch?v=nPJfI7r_AGg)