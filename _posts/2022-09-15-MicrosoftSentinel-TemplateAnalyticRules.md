---
layout: post
title: Templating Microsoft Sentinel Analytic Rules using Powershell
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

So the general idea is to provide a starting point for templating Microsoft Sentinel components, in this case analytic rules. An ARM-template generator [already exists for Playbooks](https://github.com/Azure/Azure-Sentinel/blob/master/Tools/Playbook-ARM-Template-Generator/src/Playbook_ARM_Template_Generator.ps1) and works great, but it's just above 600 lines of code. If we plan to use the scripts in our CI/CD pipelines it should be kept to it's bare minimum in terms of length and complexity.

**So what do we do?** 
* First of all, we pull analytic rules down from our Microsoft Sentinel workspace using the API.
* Secondly, we'll parse all the analytic rules to get rid of any unwanted data.

# Functions

Let's write some Powershell-functions to download and parse our analytic rules into templates. For the time being my goal is to make the function be able to run locally on my PC, while we while adapt them to a pipeline further down in this post.

## Get-MicrosoftSentinelAnalyticRules

Short and sweet function that requires the `Azure Powershell`-module. 

**It takes the following input:**
* `ResourceGroup` - where the Sentinel-workspace is located.
* `SubscriptionId` - where the resource group is located.
* `Prefix` / `Suffix` - allows you to use prefix or suffixes in your naming standards to decide what analytic rules to include in your automation. This is optional, without this the script will just download everything.

```powershell
function Get-MicrosoftSentinelAnalyticRules {
  PARAM(
    $resourceGroup,
    $subscriptionId,
    $prefix,
    $suffix
  )
  try {
    if($suffix) {

    } elseif($prefix) {

    } else {

    }
  } catch {
    Write-Host "An error occured in the MicrosoftSentinelAnalyticRules-function: $($_)"
  }

}
```

**The script generates the following output:**
* Multiple JSON-files containing the analytic rules.

## Parse-MicrosoftSentinelAnalyticRules

This function takes the following input:
* `FilePath` - the path where the output from the `Get-MicrosoftSentinelAnalyticRules`-function was saved.

```powershell
function Parse-MicrosoftSentinelAnalyticRules {
  PARAM(
    $FilePath
  )
  try {

  } catch {
    Write-Host "An error occured in the MicrosoftSentinelAnalyticRules-function: $($_)"
  }
}
```

**The script generates the following output:**
* Multiple JSON-files containing the templated analytic rules.

We are now able to download analytic rules and parse them into templates - the only thing you'd require to deploy them to multiple customers is to add back customer specific values in the pipelines used to create analytic rules.

You can download the full standalone script from my Github.

# Pipeline

## Preparing the functions for running in an Azure DevOps pipeline

In order to make sure the functions work in the pipeline we'll make some adaptions:

1. All parameter values should be defined either in a variable group, in parameters set when running the pipeline or in static variables in the pipelines themselves. 

Either which way, this means we need to modify our scripts to use the values defined in the pipeline. For Azure DevOps pipeline agents this is defined in the environment variables: 

```powershell
123
```

2. All files need to be accessible for the entire pipeline-run.

This either means that the pipeline must run in just one stage (multiple stages are split across multiple agents in Azure DevOps) or we must publish an artifact after the stage downloading the files has completed that can be consumed by the second stage that will parse the files:

```powershell
456
```

You can download the full script, modified for usage in Azure DevOps pipelines on my Github.

## Creating a pipeline

Creating a pipeline will be 

```yaml
blah
```

The entire pipeline.yaml file can be downloaded on my Github.
