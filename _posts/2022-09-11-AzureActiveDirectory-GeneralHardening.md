---
layout: post
title: Hardening Azure Active Directory Part 1
subtitle: Going over some attack paths for Azure Active Directory (that I know of) and how to harden your environment to avoid exploitation (or just minimize the risk slightly). In this first part the focus is app registrations and basic enumeration.
tags:
  - Microsoft Sentinel
  - Azure Active Directory
  - Hardening
  - Logging
  - App registration
  - Enterprise applications
  - Consent
published: true
author: author_infernux
image: /img/azad.png
---

# Introduction

The general idea of this "series" of posts is to list up some attack paths or scenarios where each post will outline a bunch of attack paths and it's mitigation. It will not be the most comprehensive guidance (there are great resources to dig deeper into this than I can provide) and the solutions might not cover your needs completely. If you have an idea for improvement or a question, please reach out.

**Update:** 
*For starters I will try to keep all the hardening I recommend free, but I might mention options or alternatives that are available under different licenses.*

# Azure AD - App registrations

## Attack path - external app registration

Consider tooling like [o365-attack-toolkit](https://github.com/mdsecactivebreach/o365-attack-toolkit). It uses an Azure AD App registration in order to trick users into consenting to giving the attacker certain rights in the context of the user, as shown in the image below:

![](https://github.com/mdsecactivebreach/o365-attack-toolkit/raw/master/images/Architecture.png)

Users will be presented with a screen that looks like the following (courtesy of [Jeff Schertz](https://blog.schertz.name/2020/04/enterprise-application-consent-requests-in-azure/)) - but usually asking for way more in terms of permissions:

![](https://blog.schertz.name/wp-content/uploads/2020/04/image-7.png)

Obviously we would like all users to be able to spot permissions that are a over the top and sketchy, like the ability to read a users mail, but that will not always be the case. 

---

**If you want to test how this works for yourself:**

* Go into Azure AD - make a note of the `TenantId`
* Select the "App registrations"-blade
* Click "Create" and fill in like in the image below:

![](/img/AzureAD/CreateAppReg.PNG)
* Under "API permissions" you can add permissions to the app - keep in mind, these are the permissions the app will "require" a user consents to
* Construct an URI for consenting to the application, using this base layout:

```
https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/authorize?client_id={clientId}&response_type=code&response_mode=query&scope={scopes}&state=12345&prompt=consent
```

* `tenantId` is the Azure AD tenant ID, which we made a note of earlier (or you can just grab it from the "Overview"-blade of the app registration)
* `clientId` is the Application (client) ID, which we can find in the "Overview"-blade of the app registration
* `scopes` are the permissions defined under the "API-permissions"-blade, by default this is `User.Read`

You can also test the admin consent flow (which directly requests an admin to approve the application for the entire organization) using the following URI:

```
https://login.microsoftonline.com/{tenant-id|domain}/adminconsent?client_id={client-id}
```

The important thing for users to be vary of are scopes/permissions such as `email`, `offline_access` and other things that application should not normally require.

## Fix - external app registration

We can limit the ability for users to consent to apps, and require admin approval:

* Go into Azure AD
* Select the "Enterprise applications"-blade
* Under "Consent and permissions" you will find the following options:

![](/img/AzureAD/UserConsentSettings.PNG)

* For the strictest option, which is the one I would recommend for small companies or very hardened tenants - select `Do not allow user consent`.
* Generally the option selected in the image above represents the best solution - all apps will require verification to use.
   * *Note, this leaves us vulnerable to the option of an attacker using app registrations internally to pivot - more on that later.*
* At this point we can select the "User settings"-blade, where you will be presented with the following options:

![](/img/AzureAD/EnterpriseAppUserSettings.PNG)
* Generally my advice here is to allow users to request consent to apps they can't consent to by changing the "Admin consent requests" to `yes`, but it all depends on the choice made in terms of allowing users to consent to some or no apps at all. 

Users will now recieve a different screen, stating that you need admin approval:

![](https://preview.redd.it/67pnac6f8i931.png?width=451&format=png&auto=webp&s=6c5523a26fbd113b0a1255177194448512c87684)

## Attack path - internal app registration

All users can by default register new app registrations, so attackers can use app registrations to pivot inside the organization after gaining initial access. You can use the aforementioned o365-attack-toolkit, but now the app registration will originate from inside the organization which means the chance users click it will increase drastically.

As mentioned above, allowing users to consent to apps from verified customers also allows users to consent to app registrations from your own organization - leaving us vulnernable to this attack path.

## Fix - internal app registration

We can limit the ability for users to create app registrations:

* Go into Azure AD
* Select the "User settings"-blade, here you will find the following option:

![](/img/AzureAD/UserSettingsAppReg.PNG)
* Set this to `No`.

# Azure AD enumeration

After gaining access to an account, the first step attackers normally will do is enumerate further. Who has admin access, what groups are there, maybe some memberships are dynamic? Are there any paths to higher privileges (usually courtesy of tools like [AzureHound](https://github.com/BloodHoundAD/AzureHound)) that we can abuse?

### Fix - Azure AD enumeration

We can obviously not fix this entirely, but we can limit the amount of information someone with basic access can grab by default. Limiting the access to Azure AD in the Azure portal to admin users only:

* Go into Azure AD
* Select the "User settings"-blade, here you will find the following option:

![](/img/AzureAD/UserSettingsPortal.PNG)
* Set this to `Yes`.

Please note that this only disables the GUI-access to the Azure Portal, so it will not affect Powershell or other types of API-access. In order to limit API and Azure Portal access we'd need to use a [conditional access policy](https://docs.microsoft.com/nb-no/azure/active-directory/fundamentals/users-default-permissions#restrict-member-users-default-permissions) - this however is behind the Azure AD Premium P1 license.

Please also note that this CA-policy only works for Azure Powershell, not Azure AD Powershell as it uses the Graph API. Limiting access to Graph API is a bit more tricky, but we can do things like [limiting the access users has to each others data](https://docs.microsoft.com/en-us/powershell/module/msonline/set-msolcompanysettings?view=azureadps-1.0) by using the `Set-MsolCompanySettings`-cmdlet with the `-UsersPermissionToReadOtherUsersEnabled` parameter set to `$false`.

# Sources

* [Death from Above: Lateral Movement from Azure to On-Prem AD by SpecterOps](https://posts.specterops.io/death-from-above-lateral-movement-from-azure-to-on-prem-ad-d18cb3959d4d)
* [Azure Dominance Paths by Fabian Bader](https://cloudbrothers.info/en/azure-dominance-paths/)
* [Introduction of Azure AD Attack & Defense Playbook by Sami Lamppu](https://samilamppu.com/2022/03/22/introduction-of-azure-ad-attack-defense-playbook/)
* [How Attackers Move from Azure Active Directory to On-Prem AD by Andy Robbins](https://thenewstack.io/how-attackers-move-from-azure-active-directory-to-on-prem-ad/)