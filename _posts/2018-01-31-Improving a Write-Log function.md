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
![Preview](/img/posts/Write-Log/Write-Log Snippet.gif){:width="600"}


<!--End_Preview-->

## Dealing with the log file
#### Testing if the log file exists
In the [previous post]({% post_url 2018-01-25-Building-WriteLog %}) we had to create _"manually"_ the log file. This was not mandatory because `Out-File` cmdlet would have created the file anyway, wouldn't it ?

In fact, if it depends if the folder path of the log file exists. That means if you try and the folder path does not exist, you will have an error.
See below :

```powershell
# Log Init
$LogFile = "C:\Workspace\PSSandBox\Logs\NoFile.log"
$LogFail = "C:\Workspace\PSSandBox\Logs\NoDir\NoFile.log"

# Check if $LogFile and $LogFail exits, just to show you it doesn't
Write-Output "Does the file $LogFile exist? : $(Test-Path -Path $LogFile)"
Write-Output "Does the file $LogFail exist? : $(Test-Path -Path $LogFail)"
""
# Spoiler, here is why one will succeed and the other won't
Write-Output "Does the directory containing $LogFile exist? : $(Test-Path -Path (Split-Path $LogFile))"
Write-Output "Does the directory containing $LogFail exist? : $(Test-Path -Path (Split-Path $LogFail))"
""
# Logging ComputerName
Write-Log -FilePath $LogFile -Message "Hello Multiverse"
Write-Log -FilePath $LogFail -Message "Hello Multiverse"
""
# Check if the file exists again
Write-Output "Does the file $LogFile exist? : $(Test-Path -Path $LogFile)"
Write-Output "Does the file $LogFail exist? : $(Test-Path -Path $LogFail)"
```

![LogFailOutput](/img/posts/Write-Log/LogFailOutput.png){:width="600"}

It seems we should check if the log file exists and create it if it doesn't.

First I will show you aside how we could have done this if we didn't care about creating the file but only wanted to check if it can be created. This requires the use of parameter validation. We will use `[ValidateScript()]` to check if the folder path exists. If it does, the script goes on, if it doesn't, the script throws an error. This is better than testing it later inside the function and taking the risk for something else to be done.

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

Given the fact that we want to work on an existing file, we can have a tricky behavior : we may have an uncontrolled growth in log file size. Sure, we can back our logs up and rotate them. But for now, we will focus on having the choice to have a clean fresh log when needed.
To do so, we will add a switch parameter to choose to clear the content or not.

```powershell
# We add the switch in param(). When the switch is not called in the function, its default value is $false.
[switch]$Clear

# If -Clear is used in the function, it will clear the content of the log file.
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

> Note : I prefered to add a -Clear parameter rather than a -NoClobber because it seems to me that I will more often call the function to append the log file rather than to overwrite it. Thus, it is a optional parameter I won't have to call each time I use the function.

## Starting with some infos

There are some informations you may always want to see in your scripts. For instance, you may want to know the name of the computer that runs the scripts. Even more important, you will definitely want to know what account is used during the execution of the script. So we are going to add another switch parameter to have an -StartInfo parameter.

```powershell
# We add the -StartInfo switch to have a boolean
[switch]$StartInfo

# If -StartInfo is used, I'll set a multiline content
if ($StartInfo) {

    [string[]]$StartText = @()
    $StartText += "$("#" * 50)"
    $StartText += "# Running script : $($MyInvocation.ScriptName)"
    $StartText += "# Start time : $(Get-Date)"  
    $StartText += "# Executing account : $([Security.Principal.WindowsIdentity]::GetCurrent().Name)"
    $StartText += "# ComputerName : $env:COMPUTERNAME"
    $StartText += "$("#" * 50)"
    $StartText | Out-File -FilePath $FilePath -Append
}

# Let's try this
Write-Log -FilePath $LogFile -Clear -StartInfo -Message "The script started."
Write-Log -FilePath $LogFile -Message "Everything seems alright."
Write-Log -FilePath $LogFile -Message "Everything is going wrong!" -Level Error
```

![StartInfoResult](/img/posts/Write-Log/StartInfoResult.png){:width="480"}

And here we have a nice header to our log.


>Note : This part shouldn't be consider as best practice because a cmdlet should be designed to do only what it is made for. Here it is `Write-Log`. This part should be extracted from the cmdlet and put inside the script or another cmdlet like `Initialize-Log` which would also handle the existence and clearing of the file. It would be a more conventional manner to do so.

## Logging to the host.

When you are developping or debugging a script, it might be annoying to always find the log file and open it. So a good solution is to duplicate the log to the host. We just need to add some `Write-Host` to the cmdlet.

```powershell
# We add a switch to determine which level it is and adapt the Write-Host foreground color.
switch ($Level) {
    "Info" { Write-Host $OutString; break }
    "Warning" { Write-Host $OutString -ForegroundColor Yellow; break }
    "Error" { Write-Host $OutString -ForegroundColor Red; break }
    Default { Write-Host $OutString; break }
}

# Let's try this
Write-Log -FilePath $LogFile -Message "Everything seems alright."
Write-Log -FilePath $LogFile -Message "Something seems wrong!" -Level Warning
Write-Log -FilePath $LogFile -Message "Everything is going wrong!" -Level Error
```
![WriteHostOutput](/img/posts/Write-Log/WriteHostResult.png){:width="600"}

## Faster with snippets

In my opinion, the `Write-Log` function is one of the cmdlets we are going to use a lot in our scripts, so we are going to type it again and again. This is a good call to create a snippet. Please have a look at [my post on VSCode]({% post_url 2018-01-28-VSCode %}){:target="_blank"} to see how to create one.

Here is the VSCode snippet code :

```
"Call the Write-Log function": {
	"prefix": "&wr",
	"body": [
		"Write-Log -FilePath ${1:\\$LogFile} -Message \"$2\"$3"
	],
	"description": "Creates a process with output"
}
```
The downside is that you should always ensure that the path to the log file is stored in a `$LogFile` variable. But this makes sense, so it's just a rule to apply. Just in case, I placed a placeholder on this parameter to set it.  
The second placeholder is used to type the actual message string.  
The third is put to let you choose either to add another optional parameter or to go to the next line.

>Note : I like to use a `&` prefix to have my snippet sorted when I need them with IntelliSense. But you can just use `wr` if you want.

Here is a comparison of calling the function with or without the snippet.

![WithoutSnippet](/img/posts/Write-Log/WithoutSnippet.gif){:width="600"}

![WithSnippet](/img/posts/Write-Log/WithSnippet.gif){:width="600"}

## The full function

Here is the full function with all the things we have seen so far.

```powershell
function Write-Log {
    [CmdletBinding()]
    Param
    (       
        [Parameter(Mandatory = $true)]
        [string] $FilePath,

        [Parameter(Mandatory = $true)]
        [string] $Message,

        [ValidateSet("Info", "Warning", "Error")]
        $Level = "Info",

        [switch]$Clear,

        [switch]$StartInfo
    )

    if (-not(Test-Path -Path(Split-Path $FilePath))) {
        New-Item -Path $FilePath -Force
    }

    if ($Clear) {
        Clear-Content -Path $FilePath
    }

    if ($StartInfo) {
        [string[]]$StartText = @()
        $StartText += "$("#" * 50)"
        $StartText += "# Running script : $($MyInvocation.ScriptName)"
        $StartText += "# Start time : $(Get-Date)"  
        $StartText += "# Executing account : $([Security.Principal.WindowsIdentity]::GetCurrent().Name)"
        $StartText += "# ComputerName : $env:COMPUTERNAME"
        $StartText += "$("#" * 50)"
        $StartText | Out-File -FilePath $FilePath -Append
    }

    $FormattedDate = Get-Date -Format "[yyyy-MM-dd][HH:mm:ss]"
    $OutString = "$FormattedDate - $Level - $Message"
    $OutString | Out-File -FilePath $FilePath -Append

    switch ($Level) {
        "Info" { Write-Host $OutString; break }
        "Warning" { Write-Host $OutString -ForegroundColor Yellow; break }
        "Error" { Write-Host $OutString -ForegroundColor Red; break }
        Default { Write-Host $OutString; break }
    }
    
}
```

## Going even further ?

On the next part, I will show you how to make this function overpowered.
* Taking advantage of pipelining
* Working with parameter sets
* Logging with mutex when multithreading