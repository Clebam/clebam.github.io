---
layout: post
title:  "Mastering a Write-Log function"
categories: PowerShell
tags:  function
author: Clebam
series: Write-Log
---
* content
{:toc}

<!-- A short phrase of preview with a short text -->
### _Getting the most out of Write-Log_
From now on, we have created a practical Write-Log function that gets the job done.
However, we lack some PowerShell features like pipelining for instance. Also we made some concessions about the best practices, and it would be great to make things more reliable. Please be sure to give a look at the previous articles from the Write-Log series. 
![Image](/img/xxx.xxx){:width="360"} <!--Standard width: 300px-->

<!--End_Preview-->
>Note : From now on, I will consider you will be using a module, or at least dotsourcing, to have all these cmdlets. Of course you can paste the functions into each scripts, but it can be heavy just for a logging solution.

## Building the Initialize-Log cmdlet

First of all, I suggest to extract from the Write-Log function the whole part about file creation, clearing content and appending some init message. Thus, our Write-Log function will be dedicated to actually write the log and nothing else.

Here is just a bit of copy pasting from the former Write-Log function. I just made a few tweaks :
* I changed the `New-Item` condition to trigger every time the file does not exist (and not only if the parent folder does not exist.) This is because I'm not calling each time Out-File which would have created the file for me.
* I return the FilePath so that we can affect the value to a variable or pipeline it.
* I renamed `StartInfo` parameter to `IncludeHeader` which seems more logical to me.

So here is the Initialize-Log function :

```powershell
function Initialize-Log {
    [CmdletBinding()]
    Param
    (       
        [Parameter(Mandatory = $true)]
        [string] $FilePath,

        [switch]$Clear,

        [switch]$IncludeHeader
    )

    if (-not(Test-Path -Path $FilePath)) {
        New-Item -Path $FilePath -Force
    }

    if ($Clear) {
        Clear-Content -Path $FilePath
    }

    if ($IncludeHeader) {
        [string[]]$Header = @()
        $Header += "$("#" * 50)"
        $Header += "# Running script : $($MyInvocation.ScriptName)"
        $Header += "# Start time : $(Get-Date)"  
        $Header += "# Executing account : $([Security.Principal.WindowsIdentity]::GetCurrent().Name)"
        $Header += "# ComputerName : $env:COMPUTERNAME"            
        $Header += "$("#" * 50)"
        $Header | Out-File -FilePath $FilePath -Append
    }

    # I return the FilePath since the whole idea of the cmdlet is to create a logfile
    Write-Output $FilePath
}
```

On the other hand, the Write-Log function has been quite shrunked. But I added a test on `$FilePath` to ensure the file exists. I don't want the Write-Log function to mess with a file I did not initialize.

So for now, we have this Write-Log function

```powershell
function Write-Log {
    [CmdletBinding()]
    Param
    (       
        [Parameter(Mandatory = $true)]
        [ValidateScript({if (-not (Test-Path $_)) { throw "The file $_ does not exist. Use Initialize-Log to create it." } $true})]
        [string] $FilePath,

        [Parameter(Mandatory = $true)]
        [string] $Message,

        [ValidateSet("Info", "Warning", "Error")]
        $Level = "Info"
    )

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

## 2nd paragraph

* Taking advantage of pipelining
* Working with parameter sets
* Logging with mutex when multithreading