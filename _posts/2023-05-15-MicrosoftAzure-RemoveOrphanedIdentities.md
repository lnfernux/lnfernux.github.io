---
layout: post
title: Removing orphaned Azure resource assigments
subtitle: Simple fix for removing any "identity not found" on resources in Microsoft Azure.
tags:
  - Cloud Security
  - Microsoft Azure
  - IAM
published: true
author: author_infernux
image: /img/azure.png
---

## Problem

Whenever an identity principal in Azure gets access to a resource, what happens in reality is that it's assigned a role defitinion on a defined scope. 

When that identity principal is removed, the assignment will still linger and show up under the IAM portion as the image below shows:

![](https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fazure%2Factive-directory%2Fmanaged-identities-azure-resources%2Fmedia%2Fmanaged-identity-best-practice-recommendations%2Fidentity-not-found.png&f=1&nofb=1&ipt=b5882cc06afa0fced6e249bfd16bd8ee7f64774d65d1ac681bbec3a3ec050188&ipo=images)

## Solution


```powershell
# ObjectType will be unknown
$objectType = "Unknown"
$orphanedIdentities = Get-AzRoleAssignment | Where-object -Property ObjectType -eq $objectType
foreach($identity in $orphanedIdentities) {
   # Role assignment removals will require the principal, definition name/id and scope of assignment to work
   Remove-AzRoleAssignment -ObjectId $identity.ObjectId -RoleDefinitionName $identity.RoleDefinitionName -Scope $identity.Scope
}
```

I've seen plenty of solutions for this, using both scripts as I've done above and policies. I've added links below to check out if you want to remove using policies, but I think the most simple solution would be to implement a simple script in a scheduled pipeline run.

## Source

- [Script](https://github.com/infernuxmonster/MicrosoftSentinel-Scripts/blob/main/Remove-OrphanedAzureResourceAssignments.ps1)
- [Stackoverflow thread](https://stackoverflow.com/questions/75002189/delete-orphaned-role-assignments-in-azure)
- [Removing Unknown Azure RBAC Role Assignments with PowerShell @ jloudon.com](https://jloudon.com/cloud/Removing-Unknown-Azure-RBAC-Role-Assignments-with-PowerShell/)
- [Orphaned Azure Security Principals Clean-up & Azure Policy Managed Identity Role Assignment Automation](https://mortenknudsen.net/?p=938)