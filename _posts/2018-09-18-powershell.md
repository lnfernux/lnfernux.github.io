---
layout: post
title: Securing Windows Server (70-744) scripts
subtitle: 
tags:
  - powershell
  - security
  - windows
  - server
published: true
image: /img/ps.png
---

## Some stolen scripts

Gentle reminder that these scripts are stolen from the exam ref written by Timothy L. Warner and Craig Zacker.
At best I've added some steps to make the scripts easier to re-use, like having variables be user-prompted and not hardcoded.

First of all, let's look at making sure our Windows Defender is up to date with it's signatures, and then running a scan.

~~~powershell
#Start by importing defender module
Import-Module -Name Defender

#Find yesterdays date
$TimeStampYesterday = (Get-Date).AddDays(-1)

#Find date for last signature update
$AntiVirusSignatureLastUpdate = Get-MpComputerStatus | Select-Object -ExpandProperty AntiVirusSignatureLastUpdated
if($AntiVirusSignatureLastUpdate -lt $TimeStampYesterday){
    #Update signature if not updated
    Update-MpSignature
}
#Start a new scan
Start-MpScan
~~~

Secondly a script for whitelisting apps from a certain provider.

~~~powershell
#First take argument where you define the publisher
$var = Read-Host "Enter publisher you want to check (Microsoft)"

#Get all apps from that publisher to a variable
$AllowedApps = Get-AppxPackage  -AllUsers | Where-Object {$_.publisher -match $var}

#Pipe that through fileinfo and make a new policy
$AllowedApps | Get-AppLockerFileInformation | New-AppLockerPolicy -User Everyone -Optimize

#Print out rules, to show that our policy was created and is in effect
(Get-AppLockerPolicy -Effective).Rulecollections.PublisherConditions | Sort-Object ProductName
~~~

That's all for now.
