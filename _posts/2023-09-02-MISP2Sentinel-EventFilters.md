---
layout: post
title: Figuring out MISP2Sentinel Event Filters
subtitle: How they work, how to use them and some (hopefully not horrible) examples.
tags:
  - MISP
  - Threat Intelligence
  - Microsoft Sentinel
  - IOC
published: true
author: author_infernux
image: /img/sentinel.png
---

#### Table of contents

- [Let's get hunting](#lets-get-hunting)
- [Event filtering basics](#event-filtering-basics)
- [Default MISP2Sentinel filters](#default-misp2sentinel-filters)
    - [`published`](#published)
    - [`tags`](#tags)
    - [`enforceWarninglist`](#enforcewarninglist)
    - [`includeEventTags`](#includeeventtags)
    - [`publish_timestamp`](#publish_timestamp)
- [Some honorable mentions](#some-honorable-mentions)
    - [`to_ids`](#to_ids)
    - [`type_attribute`](#type_attribute)
- [Putting it all together](#putting-it-all-together)
    - [Managed MISP](#managed-misp)
        - [TLP-levels](#tlp-levels)
        - [Workflow based tags](#workflow-based-tags)
        - [Published events](#published-events)
    - [Umanaged MISP](#umanaged-misp)
        - [On feeds](#on-feeds)
        - [Some examples](#some-examples)
- [Closing remarks](#closing-remarks)
    - [Previous writing on MISP](#previous-writing-on-misp)
    - [References](#references)

# Let's get hunting

With the new [Upload Indicators API]() coming out, the [misp2sentinel](https://github.com/cudeso/misp2sentinel)-project being [updated to use this new API](https://www.misp-project.org/2023/08/26/MISP-Sentinel-UploadIndicatorsAPI.html/) and provided [as a solution in Content Hub](https://portal.azure.com/#create/microsoftsentinelcommunity.azure-sentinel-solution-misp2sentinel), I thought it would be a good time to write a bit about how event filters work in MISP and hopefully help some people hunting for threats in their environments.

![](/img/gun.gif)

# Event filtering basics

The first thing we need to know about MISP event filters is that they are an implementation of parameters used to search the [MISP API](https://www.misp-project.org/openapi/#tag/Events/operation/restSearchEvents). A similar list of parameters can be found in the [PyMISP documentation](https://pymisp.readthedocs.io/en/latest/modules.html#pymisp.PyMISP.search). MISP allows for exporting events in the [STIX](https://stixproject.github.io/) format, which is also what the new upload indicators API accepts. The image below shows the workflow of the MISP2Sentinel project and where the event filters are used.

![](https://www.misp-project.org/img/blog/misp2sentinel-workflow.png)

The second thing we need to know is that how we implement our filter will vary a lot depending on a few factors with **how you manage MISP** being the primary factor. For brevity I've split the different management methods into two categories:

- **Managed MISP**: MISP is managed either by a dedicated team using a process for event creation. Typically this kind of instance will both pull and push from other MISP instances.
- **Unmanaged MISP**: This is what we might refer to as an feed MISP instance (or IOC hub). This kind of instance is typically used to pull data from other MISP instances and feeds, and might not have any event creation process.

# Default MISP2Sentinel filters

The [default `config.py`](https://github.com/cudeso/misp2sentinel/blob/main/config.py.default) file contains the following example filter:

```python
misp_event_filters = {
    "published": 1,
    "tags": [ "workflow:state=\"complete\""],
    "enforceWarninglist": True,
    "includeEventTags": True,
    "publish_timestamp": "14d",
}
```

These are common options, so let's go through them one by one.

## `published`

|Name | Description | Example |
|-----|-------------|---------|
|`published (Optional[bool])`| Set whether published or unpublished events should be returned. Do not set the parameter if you want both.|`"published": 1`|

This parameter represents the event publication state. If the event was published, the published value MUST be true. In any other publication state, the published value MUST be false. This parameter is present in all events. This is an optional value.

## `tags`

|Name | Description | Example |
|-----|-------------|---------|
|`tags (Optional[TypeVar(SearchParameterTypes, str, List[Union[str, int]], Dict[str, Union[str, int]])])`| Tags to search or to exclude. You can pass a list, or the output of build_complex_query|Include all events with the tag `workflow:state` set to `complete` - `"tags": [ "workflow:state=\"complete\""]`<br>Include `TLP:Green` and exclude `TLP:Red` - `"tags": ["!TLP:RED","TLP:GREEN"]`|

This parameter is used to search for events with a specific tag. You can pass a list of tags. This is an optional value, but a very strong filter to use. According to the [MISP Best Practices](https://www.circl.lu/doc/misp/best-practices/), tagging with TLP-levels is a **minimum** for events. 

If we want to filter out events with a certain tag, we can use the `!`-operator. This is useful if we want to filter out events with a certain TLP-level, like `TLP:RED` or `TLP:AMBER+STRICT`.

## `enforceWarninglist`

| Name | Description | Example |
| ---- | ----------- | ------- |
| **`enforce_warninglist`** `(Optional[bool])`| Remove any attributes from the result that would cause a hit on a warninglist entry. |`"enforceWarninglist": True`|

This is basically as false positive filter, which will remove a lot of common IPs (like those belonging to Cloud Providers) from the results. This is an optional value.

## `includeEventTags`

| Name | Description | Example |
| ---- | ----------- | ------- |
|`include_event_tags (Optional[bool])`|Include the event level tags in each of the attributes. |`"includeEventTags": True`|

Optional value that will include the event tags in the attributes. 

## `publish_timestamp`

| Name | Description | Example |
| ---- | ----------- | ------- |
| `publish_timestamp (Union[datetime, date, int, str, float, None, Tuple[Union[datetime, date, int, str, float, None], Union[datetime, date, int, str, float, None]]])`|Restrict the results by the last publish timestamp (newer than).|`"publish_timestamp": "14d"`|

This is a very useful filter, as it allows you to only pull events that have been published in the last X days. This is an optional value.

# Some honorable mentions

## `to_ids`

| Name | Description | Example |
| ---- | ----------- | ------- |
| **`to_ids`** `(Union[TypeVar(ToIDSType, str, int, bool), List[TypeVar(ToIDSType, str, int, bool)], None])`|By default all attributes are returned that match the other filter parameters, regardless of their to_ids setting. To restrict the returned data set to to_ids only attributes set this parameter to 1. 0 for the ones with to_ids set to False.|`"to_ids": 1`|

> Attributes in MISP have a boolean flag to_ids allowing you to indicate if an attribute should be used for detection or correlation actions. According to the [MISP core format data standard](https://www.misp-standard.org/rfc/misp-standard-core.html#rfc.section.2.4.2), the to_ids flag represents whether the attribute is meant to be actionable. *[https://www.vanimpe.eu/2019/09/24/tracking-false-positives-and-disabling-to_ids-in-misp/](https://www.vanimpe.eu/2019/09/24/tracking-false-positives-and-disabling-to_ids-in-misp/).*

## `type_attribute`

| Name | Description | Example |
| ---- | ----------- | ------- |
|`type_attribute (Optional[TypeVar(SearchParameterTypes, str, List[Union[str, int]], Dict[str, Union[str, int]])])`| The attribute type, any valid MISP attribute type is accepted.|`"type_attribute": "ip-src"`|

You can see all the possible attribute types [in the MISP OpenAPI spec](https://www.misp-project.org/openapi/#tag/Attributes/operation/restSearchAttributes). If we only wanted to include IP addresses, we could use this filter to only include attributes of type `ip-src` and `ip-dst`. We could also filter out unwanted types, like `domain` or `hostname`.

# Putting it all together

## Managed MISP

You will most likely end up with a filter very close to the default `config.py` when using a managed MISP, the only thing that might differ are what TLP-levels you'll send to other organizations and how you manage your event creation process, which will return in a slightly different filter.

### TLP-levels

If we use TLP-levels, we can use the `tags` parameter to include only events that have a TLP-level that we want to return. Two scenarios here are as follows:

1. You are pushing to your own Microsoft Sentinel, in this case we don't need to filter for TLP as we can include all.
2. You are pushing to a another Microsoft Sentinel, in this case we need to filter out events with `TLP:RED` and `TLP:AMBER+STRICT` as we don't want to share those outside our organization.

If you're wondering what TLP-levels are, TLP is short for Traffic Light Protocol and we currently have 5 levels. 

#### TLP:RED
![](https://www.cisa.gov/sites/default/files/styles/16x9_small/public/2023-02/tlp_teaser_red.png?h=c653ff14&itok=QOt4tAAU)

Not for disclosure, restricted to participants only.

#### TLP:AMBER+STRICT
![](https://www.cisa.gov/sites/default/files/styles/16x9_small/public/2023-02/tlp_teaser_amber_strict.png?h=c653ff14&itok=Z3k0H6iW)

Limited disclosure, restricted to participants’ organization.

#### TLP:AMBER
![](https://www.cisa.gov/sites/default/files/styles/16x9_small/public/2023-02/tlp_teaser_amber.png?h=c653ff14&itok=iqf0bzad)

Limited disclosure, restricted to participants’ organization and its clients (see Terminology Definitions).


#### TLP:GREEN
![](https://www.cisa.gov/sites/default/files/styles/16x9_small/public/2023-02/tlp_teaser_green.png?h=c653ff14&itok=zbArdsX9)

Limited disclosure, restricted to the community.

#### TLP:CLEAR
![](https://www.cisa.gov/sites/default/files/styles/16x9_small/public/2023-02/tlp_teaser_clear.png?h=c653ff14&itok=0D6kLdeZ)

Disclosure is not limited.

This TLP was formerly known as `TLP:WHITE`. 

*Microsoft Graph API uses `TLP:WHITE` instead of `TLP:CLEAR`, so when trying to sync events with `TLP:CLEAR` you might get an error.*

### Workflow based tags

For a managed MISP, we'll need to look at the way we manage events. If we have a process for event creation, then we can use the `tags` parameter to filter on the event tags we use for our process. Common tags here are the `workflow`-ones, and we can chose (like the default `config.py`) to only include ones that have completed. If we use another tag for our process, we can use that instead. 

As an example we can look to the default `config.py`.

### Published events

Another common filter is to only include published events. If your process includes publishing reviewed event, then this is a good filter to implement. If you don't have a process for publishing events, then you can leave this out.

As an example we can look to the default `config.py` again.

## Umanaged MISP

This deployment type of MISP usually consumes a lot of feeds. Your strategy will vary solely on the percieved quality of the feeds you are consuming. If you are consuming feeds from a trusted source, then you might not need to filter at all. 

### On feeds

I will not be recommending a certain set of feeds, but the following can be done to minimize false positives and "bad" feeds:

- Use the `enforceWarninglist` parameter to remove IPs belonging to cloud providers.
- Use the `publish_timestamp` parameter to only include events that have been published in the last X days.
- When enabling feeds, do your due dilligence in terms of checking the organization that is providing it.
- Do not enable low fidelity feeds just because they are free.
- Focus on feeds that are relevant to your organization.
- Map out organizations such as FIRST, your national CERT or other organizations that might allow you to pull/sync from their MISP instances and work on gaining access to those.

### Some examples

A quick note on `publish_timestamp`; depending on how often the `misp2sentinel`-sync runs, you can change the value to shorten the amount of indicators pulled and thus shorten the runtime of the process.

#### Unmanaged MISP with high quality feeds to your own Sentinel

Basic filter that allows you to pull all indicators from your MISP instance to your own Sentinel, bar the false positives.

```python
misp_event_filters = {
    "enforceWarninglist": True,
    "includeEventTags": True,
    "publish_timestamp": "14d"
}
```

#### Unmanaged MISP with high quality feeds to another Sentinel

Same as above, but we filter out all TLP-levels that we don't want to share.

```python
misp_event_filters = {
    "tags": ["!TLP:RED","!TLP:AMBER+STRICT"],
    "enforceWarninglist": True,
    "includeEventTags": True,
    "publish_timestamp": "14d"
}
```

# Closing remarks

There isn't a one size fits all solution for this, and the two methods for operating MISP only covers the two extremes. Most organizations will fall somewhere in between, and you'll have to figure out what works best for you.

Until next time, happy hunting!

## Previous writing on MISP

- [Use Update Indicators API to push Threat Intelligence from MISP to Microsoft Sentinel](https://www.infernux.no/MicrosoftSentinel-MISP2SentinelUpdate/)
- [Pushing Threat Intelligence from MISP to Microsoft Sentinel](https://www.infernux.no/MicrosoftSentinel-PushTIfromMISP/)
- [Increasing the default timeout of Azure Functions](https://www.infernux.no/MicrosoftSentinel-AzureFunctionDataConnectorsTimeout/)

## References

- [MISP Best Practices](https://www.circl.lu/doc/misp/best-practices/)
- [MISP Core Format Data Standard](https://www.misp-standard.org/rfc/misp-standard-core.html#rfc.section.2.4.2)
- [Tracking false positives and disabling to_ids in MISP](https://www.vanimpe.eu/2019/09/24/tracking-false-positives-and-disabling-to_ids-in-misp/)
- [MISP OpenAPI spec](https://www.misp-project.org/openapi/)
- [PyMISP documentation](https://pymisp.readthedocs.io/en/latest/modules.html#pymisp.PyMISP.search)
- [Traffic Light Protocol](https://www.cisa.gov/news-events/news/traffic-light-protocol-tlp-definitions-and-usage)
