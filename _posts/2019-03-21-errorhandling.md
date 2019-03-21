---
layout: post
title: A quick intro to handlings errors with try and catch
subtitle: 
tags:
  - powershell
  - try
  - catch
  - finally
  - errorhandling
published: true
image: /img/ps.png
---

# Handling errors in Powershell using try and catch

## The basic structure of a try catch

Let's first try (eeyy) something we know will work, like a simple `Write-Host`

~~~powershell
function Test-TryCatch {
    Try {
        Write-Host "Hey mom look at me"
    }
    Catch {
        Write-Host $_
    }
}
~~~

If we run this code, it outputs `Hey mom look at me` like expected.

Now, let's try something we know will fail, like finding a service that doesn't exist with `Get-Service`.

~~~powershell
function Test-TryCatch-v2 {
    Try {
        Get-Service "thisisjustabogusservicedesignedtogivemeanerror" -ErrorAction Stop
    }
    Catch {
        Write-Host $_
    }
}
~~~

Let's first try running only the the line including `Get-Service` on it's own.

~~~powershell
C:\temp $ Get-Service "thisisjustabogusservicedesignedtogivemeanerror"

Get-Service : Cannot find any service with service name 'thisisjustabogusservicedesignedtogivemeanerror'.
At line:1 char:1
+ Get-Service "thisisjustabogusservicedesignedtogivemeanerror"
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (thisisjustabogu...togivemeanerror:String) [Get-Service], ServiceCommand
   Exception
    + FullyQualifiedErrorId : NoServiceFoundForGivenName,Microsoft.PowerShell.Commands.GetServiceCommand
~~~

This produces an error, but it's quite *heavy* and hard to read. Now, let's try running our function.

~~~powershell
C:\temp $ Test-TryCatch-v2
Cannot find any service with service name 'thisisjustabogusservicedesignedtogivemeanerror'.
~~~

As you can see, it outputs only the first line of the error, making it more readable. We can also create custom messages if we so desire. 

## Retrying an action

Ok, so we've got the basics of the try catch down, let's move onto handling known errors. I've chosen to work with the `xbgm` service

~~~powershell
function Test-TryCatch-v3 {
    #Define a service that exists on your system
    $service = "xbgm"
    Try {
        $service_status = (Get-Service $service -ErrorAction Stop).Status
        if($service_status -eq "Stopped"){
            Start-Service $service -ErrorAction Stop
        }
        elseif ($service_status -eq "Running") {
            Write-Host "Service $service is already running"
        }
    }
    Catch {
        Write-Host $_
    }
}
~~~

Now, if I run this it should fail (I've chosen a service I know will fail, so cheating a bit).

~~~powershell
C:\temp $ Test-TryCatch-v3
Service 'Xbox Game Monitoring (xbgm)' cannot be started due to the following error: Cannot open xbgm service on computer '.'.
~~~

So let's handle that error! If we know a scenario where our code may fail we can handle it gracefully. We can also see if we can retry it using a `do until`, but that's for another post.

This time we want the following:

* Handling known errors
* Handling of unknown errors
* Pause the script if we get unknown errors

Let's take our v3 function and upgrade it a bit!

~~~powershell
function Test-TryCatch-v4 {
    #Define a service that exists on your system
    $service = "xbgm"
    Try {
        $service_status = (Get-Service $service -ErrorAction Stop).Status
        if($service_status -eq "Stopped"){
            Start-Service $service -ErrorAction Stop
        }
        elseif ($service_status -eq "Running") {
            Write-Host "Service $service is already running"
        }
    }
    Catch {
        if ($_ -like "Service 'Xbox Game Monitoring (xbgm)' cannot be started due to the following error: Cannot open xbgm service on computer '.'.") {
            Write-Host "The chosen service cannot be started on this computer due to missing depencies - this is expected!"
        }
        else {
            Write-Host "ErrorMessage: $_"
            Pause
        }
    }
}
~~~

This way, if we run our function it should output our known error and let the user know what is happening and why!

~~~powershell
C:\temp $ Test-TryCatch-v4
The chosen service cannnot be started on this computer due to missing depencies - this is expected!
~~~

For the last part of the `try catch` we finally come to the last part, the `finally` block. This must be defined straight after the `catch` and runs no matter what happens - error or not!

So let's make that function into a v5!

~~~powershell
function Test-TryCatch-v5 {
    #Define a service that exists on your system
    $service = "xbgm"
    Try {
        $service_status = (Get-Service $service -ErrorAction Stop).Status
        if($service_status -eq "Stopped"){
            Start-Service $service -ErrorAction Stop
        }
        elseif ($service_status -eq "Running") {
            Write-Host "Service $service is already running"
        }
    }
    Catch {
        if ($_ -like "Service 'Xbox Game Monitoring (xbgm)' cannot be started due to the following error: Cannot open xbgm service on computer '.'.") {
            Write-Host "The chosen service cannot be started on this computer due to missing depencies - this is expected!"
        }
        else {
            Write-Host "ErrorMessage: $_"
            Pause
        }
    }
    Finally {
        #Show the user how long until the new Game of Thrones season
        [datetime]$Countdown="04/14/2019 03:00"
        $data = ($Countdown-(Get-Date))
        Write-Host "Script ran... but did you know that it's" $data.Days "days" $data.Hours "hours" $data.Minutes "minutes until the new Game of Thrones season??" -ForegroundColor DarkGreen
    }
}
~~~

And this is the output.

~~~powershell
C:\temp $ Test-TryCatch-v5
The chosen service cannot be started on this computer due to missing depencies - this is expected!
Script ran... but did you know that it's 23 days 16 hours 50 minutes until the new Game of Thrones season??
~~~

Using `try catch finally` you can build more robust functions and handle errors better.