
## Audit activities

We can solve auditing activites by sending `AzureActivity` logs to Microsoft Sentinel. These are enabled by default on the subscription-level and will provide insight into actions taken on the management level (from the outside). We can also [enable resource logs (or diagnostic logs) on specific resources](https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings?tabs=portal#create-diagnostic-settings) and [Azure Active Directory logs](https://docs.microsoft.com/en-us/azure/active-directory/reports-monitoring/concept-audit-logs) such as `SignIn` and `Audit`.


![](https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/media/platform-logs-overview/logs-overview.png)


For a full list over what actions that will be auditing, see [Microsoft Sentinel data included in Azure Activity logs.](https://docs.microsoft.com/en-us/azure/sentinel/audit-sentinel-data#microsoft-sentinel-data-included-in-azure-activity-logs)


## AzureActivity

A lot of possible scenarios here, but some simple examples from [Auditing with Azure Activity logs](https://docs.microsoft.com/en-us/azure/sentinel/audit-sentinel-data#auditing-with-azure-activity-logs) include searching for delete jobs:

```kql
AzureActivity
| where OperationNameValue contains "SecurityInsights"
| where OperationName contains "Delete"
| where ActivityStatusValue contains "Succeeded"
| project TimeGenerated, Caller, OperationName
```

We can also extend any query here into multiple workspaces and use it to create alerts that span multiple workspaces (in this case workspace `B` and `C`):

```kql
union isfuzzy=true AzureActivity, workspace("B").AzureActivity, workspace("C").AzureActivity
| where OperationNameValue contains "SecurityInsights"
| where OperationName contains "Delete"
| where ActivityStatusValue contains "Succeeded"
| project TimeGenerated, Caller, OperationName
```

This requires the user running the query (or creating the analytic rule) to have access to workspace `B` and `C` when running the query or when creating the analytic rule.