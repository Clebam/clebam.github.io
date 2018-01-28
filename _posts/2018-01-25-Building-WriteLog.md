---
layout: post
title:  "Building a Write-Log function"
categories: PowerShell
tags:  beginner function
author: Clebam
series: Write-Log
---
* content
{:toc}

### _Logging your scripts easily_ 

When a sysadmin creates scripts, he sometimes need to know what happened during the execution of the latter. A common way to do so is to log the actions perfomed by the script. Here is a sample of a Write-Log function usage. We will see how to build this Write-Log function. Sample below :

```powershell
$LogFile = "C:\Logs\Scripts.log"
$MyDir = Get-ChildItem -Path "C:\" -Directory
Write-Log -Path $LogFile -Message "There are $($MyFiles.Count) directories in C:\"
if (Test-Connection -ComputerName "MyServer" -Quiet -Count 1) {
    Write-Log -Path $LogFile -Message "MyServer responded."
}
else {
    Write-Log -Path $LogFile -Message "Can't reach my server." -Level Error
}
```

<!--End_Preview-->

## Logging without a Write-Log function

Let's say you have a short script that test if a folder exists. You may want to give some consistency to your logs, like giving a time stamp or an error level. Here is how you could go.

```powershell
# Log Init
$LogFile = "C:\Workspace\PSSandBox\Logs\Check.log"
"Check MyImportantFolder" | Out-File -FilePath $LogFile
"$(get-date) - Info - Beginning of the script." | Out-File -FilePath $LogFile -Append

# Logging ComputerName
"$(get-date) - Info - Running on Computer : $env:COMPUTERNAME" | Out-File -FilePath $LogFile -Append

# Testing if MyImportantFolder exists
if (Test-Path "C:\Workspace\PSSandBox\MyImportantFolder") {
    "$(get-date) - Info - MyImportantFolder exists." | Out-File -FilePath $LogFile -Append
}
else {
    "$(get-date) - Error - MyImportantFolder does not exist!" | Out-File -FilePath $LogFile -Append
}

# End of the script
"$(get-date) - Info - End of the script." | Out-File -FilePath $LogFile -Append
```
You can see that it is complicated to maintain since you have to maintain the string formatting each time you call the script. For example, each time you want to set a different error level (Info, Error or Warning for instance), you have to manually edit the string. Moreover, you won't have any help from [IntelliSense](https://msdn.microsoft.com/en-us/library/hcw1s69b.aspx){:target="_blank"}.  
You have some good reasons here to have the wish to use a Write-Log function.

## Building a Write-Log function

I suggest you to use VSCode to write your PowerShell code but for now, we'll go with the integrated Windows PowerShell ISE.  
One cool thing with ISE is that there are some snippets ready-to-use and one of them is a function template. Use `CTRL + J` to access the snippets and select `Cmdlet (advanced function)`

![ISESnippet](/img/ISESnippet.gif){:width="480"}

Many things are interesting in this function template. For now, let's focus on the function part.
We will set up a name using [approved verbs](https://msdn.microsoft.com/en-us/library/ms714428(v=vs.85).aspx){:target="_blank"}. Spoiler, it is going to be `Write-Log`.
In order to keep it simple, I will clean some parts of the code, like the advanced parameters and the begin\process\end template.

So we have three parts:
* The first one is the [Comment Based Help](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_comment_based_help?view=powershell-5.1){:target="_blank"}. We describe the function and provide some example. This is important for code maintaining and give some help to anyone who would use it, including your future you in 3 years. With this formatted block, you can also use `Get-Help Write-Log`.
* Then, we declare the parameters. Basically, you could just put variables with nothing else, but we have some options that are pretty useful. The `Mandatory` parameter allows to automatically check if the variable is empty. Thus you do not have to handle it in your code. The `[ValidateSet()]` parameter limits the inputs to a given set.
* Finally we have the code. There I use my parameters to create the wanted output. Here it is a string sent to a file.

I adapted the former code to use Write-Log instead.
Here is the result:

```powershell
<#
.Synopsis
   Performs a write to a log file.
.DESCRIPTION
   Performs a write to a log file.
.EXAMPLE
   Write-Log -Path "C:\Logs\MyScript.log" -Message "Everything is fine"
.EXAMPLE
   Write-Log -Path "C:\Logs\MyScript.log" -Message "Something went wrong!" -Level Error
#>
function Write-Log # You can collapse the function
{
    Param
    (
        [Parameter(Mandatory=$true)]
        [string] $FilePath,

        [Parameter(Mandatory=$true)]
        [string] $Message,

        [ValidateSet("Info","Warning","Error")]
        $Level = "Info"
    )

    $FormattedDate = Get-Date -Format "[yyyy-MM-dd][HH:mm:ss]"
    $OutString = "$FormattedDate - $Level - $Message"
    $OutString | Out-File -FilePath $FilePath -Append
}

# Log Init
$LogFile = "C:\Workspace\PSSandBox\Logs\Check.log"
New-Item -Path $LogFile -Force # Here we create a new log file with -Force parameter to overwrite the old on if it exists
Write-Log -FilePath $LogFile -Message "Check MyImportantFolder"
Write-Log -FilePath $LogFile -Message "Beginning of the script."

# Logging ComputerName
Write-Log -FilePath $LogFile -Message "Running on Computer : $env:COMPUTERNAME" -Level Warning

# Testing if MyImportantFolder exists
if (Test-Path "C:\Workspace\PSSandBox\MyImportantFolder") {
    Write-Log -FilePath $LogFile -Message "MyImportantFolder exists."
}
else {
    Write-Log -FilePath $LogFile -Message "MyImportantFolder does not exist!" -Level Error
}

# End of the script
Write-Log -FilePath $LogFile -Message "End of the script."
```

You can see the code is much easy to read now and even more easy to type with autocompletion and IntelliSense.

![IntelliSense](/img/WriteLogIntelliSense.gif)

Note the use of `[ValidateSet("Info","Warning","Error")]` to bind the inputs which can tabbed through then. Also, I did not set it to mandatory and gave it a default value, so that if you call the function without this parameter, it will be "Info" by default.


## Going further

This function is pretty useful but it lacks a few things to make it sustainable.
On the next part, we will add a few functionnalities:  
- A test on $FilePath to decide if we have to create it or to append.  
- An output to the Host to monitor the log directly on the console.  
- A premade init message.  
- Use snippets to get things even easier.

