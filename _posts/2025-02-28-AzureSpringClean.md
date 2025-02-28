---
layout: post
title: 'Azure Spring Clean - Maestering Azure Tenant Security'
subtitle: A look into how we can utilize Maester to secure our Azure Tenant with a sprinkle of AI on top
tags:
  - Cyber Security
  - Entra ID
  - Maester
  - Azure
  - Microsoft 365
published: true
author: author_infernux
image: /img/azure.png
---

Welcome back! This time, I'm writing a contribution to the Azure Spring Clean running in March 2025. It's a community effort, so very happy to contribute. 

I've chosen to cover the topic of Azure Tenant Security and how we can use Maester to achieve that. I've written on the topic of testing your own infrastructure before in my post [Test Yourself: The Prelude](https://www.infernux.no/TestYourself/). I also expanded on the topic later in [Test Yourself Part 1: Identity](https://www.infernux.no/TestYourself-Pt2/) (where the URL references -Pt2, which is just me being very good at naming conventions). In that part I covered a Maester and how to run tests, so I recommend also checking that out.

Now, covering tenant security and Maester has it's limits. Maester has a bunch of tests, and going through them one by one isn't helping anyone - **least of all my sanity**. I did cover some tests in some previous posts, so I will summarize and link to those when we need to.

So what's the point of this post? Well, given that we are diving into something as important as Azure Tenant Security, we need to make sure we're doing it right. We need to make sure we're prioritizing the right things, and that we're not missing anything. This is where AI can come in handy. I say can, I mean *"might"* and *"could"* ... most likely *"will not"*. But let's see how it goes.

So, let's get started!

# Let's define Azure tenant security

First of all, an Azure tenant consists of two logical pieces in my mind:
1. **Azure tenant**
2. **Microsoft 365 tenant**

Both of these two components are connected in that they share the same **identity**, but they are separate in that they provide access differently. To put it very generically, **Azure uses Entra ID and Azure role based access control (RBAC)** to provide access to resources, while **Microsoft 365 uses Entra ID and the Entra ID roles** to provide access.

There are **some exceptions of course**, like Defender (unified) RBAC, or Global Admins being able to essentially become owners of all Azure resources, but for the most part this is how it works.

## Azure tenant security

In the context of this blog post, I have chosen to define the tenant security settings as everything covered by Maester. For a full list, check out [Maester's documentation](https://maester.dev/docs/tests/). Currently, the tests are split into four broad categories:

1. 🔥 Maester Tests
   * These tests validate if your Microsoft configuration is following generally recommended best practices.
2. 🛡️ Entra ID Security Config Analyzer Tests
   * These tests validate tenant security config and are based on the Entra ID Security Config Analyzer
3. 🦅 CISA Tests
   * The tests in this section verifies that a Microsoft 365 tenant’s configuration conforms to the policies described in the Secure Cloud Business Applications (SCuBA) Security Configuration Baseline documents.
4. 🌀 CIS Tests
   * The tests in this section verifies that a Micorosft 365 tenant's configuration conforms to the CIS Microsoft 365 Foundations Benchmark recommendations (v3.1.0).
   
   These settings are core because they are important **even if we don't have any resources deployed**. Anything that uses single sign-on (SSO) will most likely be affected by these settings.

## Talking context

To put the context of this blog post into perspective it's important to note that I'm generally going to advocate strict security settings, but we also need to consider that we need to balance security with usability. So please keep that in mind. **At scale some of these settings might cause issues with less tech-savvy users, or just be too much to manage**. 

# Maester

## Introduction to Maester

So what is [Maester.dev](https://maester.dev/)? Well, it's a test framework built on top of Pester, a Powershell test framework. Maester allows you to test your infrastructure by allowing you to run tests against it. I've covered [ScubaGear](https://www.infernux.no/ToolsYouShouldKnow-ScubaGear/) from CISA before, and the Secure Cloud Business Applications (SCuBA) tests are all part of Maester, along with some Maester-specific tests, the [Entra ID Security Config Analyzer (EIDSCA) tests](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense/blob/main/AADSecurityConfigAnalyzer.md) from the [Entra ID Attack and Defense Playbook](https://github.com/Cloud-Architekt/AzureAD-Attack-Defense) and finally the option to write your [own custom tests!](https://maester.dev/docs/writing-tests)

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
Invoke-Maester -Verbosity Normal -OutputFolder "C:\temp\AzureSpringClean\output"
```

This will generate a report in the folder `C:\temp\AzureSpringClean\output` that we can open in a browser.

## Reading Maester tests

Now, I've already gone through some tests in my previous post ([which you can view over here](https://www.infernux.no/TestYourself-Pt2/#going-through-the-report)). Please check out that article, if not now, then at least later. It was a lot of work to write, since I also went through the process of remediating multiple tests. 

For sake of clarity, last time we walked through and made changes to the following tests:

#### Guest Access Settings
- **EIDSCA.AP04: Default Authorization Settings - Guest invite restrictions**
  - Initially failed due to incorrect guest invite restrictions settings.
  - Updated to "Only users assigned to specific admin roles can invite guest users" to pass the test.

- **EIDSCA.AP07: Default Authorization Settings - Guest user access**
  - Initially failed due to incorrect guest user access settings.
  - Updated to "Guest users access is restricted to properties and memberships of their own directory objects (most restrictive)" to pass the test.

#### Password Policy
- **EIDSCA.PR05: Default Settings - Password Rule Settings - Smart Lockout - Lockout duration in seconds**
  - Initially failed due to not explicitly setting the lockout duration.
  - Updated to 120 seconds to pass the test.

- **EIDSCA.PR06: Default Settings - Password Rule Settings - Smart Lockout - Lockout threshold**
  - Initially failed due to not explicitly setting the lockout threshold.
  - Updated to 5 attempts to pass the test.

#### Conditional Access
- **MS.AAD.3.1: Phishing-resistant MFA SHALL be enforced for all users**
  - Failed due to not enforcing phishing-resistant MFA for all users.
  - Discussed the balance between security and usability.

#### Consent and Permissions
- **MS.AAD.5.3: An admin consent workflow SHALL be configured for applications**
  - Failed due to not configuring an admin consent workflow for applications.
  - Updated the settings to pass the test.

#### Authentication Methods
- **EIDSCA.AG01: Authentication Method - General Settings - Manage migration**
  - Failed due to the migration status being "migrationInProgress".
  - Updated to "Complete" to pass the test.

- **EIDSCA.AG02: Authentication Method - General Settings - Report suspicious activity - State**
  - Failed due to the setting being "Microsoft Managed".
  - Updated to "Enabled" to pass the test.

- **EIDSCA.AF01: Authentication Method - FIDO2 security key - State**
  - Failed due to the FIDO2 security key not being enabled.
  - Enabled the FIDO2 security key and registered it to pass the test.

- **EIDSCA.AM06: Authentication Method - Microsoft Authenticator - Show application name in push and passwordless notifications**
  - Failed due to the setting being "Microsoft Managed".
  - Updated to "Enabled" to pass the test.

- **EIDSCA.AM09: Authentication Method - Microsoft Authenticator - Show geographic location in push and passwordless notifications**
  - Failed due to the setting being "Microsoft Managed".
  - Updated to "Enabled" to pass the test.

- **EIDSCA.AT01: Authentication Method - Temporary Access Pass - State**
  - Failed due to the setting being "False".
  - Enabled Temporary Access Pass and configured additional settings to pass the test.

#### Authorization Settings
- **EIDSCA.AP01: Default Authorization Settings - Enabled Self service password reset for administrators**
  - Failed due to the setting being "True".
  - Updated to "False" to pass the test.

### Results from last time 

From last time, our results looked like this:

```
Tests completed in 36.14s
Tests Passed: 57, Failed: 65, Skipped: 11, Inconclusive: 0, NotRun: 5
🔥 Maester test report generated at C:\temp\AzureSpringClean\output\TestResults-2024-10-26-124008.html
```

![](/img/Maester/newsummary.png)

We moved from a 32% pass rate to a 47% pass rate. This is a good improvement, but we still had a long way to go.

## Getting back to work

So, with some summary from last time, now it's time to get to .. **spring cleaning?** ***I guess?*** Let's see what we can do to improve our security settings. But let's put a little spin on it. **I'm normally not a fan of "AI-powered"-everything**, but for this use case it might be a good tool to use. I usually advocate AI as a good tool to summarize text and aggregata data. Imagine we have a limited supply of people to do the work, and manually reviewing and prioritizing all the tests might be a bit much. So, let's use some AI to help us out.

To put it in the immortal words of Ludwig: *"Boys, the plan is simple"*. 

![](/img/lud.gif)

We'll run Maester and then make the results available to either Security Copilot or Azure OpenAI to help us prioritize what to fix.

### Setting up the data source

We will use a storage account as a data source - so we'll start creating that. 

![](/img/AzureSpringClean/storageacc.png)

For the data to appear in the storage account, we'll need to either upload the data manually, or, if we're running Maester in a pipeline we can modify it to upload the data to the storage account. I'm using [Azure DevOps, set up according to the Maester guidance](https://maester.dev/docs/monitoring/azure-devops), but you can use Github or Gitlab if you feel like it.

A small note here, if you're running Azure DevOps just to test like me, you need to apply for a free tier pipeline. You can do that [here](https://aka.ms/azpipelines-parallelism-request).

Now, the pipeline itself already publishes the report as an artifact using the following task:

```yaml
- publish: $(System.DefaultWorkingDirectory)/test-results
    displayName: Publish Maester Html Report
    artifact: TestResults
```

All the test-results are saved to that same folder, so in order to upload the report to the storage account, we can add a task to the pipeline like this:

```yaml
- task: AzureFileCopy@4
  inputs:
    SourcePath: '$(System.DefaultWorkingDirectory)/test-results/*'
    azureSubscription: 'maester'
    Destination: 'AzureBlob'
    storage: 'infexaistorageacc'
    ContainerName: 'maester'
    BlobPrefix: ''
```

**The `SourcePath` variable supports wildcards**, so we can upload all the files in the folder. Also note that I made a container in the storage account named `maester`, so we're uploading the files there.

This will upload the report to the storage account:

![](/img/AzureSpringClean/testresults.png)


Next, we'll spin up an Azure AI Search instance using the basic tier. 

![](/img/AzureSpringClean/aisearch.png)

**Small note here** - you might need to configure an Azure OpenAI instance and set up a supported model (the vector field must use `text-embedding-ada-002`.
) to make it all work. I followed [this guide](https://learn.microsoft.com/en-us/azure/search/search-get-started-portal-import-vectors?tabs=sample-data-storage%2Cmodel-aoai%2Cconnect-data-storage/?wt.mc_id=SEC-MVP-5005030) to set everything up and be able to upload vectorized data.

Now, let's **jump into Azure AI search and continue there**. Inside Azure AI search we can simply select import and vectorize data and add our storage account as a data source:

![](/img/AzureSpringClean/importandvectorize.png)

**Follow the wizard**. For the sake of brevity, keep **everything as default**. If all is well in the world of automagic, you should see your data source indexed, vectorized and searchable. If you're like me and these terms confuse you and make you feel a bit scared, don't worry. **We're in this together.**

![](/img/pal.png)

*I'm afraid I can't let you leave just yet, Dave. We need to set up the AI to help us prioritize the tests 🤖🔧.*

### Option 1: Security Copilot

Let's spin up a Security Copilot instance following [Microsoft's guide](https://learn.microsoft.com/en-us/copilot/security/get-started-security-copilot/?wt.mc_id=SEC-MVP-5005030). Now, to make our files available to Security Copilot, we're going to use Azure AI Search to index the files and make them searchable. We've already created a storage account, so we'll upload the files there.

Heading over to [https://securitycopilot.microsoft.com/](https://securitycopilot.microsoft.com/), we can click on sources to add a new source:

![](/img/AzureSpringClean/sources.png)

This opens up a window, where we can select among a few different sources. The important one is Azure AI Search, which we'll select:

![](/img/AzureSpringClean/secaisearch.png)

Clicking "Set up" takes us to fill out some data:

![](/img/AzureSpringClean/settings.png)

For more detailed guidance on this part, I recommend checking out the [Microsoft guide](https://learn.microsoft.com/en-us/copilot/security/plugin-azure-ai-search/?wt.mc_id=SEC-MVP-5005030). 

**Note:** Requirements for Azure AI Search Plugin

To use the Azure AI Search plugin in Copilot, ensure that the Azure AI Search index is configured as follows:

- The text field must be searchable.
- The title field must be filterable.
- The vector field must use `text-embedding-ada-002`.

Given all this, the following values are present in our index `vector-1740331313661`:

![](/img/AzureSpringClean/datafields.png)

So the values we would use is as follows:

| Field Name | Value |
|------------|-------|
| Azure AI Search Service| `infex-ai-search` |
| Index Name | `vector-1740331313661` |
| Name of vector field| `text_vector` |
| Name of title field | `title` |
| Name of text field | `chunk` |
| Value | This will be your API key from going into Azure AI search, selecting keys and copying an admin key.|

According to [Microsoft guidance](https://learn.microsoft.com/en-us/copilot/security/connect-org-kb#prompting-for-a-knowledge-base-connected-using-azure-ai-search/?wt.mc_id=SEC-MVP-5005030), we can now prompt for a knowledge base connected using Azure AI Search. Please refer to guide for more information.

#### Just upload the files manually

We can also just, you know, upload the file manually in Security Copilot:

![](/img/AzureSpringClean/file.png)

Simple as that.

### The Other Option (2): Azure OpenAI

If you're not interested in shelling out to use Security Copilot, but you still don't want to upload what is basically your security configuration to ChatGPT directly, we can spin up an Azure OpenAI instance.

![](/img/AzureSpringClean/openai.png)

From here, we can head to [https://ai.azure.com](https://ai.azure.com) and head over to the playground. Here we can head into the playground and add our data. 

![](/img/AzureSpringClean/chatplayground.png)

We can either chose to upload the data manually, or we can use a data source like the storage account or Azure AI search. In this case, we'll use the Azure AI search since we have already set up uploading from our pipeline.

![](/img/AzureSpringClean/datasource.png)

We can now select the data source we set up earlier:

![](/img/AzureSpringClean/datasource2.png)

## Let's prompt 

Now, I'm not an AI-expert, nor am I really convinced this is the best course of action, but let's see what the AI says.

First out, let's check with Security Copilot! **Our prompt will be this:**

```
We have run Security tests using a tool called Maester on our Azure tenant and have uploaded the results to Azure AI Search. We would like to prioritize the tests based on the results. Can you help us with that?
```

### Security Copilot Output

Prioritized Test Results

**High Priority (Failed Tests)**

1. EIDSCA.AF04: Authentication Method - FIDO2 security key - Enforce key restrictions.
    - Status: Failed
    - Impact: Ensuring key restrictions are enforced is critical for security.

2. EIDSCA.AG03: Authentication Method - General Settings - Report suspicious activity - Included users/groups.
    - Status: Failed
    - Impact: Reporting suspicious activity is essential for monitoring and responding to potential security threats.

3. Permanent active role assignments without expiration to privileged roles.
    - Status: Failed
    - Impact: Permanent assignments to privileged roles increase the risk of exposure if accounts are compromised.

**Medium Priority (Passed Tests)**

- EIDSCA.AF01: Authentication Method - FIDO2 security key - State.
    -    Status: Passed
    -    Impact: Ensuring the FIDO2 security key state is enabled is important for secure authentication.

- EIDSCA.AF02: Authentication Method - FIDO2 security key - Allow self-service set up.
     -   Status: Passed
    -  Impact: Allowing self-service setup for FIDO2 keys enhances user convenience and security.

- EIDSCA.AF03: Authentication Method - FIDO2 security key - Enforce attestation.
    -    Status: Passed
    -    Impact: Enforcing attestation ensures the authenticity of security keys.

- EIDSCA.AG01: Authentication Method - General Settings - Manage migration.
    -    Status: Passed
    -    Impact: Managing migration settings is crucial for a smooth transition and maintaining security.

- EIDSCA.AG02: Authentication Method - General Settings - Report suspicious activity - State.
    -   Status: Passed
    -   Impact: Having a state for reporting suspicious activity is vital for security monitoring.

**Summary**

  Total Tests: 138
  Passed: 57
  Failed: 65
  Not Run: 11

### Azure OpenAI Output

Now we give Azure OpenAI the same prompt using the Azure AI foundry and the chat integration:

To prioritize the security tests based on the results from your Azure tenant using Maester, we can categorize the tests into "Pass" and "Fail" based on their outcomes. Here’s a summary of the test results:

**Tests with Failures (High Priority)**

1. MT.1024: Entra Recommendation - Ensure all users can complete multifactor authentication.
    -  Overview: 12 of 13 users are not registered with MFA.
    -   Recommendation: Ensure all users complete MFA for improved security.
2.  MT.1029: Stale accounts are not assigned to privileged roles.
    -    Overview: 1 account in a privileged role has not signed in for over 30 days.
    -   Recommendation: Regularly review accounts with privileged roles and remove unnecessary assignments. 
3. EIDSCA.AP07: Default Authorization Settings - Guest user access.
    - Overview: Guest users may have excessive access to Azure AD directory objects.
    - Recommendation: Limit guest user access as per CISA SCuBA 2.18 guidelines.
4. MT.1024: Entra Recommendation - Protect all users with a user risk policy.
    - Overview: All users lack a user risk policy, which is essential for detecting account compromise.

**Tests with Passes (Lower Priority)**

- EIDSCA.CP03: Default Settings - Consent Policy Settings - Block user consent for risky apps.
    -    Status: Passed; configuration set to 'true'.
    - Authentication method configurations (Voice)
    -   Status: Passed; configuration set to 'disabled'.

Prioritization Strategy

- High Priority: Focus on tests that failed, especially those related to user authentication and access management, as they pose immediate security risks.
- Medium to Low Priority: Address tests that passed but may require ongoing monitoring or adjustments to improve overall security posture.

By addressing the high-priority tests first, you can significantly enhance the security of your Azure tenant. If you need further assistance or specific action steps for each test, please let me know!

## Conclusion on AI Prioritization

Well, reviewing some of the recommendations from the AI, it seems like we have some work to do. For instance, if we investigate the **second priority from Security Copilot, `EIDSCA.AG03`** which refers to the setting "Report suspicious activity - Included users/groups". **It supposedly failed**, but looking into the file manually we see that **this is not actually the case:**

![](/img/AzureSpringClean/actually.png)

In this case, **Azure OpenAI was able to get all four recommendations correct.** This is a good example of why it's **important to verify the results from AI**, as they might not always be correct. It's also probably a ***good idea to have better prompt engineering skills than me***, but that's a different story. Things like asking the AI to prioritize based on the number of users affected, or the potential impact of the setting might be a good idea.

# What have we learned?

Through this post, we have looked at the following things:

- A brief introduction to Azure Tenant Security
- A brief introduction to Maester
- How to set up Maester
- How to run Maester
- How to read Maester tests (linked to previous posts, with summary of results)
- How to upload Maester test results to Azure AI Search (storage account) via Azure DevOps pipelines
    - Please note that you need to configure your indexer to run on a schedule to keep the data up to date.
- Settings up Security Copilot and Azure OpenAI to use Azure AI Search
- Prompting Security Copilot and Azure OpenAI for prioritization of tests
- Minor review of the results from the AI

Some reflections here - Maester is a godlike tool. We haven't discussed it, but as long as something has an API we can also write customs tests which is well documented on the Maester page. **There's also potential for some good reporting here, by uploading multiple reports and comparing them over time.** This can be a great tool for improving security settings over time, especially if you're giving the AI a bit more to work with in terms of context. 

All in all, this was a fun little experiment. I look forward to experimenting with it more, and maybe setting it up to automatically upload the reports to Azure AI Search every week, then having a weekly report generated by the AI using a logic app or something like that.

**Have a nice spring cleaning, everyone!**

Please visit [Azure Spring Clean](https://www.azurespringclean.com/) for more content from the community.