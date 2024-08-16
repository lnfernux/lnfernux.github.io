---
layout: post
title: 'Adding Graph API permissions to Managed Identities'
subtitle: Making a little note of this in Graph API so it's easy to find for using it
tags:
  - Nice to know
  - Entra ID
  - Azure
  - Logic App
  - Managed Identity
published: true
author: author_infernux
image: /img/devops.png
---

In this post, we will go over how to simply add a Graph API permission to a managed identity. You can view the permissions from the Enterprise Application blade in Entra ID, but not add any new permissions. Instead we have to use Powershell.

**If you just want the full script, scroll to the bottom ;)**

# Step by step

Make sure the Graph API is installed, first off all!

```powershell
Install-Module Microsoft.Graph
```

## Authenticate with the proper access 

```powershell
Connect-MgGraph -Scopes Application.Read.All, RoleManagement.ReadWrite.Directory, AppRoleAssignment.ReadWrite.All
```
This should enable us to do what we need to do.

## Add the required information

First, we need to add a variable that contains the `ObjectId` of our managed identity. We can find this in the enterprise application blade.

```powershell
$objectId = "<objectId of Managed Identity>"
```

We also need the role name(s) we want to add to the application. For this example, we'll only do one role.

```powershell
$role = "<name of role, like Mail.Send>"
```

## Get the Graph API SPN

This is a simple action of getting the Graph SPN in Azure using the standard appId.

```powershell
$graphSPN = Get-MgServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'"
```

## Find the roleId

```powershell
$roleObject = $Msgraph.AppRoles| Where-Object {$_.Value -eq $roleName} 
```

## Add the role to managed identity

```powershell
New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $graphSPN -PrincipalId $objectId -ResourceId $graphSPN.Id -AppRoleId $roleObject.Id
```

# Full script

```powershell
Connect-MgGraph -Scopes Application.Read.All, RoleManagement.ReadWrite.Directory, AppRoleAssignment.ReadWrite.All
$objectId = "<objectId of Managed Identity>"
$role = "<name of role, like Mail.Send>"
$graphSPN = Get-MgServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'"
$roleObject = $Msgraph.AppRoles| Where-Object {$_.Value -eq $roleName} 
New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $graphSPN -PrincipalId $objectId -ResourceId $graphSPN.Id -AppRoleId $roleObject.Id
```