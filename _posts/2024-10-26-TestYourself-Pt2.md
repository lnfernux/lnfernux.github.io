---
layout: post
title: 'Test Yourself Part 1: Identity'
subtitle: Some tips, tricks and tools to help you get started testing your own infrastructure. This is the part 1 where we'll look into identity and how you can test it.
tags:
  - Cyber Security
  - Entra ID
  - Security Monitoring
  - Entra ID
  - Azure
  - Microsoft 365
published: true
author: author_infernux
image: /img/azure.png
---

In the last article called "Test Yourself: The Prelude", we talked about some basic principles of security that we need to keep in mind moving forward. In this blog, we're going to focus on identity and how you can start making sure you're secure in that area. If you haven't already given my article on threat modeling a read, I suggest you do so before continuing as it will give you a good foundation to take this blogpost and apply it to your own environment.

- [Test Yourself: The Prelude](https://www.infernux.no/TestYourself/)
- [Threat Modeling](https://www.infernux.no/SecurityMonitoring-DataSources/)

# Table of Contents

- [Identity](#identity)
- [Example Company: Infernux Inc](#example-company-infernux-inc)
- [Starting with the basics](#starting-with-the-basics)
- [Introduction to Maester](#introduction-to-maester)
  - [Getting started with Maester](#getting-started-with-maester)
  - [Installing Maester](#installing-maester)
  - [Running Maester](#running-maester)
- [Going through the report](#going-through-the-report)
  - [Guest Access Settings](#guest-access-settings)
  - [Password Policy](#password-policy)
  - [Conditional Access](#conditional-access)
  - [Consent and Permissions](#consent-and-permissions)
  - [Authentication Methods](#authentication-methods)
  - [Authorization Settings](#authorization-settings)
- [Rerunning Maester](#rerunning-maester)
- [Conclusion](#conclusion)

# Identity

Is identity the new, primary perimeter for security? Maybe it is, but regardless, it's becoming an increasingly important part of security. We know for a fact that active directory is at the center of a lot of breaches historically (and still is, for some reason). Even more so now, when most companies are moving to the cloud and you can work from Bali, Starbucks or Home (at least for now), cloud identity is becoming more and more important.

## Example Company: Infernux Inc

Let's take an example company, Infernux Inc. In this case, they're a pretty new startup with a cloud-first approach. They're only in the cloud, as of now. Microsoft 365 with Teams, Outlook and Sharepoint is the main way to collaborate inside the company and also to access any other business-tools they might need. For all of this, Entra ID is the main identity provider. They currently get devices from the company, but it's not managed in any way.

What does this tell us?

1. Even if the computers are handed out by the company, they're not managed and can be considered similar to BYOD-devices. 
2. The company is cloud-first, which means that the identity provider is the main way to access anything.

# Starting with the basics

I wrote an article recently about [Hardening Entra ID](https://www.infernux.no/EntraID-GeneralHardening/) which is a very light introduction to how you can start securing your ~~Azure AD~~ Entra ID. I suggest you give it a read if you haven't already. It covers some of the most important things to consider when setting up a new Azure tenant on the Entra ID side, with some references to other resources that can help you further, but is by no means exhaustive.

Now the **entire idea of this blog is very simple**. We're going to **install and run maester**, **review the results** using my previous Entra ID hardening guide as a reference, and **fix the issues** that we find as we go. I might not cover everything mentioned in the article, but the idea is to **show you how you can use maester to test your own environment and fix the issues you find**. 

It's not 🚀 science, it's 🔥 Maester.

# Introduction to Maester

So what is [Master.dev](https://maester.dev/)? Well, it's a test framework built on top of Pester, a Powershell test framework. Maester allows you to test your infrastructure by allowing you to run tests against it. I've covered [ScubaGear](https://www.infernux.no/ToolsYouShouldKnow-ScubaGear/) from CISA before, and the Secure Cloud Business Applications (SCuBA) tests are all part of Maester, along with some Maester-specific tests, the [Entra ID Security Config Analyzer (EIDSCA) tests](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/AADSecurityConfigAnalyzer.md) from the [Entra ID Attack and Defense Playbook](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense) and finally the option to write your [own custom tests!](https://maester.dev/docs/writing-tests)

This is all made possible by the Maester PowerShell module, which runs the tests and generates interactive test reports for you to review outcomes. It can also be set up to run in a CI/CD pipeline, so you can run tests either on a schedule or automatically, triggered by new deployments.

![](https://maester.dev/img/home/maester-architecture.png)

## Getting started with Maester

The introduction guide for setting up Maester is solid, so I won't go into much details here apart from sharing the commands I run to generate my report. You can find the guide [here](https://maester.dev/docs/installation).

## Installing Maester

Start up a PowerShell session and run the following commands:

```powershell
Install-Module Pester -SkipPublisherCheck -Force -Scope CurrentUser
Install-Module Maester -Scope CurrentUser
```

At this point, we got a warning because I've previously installed the Maester module to do testing, so we had an old version (0.0.130) installed.

```
WARNING: Version '0.0.130' of module 'Maester' is already installed at 'C:\Users\infx\Documents\PowerShell\Modules\Maester\0.0.130'. To install version '0.3.0', run Install-Module and add the -Force parameter, this command will install version '0.3.0' side-by-side with version '0.0.130'.
```

To fix this, we can update the module:
  
```powershell
Update-Module Maester -Force
Import-Module Maester
```

Now, let's create a directory for our tests and navigate into it:

```powershell
md maester-tests
cd maester-tests
Install-MaesterTests
```

Output will look like this:

![](/img/Maester/installtests.png)

If you're like me and have already installed Maester and the tests before, you can run the following command to update the tests:

```powershell
cd maester-tests
Update-MaesterTests
```

There are a few other things we might want to include as well, because the SCuBA tests from CISA requires the Az and ExchangeOnlineManagement modules. We can install these by running the following commands:

```powershell
Install-Module Az -Scope CurrentUser
Install-Module ExchangeOnlineManagement -Scope CurrentUser
```

If the modules are already installed, you can update them by running the following commands:

```powershell
Update-Module Az -Force
Update-Module ExchangeOnlineManagement -Force
```

This is all we need to do for setup right now.

## Running Maester

To run Maester, we need to run the following command:

```powershell
Connect-Maester
```

This should bring you to a consent window, which looks like this (pardon the Norwegian there): 

![](/img/Maester/consentrequest.png)

**Note**, if we want to run all the tests, including SCuBA tests, we need to connect to all services:

```powershell
Connect-Maester -Service All
```

Now we can run Maester:

```powershell
Invoke-Maester
```

The `Invoke-Maester` command has a few parameters that can be useful, it's documented on the [maester.dev](https://maester.dev/docs/commands/Invoke-Maester) website. For our purposes, to get a bit more output we'll use the `-Verbosity` parameter and set it to normal, as well as the different output formats and output folder:

```powershell
Invoke-Maester -Verbosity Normal -OutputHtmlFile -OutputJsonFile -OutputFolder "C:\temp\TestYourselfPt2\output"
```

And we are off:

![](/img/Maester/running.png)

*Not looking hot for Infernux Inc...*

When the tests are completed, the report (HTML) should open automatically in the default browser. The command line should also show you some generic info about the tests that were run, and if there were any failures:

```
Tests completed in 60.63s
Tests Passed: 36, Failed: 77, Skipped: 20, Inconclusive: 0, NotRun: 5
🔥 Maester test report generated at C:\temp\TestYourselfPt2\output\TestResults-2024-10-26-114438.html
```

The report will look something like this:

![](/img/Maester/maesterreport.png)

# Going through the report 

The report is quite extensive, and I won't go through all of it here. But I'll show you some of the things that are interesting to look at. Let's first scope things out - we are first and foremost looking into Entra ID today. For the most part, we will base our review on the settings discussed in the [Hardening Entra ID](https://www.infernux.no/EntraID-GeneralHardening/) article.

## Guest Access Settings

### EIDSCA.AP04: Default Authorization Settings - Guest invite restrictions.

This test fails initially. This was an error on my part, as I had not set the guest invite restrictions. This is a good example of how Maester can help you identify things you might have missed. In my [recommendation here](https://www.infernux.no/EntraID-GeneralHardening/#guest-invite-settings) I had recommended the value to be set to `Only users assigned to specific admin roles can invite guest users` but forgot to update it.

After changing it, the test passes:

![](/img/Maester/testpass.png)

We can also see the test details, like where the recommendation is coming from, in this case it's from CISA SCuBA 2.18: "Only users with the Guest Inviter role SHOULD be able to invite guest users".

### EIDSCA.AP07: Default Authorization Settings - Guest user access.

In my article I recommended ["Guest user access is restricted to properties and memberships of their own directory objects (most restrictive)"](https://www.infernux.no/EntraID-GeneralHardening/#guest-user-access). Again, in my tenant I had forgot to change from the default "Guest users have limited access to properties and memberships of directory objects":

![](/img/Maester/testfail.png)

## Password Policy

### EIDSCA.PR05: Default Settings - Password Rule Settings - Smart Lockout - Lockout duration in seconds.

Now, this test is pretty cool because it recommends `60` (I recommended `120` in my article, but `60` is also fine). The test fails because the value is not excplicitly set in my tenant, meaning that if Microsoft changes the default in the future, it will also change in my tenant. So it's a good idea to set this explicitly to at least `60`.

### EIDSCA.PR06: Default Settings - Password Rule Settings - Smart Lockout - Lockout threshold.

This is the same as the above, where the the test will fail even if the correct treshold is set because it's not explicitly set.

## Conditional Access

### MS.AAD.1.1: Legacy authentication SHALL be blocked.

This is a pass, since we have blocked legacy authentication in our tenant using CA policies.

### MS.AAD.2.1: Users detected as high risk SHALL be blocked.

I didn't include any recommendations directly in my article, but I referenced [Daniel Chronlund's conditional access baseline work](https://danielchronlund.com/2020/11/26/azure-ad-conditional-access-policy-design-baseline-with-automatic-deployment-support/) as a reference, which inludes blocking high risk users. This is a pass.

### MS.AAD.2.3: Sign-ins detected as high risk SHALL be blocked.

Same as above, pass.

### MS.AAD.3.1: Phishing-resistant MFA SHALL be enforced for all users.

This fails because I have not enforced phishing-resistant MFA for all users. This is a good example of how Maester can help you identify things you might have missed, or things that you might not "agree" on with the recommendations. 

To put it in other words, yes, phishing resistant MFA might be the best security option, but we also have to take into account the context of the systems we have. Not all users have privileged access to systems, and not all systems are critical. So, in some cases, it might be better to have a more granular approach to MFA that makes sense for your organization and balances usability.

## Consent and Permissions

### MS.AAD.5.3: An admin consent workflow SHALL be configured for applications.

This is a fail, because I have not configured an admin consent workflow for applications. While `User consent settings` was set to `Do not allow user consent` we didn't configure the `Admin consent requests` settings:

![](/img/Maester/adminconsentflow.png)

## Authentication Methods

### EIDSCA.AG01: Authentication Method - General Settings - Manage migration.

This is a setting that many people might forget even exists, I just included it because I see a lot of tenants that are still in `migrationInProgress` mode. Information for this:

> The state of migration of the authentication methods policy from the legacy multifactor authentication and self-service password reset (SSPR) policies. In January 2024, the legacy multifactor authentication and self-service password reset policies will be deprecated and you'll manage all authentication methods here in the authentication methods policy. Use this control to manage your migration from the legacy policies to the new unified policy.
> On September 30th, 2025, the legacy multifactor authentication and self-service password reset policies will be deprecated and you'll manage all authentication methods here in the authentication methods policy. Use this control to manage your migration from the legacy policies to the new unified policy.

To fix it, we can click on the `View in Microsoft Entra admin center` link and then click on `Change` in the Migration status section. Set it to `Complete` and save:

![](/img/Maester/migrationcomplete.png)

### EIDSCA.AG02: Authentication Method - General Settings - Report suspicious activity - State.

This fails, even if the setting is "correct" because it's set to default. Here we can go in and excplitly set the setting to `Enabled` to change this to a pass. We simply navigate to [authentication method settings](https://portal.azure.com/#view/Microsoft_AAD_IAM/AuthenticationMethodsMenuBlade/~/AuthMethodsSettings) and change the state from Microsoft Managed to Enabled.

### EIDSCA.AF01: Authentication Method - FIDO2 security key - State.

This fails. Ruh-roh, I even thought I had this enabled. So I went to the [authentication methods](https://portal.azure.com/#view/Microsoft_AAD_IAM/AuthenticationMethodsMenuBlade/~/AdminAuthMethods) section and enabled it. 

I also got my FIDO2 key out from my drawer and travelled to [myaccount.microsoft.com](https://myaccount.microsoft.com) to register it.

### EIDSCA.AM06: Authentication Method - Microsoft Authenticator - Show application name in push and passwordless notifications.

This fails, because I have this set to Microsoft Managed. We can go to [authentication methods](https://portal.azure.com/#view/Microsoft_AAD_IAM/AuthenticationMethodsMenuBlade/~/AdminAuthMethods), chose Microsoft Authenticator settings and set it to enabled:

![](/img/Maester/showapplicationnamemfa.png)

That's a pass!

### EIDSCA.AM09: Authentication Method - Microsoft Authenticator - Show geographic location in push and passwordless notifications.

This fails, because I have this set to Microsoft Managed. We can go to [authentication methods](https://portal.azure.com/#view/Microsoft_AAD_IAM/AuthenticationMethodsMenuBlade/~/AdminAuthMethods), chose Microsoft Authenticator settings and set it to enabled:

![](/img/Maester/showgeographicloc.png)

What can we say to that? 

![](/img/anothaone.gif)

*Another one!*

### EIDSCA.AT01: Authentication Method - Temporary Access Pass - State.

This setting is set to `False`. The recommended setup is to enable this, and also require both compliant device and MFA on registration for new users. This requires Temporary Access Pass (TAP) to be enabled for the MFA portion during registration.

We head over to the [authentication methods](https://portal.azure.com/#view/Microsoft_AAD_IAM/AuthenticationMethodsMenuBlade/~/AdminAuthMethods) section and enable it:

![](/img/Maester/tap.png)

Here I made some changes, like changing the maximum lifetime to 2 hours and requiring one time use. I also increased the characters from 8 to 14 for some reason. Maybe it's my caveman brain going ***"more number more good"***.

## Authorization Settings

### EIDSCA.AP01: Default Authorization Settings - Enabled Self service password reset for administrators.

This fails, because I have not changed this setting. It's by default set to `True`. 

If we go to the [password reset](https://entra.microsoft.com/#view/Microsoft_AAD_IAM/PasswordResetMenuBlade/~/Properties/fromNav/) section in the Entra ID admin center we can see that there is a informational notification that says:

![](/img/Maester/adminsspr.png)

This takes us [to the Password policies and account restrictions in Microsoft Entra ID](https://learn.microsoft.com/nb-no/entra/identity/authentication/concept-sspr-policy#administrator-password-policy-differences) article on Microsoft Learn. 

In short, you can update this setting doing the following: 

You can disable the use of SSPR for administrator accounts using the Update-MgPolicyAuthorizationPolicy PowerShell cmdlet. The -AllowedToUseSspr:$true|$false parameter enables/disables SSPR for administrators. Policy changes to enable or disable SSPR for administrator accounts can take up to 60 minutes to take effect.

```powershell
Update-MgPolicyAuthorizationPolicy -AllowedToUseSspr:$false
```

Running `Get-MgPolicyAuthorizationPolicy | fl` afterwards should show that the `AllowedToUseSspr` is set to `False`:

![](/img/Maester/updatedsspradminsetting.png)

That's one more fail moved to a pass!

# Rerunning Maester

At this point, we probably could have kept going, but for the sake of brevity I think the point is made. We can now rerun Maester and see how we're doing. On our first run we had 77 fails, 36 passes and 20 skipped. Let's see how we're doing now:

```powershell
Tests completed in 36.14s
Tests Passed: 57, Failed: 65, Skipped: 11, Inconclusive: 0, NotRun: 5
🔥 Maester test report generated at C:\temp\TestYourselfPt2\output\TestResults-2024-10-26-124008.html
```

Well damn, we're getting there! We've moved 20 fails to passes, and we're down to 65 fails. We're also down to 11 skipped tests, which is good. 

To explain why we suddenly have less skipped tests and the number of failed tests isn't lowered by the same amount as our increased passes, this is because some tests are dependent on others. If a test fails, the dependent tests will be skipped. So what we have done in this case is to fix some of the root causes of the failures, which in turn has made other tests run (and likely fail).

![](/img/Maester/newsummary.png)

We have moved from a 32% pass rate to a 47% pass rate. Not bad for a few hours of work!

## All the commands for running the Maester tests

This assumes you're installing everything for the first time. If you've already installed Maester and the tests, you can skip the first part and go straight to the Maester commands.

```powershell
# Install Maester
Install-Module Pester -SkipPublisherCheck -Force -Scope CurrentUser
Install-Module Maester -Scope CurrentUser

# Create folders for Maester
md maester-tests
cd maester-tests
Install-MaesterTests

# Install SCuBA prerequisites
Install-Module Az -Scope CurrentUser
Install-Module ExchangeOnlineManagement -Scope CurrentUser

# Connect o Maester
Connect-Maester -Service All

# Run Maester
$OutputFolder = "C:\temp\TestYourselfPt2\output" # Change this
Invoke-Maester -Verbosity Normal -OutputFolder $OutputFolder
```

# Conclusion

In this blogpost we've gone through how you can use Maester to test your Entra ID tenant, and how you can use the results to fix issues in your tenant. There's a lot more things to fix about my dev tenant, but we've covered the process and how to read the results. 

For future articles, we'll try to cover some more of Azure and Azure RBAC, Microsoft 365 and it's tools like Exchange and Teams to a certain degree. We'll focus on some other tools aside from Maester. The idea is that Maester reviews your posture, but other tools can help you map out any potential attack paths (like AzureHound), help you simulate attacks to make sure they are either blocked or detected. All these things are important to include when testing your own environment.

Comments, feedback? Reach out anywhere you can find me!