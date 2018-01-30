---
layout: post
title:  "Improving a Write-Log function"
categories: PowerShell
tags: function
author: Clebam
series: Write-Log
---
* content
{:toc}

<!-- A short phrase of preview with a short text -->
### _More efficient, more resilient and handier_

In the second part of this Write-Log Series, we are going to work with some advanced parameter and take advantage of snippets to use more efficiently this function.
Such a function is going to be used in most of our scripts from now on, so we want it to be simple and easy to use.
![Description](/img/posts/Write-Log/Write-Log Snippet.gif){:width="600"}


<!--End_Preview-->

## Dealing with the log file
#### Testing if the log file exists
In the [previous post]({% post_url 2018-01-25-Building-WriteLog %}) we had to create _"manually"_ the log file. This was not mandatory because `Out-File` cmdlet would have created the file anyway, wouldn't it ?

In fact, if it depends it the folder path of the log file exists. That means if you try and the folder path does not exists you will have an error.
See below :

```powershell
# Log Init
$LogFile = "C:\Workspace\PSSandBox\Logs\NoFile.log"
$LogFail = "C:\Workspace\PSSandBox\Logs\NoDir\NoFile.log"

# Check if $LogFile and $LogFail exits, just to show you it doesn't
Write-Output "Does the file $LogFile exists ? : $(Test-Path -Path $LogFile)"
Write-Output "Does the file $LogFail exists ? : $(Test-Path -Path $LogFail)"

# Spoiler, here is why one will succeed and the other won't
Write-Output "Does the directory containing $LogFile exists ? : $(Test-Path -Path (Split-Path $LogFile))"
Write-Output "Does the directory containing $LogFail exists ? : $(Test-Path -Path (Split-Path $LogFail))"

# Logging ComputerName
Write-Log -FilePath $LogFile -Message "Hello Multiverse"
Write-Log -FilePath $LogFail -Message "Hello Multiverse"

# Check if the file exists again
Write-Output "Does the file $LogFile exists ? : $(Test-Path -Path $LogFile)"
Write-Output "Does the file $LogFail exists ? : $(Test-Path -Path $LogFail)"
```

![LogFailOutput](/img/posts/Write-Log/LogFailOutput.png){:width="600"}

It seems we should check if the log file exists and create it if it doesn't.

First I will show you aside how you could have done this if you didn't care about creating the file but only wanted to check if it can be created. This requires the use of parameter validation. We will use `[ValidateScript()]` to check if the folder path exists. If it does, the script goes on, if it doesn't, the script throws an error. This is better than testing it later inside the function and taking the risk for something else to be done.

```powershell
# Changing from this...
[Parameter(Mandatory = $true)]
[string] $FilePath

# ...to this
[Parameter(Mandatory = $true)]
[ValidateScript({Test-Path (Split-Path $_)})]
[string] $FilePath
```
![LogFailOutput2](/img/posts/Write-Log/LogFailOutput2.png){:width="720"}

You can see that the output is different from the previous one. This time, it is not the Out-File function that threw an error but our Write-Log function.

Let's get back to our need though. We want to create the file if it doesn't exist rather that throw and error. So we will handle it in the code by adding the following to the function. (The whole function will be at the end of the post)

```powershell
if (-not(Test-Path -Path(Split-Path $FilePath))) {
        New-Item $FilePath -Force
}
```

We use the `-Force` parameter to ensure that the cmdlet creates the whole needed directory tree.
Here is the output when you try to call the function with a file and its directory tree that does not exist.

![LogOutput3](/img/posts/Write-Log/LogFailOutput3.png){:width="600"}

The file, and the directory tree is created, and the log can be written.

#### Clearing the content
We check if our log file exists. This can lead to a tricky behavior : you may have an uncontrolled growth in log file size. Sure, you can back your logs up and rotate them. But for now, we will focus on having the choice to have a clean fresh log.
To do so, we will add a switch parameter to choose to clear the content or not.

```powershell
# We add the switch in param(). When the switch is not called in the function, its default value is $false.
[switch]$Clear

# If -Clear is use in the function it will clear the content.
if ($Clear) {
    Clear-Content -Path $FilePath
}

# Let's try this. I added a wait of 3 sec and line space for clarity purpose.
Write-Log -FilePath $LogFile -Message "Hello Multiverse - 1"
Get-Content $LogFile; Start-Sleep -s 3; ""
Write-Log -FilePath $LogFile -Message "Hello Multiverse - 2"
Get-Content $LogFile; Start-Sleep -s 3; ""
Write-Log -FilePath $LogFile -Message "Hello Multiverse - 3" -Clear
Get-Content $LogFile
```
![ClearContent](/img/posts/Write-Log/ClearContent.png){:width="480"}

We can see that the third time, the previous content was cleared.

## 2nd paragraph

