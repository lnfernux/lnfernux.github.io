---
layout: post
title: Create Managed Identitiy and assign roles using Azure Lighthouse
subtitle: Create Managed Identites and grant access via Azure Lighthouse using User Access Administrator delegation. 
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

- We're creating a user managed identity in a managed tenant.
- It needs to have the `Microsoft Sentinel Contributor` role.
- We want to be able to assign that role using access granted by Azure Lighthouse.

# Setup

1. Create the Lighthouse-configuration and onboard it.
2. Create the Managed Identity.
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
                    "principalId": "3kl47fff-1337-4779-b726-2cf02b05c7c4",
                    "principalIdDisplayName": "TEST",
                    "roleDefinitionId": "18d7d88d-d35e-4fb5-a5c3-7773c20a72d9",
                    //Allows the SPN to assign the following roles
                    "delegatedRoleDefinitionIds": [
                        //Microsoft Sentinel Contributor
                        "ab8e14d6-4a74-4a29-9ba8-549422addade"
                    ]
                }
            ]
}
```
The same principal also needs to have write-permission on the object the managed identity resides within - for a user assigned managed identity in a resource group, this means contributor access on the resource group.
```json
{
    //Grants contributor to the SPN
    "principalId": "3kl47fff-5655-4779-b726-2cf02b05c7c4",
    "principalIdDisplayName": "TEST",
    "roleDefinitionId": "b24988ac-6180-42a0-ab88-20f7382dd24c"
}
```
Onboard the template.

## 2. Create a new user managed identity
Login to Azure (make sure to activate all roles first if using PIM) and set the correct context - the subscriptionId you're managing using Lighthouse.
```powershell
Connect-AzAccount
Set-AzContext -SubscriptionId $subscriptionId
```
Create the managed identity.
```powershell
New-AzUserAssignedIdentity -ResourceGroupName $resourcegroup -Name $managedIdentityName -Location $location
```
Get the principalId of the managed identity
```powershell
$principalId = (Get-AzUserAssignedIdentity -Name $managedIdentityName -ResourceGroupName $resourcegroup).PrincipalId
```

## 3. Create the ARM-template

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
        //The GUID refering to the Microsoft Sentinel Contributor
        "sentinelContributor": "ab8e14d6-4a74-4a29-9ba8-549422addade"
    },
    "resources": [
        {
            "type": "Microsoft.Authorization/roleAssignments",
            "apiVersion": "2020-03-01-preview",
            "name": "[parameters('rbacGuid')]",
            "dependsOn": [
            ],
            "properties": {
                "roleDefinitionId": "[concat(subscription().id, '/resourceGroups/<resourceGroup>/providers/Microsoft.Authorization/roleDefinitions/', variables('sentinelContributor'))]",
                // The principalType property will tell Microsoft.Authorization not to perform the check for existence on your principal ID during roleAssignment creation
                "principalType": "ServicePrincipal",
                "delegatedManagedIdentityResourceId": "/subscriptions/<subscriptionId>/resourceGroups/<resourceGroup>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<managedIdentityName>",
                "principalId": "<principalId>"
            }
        }
    ]
}
```
Save as a .json file.

## 4. Deploy ARM-template

In the same PowerShell-session you logged into earlier, deploy the ARM-template created in step 3.
```powershell
New-AzResourceGroupDeployment -ResourceGroupName $resourcegroup -Name GrantRightsSPN -TemplateFile "RoleAssignment.json"
```

This should result in the following output:
```
DeploymentName          : GrantRightsSPN
ResourceGroupName       : <resourceGroup>
ProvisioningState       : Succeeded
Timestamp               : 6/8/2022 8:20:02 AM
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name             Type                       Value
                          ===============  =========================  ==========
                          rbacGuid         String                     "982c2f5d-1ee7-1337-l33t-f4b0ubd88bd55"

Outputs                 :
DeploymentDebugLogLevel :
```

## Other deployment methods

### REST API using Invoke-AzRestMethod

You should be able to use the https://management.azure.com/ endpoint to create the assignment:
```powershell
$guid = ([guid]::NewGuid()).Guid
$baseUri = "https://management.azure.com/"
$uri =  $baseUri + "subscriptions/${subscriptionId}/resourceGroups/${ResourceGroup}/providers/Microsoft.Authorization/roleAssignments/${guid}?api-version=2020-04-01-preview"
$Params = @"
properties = {
    "roleDefinitionId" : "/subscriptions/${subscriptionId}/resourceGroups/${ResourceGroup}/providers/Microsoft.Authorization/roleDefinitions/${roleDefinitionId}",
    "principalId" : "${principalId}",
    "delegatedManagedIdentityResourceId": "/subscriptions/${subscriptionId}/resourceGroups/${ResourceGroup}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${managedIdentityName}",
    "principalType" : "ServicePrincipal"
}
"@

Invoke-AzRestMethod -Method PUT -Uri $Uri -Payload $Params
```

# Sources
- [Deploy a policy that can be remediated within a delegated subscription.](https://docs.microsoft.com/en-us/azure/lighthouse/how-to/deploy-policy-remediation#create-a-user-who-can-assign-roles-to-a-managed-identity-in-the-customer-tenant)
- [Sample Lighthouse-template for policy remediation.](https://github.com/Azure/Azure-Lighthouse-samples/blob/master/templates/policy-add-or-replace-tag/addOrReplaceTag.json)
- [Deploy resources with ARM templates and Azure PowerShell.](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-powershell#deployment-scope)
- [Managed identies for Azure resources FAQ.](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/managed-identities-faq)
