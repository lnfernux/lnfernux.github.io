---
layout: post
title: Christmas Wrappers - Part 2
subtitle: How to create a wrapper script in Powershell
tags:
  - Powershell
  - pwsh
  - Cyber Security
  - MISP
published: true
author: author_infernux
image: /img/ps.png
---

Following up on my latest post, [Christmas Wrappers - Part 1](https://www.infernux.no/Christmas-Wrappers/), we'll be looking at how to expand our wrapper script in Powershell. In the last post we created a wrapper script for the MISP API. In this post we'll be adding functionality to the wrapper script.

Also, this was supposed to come out in December, but as always, stuff happens (I bought Baldur's Gate on the Steam Winter Sale...). Anyway, here it is.

![](https://media.tenor.com/vXecleSMdeUAAAAC/baldurs-gate3-bg3.gif)

### Table of contents

- [Functions in the artcle](#functions-in-the-artcle)
- [Mapping the functions](#mapping-the-functions)
- [Focusing on logging](#focusing-on-logging)
- [Creating the functions](#creating-the-functions)
  - [`Add-MISPEventTag`](#add-mispeventtag)
  - [The problem with tags](#the-problem-with-tags)
  - [`Add-MISPEventAttribute`](#add-mispeventattribute)
  - [`Create-MISPEvent`](#create-mispevent)
    - [How does the script work?](#how-does-the-script-work)
- [Closing words](#closing-words)

## Functions in the artcle

**From the last post we implemented the following functions:**

| Function | Description |
| --- | --- |
| New-MISPAuthHeader | Creates an authentication header for MISP |
| Invoke-MISPRestMethod | Invokes a REST-method against the MISP API |
| Get-MISPEvent | Gets an event from MISP |

I've also decided to postpone adding the functions for removing attributes, tags and events until a later post. This is mainly because removing tags, as far as I can read from the API, requires getting the event JSON-object and doing text-replacement on the tags in order to do a `PUT`-request. This is not very hard in essence, but from experience I usually spend way too many hours troubleshooting minor issues with the format of the body of the request. So I'll save that for a later post, if ever.

**This means that from the last part, we are missing the following functions:**

| Function | Description |
| --- | --- |
| Create-MISPEvent | Creates a new event in MISP |
| Add-MISPAttribute | Adds an attribute to an event in MISP |
| Add-MISPTag | Adds a tag to an event in MISP |

## Mapping the functions

Last time we created the base functions we need; `New-MISPAuthHeader` and `Invoke-MISPRestMethod`, which allow us to connect easily to MISP in our other functions. We also created a function to get an event from MISP; `Get-MISPEvent`. This will be used to check if an event already exists in MISP before we create it, or just plainly to download an event.

## Focusing on logging

One thing we didn't particularly focus on last time was logging. When troubleshooting, it's important to know what functions were called, how far you got and what values they were called with. So we will focus on adding simple terminal output logging to our functions this time.

## Creating the functions

Let's make sure we start in a way that makes sense. Our `Create-MISPEvent` function will need the `Add-MISPTag` and `Add-MISPAttribute` functions. So let's start with those. 

**NOTE**: It made sense to me to name the functions `Add-MISPTag` and `Add-MISPAttribute` instead of `New-MISPTag` and `New-MISPAttribute`. This is because the functions will be used to add tags and attributes to an existing event. But, we are adding them to an event, not generally to MISP. So renaming them to `Add-MISPEventTag` and `Add-MISPEventAttribute` makes more sense to me.

### `Add-MISPEventTag`

The `Add-MISPEventTag` function will be used to add a tag to an event in MISP. It will take the following parameters:

- `MISPUrl` - The URL to the MISP instance
- `MISPAuthHeader` - The authentication header for the MISP instance
- `MISPEventID` - The ID of the event we want to add the tag to
- `MISPTagId` - The ID of the tag we want to add to the event
- `LocalOnly` - This is a switch parameter that will add the tag locally to the event, but not sync it to other MISP instances

```powershell
function Add-MISPEventTag {
  PARAM(
    $MISPUrl,
    $MISPAuthHeader,
    $MISPEventID,
    $MISPTagId,
    [switch]$LocalOnly
  )
  # Which MISP API Endpoint we are working against
  $Endpoint = "events/addTag/$MISPEventID/$MISPTagId"

  # Create the body of the request
  $MISPUrl = "$MISPUrl/$Endpoint"

  # Check local only, add local only if true
  if($LocalOnly) {
    $MISPUrl = $MISPUrl0"/local:1"
  }

  # Invoke the REST method
  Write-Host "Trying to add tag $MISPTagId to event $MISPEventID"
  $return = Invoke-MISPRestMethod -Uri $MISPUrl -Header $MISPAuthHeader -Method Post
  return $return
}
```

We will reference the function in our `Create-MISPEvent` function, so we will cover usage there.

### The problem with tags

If you want to add tags to MISP, you need to add the `Id` of the tag. If we want to browse tags, we need to search or know them. To search tags, we can add a simple function `Get-MISPTags` that will return all the tags from MISP:

```powershell
function Get-MISPTags {
  PARAM(
    $MISPUrl,
    $MISPAuthHeader,
    $Tag
  )
  $Endpoint = "tags/search/$Tag"
  $MISPUrl = "$MISPUrl/$Endpoint"
  Write-Host "Trying to get ID for tag: $($Tag)"
  $return = Invoke-MISPRestMethod -Uri $MISPUrl -Headers $MISPAuthHeader -Method Get
  return $return
}
```

### `Add-MISPEventAttribute`

The `Add-MISPEventAttribute` function will be used to add an attribute to an event in MISP. It will take the following parameters:

- `MISPUrl` - The URL to the MISP instance
- `MISPAuthHeader` - The authentication header for the MISP instance
- `MISPEventID` - The ID of the event we want to add the attribute to
- `MISPAttribute` - The attribute we want to add to the event
- `MISPAttributeType` - The type of attribute we want to add
- `MISPAttributeCategory` - The category of the attribute we want to add
- `MISPAttributeComment` - The comment we want to add to the attribute

```powershell
function Add-MISPEventAttribute {
  PARAM(
    $MISPUrl,
    $MISPAuthHeader,
    $MISPEventID,
    $MISPAttribute,
    $MISPAttributeType,
    $MISPAttributeCategory,
    $MISPAttributeComment
  )
  # Which MISP API Endpoint we are working against
  $Endpoint = "attributes/add/$MISPEventID"

  # Create the body of the request
  $MISPUrl = "$MISPUrl/$Endpoint"
  $Body = @{
    value = $MISPAttribute
    type = $MISPAttributeType
    category = $MISPAttributeCategory
    comment = $MISPAttributeComment
    event_id = $MISPEventID
  }

  # Invoke the REST method
  Write-Host "Trying to add attribute $MISPAttribute to event $MISPEventID"
  $return = Invoke-MISPRestMethod -Uri $MISPUrl -Header $MISPAuthHeader -Method Post -Body ($Body |
  return $return
}
```

We will reference the function in our `Create-MISPEvent` function, so we will cover usage there.

### `Create-MISPEvent`

Now that we have the functions to add and remove tags and attributes, we can create the `Create-MISPEvent` function. This function will take the following parameters:

- `MISPUrl` - The URL to the MISP instance
- `MISPAuthHeader` - The authentication header for the MISP instance
- `MISPEventPublisher` - The publisher of the event, needs to be a valid publisher user in MISP
- `MISPTagsId` - An array of tag ids we want to add
- `MISPOrg` - The id of the organisation to add the event to
- `MISPEventName` - The title of the event
- `Publish` - A switch parameter that will publish the event if set to `$true
- `Distribution` - The distribution of the event, can be `0` (Organisation only), `1` (This community only), `2` (Connected communities), `3` (All communities) or `4` (Sharing group only),  **defaults to 0**
- `Attributes` - An array of hashtables of attributes created like this: 

  ```powershell
  $AllAttributes = @()
  $SingleAttribute = @{
    Attribute = "value"
    Type = "type"
    Category = "category"
    Comment = "comment"
  }
  $AllAttributes += $SingleAttribute
  ```

For more information on creating a MISP event, see the [MISP User Guidance on this topic](https://www.circl.lu/doc/misp/using-the-system/#creating-an-event).

```powershell
function Create-MISPEvent {
  PARAM(
    $MISPUrl,
    $MISPAuthHeader,
    $MISPEventPublisher,
    [array]$MISPTagsId,
    $MISPOrg,
    $MISPEventName,
    [switch]$Publish,
    $Distribution = 0
  )
  # Which MISP API Endpoint we are working against
  $Endpoint = "events/add"
  Write-Host "Trying to create event with title: $($MISPEventName)"

  # Check if event already exists
  $Event = Get-MISPEvent -MISPUrl $MISPUrl -MISPAuthHeader $MISPAuthHeader -MISPEventName $MISPEventName -MISPOrg $MISPOrg
  if($Event) {
    Write-Host "Event already exists, returning event"
    # Set eventID to existing event
    $MISPEventID =  $Event.Event.Id
  } else {
    # Continue script
    Write-Host "Event does not exist, creating event $MISPEventName"
    
    # Create body, we will add tlp:green as a tag for testing
    if($Publish) {
      $Publish = $true
    } else {
      $Publish = $false
    }
    $Body = @{
      info = "$MISPEventName"
      org_id = $MISPOrg
      published = $Publish
      event_creator_email = $MISPEventPublisher
      distribution = $Distribution
    }
    
    # Invoke the API to create the event
    $return = Invoke-MISPRestMethod -Uri "$MISPUrl/$Endpoint" -Header $MISPAuthHeader -Method Post -Body ($Body | ConvertTo-Json)
    
    # Get event id from return
    $MISPEventID = ($return.Content | ConvertFrom-Json).Event.Id
    
    # Add tags to event
    foreach($Tag in $MISPTagsId) {
      Add-MISPEventTag -MISPUrl $MISPUrl -MISPAuthHeader $MISPAuthHeader -MISPEventID $MISPEventID -MISPTagId $Tag
    }
  }
  # Event exists or has been created, now we can add attributes
  if($Attributes) {
    # Format of attributes is a hashtable with the following format: $HashTable = @{Attribute = "value"; Type = "type"; Category = "category"; Comment = "comment"}
    foreach($Attribute in $Attributes) {
      Add-MISPEventAttribute -MISPUrl $MISPUrl -MISPAuthHeader $MISPAuthHeader -MISPEventID $MISPEventID -MISPAttribute $Attribute.Attribute -MISPAttributeType $Attribute.Type -MISPAttributeCategory $Attribute.Category -MISPAttributeComment $Attribute.Comment
    }
  }
}
```

#### How does the script work? 

This part is generated entirely by Github Copilot:

> The provided PowerShell script defines a function `Create-MISPEvent` that is used to create an event in a MISP (Malware Information Sharing Platform) instance. MISP is an open-source threat intelligence platform used for gathering, sharing, storing and correlating Indicators of Compromise of targeted attacks, threat intelligence, financial fraud information, vulnerability information or even counter-terrorism information.
> The function takes several parameters, including the URL of the MISP instance, an authentication header, the publisher of the event, an array of tag IDs, the organization ID, the name of the event, a switch to indicate whether the event should be published, and the distribution of the event.
> The function first checks if an event with the same name already exists in the MISP instance by calling `Get-MISPEvent`. If it does, it simply returns the existing event. If not, it creates a new event with the provided parameters.
> The function then invokes a REST method to create the event in the MISP instance. It retrieves the ID of the newly created event and adds the specified tags to the event by calling `Add-MISPEventTags`.
> Finally, if any attributes are provided, the function adds these attributes to the event by calling `Add-MISPEventAttributes`. The attributes are expected to be in a specific format: a hashtable with keys for "Attribute", "Type", "Category", and "Comment".

## Closing words

This is the second part of the Christmas Wrappers series. We've now created a wrapper script that can create events in MISP, add tags and attributes to the event and check if an event already exists. We've also added some (very simple) logging to the functions, so that we can see what's happening when we run the script.

As always, the module is now updated on my Github repository:

[MISP-Scripts](https://github.com/lnfernux/MISP-Scripts)

Thanks for reading, and until next time!

![](/img/santa2.png)
*Santa, in January, making sure his inventory is stocked for next year, probably, courtesy of Midjorney and my subpar prompting skills.*