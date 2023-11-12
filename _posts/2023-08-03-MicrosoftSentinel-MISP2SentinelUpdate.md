---
layout: post
title: Use Update Indicators API to push Threat Intelligence from MISP to Microsoft Sentinel
subtitle: A quick intro on how to set up MISP, Azure Functions and Sentinel to push threat intelligence from MISP to Sentinel
tags:
  - Cloud Security
  - Microsoft Sentinel
  - Data Connectors
  - Azure Functions
  - Automation
  - MISP
  - Upload Indicators API
published: true
author: author_infernux
image: /img/sentinel.png
---

An updated guidance on how to set up the MISP2Sentinel Azure Function to push threat intelligence from MISP to Microsoft Sentinel using the new Upload Indicators API.

#### Table of Contents
- [MISP2Sentinel](#misp2sentinel)
  - [Installation documents](#installation-documents)
- [Updated guidance](#updated-guidance)
    - [MISP server](#misp-server)
    - [Azure AD App Registration](#azure-ad-app-registration)
    - [Azure Key Vault](#azure-key-vault)
    - [Microsoft Sentinel](#microsoft-sentinel)
    - [Azure Function](#azure-function)
    - [Verify successful execution](#verify-successful-execution)
- [More information](#more-information)
    - [MISP2Sentinel solution](#misp2sentinel-solution)
    - [References](#references)

# MISP2Sentinel

So my [previous post](https://www.infernux.no/MicrosoftSentinel-PushTIfromMISP/) was about pushing Threat Intelligence from MISP to Microsoft Sentinel, namely the `ThreatIntelligenceIndicators`-table. The method for doing this was using the `Microsoft Graph API`, which is in the process of being superseded by the new `Upload Indicators API`. This means we've got some updates to do!

![](https://media.tenor.com/rY95EqesaaoAAAAC/bob-the-builder.gif)

This post will *first and foremost* be a **quick** guide on how to set up and deploy the new solution. The **majority** of the work is being done by [Koen van Impe](https://github.com/cudeso/) on the python-front, I'm simply helping out in the **Azure-department**, making sure there's a version that's easily deployable that works when running as an Azure Function. You can follow the [MISP2Sentinel](https://github.com/cudeso/misp2sentinel) project on Github for more updates. 

## Installation documents

- [README.MD](https://github.com/cudeso/misp2sentinel/blob/main/AzureFunction/README.MD)
- [INSTALL.MD](https://github.com/cudeso/misp2sentinel/blob/main/docs/INSTALL.MD)

> **NOTE**: The installation guide is still a work in progress, but should be sufficient to get you up and running. Any feedback is appreciated, so feel free to open an issue on Github.

# Updated guidance

Not much will change from my previous post, so most of this guidance will be similar. The only difference is that you don't need to add `Microsoft Graph API`-permissions to your Azure AD app registration, but instead need to add a role on the workspace, namely the `Microsoft Sentinel Contributor`-role. 

> **NOTE**: This comes with the unwanted side-effect of the current implementation of the multi-tenant push **only being able to push** to **one workspace per tenant**.

## MISP server

1. Create a new Azure VM called MISP running **Ubuntu LTS 20.04**:

    ![MISP VM](/img/MISP/MISP_VM.png)

2. All default settings except for **adding an NSG with only port 22 open to my IP address for SSH access**, and changed the username to `misp`.
3. Followed the [MISP installation guide](https://misp.github.io/MISP/INSTALL.ubuntu2004) to install MISP on the VM by running the following command:
    ```bash
    wget --no-cache -O /tmp/INSTALL.sh https://raw.githubusercontent.com/MISP/MISP/2.4/INSTALL/INSTALL.sh
    ```
    ![](/img/MISP/MISP_install.png)

4. **Opened port 443** in the NSG to **allow for access to the MISP server from the Azure Function**.
5. We can now log in to the MISP server using default credentials.

    ![](/img/MISP/InitialInstall.png)

5. Go to the `Feeds` tab.

    ![](/img/MISP/Feeds.png)

6. **Enable** the two default feeds.

    ![](/img/MISP/EnabledSelectedFeeds.png)

7. **Pull data** from the feeds by clicking on the arrow pointing down next to the feed name.
8. We should be able to see events being pulled from the feeds now if we head over to the `Administration` tab and select `Jobs`.

    ![](/img/MISP/FetchFeedsJobs.png)

9. The output we need from the MISP server is the following:
    - `URL` (this will be the Azure public IP address of the VM in the format of `https://<ip address>/`)
    - `API key` (this was in the output when the install finished, but we can also add a new one by going to `Administration` and selecting `Add authentication key`.)

---

## Azure AD App Registration

1. **Create a new App Registration** in Azure AD called `MISP2Sentinel` using all default settings.

    ![](/img/MISP/MISP2Sentinel.png)

2. **Add a new secret** under Certificates & secrets - remember to take a note of the value.
3. Add the app registration to the `Microsoft Sentinel Contributor` role on your Microsoft Sentinel-workspace.
4. Output that we need to save from this step are the following:
    - `Application (client) ID`
    - `Directory (tenant) ID`
    - `Client secret`

---

## Azure Key Vault

1. **Create a new Azure Key Vault** called `MISP2Sentinel-kv` using all default settings.
2. **Add the following secrets** to the Key Vault:
    - `mispkey`
    - `mispurl`
    - `tenants`
3. The `tenants` secret is a JSON object containing the tenant ID, client id and secret of each tenant you want to push TI to:
    ```json
    [
        {
        "tenantId": "<TENANT_ID_WITH_APP_1>",
        "id": "<APP_ID>",
        "secret": "<APP_SECRET>",
        "workspaceId": "<WORKSPACE_ID>"
        },
        {
        "tenantId": "<TENANT_ID_WITH_APP_N>",
        "id": "<APP_ID>",
        "secret": "<APP_SECRET_N>",
        "workspaceId": "<WORKSPACE_ID_N>"
        }
    ]
    ```
4. You can limit access to the Key Vault by removing the public endpoint access, and only allowing access from the Azure Function by specifying the Azure Functions outbound IP addresses in the Key Vault firewall. This is not required, but recommended.

---

## Microsoft Sentinel

1. Install the **ThreatIntelligence** solution from the Microsoft Sentinel content hub.
2. Once onboarded, the data connector should show up (and turn green once data starts flowing in):
    ![](/img/MISP/TI_Enabled_DataConnector.png)

 
---

## Azure Function

1. Go to the service **Function App**
2. Click **Create** to generate a new Azure Function
- Give the function a descriptive name like `MISP2Sentinel`
- Choose at Publish for **Code**, and **Python** as the Runtime Stack.
- OS can remain Linux
- At plan type choose **App service plan** or **Premium** for production workloads. *I'll use consumption for this demo.*
- Other settings can be left to default values. Click **Review + Create**
3. After the creation of the Azure Function, add a [**system managed identity to the Azure Function**](https://learn.microsoft.com/EN-us/azure/app-service/overview-managed-identity?toc=%2Fazure%2Fazure-functions%2Ftoc.json&tabs=portal%2Chttp#add-a-system-assigned-identity). This will be used to authenticate with the Key Vault.

    ![](/img/MISP/AzureFunctionManagedIdentity.png)

4. Give the managed identity the **Reader** role on the Key Vault.
5. Go to the Key Vault and click on **Access policies**.
6. Click on **Add Access Policy**.
7. Select the **Secret permissions** tab and choose **Get** and **List** from the options.
8. Select the **Select principal** tab and search for the name of the Azure Function.
9. Click **Add** and **Save**.
10. Go back to the Azure Function and click on **Configuration**.
10. Add a new application setting with the the following settings:
    - **Name**: `tenants` 
    - **Value**: `@Microsoft.KeyVault(SecretUri=https://<keyvaultname>.vault.azure.net/secrets/tenants/)`
11. Add a new application setting with the the following settings:
    - **Name**: `mispkey`
    - **Value**: `@Microsoft.KeyVault(SecretUri=https://<keyvaultname>.vault.azure.net/secrets/mispkey/)`
12. Add a new application setting with the the following settings:
    - **Name**: `mispurl` 
    - **Value**: URL of your MISP instance (e.g. `https://<url>` or `https://<ip address>`)
13. Add a new application setting with the the following settings:
    - **Name**: `timerTriggerSchedule`
    - The `timerTriggerSchedule` takes a cron expression. For more information, see [Timer trigger for Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer?tabs=python-v2%2Cin-process&pivots=programming-language-python).
    * Run once every two hours cron expression: `0 */2 * * *`
14. ***OPTIONAL*** - Add a new application setting with the the following settings:
    - **Name**: `AzureFunctionsJobHost__functionTimeout` 
    - **Value**: `00:10:00` if using the consumption plan, or `02:00:00` if using premium or dedicated plans. *This setting is required to prevent the function from timing out when processing large amounts of data.*

This is how the application settings should look like (*I like to start of with a low frequency on the timer trigger to make sure everything is working as expected*):

![](/img/MISP/AppSettings.png)

---

## Upload the Function Code with Visual Studio Code

1. **Download and install** [Visual Studio Code](https://code.visualstudio.com/)
2. **Install** the [Azure Functions extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions)

![](/img/MISP/AzureFunctionsVSCode.png)

3. **Clone [this repo](https://github.com/cudeso/misp2sentinel)** and open the folder in Visual Studio Code.
3. If required, **make changes to `config.py`**. This will mainly consist of updating the filter and lifetime of the IOCs. 
   - The parameters for the filter object can be found [here](https://buildmedia.readthedocs.org/media/pdf/pymisp/latest/pymisp.pdf) on page 34 and onward.
4. Right click on the folder called `Azure Function` and select **Deploy to Function App...**
5. Select the Azure Function you created in the previous steps and click **Deploy**
6. You should see **Deployment succesful** in the output window after a short while.
7. The `MISP2Sentinel` function should also show up under the Function App.

    ![](/img/MISP/Function.png)

---

## ZIP-deploy of the Function Code with Powershell, AZ CLI or CI/CD pipelines

Added the option to do a ZIP-deploy. If you want to make changes to the ZIP-file, simply send the contents of the `AzureFunction`-folder (minus any `.venv`-folder you might have created) to a ZIP-file and upload that.

   
### Powershell

```pwsh
Publish-AzWebApp -ResourceGroupName <ResourceGroupName> -Name <FunctionName> -ArchivePath .\AzureFunction.zip -Force
```

### AZ CLI

```azcli
az functionapp deployment source config-zip --resource-group <resourcegroupname> --name <functionappname> --src <path to zip file>`.
```

### WEBSITE_RUN_FROM_PACKAGE

You can also use the [`WEBSITE_RUN_FROM_PACKAGE`](https://learn.microsoft.com/en-us/azure/azure-functions/functions-app-settings#website_run_from_package) configuration setting, which will allow you to upload the ZIP-file to a storage account (or Github repository) and have the Azure Function run from there. This is useful if you want to use a CI/CD pipeline to deploy the Azure Function, meaning you can just update the ZIP-file and have the Azure Function automatically update.

## Adding multi-tenancy support

0. Add a **redirect URI** to the app registration we created earlier, like `https://portal.azure.com`

    ![](/img/MISP/RedirectURIs.png)

1. To make the app registration work in the other tenants you will need to grant admin consent to the enterprise app in each tenant. This can be done **by navigating to the following URL**:
 
```
https://login.microsoftonline.com/common/adminconsent?client_id=<APP_ID>&sso_reload=true
```
2. If done correctly, you should see the following page:

    ![](/img/MISP/AskForConsentViaLink.png)

3. The enterprise application should now show up in the customer-environment. Grant this enterprise application the `Microsoft Sentinel Contributor`-role on the workspace. 
    * This can be accomplished by navigating to the IAM-menu and adding a role assignment.
    * Search for the **name** or **ObjectId** of the enterprise application (this needs to be the object ID of the enterprise application in the customer tenant).

3. Update the `tenants` secret in the Key Vault to include the new tenant ID. The client ID and secret should remain the same.
5. Make sure the **ThreatIntelligence data connector is enabled** in the new tenant (install the Threat Intelligence solution if it's not already installed).

---

## Verify successful execution

1. Go to the Azure Function and click on **Monitor**
2. Click on **Logs** to see the output of the function live, or check the **Invocations** to see the execution history.

    ![](/img/MISP/FunctionRuns.png)

3. You should see that indicators are being pushed to the tenants you specified in the `tenants` secret
4. You can also check the `ThreatIntelligenceIndicator` table in the Log Analytics workspace to see the indicators that have been pushed to Sentinel. 

    ![](/img/MISP/ThreatIntelligenceIndicatorTable.png)

---

# More information

## MISP2Sentinel solution

MISP2Sentinel is now also available as a solution!

![](/img/MISP/MISP2SentinelSolution.png)

You can find it in the content hub and it will show up as a new data connector (this helps your verify that you are actually ingesting MISP-events over other TI-sources):

![](/img/MISP/dataConnectorNew.png)

## References

- [Koen's blog post on the MISP-project site](https://www.misp-project.org/2023/08/26/MISP-Sentinel-UploadIndicatorsAPI.html/)
- [misp2sentinel repository](https://github.com/cudeso/misp2sentinel)
