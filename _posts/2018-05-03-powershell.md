---
layout: post
title: Reading SecureString credentials as cleartext
subtitle: this probably shouldn't be possible
tags:
  - powershell
  - getnetworkcredential
published: true
---

Let's say I want to make a script in which the user has to input credentials. Oh, there's a cmdlet for that? Great.

~~~
PS C:\> $totallysecurecredentials = Get-Credential
~~~

This gives us this prompt:

![prompt](http://www.powershellmagazine.com/wp-content/uploads/2013/02/Get-Credential.png)

So what happens here is that the Get-Credential cmdlet retrieves a PSCredentialObject. 
We can use the Get-Member cmdlet to look at what that entails:

~~~
PS C:\> $totallysecurecredentials | Get-Member 

   TypeName: System.Management.Automation.PSCredential

Name                 MemberType Definition

----                 ---------- ----------

Equals               Method     bool Equals(System.Object obj)
GetHashCode          Method     int GetHashCode()
GetNetworkCredential Method     System.Net.NetworkCredential GetNetworkCredential()
GetObjectData        Method     void GetObjectData(System.Runtime.Serialization.S...
GetType              Method     type GetType()
ToString             Method     string ToString()
Password             Property   securestring Password {get;}
UserName             Property   string UserName {get;}
~~~

Trying to read the password with the .password method gives us the following:

~~~
PS C:\> $totallysecurecredentials.Password
System.Security.SecureString
~~~

However, if I use the .GetNetworkCredential-method, the PSCredentialObject turns into a NetworkCredential. Why is this important? Well, first of all this is a _feature_, believe it or not. PSCredentialObjects only work with other cmdlets or functions that know how to interpret the objects. Converting it to a more generic NetworkCredential-object allows us to work with .NET and other functions with no native PSCredentialObject support. 

Now, let's try Get-Member with the GetNetworkCredential method:

~~~
PS C:\> $totallysecurecredentials.GetNetworkCredential() | Get-Member 

   TypeName: System.Net.NetworkCredential

 

Name           MemberType Definition

----           ---------- ----------

Equals         Method     bool Equals(System.Object obj)
GetCredential  Method     System.Net.NetworkCredential GetCredential(uri uri, str...
GetHashCode    Method     int GetHashCode()
GetType        Method     type GetType()
ToString       Method     string ToString()
Domain         Property   string Domain {get;set;}
** Password       Property   string Password {get;set;}
SecurePassword Property   securestring SecurePassword {get;set;}
UserName       Property   string UserName {get;set;}
~~~

Excuse me? SecurePassword still shows up as a securestring, but our Password now is a plain string? What happens if we try to play with that?

~~~
PS C:\> $totallysecurecredentials.GetNetworkCredential().password
123456
~~~

Oh. Okay. So don't parse credentials in PowerShell using Get-Credential kids, it's dangerous.
