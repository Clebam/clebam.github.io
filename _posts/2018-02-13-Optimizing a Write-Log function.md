---
layout: post
title:  "Optimizing a Write-Log function"
categories: PowerShell
tags:  function reddit
series : Write-Log
author: Clebam
---
* content
{:toc}

<!-- A short phrase of preview with a short text -->
### _Learning from others_
When I posted my Write-Log function post on reddit, I was glad to share it. In return, I have been given many good advices that I want you to know.
Thanks to my fellow powershellers from /r/powershell.
As promised, I also added a way to back the logs up and even a Send-Log function to mail the log if needed.

![Image](/img/posts/Write-Log/reddit.png){:width="360"} <!--Standard width: 300px-->

<!--End_Preview-->

## Refactoring Initialize-Log

There were a few things that were not ideal in this function. I will list them and show what I changed.

* **Function Parameters**

    I have been adviced that it was not necessary to add `= $true` to each statement of parameters, so I removed them.  
    I also reconsidered using Pipeline on Filepath as I think there are no valid use case. So I dropped the parameter.

    _Before_
    ```powershell
    [Parameter(Mandatory = $true, ValueFromPipeline = $true, ParameterSetName = "FilePath")]
    [ValidateNotNullOrEmpty()]
    [System.IO.FileInfo]$FilePath
    ```

    _After_
    ```powershell
    [Parameter(Mandatory)]
    [ValidateNotNullOrEmpty()]
    [System.IO.FileInfo]$FilePath,
    ```

* **MutexCall**

    I changed the way mutex are called. I set a default name in each involved function rather than having to set a name at each call. So, I also changed the type from `[string]` to `[switch]`. I also removed the `CloseMutex` parameter, and therefore the ParameterSetName was also removed.  
    Another detail is that using `| Out-Null` is less optimized that using `[void]` or  `$null =` . Given the Log functions, especially Write-Log, might be called dozens of time in a script, it was not superfluous.

    _Before_  
    ```powershell
    # In params
    [Parameter(ParameterSetName = "FilePath")]
    [ValidateNotNullOrEmpty()]
    [string]$NewMutex,

    # In code
    if ($NewMutex) {
        If ( -not ([System.Threading.Mutex]::TryOpenExisting($NewMutex, [ref]$null))) {
            $Mutex = New-Object System.Threading.Mutex($false, $NewMutex)
            $Mutex.WaitOne() | Out-Null
            $Mutex.ReleaseMutex()
        }    
    }
    ```

    _After_
    ```powershell
    # In params
    [switch]$UseMutex

    # In code
    if ($UseMutex) {
        if ( -not ([System.Threading.Mutex]::TryOpenExisting("LogMutex", [ref]$null))) {
            $Mutex = New-Object System.Threading.Mutex($false, "LogMutex")
            [void]$Mutex.WaitOne()
            [void]$Mutex.ReleaseMutex()
        }    
    }
    ```

* **Using LiteralPath**

    I struggle on this one. (Not for this function, but you will see below.)
    So I wanted to deal with files like _MyLog[].log_. This files would have generated and error with the `*-Path` for instance because `[` and `]` are common regex pattern characters.

    With the parameter LiteralPath, I was sure that no regex would be performed and that the name would be kept.

    _Before_
    ```powershell
    if (-not(Test-Path -Path $FilePath)) {
        New-Item -Path $FilePath -Force | Out-Null
    }
    ```

    _After_
    ```powershell
    if (-not(Test-Path -LiteralPath $FilePath)) {
        $null = New-Item -Path $FilePath -Force
    }
    # Note the replacement of "| Out-Null" also.
    ```

* **Replacing Clear-Content**

    With my test, I used glogg. This program seems to lock the file it opens. So `Clear-Content` threw _The process cannot access the file_ errors. So I decided to use `Out-File` with the `-Force` parameter.

    _Before_
    ```powershell
    if ($Clear) {
        Clear-Content -Path $FilePath
    }
    ```

    _After_
    ```powershell
    if ($Clear) {
        $null | Out-File -LiteralPath $FilePath -Force -Encoding utf8
    }
    ```

* **Header formatting**

    I changed the Header construction to something more conventional. In fact, I rebuilded the whole Log functions to output csv-like log. (tab separated lines)

    I was also told to prefer _Here-String_ to array for multiline string. This make sense, but I don't like the indenting issues with _Here-String_ and given we call it only once per script, it is not a big issue.

    _Before_
    ```powershell
    if ($IncludeHeader) {
        [string[]]$Header = @(
            "$("#" * 50)"
            "# Running script : $($MyInvocation.ScriptName)"
            "# Start time : $(Get-Date -Format "F")"  
            "# Executing account : $([Security.Principal.WindowsIdentity]::GetCurrent().Name)"
            "# ComputerName : $env:COMPUTERNAME"            
            "$("#" * 50)"
        )           
        $Header | Out-File -FilePath $FilePath -Append
    } 
    ```

    _After_
    ```powershell
    if ($IncludeHeader) {
        $Now = [System.DateTime]::Now
        $FormattedDate = $Now.ToString('s')
        $FullDate = $Now.ToString('F')

        [string[]]$Header = @(
            "{0}`t{1}`t{2}" -f $FormattedDate, "Info", "Running script : $($MyInvocation.ScriptName)"                    
            "{0}`t{1}`t{2}" -f $FormattedDate, "Info", "Start time : $FullDate"  
            "{0}`t{1}`t{2}" -f $FormattedDate, "Info", "Executing account : $([Security.Principal.WindowsIdentity]::GetCurrent().Name)"
            "{0}`t{1}`t{2}" -f $FormattedDate, "Info", "ComputerName : $env:COMPUTERNAME"            
        )

        $Header | Out-File -LiteralPath $FilePath -Append -Encoding utf8
    }   
    ```

* **Added the possibility to set *-Log function**

    With this one, I am a bit puzzled. I find it very handy and useful, but I have to set a `$global` variable (_$script_ scope did not work as intended). However, through testing, I did not find any issues. But this should be used with caution.

    These parameters allow to set default values, script wide, to parameters. Thus, you can omit the `-FilePath` parameter when you call `Write-Log` for instance.

    _After_
    ```powershell
    # In param
    [ValidateSet("FilePath", "UseMutex", "All")]
    [string]$ForceGlobalLogCmdletParameter,

    [switch]$ForceGlobalLogCmdletPassthru,

    # In code
    switch ($ForceGlobalLogCmdletParameter) {
        
        {$true} {
            Write-Verbose -Message "You forced DefaultParameter of bound *-Log function. These are global scope variables. Use with caution!" 
        }
        "FilePath" {
            $global:PSDefaultParameterValues['Write-Log:FilePath'] = $FilePath.FullName
            $global:PSDefaultParameterValues['Send-Log:FilePath'] = $FilePath.FullName
            $global:PSDefaultParameterValues['Complete-Log:FilePath'] = $FilePath.FullName
            break
        }
        "Mutex" {
            $global:PSDefaultParameterValues['Write-Log:UseMutex'] = $true
            break
            
        }
        "All" {
            $global:PSDefaultParameterValues['Write-Log:UseMutex'] = $true
            $global:PSDefaultParameterValues['Write-Log:FilePath'] = $FilePath.FullName
            $global:PSDefaultParameterValues['Send-Log:FilePath'] = $FilePath.FullName
            $global:PSDefaultParameterValues['Complete-Log:FilePath'] = $FilePath.FullName
            break
        }            
        Default {break}
    }

    if ($ForceGlobalLogCmdletPassthru) {
        $global:PSDefaultParameterValues['Write-Log:Passthru'] = $true  
        $global:PSDefaultParameterValues['Write-Log:InformationAction'] = "Continue"
        $global:PSDefaultParameterValues['Write-Log:WarningAction'] = "Continue"
        $global:PSDefaultParameterValues['Write-Log:ErrorAction'] = "Continue"
    }
    ```

## Refactoring Write-Log

There were many advices given for this function. Some of them were treated above, so I won't repeat myself, but Write-Log has been updated the same way.

Let's take a look at the others.

* **Mutex**

    I tried at first to remove the Mutex parameter from the function and let it handle with some if statements. Like `if ([System.Threading.Mutex]::TryOpenExisting("LogMutex", [ref]$null))` but when several threads try to call this, it tends to fail for whatever reason. So I found it was more reliable to use a switch parameter. No name is needed as I defined a default one.

    _Before_
    ```powershell
    # In params
    [string]$MutexName
    # In code
    if ($MutexName) {
        $Mutex = New-Object System.Threading.Mutex($false, $MutexName)
        $Mutex.WaitOne() | Out-Null
    }
    ```

    _After_
    ```powershell
    # In params
    [switch]$UseMutex
    # In code
    if ($UseMutex) {
        try {
            $Mutex = New-Object System.Threading.Mutex($false, "LogMutex")
            [void]$Mutex.WaitOne()
        }
        catch [System.Threading.AbandonedMutexException] {
            # It may happen if a Mutex is not released correctly, but it will still get the Mutex.
        }
    }
    ```

* **Overall log formatting**

    I also changed the formatting of output to tab separated lines. This way, it is parsable as CSV and still readable by a human.

    I also chose to use a conventional datetime format.

    (Example in the next paragraph)

* **Using Add-Content over Out-File ?**

    I was told to use Add-Content and tried to do so. But I realised that it need the file to not be locked. And like `Clear-Content` above, it lead file access error.

    Moreover, the main problem was the way output were treated since `Add-Content` transforms the input to string while `Out-File` just outputs the objects. (Which can lead to strange behaviors, like truncated results) But in my function, every object is transformed to string before going to `Out-File` so there is not worries about that.

    However, I changed the formatting to UTF8 rather than UTF16, since it was not needed and is more standard in the end.

    _Before_
    ```powershell
    $FormattedDate = Get-Date -Format "[yyyy-MM-dd][HH:mm:ss]"
    $OutString = "$FormattedDate - $Level - $Message"
    $OutString | Out-File -FilePath $FilePath -Append
    ```

    _After_
    ```powershell
    $FormattedDate = [System.DateTime]::Now.ToString('s')
    $OutString = "{0}`t{1}`t{2}" -f $FormattedDate, $Level, $Message
    $OutString | Out-File -LiteralPath $FilePath -Append -Encoding utf8
    ```

* **Apologies for Write-Host**

    Ok... This was a bad idea to use Write-Host. I learned. So I redesigned the output with standard cmdlet (beware, Write-Information requires PSv5)

    Also, it relies on the `$InformationAction`, `$WarningAction` and `$ErrorAction` script setting by default. So if they are set to SilentlyContinue, nothing will ever print to the screen. That's why I force them is `Initialize-Log`.

    I also wrapped it in a switch parameter `Passthru` to have the possibility not to use it.

    _Before_
    ```powershell
    switch ($Level) {
        "Info" { Write-Host $OutString; break }
        "Warning" { Write-Host $OutString -ForegroundColor Yellow; break }
        "Error" { Write-Host $OutString -ForegroundColor Red; break }
        Default { Write-Host $OutString; break }
    }  
    ```

    _After_
    ```powershell
    if ($Passthru) {
        switch ($Level) {
            "Info" { Write-Information $OutString; break }
            "Warning" { Write-Warning $OutString; break }
            "Error" { Write-Error $OutString; break }
        }    
    }
    ```

## Building Send-Log

A user shared his github PowerShell Logging Module. You should take a look, it is very interesting too. [See repository](https://github.com/EsOsO/Logging){:target="_blank"}

This gave me the idea to send the log via mail so I built a simple Send-Log command. This cmdlet works because I formatted the log in a specific way. It needs the log to be csv parsable and to have all the ErrorLevels listed.

The cmdlet parameters requires the Filepath and a MinLevel. MinLevel is the Level at which your log file will be sent via mail. Say you specify MinLevel = "Error", you will only receive a mail if an error is in the log. To do this, I use the Level column when importing as a csv. Hence the need to have a csv parsable log file.

Of course the mail parameters is to be set at least once. I put the password in clear text, but you should do this another way.

>Note : Remember when I said I struggle with fancy filenames containing regex character ? Well, the Send-MailMessage -Attachments does not have a way to deal with this, so I used .NET objects.

Depending on which Level you set in Write-Log function, be sure to list all of them over here.

```powershell
# In params
[ValidateSet("Info", "Warning", "Error")]
[string]$MinLevel = "Error"
# In code
switch ($MinLevel) {
    "Info" {$Level = "Info", "Warning", "Error"; break}
    "Warning" {$Level = "Warning", "Error"; break}
    "Error" {$Level = "Error"; break}
}
```

The full function will be available at the end of the post.

## Building Complete-Log

I think I spent more time finding the right approved verb that anything else with this function. Feel free to use another, but I did not feel like _Rotate-Log_ or _Backup-Log_. One is not an approved verb, the other is not really on purpose in the end, so let's stick with Complete.

So `Complete-Log` cmdlet is mainly used to make a log rotation, but it also close the mutex, if ever needed. (In fact, it is not mandatory since the mutex will be killed once the current PowerShell process is terminated, for instance, when the script ends if not launched from an interactive session.)

On the log rotation part, I think there are many ways to do this. I sticked to a relatively simple one.

There are 2 triggers :
* MaxFileSize : If the file reaches this size, it rotates. (You can use powershell [int] size like `-MaxFileSize 10MB`)
* MaxFileAge : If the file is older that the timespan, it rotates. (Here, I use a Timespan, so use New-TimeSpan when call the function.)

And there is a limit to the number of rotation. If there are more logs than the limit, the older one is removed.

You can set an ArchivesPath, otherwise it will be the current log file path by default.

I chose random default value, this should be tweaked according to your need.

I rename the filename with a formatted date, so if we look in the explorer, they will be order from the oldest to the newest.

>Note : To remove the oldest file, I use the name instead of CreationDate because sometimes this value might be wrong. (If the file was copied without keeping metadata for instance.) But this depends on your naming convention, so beware.

>Note : Remember when I said I struggle with fancy filename ? I had problems with this cmdlet too. I had to use this _Where-Object Name -Match ([regex]::escape($CurrentLog.BaseName.ToString()))_ because the _-Match_ parameter triggers with Regex.

The full function will be available at the end of the post.

## Building a Log module
So now we have 4 cmdlets to play with. The best way to keep them available easily is to wrap them up in a powershell module.

I won't go into the details of powershell module but here is a very [complete post](https://kevinmarquette.github.io/2017-05-27-Powershell-module-building-basics/){:target="_blank"} from Kevin Marquette.

Create a .psm1 file and dot source the functions. Here I called it Bamshell.Logging.psm1

```powershell
. $PSScriptRoot\Public\Initialize-Log.ps1
. $PSScriptRoot\Public\Write-Log.ps1
. $PSScriptRoot\Public\Complete-Log.ps1
. $PSScriptRoot\Public\Send-Log.ps1
```

Then in your script, you can use either :

```powershell
Import-Module "C:\Workspace\MyModules\Bamshell.Logging.psm1" -Force
```

Or if you create a right module manifest and put the module in one of the `$env:PSModulePath` you can do this.

```powershell
#Requires -Modules Bamshell.Logging
```

## Some examples
With all of the new parameters added, you can see below some working code examples

* **Taking advantage of PSDefaultParameterValues**

    ```powershell
    Import-Module Bamshell.Logging -Force

    # See the filename with Regex chars.
    Initialize-Log -FilePath C:\Temp\Script[].log -Clear -IncludeHeader -ForceGlobalLogCmdletParameter FilePath -ForceGlobalLogCmdletPassthru

    # I never call FilePath parameter
    Write-Log "Ok"
    Write-Log "An error occured" -Level Error
    Write-Log "Something is not really correct." -Level Warning

    Send-Log

    Complete-Log
    ```

* **Easier Mutex**

    ```powershell
    Import-module Bamshell.Logging -Force
    Initialize-Log -FilePath "C:\Temp\ParallelJobs.log" -IncludeHeader -Clear -UseMutex

    $ScriptBlock = {
        param (
            $Path
        )
        Import-module Bamshell.Logging -Force
        $ChildItem = Get-Childitem -Path $Path
        $ChildItem.FullName | Write-Log -FilePath "C:\Temp\ParallelJobs.log" -UseMutex
    }

    (Get-ChildItem 'C:\Program Files' -Directory).FullName | ForEach-Object {
        Start-Job -ScriptBlock $ScriptBlock -ArgumentList $_
    }

    Get-job | Wait-Job | Receive-Job
    ```

## Full functions

Once again, I put them on gist. They might be updated and be slightly different from code above.

<script src="https://gist.github.com/Clebam/ad7a92d5695df9c845687f0c223111ae.js"></script>

## End of this Write-Log series

I am glad that I have been given so many advice. A huge thanks to [/r/PowerShell](https://www.reddit.com/r/PowerShell/){:target="_blank"}

Especially to  [Lee\_Daily](https://www.reddit.com/user/Lee_dailey){:target="_blank"}, [Ta11ow](https://www.reddit.com/user/Ta11ow){:target="_blank"} and [da\_chicken](https://www.reddit.com/user/da_chicken){:target="_blank"}

They all put some time to analyse and give some advice. Thanks!

I think this time the series is over since we covered the main aspect of writing logs and handling files.