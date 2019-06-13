---
layout: post
title: Building a function
subtitle: 
tags:
  - powershell
  - try
  - catch
  - finally
  - errorhandling
  - function
  - trap
published: true
image: /img/ps.png
---

# Building a function

So last time we tested out errorhandling in PowerShell using `try`, `catch` and `finally`.
This time we are building on that by introducing some more ways of handling errors and building a function.

Lets start out with a basic structure and explain the parts of it.

## The basic structure of a function

~~~powershell
$ErrorActionPreference = "Stop"
Trap {
        Write-Host "Error!" -ForegroundColor Red
        $Error [0]
        Write-host "Opening new powershell window to troubleshoot!"
        Start-Process Powershell.exe
        Pause
}
Function Example-Function {
    <#
        .SYNOPSIS
        Example function

        .DESCRIPTION
        Again, example function

        .EXAMPLE
                PS C:\> Example-Function -Dynamic "test"
                Does nothing, but it looks cool
            
        .PARAMETER Dynamic
            Example parameters
    
        .NOTES
            Author: infernux 29.05.2019
        #>

        [CmdletBinding()]
        PARAM(
        [string]$Email,
        [int]$Phone,
        [switch]$DeveloperMode
        )

        BEGIN {
                $RetrySeconds = 30
        }
        PROCESS {
                Do {
                        Try {
                                #We define our varibles
                                $Service = "DummyService"
                                $ServiceStatus = Get-Service $Service -ErrorAction SilentlyContinue
                                If($ServiceStatus.Status -eq "Running") {
                                        Stop-Service $Service
                                        Start-Sleep 5
                                        $ServiceStatus = Get-Service $Service -ErrorAction SilentlyContinue
                                        If ($ServiceStatus.Status -eq "Stopped") {
                                                $Retry = $false
                                        } Else {
                                                Retry = $true
                                        }
                                } Else {
                                        Write-Host "Service does not exist, no need to run script"
                                        $Retry = $false
                                }
                        } Catch {
                                If ($_ -match "something") {
                                        Write-Host "something"
                                        $Retry = $true
                                } Elseif ($_ -match "something_else") {
                                        Write-Host "$_"
                                        Write-Host "here I could potentially have some error handling"
                                        $Retry = $true
                                } Else {
                                        $ErrorLine = $_.InvocationInfo.ScriptLineNumber
                                        Write-Host "Error was $_"
                                        Write-Host "Error was on line $ErrorLine"
                                        $Retry = $true
                                        Pause
                                }
                        } Finally {
                                If ($Retry -eq $true) {
                                        Write-Host "retrying"
                                        $Var | Out-File var.txt
                                        Start-Sleep -Seconds $RetrySeconds
                                } Else {
                                        Write-Host "Done"
                                }
                        }
                } Until ($Retry -eq $False)
        }
}
~~~

Let's break it down into understandable pieces.

## ErrorActionPreference

`ErrorActionPreference` defines what you want to happen when a function throws an error. I usually have this set to stop, so I can troubleshoot and see if I can solve the problem, eventually building the problemsolving itself into my function.

## Trap

When PowerShell recieves a terminating error, depending on the errorhandling and action preference it might stop running the function in the current pipeline. A `trap` allows to specify a list of statements or actions to run whenever a fatal terminating error occurs. In our example function, it starts another window of PowerShell for you to troubleshoot in.

~~~powershell
Trap {
        Write-Host "Error!" -ForegroundColor Red
        $Error [0]
        Write-host "Opening new powershell window to troubleshoot!"
        Start-Process Powershell.exe
        Pause
}
~~~

We will get a little bit more into `try` and `catch` using different types later, but we can also do the same for the trap function. To only trap errors from the `[System.Management.Automation.CommandNotFoundException]` class, we can do something like this:

~~~powershell
Trap [System.Management.Automation.CommandNotFoundException] {
    Write-Host "something cool"
}
~~~

Obviously this is just scratching the surface, but you can read more about traps [here.](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_trap?view=powershell-6)

## Defining the function and parameters

At the start of the function itself we define the name of our function and what parameters we want it to take. We can also set the `cmdletbinding` which is a special attribute class that allows us to utilize more advanced script functions in PowerShell, such as the `-Confirm` flag.

~~~powershell
Function Example-Function {
        <#
        .SYNOPSIS
        Example function

        .DESCRIPTION
        Again, example function

        .EXAMPLE
                PS C:\> Example-Function -Dynamic "test"
                Does nothing, but it looks cool
            
        .PARAMETER Dynamic
            Example parameters
    
        .NOTES
            Author: infernux 29.05.2019
        #>
        [CmdletBinding()]
        PARAM (
                #SOMETHING
        )
~~~

Looking at this function it's clear that the cmdlet-name will be `Example-Function`. After defining the function we specify the `cmdletbinding` attribute (this needs to be first) and the parameters attribute, using `PARAM`.

An important takeaway here is that `PARAM` should be the first thing in your function, always, unless you are using `cmdletbinding`, in which case it should be second.

But, what about `.SYNPOSIS`, `.DESCRIPTION`, `.EXAMPLE`, `.PARAMETER` and `.NOTES`? Well, this is mostly for information purposes and help-text. It doesn't affect the function in any other way and as such is not needed, but welcome for the people using your functions later.

## Parameters

Going into parameters and parameter sets is something to almost be reserved for it's own post, but for more details on parameters and parametersets check out a great blogpost on that [here.](https://blog.simonw.se/powershell-functions-and-parameter-sets/)

For our simple use, consider the following from our script

~~~powershell
        [CmdletBinding()]
        PARAM(
        [string]$Email,
        [int]$Phone,
        [switch]$DeveloperMode
        )
~~~

This defines three parameters, none of which is mandatory. If we wanted to make one of the parameters mandatory, we could do something like

~~~powershell
[Parameter(Position=0,mandatory=$true)]
        [string] $aMandatoryParam
~~~

Now, for our simple needs let's imagine we have search function that lets us search for a users email based on the phone, and vice versa. Here I've defined the `email` parameter as a string - which means that whatever I pass to it will be in the variable `$email` for the entire time the script runs. Likewise `phone` is in a integer which will hold a number in the `$Phone` variable. I can add error-handling to the parameters themselves (define what length and format input should be in), or I can handle it using `if` in the script itself.

Lastly, we have a switch. This is basically just a `$true` or `$false`. If the switch `-developermode` mode is set, then the variable `$developermode` will be set to `$true`.

## BEGIN, PROCESS and END

Function input processing methods avaible in PowerShell, all three have different usecases.

`BEGIN` allows us to define optional one-time pre-processing for the function. This means that everytime the module or function is loaded, everything in the `BEGIN` block is used once per instance of the function in the pipeline.

`PROCESS` is the "doing" part of the script and is used on a record-to-record basis. This means everytime the function is called or something is looped through the function, this block runs.

`END` is the same as begin, just one-time post-processing for each instance of the function in the pipeline.

## Do, until death does us apart

Next up is the `do until` we have defined. This is basically our retry-function, we define a variable called `$retry` that is set to `$false` whenever a desired state is reached. If a desired state is not reached when running through the script, the variable is set to `true`.

Consider our example function, where we have dummy-service that we are trying to stop:

~~~powershell
Do {
        Try {
                #We define our varibles
                $Service = "DummyService"
                $ServiceStatus = Get-Service $Service -ErrorAction SilentlyContinue
                If($ServiceStatus.Status -eq "Running") {
                        Stop-Service $Service
                        Start-Sleep 5
                        $ServiceStatus = Get-Service $Service -ErrorAction SilentlyContinue
                        If ($ServiceStatus.Status -eq "Stopped") {
                                $Retry = $false
                        } Else {
                                $Retry = $true
                        }
                } Else {
                        Write-Host "Service does not exist, no need to run script"
                        $Retry = $false
                }
        } Catch {
                If ($_ -match "something") {
                        Write-Host "something"
                        $Retry = $true
                } Elseif ($_ -match "something_else") {
                        Write-Host "$_"
                        Write-Host "here I could potentially have some error handling"
                        $Retry = $true
                } Else {
                        $ErrorLine = $_.InvocationInfo.ScriptLineNumber
                        Write-Host "Error was $_"
                        Write-Host "Error was on line $ErrorLine"
                        $Retry = $true
                        Pause
                }
        } Finally {
                If ($Retry -eq $true) {
                        Write-Host "retrying"
                        $Var | Out-File var.txt
                        Start-Sleep -Seconds $RetrySeconds
                } Else {
                        Write-Host "Done"
                }
        }
} Until ($Retry -eq $False)
~~~

As we learned in the last post when we looked at `try`, `catch` and `finally` these blocks helps us try to run our code and will handle any errors we help them handle. In other words, this part will handle errors for us only as well as we can write them. If there's something that might be timing-based, such as a service starting slower on certain older hardware, a retry might be necesarry.

In our function we see that in the `finally` block, if `$retry` is set to true, we move on and try again. Everything inside the `do until` is looped until we reach a state where `$retry` is set to false.

Keep in mind this is a "dummy" function, it does nothing as it is now. The concept of this function is simple - if the service exists, stop it. If it's stopped when we check, good - we're done. If there's no service, we're done. If the service for some reason doesn't stop or there's an error, we either move to our `trap` or the function should loop.

### Disclaimer

I'm not a PowerShell guru, so there might be some errors in here. There's also no need to get bogged down by having all of the things I've mentioned above in your function, it will work well with only `PARAM` and `try catch`. The moral is to write good scripts that handle unexpected situtations, that are easily readable and can be expanded on to include errorhandling and the likes. If you spot anything that's wrong, let me know!
