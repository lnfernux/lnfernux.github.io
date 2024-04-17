---
layout: post
title: 'Download Azure DevOps Repositories using a Managed Identity and REST API'
subtitle: Everything you need to know to download Azure DevOps repositories using a Managed Identity and REST API
tags:
  - Nice to know
  - Entra ID
  - Azure
  - Azure DevOps
  - Managed Identity
published: true
author: author_infernux
image: /img/devops.png
---

In this post, we will go over how to download Azure DevOps repositories using a Managed Identity and REST API. This is a follow-up to the post on [how to authenticate to Azure DevOps using a Managed Identity](https://www.infernux.no/AuthenticateToAzureDevops/).

## List all repositories

```powershell
Connect-AzAccount -Identity
$token = (Get-AzAccessToken -ResourceUrl "499b84ac-1321-427f-aa17-267ca6975798/.default").Token
$org = "MyOrg"
$project = "SecretSauce"
$uri = "https://dev.azure.com/$org/$project/_apis/git/repositories?api-version=7.0" 
$header = @{
    'Authorization' = "Bearer $token"
    'Content-Type' = 'application/json'
}
$response = Invoke-RestMethod -Uri $uri -Headers $header -Method GET
```

Now, the response contains the repositories in the project `SecretSauce` in the organization `MyOrg`. The response is a JSON object that contains a list of repositories and most importantly, their IDs. 

## Download a specific repository

Let's assume we want to download the repository named **`MyRepo`**. We can do this by using the following code:

```powershell
$repoId = ($response.value | Where-Object { $_.name -eq "MyRepo" }).id
$uri = "https://dev.azure.com/$org/$project/_apis/git/repositories/$repoId/items?path=/&"+'$format=zip&download=true'
$repo_response = Invoke-RestMethod -Uri $uri -Headers $header -Method GET -OutFile "MyRepo.zip"
```

This will download the repository named `MyRepo` as a zip file to the current directory.