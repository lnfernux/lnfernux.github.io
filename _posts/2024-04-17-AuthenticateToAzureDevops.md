---
layout: post
title: 'Authenticate to Azure DevOps using Managed Identity and REST API'
subtitle: How to add a managed identity to Azure DevOps and get access tokens for Azure Devops
tags:
  - Entra ID
  - Azure
  - Managed Identity
  - Identity and Access Management
  - Cloud Security
  - Azure DevOps
published: true
author: author_infernux
image: /img/devops.png
---

This one is very short and sweet - how to authenticate to Azure DevOps using a Managed Identity. This can be done from a virtual machine, Azure Function, or any other Azure service that supports Managed Identities. **Usually** when we authenticate to Azure DevOps **we are stuck using a Personal Access Token (PAT)**. Not only is this token personal (as indicated by the name) and bound to a specific user, it's hard to control using policies and it's permissions are also not as granular as we would like. This is where Managed Identities come in. **Managed Identities are bound to a specific resource** and can be given permissions to access limited scopes of resources in Azure DevOps. 

## Adding a Managed Identity to Azure DevOps

This is as simple as it gets. Managed identities are named after the resource they are attached to. So if you have a virtual machine named `MyVM` and you want to add a managed identity to it, the managed identity will be named `MyVM`.

Simply go to the Azure DevOps organization and invite the managed identity `MyVM` as a user. After this you are free to assign it any permissions you want.

For this example let's say I gave `MyVM` access to a custom `Read Only` role in a project called `SecretSauce`. For the purpose of example, the org is called `MyOrg`.

## Connect to Azure

When authenticating to Azure using a managed identity you can use any flavor you want of programming language as long as it has a library or module that supports authentication using managed identities. In this case, we are going to use the `Az`-module for PowerShell.

```powershell
Connect-AzAccount -Identity
```

## Authenticating to Azure

Now we have authenticated to Azure, but we still need to authenticate to Azure DevOps. For Azure resources, this is done by obtaining a token. This defaults to the resource `https://management.azure.com/`:

```powershell
$token = (Get-AzAccessToken).Token
```

We can also do the same operation in the Azure CLI:

```bash
token=$(az account get-access-token --query accessToken --output tsv)
```

Or we can also do it directly using cURL or any other HTTP client:

```powershell
$response = Invoke-RestMethod -Uri "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com" -Headers @{Metadata="true"} -Method GET
$token = ($response.Content | ConvertFrom-Json).access_token
```

## Authenticating to Azure DevOps

The missing piece in all of this is how to authenticate to Azure Devops. Referenced only by way of "oh, by the way", we can find that in the Microsoft documentation for Azure DevOps services REST API regarding managing PATs, [there's a reference to aquire an acces token for the API.](https://learn.microsoft.com/en-us/rest/api/azure/devops/tokens/pats?view=azure-devops-rest-7.1/?wt.mc_id=SEC-MVP-5005030)

In short, this scope, `499b84ac-1321-427f-aa17-267ca6975798/.default`, is the scope for Azure DevOps. This is the scope you need to use to get an access token for Azure DevOps. 

*We can also use just `499b84ac-1321-427f-aa17-267ca6975798` as the scope.* 

This is how you do it in PowerShell:

```powershell
$token = (Get-AzAccessToken -ResourceUrl "499b84ac-1321-427f-aa17-267ca6975798").Token
```

## Putting it into a script

Here is a simple script that will authenticate to Azure using a managed identity, then get a token for Azure DevOps and list all repositories the identity has access to:

```powershell
Connect-AzAccount -Identity
$token = (Get-AzAccessToken -ResourceUrl "499b84ac-1321-427f-aa17-267ca6975798").Token
$org = "MyOrg"
$project = "SecretSauce"
$uri = "https://dev.azure.com/$org/$project/_apis/git/repositories?api-version=7.0" 
$header = @{
    'Authorization' = "Bearer $token"
    'Content-Type' = 'application/json'
}
$response = Invoke-RestMethod -Uri $uri -Headers $header -Method GET
$response.value
```

That's it. You can now authenticate to Azure DevOps using a managed identity. This is a great way to authenticate to Azure DevOps without having to use a PAT. This is especially useful because it allows you to scope access down to single projects, and resources inside those projects. This is a great way to secure your Azure DevOps environment.