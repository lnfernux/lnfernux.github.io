---
layout: post
title: Adding a Key Vault to your Microsoft Sentinel Data Connector ARM-template
subtitle: A subset of Data Connector for Sentinel come in the form of Azure Functions deployed using an ARM-template. Most if not all of these functions avoid actually implementing a Key Vault to secure your variables, so here's the snippets to implement it yourself.
tags:
  - Microsoft Sentinel
  - ARM-templates
  - Azure Functions
  - Data connectors
  - Key vault
published: true
author: author_infernux
image: /img/sentinel.png
---

## Introduction

Let's take the [Qualys Vulnerability Management](https://github.com/Azure/Azure-Sentinel/tree/master/Solutions/QualysVM) data connector as an example. 

![](/img/MicrosoftSentinel/QualysVMprereq.PNG)

Reviewing the [template](https://raw.githubusercontent.com/Azure/Azure-Sentinel/master/Solutions/QualysVM/Data%20Connectors/azuredeploy_QualysVM_API_FunctionApp_V2.json) we find that it doesn't set up a key vault - it just stores variables, such as the `workspaceKey`, `apiUserName` and `apiPassword` in the configuration of the Azure Function.

![](/img/MicrosoftSentinel/QualysVMOptionalStep.PNG)

Adding a key vault is set up as an optional step, one that perhaps many people will miss. Just below the text on the image it links to [some guidance](https://docs.microsoft.com/nb-no/azure/app-service/app-service-key-vault-references?WT.mc_id=Portal-fx&tabs=azure-cli) that can help you add a key vault - I recommend giving it a read.

In this post we will go through all the steps required to add a key vault to a data connector on an actual data connector template and end up with something you could deploy yourself.

## Adding the Key Vault

We will add our key vault at the bottom, below the last resource block.

For the key vault we will add three values into the keyvault:
1. `workspaceKey`
2. `APIUsername`
3. `APIPassword`

The resource block for the key vault itself will look like this:

```json
{
  "type": "Microsoft.KeyVault/vaults",
  "apiVersion": "2022-07-01",
  "name": "[variables('FunctionName')]",
  "location": "[resourceGroup().location]",
  "dependsOn": [
    "[resourceId('Microsoft.Web/sites', variables('FunctionName'))]"
    ],
  "properties": {
    "accessPolicies": [
      {
        "objectId": "[reference(resourceId('Microsoft.Web/sites', variables('FunctionName')), '2019-08-01', 'full').identity.principalId]",
        "permissions": {
          "secrets": [ "get",
                      "list"
                    ]
        },
        "tenantId": "[subscription().tenantId]"
      }
    ],
    "enabledForDeployment": "false",
    "enabledForDiskEncryption": "false",
    "enabledForTemplateDeployment": "true",
    "enableSoftDelete": "true",
    "sku": {
      "family": "A",
      "name": "Standard"
    },
    "tenantId": "[subscription().tenantId]",
  }
}
```

This is as basic as it probably can get for key vaults, it creates a key vault dependant on the `Microsoft.Web/sites` created earlier. 

It also adds an access policy to the Managed Identity of the Azure Function itself, so that it can retrieve the secrets stored in the key vault. For this to work we just need to make sure that the following is set under the `Microsoft.Web/sites` resource block:

```json
"identity": {
    "type": "SystemAssigned"
},
```

## Adding secrets to the vault
Moving on, we need to add the secrets themselves. Secret blocks are added as resources under the key vault block:

```json
...
    },
    "tenantId": "[subscription().tenantId]",
  },
  "resources": [
    {
       "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2021-04-01-preview",
      "name": "[variables('workspaceKey')]",
      "properties": {
        "value": "[parameters('workspaceKey')]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName'))]"
      ]
    },
    {
       "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2021-04-01-preview",
      "name": "[variables('APIUsername')]",
      "properties": {
        "value": "[parameters('APIUsername')]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName'))]"
      ]
    },
    {
       "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2021-04-01-preview",
      "name": "[variables('APIPassword')]",
      "properties": {
        "value": "[parameters('APIPassword')]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName'))]"
      ]
    }
  ]
```

## Adding secrets to the function app

First, we need to add dependencies to the `config` resource block under the `Microsoft.Web/sites` block:

```json
"dependsOn": [
    "[concat('Microsoft.Web/sites/', variables('FunctionName'))]",
    "[resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName'))]",
    "[resourceId('Microsoft.KeyVault/vaults/secrets', parameters('keyVaultName'), variables('workspaceKey'))]",
    "[resourceId('Microsoft.KeyVault/vaults/secrets', parameters('keyVaultName'), variables('APIUsername'))]",
    "[resourceId('Microsoft.KeyVault/vaults/secrets', parameters('keyVaultName'), variables('APIPassword'))]"
],
```

Next step is change the static variables of the parameters we chose into dynamic key vault references.

This is how it currently looks:

```json
"workspaceKey": "[parameters('WorkspaceKey')]",
"apiUsername": "[parameters('APIUsername')]",
"apiPassword": "[parameters('APIPassword')]",
```

When referencing key vault secrets in ARM-templates, we can use the [following syntax:](https://docs.microsoft.com/nb-no/azure/app-service/app-service-key-vault-references?WT.mc_id=Portal-fx&tabs=azure-cli#reference-syntax)

```json
"workspaceKey": "[concat('@Microsoft.KeyVault(SecretUri=', reference(variables('WorkspaceKey')).secretUriWithVersion, ')')]",
"apiUsername": "[concat('@Microsoft.KeyVault(SecretUri=', reference(variables('APIUsername')).secretUriWithVersion, ')')]",
"apiPassword": "[concat('@Microsoft.KeyVault(SecretUri=', reference(variables('APIPassword')).secretUriWithVersion, ')')]",
```
## Modifying input from string to securestring

Modyfing the input parameters to be `SecureString` in place of `String` will give the same result, but the input will be shielded and the value of the parameter [won't be saved to deployment history or logged.](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/data-types#secure-strings-and-objects)

We'll modify two of the three parameters we put into the key vault:
* `workspaceKey`
* `APIPassword`

```json
"WorkspaceKey": {
    "type": "securestring",
    "defaultValue": "<workspaceKey>"
},
"APIUsername": {
    "type": "string",
    "defaultValue": "<apiUsername>"
},
"APIPassword": {
    "type": "securestring",
    "defaultValue": "<apiPassword>"
},
```

## Full key vault resource block


```json
{
  "type": "Microsoft.KeyVault/vaults",
  "apiVersion": "2022-07-01",
  "name": "[variables('FunctionName')]",
  "location": "[resourceGroup().location]",
  "dependsOn": [
    "[resourceId('Microsoft.Web/sites', variables('FunctionName'))]"
    ],
  "properties": {
    "accessPolicies": [
      {
        "objectId": "[reference(resourceId('Microsoft.Web/sites', variables('FunctionName')), '2019-08-01', 'full').identity.principalId]",
        "permissions": {
          "secrets": [ "get",
                      "list"
                    ]
        },
        "tenantId": "[subscription().tenantId]"
      }
    ],
    "enabledForDeployment": "false",
    "enabledForDiskEncryption": "false",
    "enabledForTemplateDeployment": "true",
    "enableSoftDelete": "true",
    "sku": {
      "family": "A",
      "name": "Standard"
    },
    "tenantId": "[subscription().tenantId]"
  },
  "resources": [
    {
       "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2021-04-01-preview",
      "name": "[variables('workspaceKey')]",
      "properties": {
        "value": "[parameters('workspaceKey')]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName'))]"
      ]
    },
    {
       "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2021-04-01-preview",
      "name": "[variables('APIUsername')]",
      "properties": {
        "value": "[parameters('APIUsername')]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName'))]"
      ]
    },
    {
       "type": "Microsoft.KeyVault/vaults/secrets",
      "apiVersion": "2021-04-01-preview",
      "name": "[variables('APIPassword')]",
      "properties": {
        "value": "[parameters('APIPassword')]"
      },
      "dependsOn": [
        "[resourceId('Microsoft.KeyVault/vaults', parameters('keyVaultName'))]"
      ]
    }
  ]
}
```

## Putting it all together

Using this resource block you can add a key vault to any function app deployment.
The steps you will need to take to make sure this works is:

1. [Add the block](https://github.com/infernuxmonster/MicrosoftSentinel-Templates/blob/main/AzureFunction_AddKeyVault.json)
2. Make sure the `Microsoft.Web/sites` block has a Managed Identity configured
3. Add dependencies for the key vault and secrets to the `config` block of the `Microsoft.Web/sites` block
4. Change the references to the variables you wish to add to the key vault in the `config` block
5. [Optional] Change input type from `String` to `SecureString` for all applicable parameters

### Full template

The full template can be found [here, on my Github.](https://github.com/infernuxmonster/MicrosoftSentinel-Templates/blob/main/QualysVM.json)

## Sources

* [Deploy Key Vault with a list of secrets](https://github.com/infernuxmonster/MicrosoftSentinel-Templates/blob/main/QualysVM.json)
* [Reference secrets from Key Vault](https://docs.microsoft.com/nb-no/azure/app-service/app-service-key-vault-references?tabs=azure-powershell)