---
layout: post
title: Auditing Microsoft Azure queries in a Lighthouse-environment
subtitle: Quick introduction to auditing Microsoft Sentinel queries in a cross-tenant scenario - and some things to be aware of. 
tags:
  - Microsoft Sentinel
  - Azure Lighthouse
  - LAQueryLogs
  - AzureActivity
  - Audit
published: true
author: author_infernux
image: /img/sentinel.png
---

# Scenario

- We need to audit queries performed in Microsoft Sentinel
- We also need to consider queries performed in a multi-tenant Azure Lighthouse-environment

# Configure auditing

## Audit queries

In order to audit queries we need to enable the `LAQueryLogs` table. 
This can be achieved by going into the diagnostic settings of the Log Analytics workspace and creating a new diagnostic setting where we set the category `Audit` to `$true`.
You can also configure it [using ARM-templates.](https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/resource-manager-diagnostic-settings?tabs=json#diagnostic-setting-for-log-analytics-workspace)

An audit record will be created each time a query is ran. For more information about the data structure and it's properties, see [this page.](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/query-audit#audit-data)


# Example scenarios for auditing

## LAQueryLogs

We can either use this table to look for what searches users are performing, like a simple search across the last seven days:

```kql
LAQueryLogs
| where TimeGenerated > ago(7d)
```

Or we can use it to create alerts for queries into sensitive tables we would like to limit access to with this query:
```kql
LAQueryLogs
| where QueryText contains "[Name of sensitive table]"
| where TimeGenerated > ago(1d)
| extend User = AADEmail, Query = QueryText
| project User, Query
```

For more information please see this article by Microsoft; [Auditing with LAQueryLogs.](https://docs.microsoft.com/en-us/azure/sentinel/audit-sentinel-data#auditing-with-laquerylogs)

# Considerations

## LAQueryLogs in a multi-workspace Azure Lighthouse environment

Let's imagine we have multiple workspaces onboarded using Azure Lighthouse, as shown in the image below:

![](https://docs.microsoft.com/en-us/azure/sentinel/media/extend-sentinel-across-workspaces-tenants/cross-workspace-architecture.png)

If we look at the [considerations](https://docs.microsoft.com/en-us/azure/azure-monitor/logs/query-audit#considerations) for using `LAQueryLogs` we can see that:

> *For queries that include data from multiple workspaces, the query will only be captured in those workspaces to which the user has access.*

This means that any query performed cross workspace will be capture in all the workspaces the query is ran. 

Consider this query:
```kql
union isfuzzy=true AzureActivity, workspace("B").AzureActivity, workspace("C").AzureActivity
| limit 1
```

*We include `isfuzzy=true` in cross workspace-queries as a best practice, because if a table doesn't exist in one tenant the query will still run.* 

This would show up in the workspace it was ran in, as well as tenant B and C. For a company managing multiple of it's own workspaces, this is fine. 

If we consider an MSSP, this would mean that the workspace-names of multiple customers will be visible to everyone with the `LAQueryLogs` enabled. Workspace-names are not considered PII-data, so it's more of an inconvenience than an actual problem.

### Obfuscating queries

We could consider obfuscating the output to hide the workspaces, but if we look again at the considerations we find that this is not possible:

> *The h hint on strings that obfuscates string literals will not have an effect on the query audit logs. The queries will be captured exactly as submitted without the string being obfuscated. You should ensure that only users who have compliance rights to see this data are able to do so using the various Kubernetes RBAC or Azure RBAC modes available in Log Analytics workspaces.*

Similarly we could do ingestion time transformation of the logs, but messing with audit logs is not a good practice. This leaves us with one option, **using functions**. The query itself is exposed in a field called `QueryText`, so creating a function would be same as obfuscating the query itself.

Consider the same query we used as an example earlier:
```kql
union isfuzzy=true AzureActivity, workspace("B").AzureActivity, workspace("C").AzureActivity
| limit 1
```

Saving the query above as a function `CrossWorkspaceAzureActivity` would allow us to run the query without exposing the workspace names of our customers in the query, like so:

```kql
CrossWorkspaceAzureActivity
```

This solves the problem of the `QueryText`-field, however there is one more issue. 

### RequestContext

If we look at the [reference](https://docs.microsoft.com/en-us/azure/azure-monitor/reference/tables/laquerylogs) for `LAQueryLogs` we find that in addition to the `QueryText` field, we also have the `RequestContext`-field. This is described as:

> *ResourceId of all referenced workspaces, applications, and resources across which the query was requested by the caller to be executed.*

This means that for each workspace searched across (that has `LAQueryLogs` enabled), we would be includig the following information for each of them:
- subscriptionId
- resourceGroupName
- workspaceId

Again, this is not really an issue as neither of these three values are considered PII, but it's something to be aware of.

### Summary

In sum I'd say the value of cross workspace queries in Sentinel greatly outweighs this minor inconvenience, however it does leave something to be desired in terms of configuration options on the level of verbosity for the logs themselves. 


