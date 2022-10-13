---
layout: post
title: Enable Defender for DevOps in Azure DevOps pipelines
subtitle: Quick introduction to Defender for DevOps and how to enable it in an Azure DevOps pipeline.
tags:
  - Cloud Security
  - Defender for Cloud
  - Defender for DevOps
  - Azure DevOps
published: true
author: author_infernux
image: /img/devops.png
---

# Introduction

Defender for DevOps is now in [public preview](https://www.microsoft.com/en-us/security/business/cloud-security/microsoft-defender-devops) - this means we are able to integrate with Azure DevOps and Github to surface any issues with our CI/CD pipelines in Defender for Cloud. In this short post I'll quickly get you up and running with Defender for DevOps.

There's three high level steps to this process, as mentioned in the guidance from Microsoft if you click the Defender for DevOps blade in Defender for Cloud:

![](/img/DefenderForDevops/Connect.PNG)


# Installation

## Connect DevOps environments

The following features will be enabled (from [Microsoft](https://learn.microsoft.com/en-us/azure/defender-for-cloud/quickstart-onboard-devops)):

* Defender for Cloud's CSPM features - Assesses your Azure DevOps resources according to ADO-specific security recommendations. These recommendations are also included in your secure score. Resources will be assessed for compliance with built-in standards that are specific to DevOps. Defender for Cloud's asset inventory page is a multicloud enabled feature that helps you manage your Azure DevOps resources alongside your Azure resources.

* Microsoft Defender for DevOps - Extends Defender for Cloud's threat detection capabilities and advanced defenses to your Azure DevOps resources.

### Connect Azure DevOps to Defender for Cloud

1. Sign in to the Azure portal and Navigate to Microsoft Defender for Cloud > Environment Settings.
2. Select Add environment and Azure DevOps.

   ![](/img/DefenderForDevops/AddEnv.PNG)

3. Enter a name, select a subscription, resource group, and region (this is not very important for the connection, but where Azure will store information about the connection)
4. Select Next: Select plans (currently there's only a free preview plan).

   ![](/img/DefenderForDevops/Free.PNG)

5. Select Next: Authorize connection and Authorize.

   ![](/img/DefenderForDevops/Auth.PNG)

6. Click Accept.
7. Select your organization from the drop-down menu and select either auto-discover or a specific project.
8. Select Next: Review and create, then click create.

You can also view the full quickstart from Microsoft [here](https://learn.microsoft.com/en-us/azure/defender-for-cloud/quickstart-onboard-devops#connect-your-azure-devops-organization
) with some pictures.

## Azure DevOps

1. Inside your Azure DevOps organization, go to marketplace and manage extensions from the dropdown in the upper right.
2. Find and install the Microsoft Security DevOps extension for your organization.
3. (Optional) Find and install the SARIF SAST Scans Tab for your organization.
   * This allows you to view all outputs from the scan in a separate tab in the pipeline results.

### Azure DevOps pipelines

For general usage, you can plug the following task into your pipelines:

```yaml
- task: MicrosoftSecurityDevOps@1
```

There's also a bunch of flags in order to limit the amount of scans your run: 

```yaml
- task: MicrosoftSecurityDevOps@1
  inputs:
    categories: 'all'
    languages: 'all'
    tools: 'all'
    break: true
```

#### Categories

This value defaults to the value `all`, but can be set to the following values:

|Value|
|--|
|`secrets`|
| `code`|
| `artifacts` |
| `IaC`|
| `containers`|

#### Languages

This value also defaults to `all`, but you can set what language to look for. The task only gives two examples, `JavaScript` and `TypeScript`.
We can assume some other values from the tools under, like `Python` from the tool `Bandit`.

#### Tools

Again, defaults to `all`, but can be set to the following values:

|Value|More information|
|--|--|
|`bandit`|[Covers Python](https://github.com/PyCQA/bandit)
|`binskim`|[Covers binaries - Windows, ELF](https://github.com/Microsoft/binskim)
|`eslint`|[Covers JavaScript](https://github.com/eslint/eslint)
|`credscan`*|[Microsoft-tool to identity credential leaks](https://learn.microsoft.com/en-us/azure/defender-for-cloud/detect-credential-leaks)
|`template-analyzer`|[Covers ARM, Bicep](https://github.com/Azure/template-analyzer)
|`terrascan`|[Covers Terraform (HCL2), Kubernetes (JSON/YAML), Helm v3, Kustomize, Dockerfiles, Cloud Formation	](https://github.com/accurics/terrascan)
|`trivy`|[Covers container images, file systems, git repositories](https://github.com/aquasecurity/trivy)

**Credscan doesn't show up as a supported value in the description of the task, so might just be built in.*

#### Break

Stops the task if set to `true`, defaults to `false`.

# List of recommendations for Defender for DevOps (in the Defender for Cloud recommendations)

|Recommendation|	Description|	Severity|
|--|--|--|
|(Preview) Code repositories should have code scanning findings resolved|	Defender for DevOps has found vulnerabilities in code repositories. To improve the security posture of the repositories, it is highly recommended to remediate these vulnerabilities. (No related policy)	|Medium
|(Preview) Code repositories should have secret scanning findings resolved	|Defender for DevOps has found a secret in code repositories.  This should be remediated immediately to prevent a security breach.  Secrets found in repositories can be leaked or discovered by adversaries, leading to compromise of an application or service. For Azure DevOps, the Microsoft Security DevOps CredScan tool only scans builds on which it has been configured to run. Therefore, results may not reflect the complete status of secrets in your repositories. (No related policy)|	High
|(Preview) Code repositories should have Dependabot scanning findings resolved	|Defender for DevOps has found vulnerabilities in code repositories. To improve the security posture of the repositories, it is highly recommended to remediate these vulnerabilities. (No related policy)|	Medium
|(Preview) Code repositories should have infrastructure as code scanning findings resolved|	Defender for DevOps has found infrastructure as code security configuration issues in repositories. The issues shown below have been detected in template files. To improve the security posture of the related cloud resources, it is highly recommended to remediate these issues. (No related policy)|	Medium
|(Preview) GitHub repositories should have code scanning enabled	|GitHub uses code scanning to analyze code in order to find security vulnerabilities and errors in code. Code scanning can be used to find, triage, and prioritize fixes for existing problems in your code. Code scanning can also prevent developers from introducing new problems. Scans can be scheduled for specific days and times, or scans can be triggered when a specific event occurs in the repository, such as a push. If code scanning finds a potential vulnerability or error in code, GitHub displays an alert in the repository. A vulnerability is a problem in a project's code that could be exploited to damage the confidentiality, integrity, or availability of the project. (No related policy)|	Medium
|(Preview) GitHub repositories should have secret scanning enabled	|GitHub scans repositories for known types of secrets, to prevent fraudulent use of secrets that were accidentally committed to repositories. Secret scanning will scan the entire Git history on all branches present in the GitHub repository for any secrets. Examples of secrets are tokens and private keys that a service provider can issue for authentication. If a secret is checked into a repository, anyone who has read access to the repository can use the secret to access the external service with those privileges. Secrets should be stored in a dedicated, secure location outside the repository for the project. (No related policy)|	High
|(Preview) GitHub repositories should have Dependabot scanning enabled	|GitHub sends Dependabot alerts when it detects vulnerabilities in code dependencies that affect repositories. A vulnerability is a problem in a project's code that could be exploited to damage the confidentiality, integrity, or availability of the project or other projects that use its code. Vulnerabilities vary in type, severity, and method of attack. When code depends on a package that has a security vulnerability, this vulnerable dependency can cause a range of problems. (No related policy)	|Medium

From [DevOps recommendations.](https://learn.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference#devops-recommendations) Each tool mentioned above will also surface their own recommendations locally in Azure DevOps and in Defender for Cloud.