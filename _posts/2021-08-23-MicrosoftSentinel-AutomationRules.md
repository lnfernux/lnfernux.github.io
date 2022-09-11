---
layout: post
title: Deploying Automation Rules via API
subtitle: Automate more of your Azure Sentinel deployment by combining the Az Powershell-module and the 2019-01-01-preview API to deploy Automation Rules from JSON-templates. 
tags:
  - Azure REST API
  - Azure Sentinel
  - SecurityInsights
  - Automation Rules
  - PowerShell
published: true
author: author_infernux
image: /img/sentinel.png
---

# Introduction to Automation Rules

Automation Rules is a resource in Azure, mainly used to automatically manage incidents and trigger playbooks in Azure Sentinel. 
All examples in this post are based purely on Azure Sentinel use-cases.

# Calling the API

We can call the API using this path:

```
https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{operationalInsightsResourceProvider}/workspaces/{workspaceName}/providers/Microsoft.SecurityInsights/automationRules
```

Using this API we can perform the following actions:

- Get all automation rules
- Get specific automation rule
- Create automation rules
- Delete automation rules

Let's get into some examples.

## Scenario 1: Get all automation rules

We need the following info:

- subscriptionId
- resourceGroupName
- workspaceName

Since we are working with Azure Sentinel the `{operationalInsightsResourceProvider}` variable is `Microsoft.OperationalInsights`.
Please keep in mind you need to have the Az-module (`Install-Module -Name Az -Scope CurrentUser -Repository PSGallery -Force`) and be signed in (`Connect-AzAccount`) for this snippet to work.

### Scenario 1 Script

```powershell
$baseUri = "/subscriptions/$subId/resourceGroups/{$rg}/providers/Microsoft.OperationalInsights/workspaces/{$law}"
$subId = Read-Host "Enter subscriptionId"
$rg = Read-Host "Enter resourceGroupName"
$law = Read-Host "Enter workspaceName"
$uri = "$baseUri/providers/Microsoft.SecurityInsights/automationRules?api-version=2019-01-01-preview"
$result = (Invoke-AzRestMethod -path $uri -Method GET).Content | ConvertFrom-Json
```

The `$result` variable should hold the returned Automation Rules. These can also be the basis for a JSON-template.

## Scenario 2: Create automation rule from JSON-template

We will need the information from the first example, as well as a Automation Rule-template in JSON-format. 
The best way to get this is creating an Automation Rule using the Azure Sentinel GUI, then downloading that using the snippet from scenario 1.
For this example we will just steal the example code from the preview-repo linked below in sources.

### Scenario 2 Script

```powershell
$baseUri = "/subscriptions/$subId/resourceGroups/{$rg}/providers/Microsoft.OperationalInsights/workspaces/{$law}"
$subId = Read-Host "Enter subscriptionId"
$rg = Read-Host "Enter resourceGroupName"
$law = Read-Host "Enter workspaceName"
$jsonFilePath = Read-Host "Full file-path to JSON-template"
$rawData = Get-Content $jsonFilePath -Raw
$ruleId = (New-Guid).Guid #creates a new random GUID, every automation rule needs an unique guid
$uri = "$baseUri/providers/Microsoft.SecurityInsights/automationRules/{$ruleId}?api-version=2019-01-01-preview"
$result = (Invoke-AzRestMethod -Path $uri -Method PUT -Payload $rawData).Content | ConvertFrom-Json
```

The `$result` variable should hold the returned information. If you want to avoid creating random GUIDs you can also include an `"id": "73e01a99-5cd7-4139-a149-9f2736ff2ab5"` or `"automationRuleId": "73e01a99-5cd7-4139-a149-9f2736ff2ab5"` at the top level in the json-template below.

### Scenario 2 JSON-template

Everything in the JSON-template pertains to a configurable element in the Automation Rule GUI. To read more about Automation Rules I'd suggest starting [here.](https://techcommunity.microsoft.com/t5/azure-sentinel/what-s-new-automation-rules/ba-p/2216926)

```json
{
  "etag": "\"0300bf09-0000-0000-0000-5c37296e0000\"",
  "properties": {
    "order": 1,
    "displayName": "High severity incidents escalation",
    "triggeringLogic": {
      "isEnabled": true,
      "triggersOn": "Incidents",
      "triggersWhen": "Created",
      "conditions": [
        {
          "conditionType": "Property",
          "conditionProperties": {
            "propertyName": "IncidentRelatedAnalyticRuleIds",
            "operator": "Contains",
            "propertyValues": [
              "/subscriptions/d0cfe6b2-9ac0-4464-9919-dccaee2e48c0/resourceGroups/myRg/providers/Microsoft.OperationalInsights/workspaces/myWorkspace/providers/Microsoft.SecurityInsights/alertRules/fab3d2d4-747f-46a7-8ef0-9c0be8112bf7",
              "/subscriptions/d0cfe6b2-9ac0-4464-9919-dccaee2e48c0/resourceGroups/myRg/providers/Microsoft.OperationalInsights/workspaces/myWorkspace/providers/Microsoft.SecurityInsights/alertRules/8deb8303-e94d-46ff-96e0-5fd94b33df1a"
            ]
          }
        }
      ]
    },
    "actions": [
      {
        "order": 1,
        "actionType": "ModifyProperties",
        "actionConfiguration": {
          "severity": "High"
        }
      },
      {
        "order": 2,
        "actionType": "RunPlaybook",
        "actionConfiguration": {
          "tenantId": "ee48efaf-50c6-411b-9345-b2bdc3eb4abc",
          "logicAppResourceId": "/subscriptions/d0cfe6b2-9ac0-4464-9919-dccaee2e48c0/resourceGroups/myRg/providers/Microsoft.Logic/workflows/IncidentPlaybook"
        }
      }
    ]
  }
}
```

# Sources

* [The 2019-01-01-preview API github](https://github.com/Azure/azure-rest-api-specs/blob/master/specification/securityinsights/resource-manager/Microsoft.SecurityInsights/preview/2019-01-01-preview/AutomationRules.json)
* [Automation Rule example-calls](https://github.com/Azure/azure-rest-api-specs/tree/master/specification/securityinsights/resource-manager/Microsoft.SecurityInsights/preview/2019-01-01-preview/examples/automationRules)
