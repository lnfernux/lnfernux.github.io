---
layout: post
title: Templating Microsoft Sentinel Analytic Rules using Powershell and CI/CD pipelines
subtitle: Using the Microsoft Sentinel API and Powershell we can download all the components we want and template them for deployment - this allows you to create Analytic Rules in the Azure Portal and deploy them to multiple customers using CI/CD pipelines.
tags:
  - Microsoft Sentinel
  - Azure DevOps
  - Analytic Rules
  - Powershell
  - Microsoft Sentinel API
  - ARM-templates
published: true
author: author_infernux
image: /img/sentinel.png
---

# Templating

So the general idea is to provide a starting point for templating Microsoft Sentinel components, in this case analytic rules. An ARM-template generator [already exists for Playbooks](https://github.com/Azure/Azure-Sentinel/blob/master/Tools/Playbook-ARM-Template-Generator/src/Playbook_ARM_Template_Generator.ps1) and works great. However, the script is not adapted for pipelines and it's just above 600 lines of code. When using scripts in CI/CD pipelines I personally like to keep the length and complexity to a minimum.

**So what do we do?** 
* First of all - pull analytic rules down from our Microsoft Sentinel workspace using the API.
* Secondly - parse all the analytic rules to get rid of any unwanted data.

# Script

Let's write some Powershell-functions to download and parse our analytic rules into templates. For the time being my goal is to make the function be able to run locally on my PC, while we while adapt them to a pipeline further down in this post.

## Get-MicrosoftSentinelAnalyticRulesStandalone.ps1

**Requirements:**
Scripts were tested locally using Powershell Core 7.2.5 and the Azure module (`Install-Module Az`).

**It takes the following input:**
* `ResourceGroup` - where the Sentinel-workspace is located.
* `SubscriptionId` - where the resource group is located.
* `workspaceName` - name of the Sentinel-workspace.
* `outputPath` - filepath to write all the parsed json-files to.

We can run the script by dot sourcing it in Powershell:
```powershell
. ./Get-MicrosoftSentinelAnalyticRulesStandalone.ps1
Get-MicrosoftSentinelAnalyticRules -outputPath $outputPath -resourceGroup $resourceGroup -subscriptionId $subscriptionId -workspaceName $workstpaceName
```

The script itself is around 49 lines, so quite an improvement in terms of length (*if we were to do playbooks it would be a tad longer, because there are more fields to parse*):

```powershell
function Parse-MicrosoftSentinelAnalyticRules {
    PARAM(
      $DownloadedRules,
      $outputPath
    )
    try {
        foreach($DownloadedRule in $DownloadedRules.Value) {
            # Set name variable
            $Name = $DownloadedRule.Properties.DisplayName

            # Remove all spaces and colons from names
            $Name = $Name.Replace(" ", "")
            $Name = $Name.Replace(":", "")    
            $File = "$Name.json"
            if($Name) {
                Write-Host "Parsing rule: $($Name) and saving to $($Name).json"
                $DownloadedRule.PSObject.Properties.Remove("id")
                $DownloadedRule.PSObject.Properties.Remove("etag")
                $DownloadedRule.properties.PSObject.Properties.Remove("lastModifiedUtc")
                $DownloadedRule | ConvertTo-Json -Depth 15 | Out-File $outputPath/$File
            }
        } 
    } catch {
      Write-Host "An error occured in the MicrosoftSentinelAnalyticRules-function: $($_)"
    }
}
function Get-MicrosoftSentinelAnalyticRules {
    PARAM(
      $resourceGroup,
      $subscriptionId,
      $outputPath,
      $workspaceName
    )
    try {
        # First, craft the URI for downloading the analytic rules
        $uri = "/subscriptions/${subscriptionId}/resourceGroups/${resourceGroup}/providers/Microsoft.OperationalInsights/workspaces/${workspaceName}/providers/Microsoft.SecurityInsights/alertRules?api-version=2022-01-01-preview"
        
        # Download all analytic rules
        $DownloadedRules = (Invoke-AzRestMethod -Path $uri).Content | ConvertFrom-Json -Depth 15

        # Check that outputPath exists, if not create
        If(!(Test-Path $outputPath)) {
            New-Item -ItemType Directory -Path $outputPath
        }
        Parse-MicrosoftSentinelAnalyticRules -DownloadedRules $DownloadedRules -outputPath $outputPath

    } catch {
        Write-Host "An error occured in the MicrosoftSentinelAnalyticRules-function: $($_)"
    }
}
```

**The script generates the following output:**
* Multiple JSON-files containing the templated analytic rules.

We are now able to download analytic rules and parse them into templates - the only thing you'd require to deploy them to multiple customers is to add back customer specific values in the pipelines used to create analytic rules.

> You can download the [full standalone script from my Github.](https://github.com/infernuxmonster/MicrosoftSentinel-Scripts/blob/main/Get-MicrosoftSentinelAnalyticRulesStandalone.ps1)

# Pipeline

We will use Azure DevOps pipelines to accomplish our CI/CD goal - for more information on setting up your own ADO-organization (and other free Azure resources), scroll to the bottom of this post.

## Creating a pipeline

The pipeline will do the following:
* Run manually (define `trigger: none`)
* Download scripts from a [repository resource](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/resources?view=azure-devops&tabs=example#define-a-repositories-resource) using a service connection
* Run on a windows build agent
* Take inputs from a [variable group and variables defined in the pipeline](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=yaml)
   * `resourceGroup`, `workspaceName` and `subscriptionId` are all defined as `RG`, `WSN` and `SUBID` in the variable group `demo`
   * Rest of the variables are set in the `variables:` block
* Perform certain steps:
   * Checkout (download) the defined `Scripts` repository
   * Run an Azure Powershell task, which will run the `Get-MicrosoftSentinelAnalyticRulesPipeline.ps1` script using a service connection
   * Run the publish build artifact task, which is where we can download all the parsed analytic rules after running

**The demo variable group looks like this:**

![](/img/AzureDevops/VariableGroupDemo.PNG)

*The `subscriptionId` isn't a secret in my environment, just was easier to set it to secret for the screenshot. If you want to keep it as a secret, you should be able to add a Powershell task to add the secret variable to `env:` and it should persist through all the tasks, as long as you don't split them into stages:*

```json
steps:
- checkout: Scripts
- task: PowerShell@2
  inputs:
    targetType: 'inline'
    script: |
        Write-Host "Adding subscriptionId to env"
  env:
    SUBID: $(SUBID) 
```

**And if we go to our project settings and look at the service connections, they look like this:**

![](/img/AzureDevops/ServiceConnections.PNG)

The `azure_demo_tenant` is just a SPN (Azure AD app registration) which is given read-access to the subscription where my Sentinel-workspace is defined.

**This leaves us with the following pipeline:**

```yaml
trigger: none

resources:
  repositories:
  - repository: Scripts
    type: github
    name: infernuxmonster/MicrosoftSentinel-Scripts
    endpoint: infernuxmonster

pool:
  vmImage: 'windows-latest'

variables:
  - group: demo
  - name: Scripts
    value: '$(Build.Repository.LocalPath)/'
  - name: ServiceConnection
    value: 'azure_demo_tenant'
  - name: outputPath
    value: '$(Pipeline.Workspace)/OutFiles'

steps:
- checkout: Scripts
- task: AzurePowerShell@5
  displayName: 'Download and parse analytic rule'
  inputs:
    azureSubscription: $(ServiceConnection)
    ScriptPath: '$(Scripts)/Get-MicrosoftSentinelAnalyticRulesPipeline.ps1'
    ScriptArguments: '-outputPath $(outputPath)'
    azurePowerShellVersion: LatestVersion
    pwsh: true

- task: PublishBuildArtifacts@1
  displayName: 'Publish parsed analytic rules artifact'
  inputs: 
    PathtoPublish: $(outputPath)
    ArtifactName: ParsedRules
```

> The entire [Get-MicrosoftSentinelAnalyticRules.yaml file can be downloaded on my Github.](https://github.com/infernuxmonster/MicrosoftSentinel-Templates/blob/main/Pipelines/Get-MicrosoftSentinelAnalyticRules.yaml)

## Preparing the script for running in an Azure DevOps pipeline

In order to make sure the functions work in the pipeline we'll make some adaptions:

* All parameter values should be defined either in a variable group, in parameters set when running the pipeline or in static variables in the pipelines themselves. 

Either which way, this means we need to modify our scripts to use the values defined in the pipeline. For Azure DevOps pipeline agents this is defined in the environment variables, which means we can reference them in Powershell using `$env:VariableName` in our scripts: 

```powershell
PARAM(
    $outputPath
)
function Parse-MicrosoftSentinelAnalyticRules {
  ...
}
function Get-MicrosoftSentinelAnalyticRules {
  ...
}
try {
    # We define variables either in variable groups, in keyvaults or in the pipeline itself depending on the sensitivity of the variable
    # All pipeline variables are stored in env: 
    $resourceGroup = $env:RG
    $subscriptionId = $env:SUBID
    $workspaceName = $env:WSN
    Write-Host "resourceGroup: $($resourceGroup)"
    Write-Host "subscriptionId: $($subscriptionId)"
    Write-Host "workspaceName: $($workspaceName)"

    Get-MicrosoftSentinelAnalyticRules -outputPath $outputPath -resourceGroup $resourceGroup -subscriptionId $subscriptionId -workspaceName $workspaceName
    Write-Host "##vso[task.complete result=Succeeded;]DONE"
} catch {
    Write-Host "##vso[task.LogIssue type=error;]$_"
}
```

We also added some [logging commands](https://docs.microsoft.com/en-us/azure/devops/pipelines/scripts/logging-commands?view=azure-devops&tabs=powershell) in order to raise an alert in the pipeline-run if the script errors out.

> You can download the [full script, modified for usage in Azure DevOps pipelines on my Github.](https://github.com/infernuxmonster/MicrosoftSentinel-Scripts/blob/main/Get-MicrosoftSentinelAnalyticRulesPipeline.ps1)

## Running the pipeline

For testing, I've created an Analytic Rule for testing:

![](/img/AzureDevops/TestRule.PNG)

We can now run the pipeline:

![](/img/AzureDevops/PipelineRun.PNG)

If everything is set up correctly, we should be looking at a finished job:

![](/img/AzureDevops/SuccessfulRun.PNG)

The output tells us we've found two rules, the default fusion rule and my test rule:

![](/img/AzureDevops/Output.PNG)

If we look at the artifact step we can see that it has also published both files as an artifact:

![](/img/AzureDevops/UploadArtifact.PNG)

If we go to the pipeline overview we can see that one artifact has been published:

![](/img/AzureDevops/ArtifactPipelineOutput.PNG)

And now we can download our artifacts:

![](/img/AzureDevops/Artifacts.PNG)

End result is a json file that looks like this:

```json
{
  "name": "4f798eac-ac88-4dd9-a8f9-8f069fea566d",
  "type": "Microsoft.SecurityInsights/alertRules",
  "kind": "Scheduled",
  "properties": {
    "queryFrequency": "PT5H",
    "queryPeriod": "PT5H",
    "triggerOperator": "GreaterThan",
    "triggerThreshold": 0,
    "eventGroupingSettings": {
      "aggregationKind": "SingleAlert"
    },
    "severity": "Medium",
    "query": "AzureActivity\r\n| take 1",
    "suppressionDuration": "PT5H",
    "suppressionEnabled": false,
    "incidentConfiguration": {
      "createIncident": true,
      "groupingConfiguration": {
        "enabled": false,
        "reopenClosedIncident": false,
        "lookbackDuration": "PT5H",
        "matchingMethod": "AllEntities",
        "groupByEntities": [],
        "groupByAlertDetails": [],
        "groupByCustomDetails": []
      }
    },
    "tactics": [],
    "techniques": [],
    "displayName": "infernux-testrule-dev",
    "enabled": true,
    "description": "",
    "alertRuleTemplateName": null
  }
}
```

# Conclusion

We can now download and template analytic rules standalone or in a CI/CD pipeline. You can also modify the script to push the templates directly into a repository, and use a similar setup to [push analytic rules to Sentinel.](https://github.com/javiersoriano/sentinelascode)

It was tested in a free Azure DevOps organization (using Github to sign in). Connections to Azure and Github (repository for hosting our pipeline and scripts) connected via [service connection.](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml)

**If you wanted to try this out yourself:**
* [Free Azure DevOps organization](https://azure.microsoft.com/en-us/products/devops/)
* [1800 minutes of free Azure DevOps pipeline runtime](https://aka.ms/azpipelines-parallelism-request)
* [Azure 30 day trial](https://azure.microsoft.com/en-gb/free/)
   * [Azure for students](https://azure.microsoft.com/en-gb/free/students/)
* [Enterprise Mobility + Security E5 Trial](https://signup.microsoft.com/get-started/signup?OfferId=87dd2714-d452-48a0-a809-d2f58c4f68b7&ali=1&products=87dd2714-d452-48a0-a809-d2f58c4f68b7)