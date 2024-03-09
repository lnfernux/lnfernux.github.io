---
layout: post
title: 'Tools You Should Know: ScubaGear'
subtitle: Developed by CISA, ScubaGear is an assessment tool that verifies a Microsoft 365 (M365) tenant’s configuration conforms to the policies described in the Secure Cloud Business Applications (SCuBA) Security Configuration Baseline documents.
tags:
  - Cyber Security
  - Entra ID
  - Security Monitoring
  - Entra ID
  - Azure
  - Microsoft 365
  - ScubaGear
published: true
author: author_infernux
image: /img/azure.png
---

**Welcome to the first post in the series "Tools You Should Know"**. In this series, I'll go over some of the tools and services that I've found useful in the past. The focus will be on **Microsoft 365 and Azure**, but I might look to expand with other cloud providers in the future. First out is **[ScubaGear](https://github.com/cisagov/ScubaGear)**, a tool developed by CISA. 

## What is ScubaGear?

It's an assessment tool that verifies a Microsoft 365 (M365) tenant’s configuration to a baseline. The baseline (or policy) in question are the policies described in the [Secure Cloud Business Applications (SCuBA)](https://github.com/cisagov/ScubaGear/tree/main/baselines) Security Configuration Baseline documents.

This is important because of many reasons, but the rapid pace of change might be chief among them. **Cloud is always being updated** and it's hard to know if you've done everything right configuring everything (and even if you did, it might be outdated already). This is where testing comes in, and tools like ScubaGear can be a great help.

## How does it work?

ScubaGear is a PowerShell module that uses the [Microsoft Graph API](https://docs.microsoft.com/en-us/graph/overview?view=graph-rest-1.0) to access the M365 tenant. It then compares the configuration to the SCuBA baseline documents.

![ScubaGear](https://github.com/cisagov/ScubaGear/raw/main/images/scuba-architecture.png)

It can be run either in interactive mode or in a non-interactive mode. The interactive mode is run in the context of a user, and the non-interactive mode is run in the context of an application. Before starting, the user or application must have the necessary permissions to access the M365 tenant.

> **Tip**: When reviewing Graph permissions, you can use the [Microsoft Graph Permission Explorer](https://graphpermissions.merill.net/) to see what permissions are required for a specific operation. Example for the `Sites.FullControl.All` permission: [https://graphpermissions.merill.net/permissions/Sites.FullControl.All](https://graphpermissions.merill.net/permissions/Sites.FullControl.All)

### User Permissions

The user must have the following permissions:

|Access/Role|Description|
|--|--|
|Global Reader|A role that grants read-only access to all administrative units and resources.|
|Power Platform Administrator with a "Power Apps for Office 365" license|A role that grants access to the Power Platform admin center.|
|SharePoint Administrator|SharePoint admin role.|
|Directory.Read.All|Microsoft Graph permission to read Entra ID Directory.|
|GroupMember.Read.All|Microsoft Graph permission to read group members.|
|Organization.Read.All|Microsoft Graph permission to read organization information.|
|Policy.Read.All|Microsoft Graph permission to read policies.|
|RoleManagement.Read.Directory|Microsoft Graph permission to read role management.|
|User.Read.All|Microsoft Graph permission to read users.|
|PrivilegedEligibilitySchedule.Read.AzureADGroup|Microsoft Graph permission to read privileged eligibility schedule (this means PIM assignments in cleartext).|

### Application Permissions

The application must have the following permissions:

|Access/Role|Description|
|--|--|
|Global Reader|A role that grants read-only access to all administrative units and resources.|
|Sites.FullControl.All|Microsoft Graph permission to read and write items and lists in all site collections|
|Directory.Read.All|Microsoft Graph permission to read Entra ID Directory.|
|GroupMember.Read.All|Microsoft Graph permission to read group members.|
|Organization.Read.All|Microsoft Graph permission to read organization information.|
|Policy.Read.All|Microsoft Graph permission to read policies.|
|RoleManagement.Read.Directory|Microsoft Graph permission to read role management.|
|User.Read.All|Microsoft Graph permission to read users.|
|PrivilegedEligibilitySchedule.Read.AzureADGroup|Microsoft Graph permission to read privileged eligibility schedule (this means PIM assignments in cleartext).|

---

## Setup

The [usage](https://github.com/cisagov/ScubaGear?tab=readme-ov-file#usage)-section on the ScubaGear-github page covers most of the setup and eventual troubleshooting. Here's a quick rundown:

1. To get started, visit the [ScubaGear GitHub repository](https://github.com/cisagov/ScubaGear) and head over to download the [latest release](https://github.com/cisagov/ScubaGear/releases/latest). 
2. Extract the files to a folder of your choice.
3. Open a PowerShell prompt and navigate to the folder where you extracted the files.
4. Start by running the `SetUp.ps1` script. This will install all the dependencies.

    ![ScubaGear Setup](/img/Tools/ScubaGear/setupps1.png)

5. Import the module

    ```pwsh
    Import-Module -Name .\PowerShell\ScubaGear
    ```

6. We should now be able to see the module being loaded

    ```pwsh
    (Get-Module ScubaGear).ExportedCommands

    Key                        Value
    ---                        -----
    Copy-ScubaBaselineDocument Copy-ScubaBaselineDocument
    Disconnect-SCuBATenant     Disconnect-SCuBATenant
    Invoke-RunCached           Invoke-RunCached
    Invoke-SCuBA               Invoke-SCuBA
    ```

## Usage

ScubaGear can be run either against all products covered (Entra ID, Defender, Exchange, PowerBI/Power Platform, Sharepoint and OneDrive, Teams) by using the `Invoke-SCuBA` command, or against a specific product by using the `Invoke-SCuBA` command with the `-ProductNames` parameter.

### Run against all products (minus PowerPlatform)

```pwsh
Invoke-SCuBA
```

### Run against a specific product

```pwsh
Invoke-SCuBA -ProductNames aad
```

### Run against multiple products

```pwsh
Invoke-SCuBA -ProductNames aad,defender
```

#### Product Name Mapping

|Product Name|Mapping|
|--|--|
|Entra ID|aad|
|Defender|defender|
|Exchange|exo|
|PowerBI / Power Platform|powerplatform|
|Sharepoint and OneDrive|sharepoint|
|Teams|teams|

---

## Results

After running the tool, you'll get an output in the form of a folder with the naming scheme `M365BaselineConformance_YEAR_MONTH_DAY_HOUR_MINUTE_SECOND`. This folder will be located on the path where you executed the tool, or in the path you specified with the `-OutputPath` parameter. The folder contains the following:

|File|Description|
|--|--|
|TestResults.json|The results of the test in JSON format.|
|TestResults.csv|The results of the test in CSV format.|
|BaseLineReports.html|A HTML-report of the test results.|
|ProviderSettingsExport.json|All exported settings.|
|IndividualReports|A folder containing individual reports for each product.|

---

> If you're a dumb-dumb like me, and ran the tool in the wrong version of Powershell (7) you might not get any Entra ID or SharePoint-results. This can be resolved by following the documentation and running in the correct version of Powershell, which is **5**.

### Reading the HTML-report

The main way to view the results is by opening the `BaseLineReports.html`-file. This file contains a summary of the results, and you can drill down into the individual reports from here.

![ScubaGear HTML-report](/img/Tools/ScubaGear/results.png)

For brevity sake, let's take a look at the summary of the Entra ID-report. The summary contains a list of all the checks that were performed, and the result of each check.

![ScubaGear Entra ID-report](/img/Tools/ScubaGear/aadresults.png)

The colors correspond to the following:

|Color|Description|
|--|--|
|🟢 (Green)|Checks passed|
|🟡 (Yellow)|Checks passed with warnings|
|🔴 (Red)|Checks failed|
|⚪ (Imagine this is grey, *please*)|Checks not applicable, or manual check required|

---

### Updating settings

Let's check a couple of the checks that failed, and how to remedy them.

|Control ID|Requirement|Result|Criticality|Details|
|--|--|--|--|--|
|MS.AAD.7.1v1	|A minimum of two users and a maximum of eight users SHALL be provisioned with the Global Administrator role.	|Fail	|Shall	|1 global admin(s) found: 🤡|
|MS.AAD.7.5v1	|Provisioning users to highly privileged roles SHALL NOT occur outside of a PAM system.	|Fail	|Shall	|2 role(s) assigned to users outside of PIM: Global Administrator, SharePoint Administrator|
|MS.AAD.8.2v1	|Only users with the Guest Inviter role SHOULD be able to invite guest users.|	Warning	|Should	|Permission level set to "everyone" (authorizationPolicy)|

First of all, the critically of the checks are important. If a check is marked as **"Shall", it's a requirement**. If it's marked as **"Should", it's a strong recommendation**.

#### MS.AAD.7.1v1

Here, we are lacking assignments to the Global Administrator role. This is a requirement, and we should remedy this by assigning the role to at least two users. This is to avoid a single point of failure.

#### MS.AAD.7.5v1

Here, we have two roles assigned to users outside of PIM. **This is a requirement, and we should remedy this by assigning the roles to users through PIM**. It's important to note that Break Glass accounts should be excluded from all requirements when it comes to Conditional Access, as they should have no resistance in being able to achieve their purpose and gain Global Admin rights when needed.

Assigning the user roles via PIM **doesn't get in the way of this in theory**. Practially speaking, if PIM (like recently) has been experiencing issues and slow load times then it might impede the function of a BTG-user. This is someting everyone should be aware of and have a plan for.

#### MS.AAD.8.2v1

Here we have a warning, which tells us that the setting that allows users to invite guests is currently set to everyone. This is a strong recommendation, and we should remedy this by **turning off the ability to invite guests for everyone**, and instead only allow administrators with the correct role to invite guests.

## Conclusion

**ScubaGear is a great tool to verify that your M365 tenant is configured according to the SCuBA baseline documents**. It's a great way to ensure that you're not missing anything, and that you're not falling behind on the rapid pace of change that is cloud.

Be careful **not to follow the recommendations blindly**, however. Every environment is different, and some settings might not be applicable to your organization. **It's important to understand the recommendations and the reasoning behind them.**