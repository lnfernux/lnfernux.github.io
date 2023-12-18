---
layout: post
title: Increasing the default timeout of Azure Functions
subtitle: Azure Functions are used for most data connectors, but some of them have a very low default timeout.
tags:
  - Cloud Security
  - Microsoft Sentinel
  - Data Connectors
  - Azure Functions
  - Automation
  - MISP
published: true
author: author_infernux
image: /img/sentinel.png
---

# Background

During the last few months I've been working on sending indicators from a [MISP-server](https://www.misp-project.org/) to Sentinel. First porting it to an Azure Function myself, then stealing some code from [zolderik](https://github.com/zolderio/misp-to-sentinel/tree/main) to make the code work for multi-tenant pushing. I have also upgraded it to include the work of [cudeso](https://github.com/cudeso/misp2sentinel) and also hope to upgrade it to use the new [upload indicators API](https://learn.microsoft.com/en-us/azure/sentinel/whats-new#may-2023/?wt.mc_id=SEC-MVP-5005030) that just went into general availability.

You can follow the 'official' repo [here](https://github.com/cudeso/misp2sentinel) and follow development on the upload indicators version of the same script [here](https://github.com/cudeso/misp2sentinel/tree/upload_indicators_api). I'm planning to push my current multi-tenant implementation as a PR (soonTM) and hopefully expand it to this new API as well. 

_When (if) I'm done, I'll write a new post about it._

# Function timeouts

![](/img/timeout.gif)

[According to Microsoft](https://learn.microsoft.com/en-us/azure/azure-functions/functions-versions?tabs=v4&pivots=programming-language-csharp#timeout/?wt.mc_id=SEC-MVP-5005030) there's a default timeout of 5 minutes for Azure Functions on the consumption plan with a maximum duration of 10 minutes. This is fine for most functions, but when you're working with data connectors, you might run into issues, especially if you're working with large amounts of data in a single run.

If we upgrade to the premium plan, however, we can increase this timeout to 30 minutes default or unlimited maximum. This might be a bit overkill in some cases, but for the MISP2Sentinel function it's a must.

# Configure the timeout

No matter what plan you're on, you have two options to configure the timeout: 

1. Modify the `functionTimeout` in the `host.json` [file](https://learn.microsoft.com/en-us/azure/azure-functions/functions-host-json#functiontimeout/?wt.mc_id=SEC-MVP-5005030)
2. Add the `AzureFunctionsJobHost__functionTimeout` app setting in the configuration of the function in the Azure Portal

* Please note that trying to extend the timeout beyond the maximum will not work. This does not really apply to the premium or dedicated plans, but will apply to the consumption plan. *

## host.json

The `host.json` file is located in the root of the function app. If you're using the Azure Portal, you can find it under the `Configuration` tab of the function app. Most data connectors will run with the `WEBSITE_RUN_FROM_PACKAGE` app setting, which means you can modify it there. If you're using Visual Studio Code, you can find it in the root of the function app.

Add the following to the `host.json` file (example is 1 hour):

```json
{
    "functionTimeout": "01:00:00"
}
```

## Azure Portal

If you're using the Azure Portal, you can find the `AzureFunctionsJobHost__functionTimeout` app setting under the `Configuration` tab of the function app. You can add it there and set it to the desired value.

Add the following key value pair to the app setting (example is 2 hours):

```
"AzureFunctionsJobHost__functionTimeout": "02:00:00"
```

**Note:** Thanks to the user [paulbatum](https://github.com/Azure/azure-functions-host/issues/6111) which presented this solution on the Azure github.