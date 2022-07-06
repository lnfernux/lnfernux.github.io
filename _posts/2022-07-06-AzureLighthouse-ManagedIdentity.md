---
layout: post
title: Assign roles managed identities in Microsoft Sentinel playbooks using Azure Lighthouse
subtitle: Grant access via Azure Lighthouse using User Access Administrator delegation, ARM-templates, pipelines and powershell. 
tags:
  - Azure REST API
  - Microsoft Sentinel
  - Azure Lighthouse
  - ARM Template
  - PowerShell
  - Managed Identity
  - User Access Administrator
published: true
author: author_infernux
image: /img/ps.png
---

# Scenario

- We're creating playbooks in a managed Microsoft Sentinel-workspace.
- It needs to have the `Microsoft Sentinel Responder` role.
- We want to be able to assign that role using access granted by Azure Lighthouse.
- We should do as much of this as possible in a Azure DevOps pipeline.

# Setup

1. Create the Lighthouse-configuration and onboard it.
2. Deploy playbooks with managed identities enabled.
3. Create ARM-template for assigning rights (or use Azure CLI, API).
4. Deploy ARM-template.

## 1. Lighthouse-configuration
Create a Lighthouse-configuration and grant the User Access Administrator to a principal (User, Group, SPN):
```json
"authorizations": {
	        "type": "array",
            "metadata": {
                "description": "Specify an array of objects, containing tuples of Azure Active Directory principalId, a Azure roleDefinitionId, and an optional principalIdDisplayName. The roleDefinition specified is granted to the principalId in the provider's Active Directory and the principalIdDisplayName is visible to customers."
            },
            "defaultValue": [
                {
                    //Grants user access administrator to the SPN
                    "principalId": "3kl47fff-1337-1337-b726-2cf02b05c7c4",
                    "principalIdDisplayName": "Test-SPN",
                    "roleDefinitionId": "18d7d88d-d35e-4fb5-a5c3-7773c20a72d9",
                    //Allows the SPN to assign the following roles
                    "delegatedRoleDefinitionIds": [
                        //Microsoft Sentinel Responder
                        "3e150937-b8fe-4cfb-8069-0eaf05ecd056"
                    ]
                }
            ]
}
```
The same principal also needs to have write-permission on the object the managed identity resides within - for a playbook with a managed identity we should be fine with Microsoft Sentinel Contributor.
```json
{
    //Grants contributor to the SPN
    "principalId": "3kl47fff-5655-4779-b726-2cf02b05c7c4",
    "principalIdDisplayName": "Test-SPN",
    "roleDefinitionId": "ab8e14d6-4a74-4a29-9ba8-549422addade"
}
```
At this point we can onboard the template. 

> *Note: to read more about onboarding using Azure Lighthouse, click [here.](https://docs.microsoft.com/en-us/azure/lighthouse/how-to/onboard-customer)*

## 2. Deploy playbooks with managed identities enabled
Using the Sentinel-as-Code project (https://github.com/javiersoriano/sentinelascode) as inspiration, we can deploy playbooks with a simple push-pipeline:

Assuming a simple Azure DevOps setup similar to the one mentioned above with the SPN from the above Lighthouse-configuration defined as the `service-connection` or the subscription-object in the Azure Powershell-task:

### Example pipeline

Simple pipeline that downloads the local repository and runs the `CreatePlaybooks.ps1`-script with inputs.

```yaml
name: build and deploy Playbooks

trigger:
 paths:
   include:
     - Playbooks/*

stages:
- stage: deploy_playbooks
  jobs:
    - job: AgentJob
      pool:
       name: Azure Pipelines
       vmImage: 'windows-latest'
      variables: 
      - group: DeployGroup
      - name: RepositoryLocation
        value: '$(Build.Repository.LocalPath)/SentinelAsCode'
      steps:
      - checkout: self
      - task: AzurePowerShell@5
        displayName: 'Create and Update Playbooks'
        inputs:
         azureSubscription: $(subscription)
         ScriptPath: '$(RepositoryLocation)/Scripts/CreatePlaybooks.ps1'
         ScriptArguments: '-resourceGroup $(ResourceGroup) -PlaybooksFolder $(RepositoryLocation)/Playbooks'
         azurePowerShellVersion: LatestVersion
         pwsh: true
```

### Example deployment script 

Script that takes a path as input and loops through all files and tries to deploy `.json` files to Azure as ARM-templates.

```powershell
param(
    [Parameter(Mandatory=$true)]$resourceGroup,
    [Parameter(Mandatory=$true)]$PlaybooksFolder
)

Write-Host "Folder is: $($PlaybooksFolder)"

$armTemplateFiles = Get-ChildItem -Path $PlaybooksFolder -Filter *.json

Write-Host "Files are: " $armTemplateFiles

foreach ($armTemplate in $armTemplateFiles) {
    try {
        New-AzResourceGroupDeployment -ResourceGroupName $resourceGroup -TemplateFile $armTemplate 
    }
    catch {
        $ErrorMessage = $_.Exception.Message
        Write-Error "Playbook deployment failed with message: $ErrorMessage" 
    }
}
```
> *Note: script courtesy of Javier Soriano (https://github.com/javiersoriano/sentinelascode/blob/master/Scripts/CreatePlaybooks.ps1)*

### Example playbook template

Sample playbook for changing severity:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "PlaybookName": {
            "defaultValue": "Change-Incident-Severity",
            "type": "string"
        }
    },
    "variables": {
        "AzureSentinelConnectionName": "[concat('azuresentinel-', parameters('PlaybookName'))]"
    },
    "resources": [
        {
            "type": "Microsoft.Web/connections",
            "apiVersion": "2016-06-01",
            "name": "[variables('AzureSentinelConnectionName')]",
            "location": "[resourceGroup().location]",
            "kind": "V1",
            "properties": {
                "displayName": "[parameters('PlaybookName')]",
                "customParameterValues": {},
                "parameterValueType": "Alternative",
                "api": {
                    "id": "[concat('/subscriptions/', subscription().subscriptionId, '/providers/Microsoft.Web/locations/', resourceGroup().location, '/managedApis/azuresentinel')]"
                }
            }
        },
        {
            "type": "Microsoft.Logic/workflows",
            "apiVersion": "2017-07-01",
            "name": "[parameters('PlaybookName')]",
            "location": "[resourceGroup().location]",
            "tags": {
                "LogicAppsCategory": "security"
            },
            "identity": {
                "type": "SystemAssigned"
            },
            "dependsOn": [
                "[resourceId('Microsoft.Web/connections', variables('AzureSentinelConnectionName'))]"
            ],
            "properties": {
                "state": "Enabled",
                "definition": {
                    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
                    "actions": {
                        "Entities_-_Get_Accounts": {
                            "inputs": {
                                "body": "@triggerBody()?['object']?['properties']?['relatedEntities']",
                                "host": {
                                    "connection": {
                                        "name": "@parameters('$connections')['azuresentinel']['connectionId']"
                                    }
                                },
                                "method": "post",
                                "path": "/entities/account"
                            },
                            "runAfter": {},
                            "type": "ApiConnection"
                        },
                        "For_each": {
                            "actions": {
                                "Condition": {
                                    "actions": {
                                        "Update_incident": {
                                            "inputs": {
                                                "body": {
                                                    "incidentArmId": "@triggerBody()?['object']?['id']",
                                                    "severity": "High"
                                                },
                                                "host": {
                                                    "connection": {
                                                        "name": "@parameters('$connections')['azuresentinel']['connectionId']"
                                                    }
                                                },
                                                "method": "put",
                                                "path": "/Incidents"
                                            },
                                            "runAfter": {},
                                            "type": "ApiConnection"
                                        }
                                    },
                                    "expression": {
                                        "or": [
                                            {
                                                "contains": [
                                                    "@items('For_each')?['Name']",
                                                    "admin"
                                                ]
                                            }
                                        ]
                                    },
                                    "runAfter": {},
                                    "type": "If"
                                }
                            },
                            "foreach": "@body('Entities_-_Get_Accounts')?['Accounts']",
                            "runAfter": {
                                "Entities_-_Get_Accounts": [
                                    "Succeeded"
                                ]
                            },
                            "type": "Foreach"
                        }
                    },
                    "contentVersion": "1.0.0.0",
                    "outputs": {},
                    "parameters": {
                        "$connections": {
                            "defaultValue": {},
                            "type": "Object"
                        }
                    },
                    "triggers": {
                        "Microsoft_Sentinel_incident": {
                            "inputs": {
                                "body": {
                                    "callback_url": "@{listCallbackUrl()}"
                                },
                                "host": {
                                    "connection": {
                                        "name": "@parameters('$connections')['azuresentinel']['connectionId']"
                                    }
                                },
                                "path": "/incident-creation"
                            },
                            "type": "ApiConnectionWebhook"
                        }
                    }
                },
                "parameters": {
                    "$connections": {
                        "value": {}
                        }
                }
            }
        }
    ]
}
```

In order to enable managed identites the following block must be configured under the `"type": "Microsoft.Logic/workflows"`-resource:

```json
"identity": {
  "type": "SystemAssigned"
},
```

> *Note: template courtesy of Yaniv Shasha (https://github.com/Azure/Azure-Sentinel/tree/master/Playbooks/Change-Incident-Severity)*

## 3. Create the ARM-template

In order to assign roles using the SPN mentioned above we need to create an ARM-template for the roleAssignment:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2018-05-01/subscriptionDeploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        //Generates a new random guid to be used as assignmnent ID
        "rbacGuid": {
            "type": "string",
            "defaultValue": "[newGuid()]"
        } 
    },
    "variables": {
        //The GUID refering to the Microsoft Sentinel Responder
        "responder": "3e150937-b8fe-4cfb-8069-0eaf05ecd056"
    },
    "resources": [
        {
            "type": "Microsoft.Authorization/roleAssignments",
            "apiVersion": "2020-03-01-preview",
            "name": "[parameters('rbacGuid')]",
            "dependsOn": [
            ],
            "properties": {
                "roleDefinitionId": "[concat(subscription().id, '/resourceGroups/###resourceGroup###/providers/Microsoft.Authorization/roleDefinitions/', variables('responder'))]",
                // The principalType property will tell Microsoft.Authorization not to perform the check for existence on your principal ID during roleAssignment creation
                "principalType": "ServicePrincipal",
                "delegatedManagedIdentityResourceId": "###delegatedresourceid###",
                "principalId": "###principalId###"
            }
        }
    ]
}
```
Save as a .json file.

## 4. Deploy ARM-template

### Create a script for parsing the roleAssignment-template

You grab the content of the template in Powershell:

```powershell
$armTemplate = Get-Content "$templatePath/API-Connectors/ManagedIdentityAccess.json" 
```

You then grab the resourceId of the playbook(s) you're trying to change - just change the matching of the name to grab more of them:

```powershell
$Playbook = Get-AzResource -ResourceType Microsoft.Logic/workflows | Where-Object {$_.Name -match "PlaybookName-*"}
```

We can then do some simple parsing of the template, grab the principalId of the managedIdentity and insert:

```
$ID = $Playbook.Identity.PrincipalId
$parsedTemplate = $armTemplate
$parsedTemplate = $parsedTemplate -replace ("###resourceGroup###",$resourceGroup)
$parsedTemplate = $parsedTemplate -replace ("###principalId###",$ID)
$parsedTemplate = $parsedTemplate -replace ("###delegatedresourceid###",$Playbook.ResourceId)
$parsedTemplate | Set-Content "RoleAssignment.json"
```

After this we have a ready to deploy template, which can be pushed using the following line of code:
```powershell
New-AzResourceGroupDeployment -ResourceGroupName $resourcegroup -Name GrantRightsSPN -TemplateFile "RoleAssignment.json"
```

This all results in the following script:

#### Assign-Roles.ps1
```powershell
PARAM (
  $templatePath,
  $resourceGroup
)
$armTemplate = Get-Content $templatePath
$Playbook = Get-AzResource -ResourceType Microsoft.Logic/workflows | Where-Object {$_.Name -match "PlaybookName-*"}
foreach($p in $playbook) {
  $ID = $p.Identity.PrincipalId
  $parsedTemplate = $armTemplate
  $parsedTemplate = $parsedTemplate -replace ("###resourceGroup###",$resourceGroup)
  $parsedTemplate = $parsedTemplate -replace ("###principalId###",$ID)
  $parsedTemplate = $parsedTemplate -replace ("###delegatedresourceid###",$p.ResourceId)
  $parsedTemplate | Set-Content "RoleAssignment.json"
  New-AzResourceGroupDeployment -ResourceGroupName $resourcegroup -Name GrantRightsSPN -TemplateFile "RoleAssignment.json"
}
```

### Deploy the roleAssignment-template using a pipeline

Using the script from above, we can create a simple pipeline similar to the one we used for playbooks:

```yaml
name: assignRoles using Lighthouse

trigger:
 paths:
   include:
     - Playbooks/*

stages:
- stage: assign_roles
  jobs:
    - job: AgentJob
      pool:
       name: Azure Pipelines
       vmImage: 'windows-latest'
      variables: 
      - group: DeployGroup
      - name: RepositoryLocation
        value: '$(Build.Repository.LocalPath)/SentinelAsCode'
      steps:
      - checkout: self
      - task: AzurePowerShell@5
        displayName: 'Assign Roles'
        inputs:
         azureSubscription: $(subscription)
         ScriptPath: '$(RepositoryLocation)/Scripts/Assign-Roles.ps1'
         ScriptArguments: '-templatePath "$(RepositoryLocation)/API-Connectors/ManagedIdentityAccess.json" -resouceGroup $(resourceGroup)'
         azurePowerShellVersion: LatestVersion
         pwsh: true
```

This should assign the Sentinel Responder-role to all managed identities for all playbooks in the filter `Where-Object {$_.Name -match "PlaybookName-*"}` - you can also set the assignRoles-pipeline to trigger on finished runs by the Create-Playbook pipeline by adding it as a resource:
```yaml
resources:
  pipelines:
  - pipeline: Create Playbooks # Name of the pipeline resource.
    source: create-playbooks # The name of the pipeline referenced by this pipeline resource.
    #project: SentinelAsCode # Required only if the source pipeline is in another project
    trigger: true # Run pipeline when any run of assignRoles completes
```


# Sources
- [Pipeline triggers in Azure DevOps.](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/pipeline-triggers?view=azure-devops#configure-pipeline-resource-triggers)
- [Sentinel as Code Github-project.](https://github.com/javiersoriano/sentinelascode)
- [Deploy a policy that can be remediated within a delegated subscription.](https://docs.microsoft.com/en-us/azure/lighthouse/how-to/deploy-policy-remediation#create-a-user-who-can-assign-roles-to-a-managed-identity-in-the-customer-tenant)
- [Sample Lighthouse-template for policy remediation.](https://github.com/Azure/Azure-Lighthouse-samples/blob/master/templates/policy-add-or-replace-tag/addOrReplaceTag.json)
- [Deploy resources with ARM templates and Azure PowerShell.](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-powershell#deployment-scope)
- [Managed identies for Azure resources FAQ.](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identities-faq)
