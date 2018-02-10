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
However, we lack some PowerShell features like pipelining for instance. Also we made some concessions about the best practices, and it would be great to make things more reliable. We will also see how to handle parallel jobs.  
 This is the 3rd part of the Write-Log series, so make sure to give a look at the previous articles.

![Image](/img/posts/Write-Log/LogPicture.png){:width="480"} <!--Standard width: 360px-->

<!--End_Preview-->
>Note : From here on, I will consider you will be using a module, or at least dotsourcing, to have all these cmdlets. Of course you can paste the functions into each scripts, but it can be heavy just for a logging solution.

## Building the Initialize-Log cmdlet

First of all, I suggest to extract from the Write-Log function the whole part about file creation, clearing content and appending some init message. This way, our Write-Log function will be dedicated to actually write the log and nothing else.

It just needs a bit of copy pasting from the former Write-Log function. I only made a few tweaks :
* I added `[ValidateNotNullOrEmpty()]` to the FilePath parameter to avoid some errors.
* I changed the `New-Item` condition to trigger every time the file does not exist (and not only if the parent folder does not exist.) This is because I'm not calling each time Out-File which would have created the file for me. I also piped it to `Out-Null` in order to not return unwanted objects.
* I added a Passthru parameter to return the FilePath so that we can affect the value to a variable.
* I renamed `StartInfo` parameter to `IncludeHeader` which is a better parameter name to me.

So here is the Initialize-Log function :

```powershell
function Initialize-Log {
    [CmdletBinding()]
    Param
    (       
        [Parameter(Mandatory = $true)]
        [ValidateNotNullOrEmpty()]
        [string]$FilePath,

        [switch]$Clear,

        [switch]$IncludeHeader,

        [switch]$Passthru
    )

    if (-not(Test-Path -Path $FilePath)) {
        New-Item -Path $FilePath -Force | Out-Null
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
    if ($Passthru) {
        Write-Output $FilePath
    }  
}
```

On the other hand, the Write-Log function has been quite shrunked. But I added a test on `$FilePath` to ensure the file exists. I don't want the Write-Log function to mess with a file I did not initialize.

So for now, we have this Write-Log function : 

```powershell
function Write-Log {
    [CmdletBinding()]
    Param
    (       
        [Parameter(Mandatory = $true)]
        [ValidateScript({
            if (-not (Test-Path $_)) {
                throw "The file $_ does not exist. Use Initialize-Log to create it." 
            } 
            $true
        })]
        [string]$FilePath,

        [Parameter(Mandatory = $true)]
        [string]$Message,

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

## Taking advantage of pipelining
Now that we have properly set our functions, let's make both of them pipeline compliant.

For the `Initialize-Log` function, we would provide a file item or a path so we are going to add pipelining to the `$Filepath` variable. In this case, I do not put the body inside a `Process` block as far as I don't see any reason to give multiple path as an input. So keep in mind, that the pipeline will go straight to (implied) `End` block.

So we just have to update `$FilePath` parameter by adding `ValueFromPipeline = $true` :

```powershell
# Inside Initialize-Log function
[Parameter(Mandatory = $true, ValueFromPipeline = $true)]
[ValidateNotNullOrEmpty()]
[string]$FilePath,
```
On the `Write-Log` function, we could provide both a path or a message through pipeline. However, it seems unlikely that we could provide both. So we can keep it simple and simply add `ValueFromPipeline = $true` to the proper parameters :

```powershell
# Inside Write-Log function
[Parameter(Mandatory = $true, ValueFromPipeline = $true)]
[ValidateScript({
    if (-not (Test-Path $_)) {
        throw "The file $_ does not exist. Use Initialize-Log to create it." 
    } 
    $true
})]
[string]$FilePath,

[Parameter(Mandatory = $true, ValueFromPipeline = $true)]
[string]$Message,
```

Unlike the `Initialize-Log` function, it is very likely that the `Write-Log` function receives many string at once as an input. So we have to handle those correctly by putting the code in a `Process` block. If it was not executed in a process block, only the last item of the pipeline will be treated.

```powershell
# Inside Write-Log function
Process {
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

With such a code, we can call the function through pipelining and this become pretty effective

```powershell
# An elegant way to start logging
$LogFile = Initialize-Log -FilePath "C:\Temp\Script.log" -IncludeHeader -Clear -Passthru
Write-Log -FilePath $LogFile -Message "Hello Multiverse"
Get-ChildItem -Path C:\Temp -Recurse | Write-Log -FilePath $LogFile

# Just because we can (even if not very useful in this case)
New-Item "C:\Temp\TempFiles.log" | Initialize-Log -Passthru | Write-Log -Message "The script begins."
```

## Relying on Mutex

#### Parallel Jobs without Mutex
Okay from now on, we kept things quite basic. In fact, you could stop there have a decent Write-Log function. Here, we are going to touch parallel execution and how we can handle the logging correctly.

First, let's try without handling mutex. I made a script to call parallel jobs that will write into the same log file.

```powershell
Import-module C:\Workspace\GitHub\PowerShellScripts\_bamshell\Write-Log\Powershell.Logging.Utility.psm1
Initialize-Log -FilePath "C:\Temp\ParallelJobs.log" -IncludeHeader -Clear

# This is the script block that will be called each time.
$ScriptBlock = {
    param (
        $Path
    )
    # I have to import the module containing my Write-Log functions because it's session wide loaded, and jobs are in another session.
    Import-module C:\Workspace\GitHub\PowerShellScripts\_bamshell\Write-Log\Powershell.Logging.Utility.psm1
    $ChildItem = Get-Childitem -Path $Path
    $ChildItem.FullName | Write-Log -FilePath "C:\Temp\ParallelJobs.log"
}

# I run parallel jobs on each Directory in C:\Program Files and get their content.
(Get-ChildItem 'C:\Program Files' -Directory).FullName | ForEach-Object {
    Start-Job -ScriptBlock $ScriptBlock -ArgumentList $_
}

# Job function are really well made for pipelining :)
Get-job | Wait-Job | Receive-Job
```

The result is really treacherous. If you look at you log file, everything seems quite normal. But you will spot that there are crossing between calls. Ideally, it should have been grouped by Directory for instance.

```
[2018-02-10][11:40:30] - Info - C:\Program Files\BCUninstaller\WinUpdateHelper.exe
[2018-02-10][11:41:00] - Info - C:\Program Files\WindowsPowerShell\Configuration
[2018-02-10][11:41:00] - Info - C:\Program Files\Oracle\VirtualBox
[2018-02-10][11:41:00] - Info - C:\Program Files\CONEXANT\CNXT_AUDIO_HDA
[2018-02-10][11:41:00] - Info - C:\Program Files\Reference Assemblies\Microsoft
[2018-02-10][11:41:00] - Info - C:\Program Files\Synaptics\SynTP
[2018-02-10][11:41:00] - Info - C:\Program Files\windows nt\accessories
[2018-02-10][11:41:00] - Info - C:\Program Files\Windows Portable Devices\sqmapi.dll
[2018-02-10][11:41:01] - Info - C:\Program Files\WindowsPowerShell\Modules
[2018-02-10][11:41:00] - Info - C:\Program Files\Wireshark\audio
[2018-02-10][11:41:01] - Info - C:\Program Files\Microsoft VS Code\bin
[2018-02-10][11:41:01] - Info - C:\Program Files\Q-Dir\Q-Dir.exe
[2018-02-10][11:41:01] - Info - C:\Program Files\Windows Photo Viewer\en-US
[2018-02-10][11:41:01] - Info - C:\Program Files\CONEXANT\DTSCONFIG
```

But this is even worse than it seems at first glance. Because the output on the console hides a bigger problem. Some objects weren't even logged. An error was thrown.

```
The process cannot access the file 'C:\Temp\ParallelJobs.log' because it is being used by another process.
    + CategoryInfo          : OpenError: (:) [Out-File], IOException
    + FullyQualifiedErrorId : FileOpenFailure,Microsoft.PowerShell.Commands.OutFileCommand
    + PSComputerName        : localhost
```

So, we have a big problem over there because we are missing some information in our logging file. We can't rely on our function in parallel jobs the way it is. But we have a designated solution: Mutex

#### Working with mutex

I learned most of this a while ago from this great [post](https://learn-powershell.net/2014/09/30/using-mutexes-to-write-data-to-the-same-logfile-across-processes-with-powershell/){:target="_blank"}

>Note : One important thing to keep in mind is that mutex works only in local parallelization. So it won't work over network, like _Invoke-Command_.

I will show you how to implement it in our *-Log functions.

So first, we need to create a named Mutex. This should be performed in our `Initialize-Log` function. So we will add `NewMutex` to create a mutex and `CloseMutex` to dispose it when we do not need it anymore. It is also a good place to use `ParameterSetName` since we want to be able to call `CloseMutex` without any other parameter. We will have to define ParameterSetName for each parameter we want to include/exclude to a ParameterSet.

>Note : _ParameterSetName_ allows us to separate my parameters in 2 groups that won't collide. This way, we won't be able to call _NewMutex_ with _CloseMutex_ (Which would make no sense)

Here is the code we add : 
```powershell
# Initialize-Log params
    [CmdletBinding(DefaultParameterSetName = "FilePath")]
    Param
    ( 
        [Parameter(Mandatory = $true, ValueFromPipeline = $true, ParameterSetName = "FilePath")]
        [ValidateNotNullOrEmpty()]
        [string]$FilePath,

        [Parameter(ParameterSetName = "FilePath")]
        [switch]$Clear,

        [Parameter(ParameterSetName = "FilePath")]
        [switch]$IncludeHeader,

        [Parameter(ParameterSetName = "FilePath")]
        [switch]$Passthru,
        
        [Parameter(ParameterSetName = "FilePath")]
        [ValidateNotNullOrEmpty()]
        [string]$NewMutex,

        [Parameter(Mandatory = $true, ParameterSetName = "CloseMutex")]
        [ValidateNotNullOrEmpty()]    
        [string]$CloseMutex
    )
```

You can see that all parameters are in the default _FilePath_ parameter set except `CloseMutex`. This means that if you call CloseMutex, you won't have the other parameters available.

Inside the body of the function, we will have to handle parameter sets with a test on `$PSCmdlet.ParameterSetName`. This is not mandatory in all function, but here, it made error handling easier. For instance, we don't want the file to be created if we call `CloseMutex`

```powershell
# Inside the Initialize-Log function
if ($PSCmdlet.ParameterSetName -eq "FilePath") {
    #//...I extracted parts of the code for clarity purpose ...//
    if ($NewMutex) {
        If ( -not ([System.Threading.Mutex]::TryOpenExisting($NewMutex, [ref]$null))) {
            $Mutex = New-Object System.Threading.Mutex($false, $NewMutex)
            $Mutex.WaitOne() | Out-Null
            $Mutex.ReleaseMutex()
        }    
    }
}
else {
    if ($CloseMutex) {
            $Mutex = New-Object System.Threading.Mutex($false, $NewMutex)
            $Mutex.WaitOne() | Out-Null
            $Mutex.ReleaseMutex()
            $Mutex.Close()
            $Mutex.Dispose()
    }
} 
```

On the Write-Log function, I won't deal with parameter sets because the `MutexName` will not be mandatory, and won't collide with other parameters. However, I will have to handle the mutex in the begin block and the end block to obtain it and release it. So I added both blocks.

>Reminder : When objects are received by the pipeline input, the begin block is executed once, the process block is executed as many times as the number of objects and then the End block is executed once.

Here is the implementation :
```powershell
Param
(       
    #//... Other params...//
    [string]$MutexName
)
Begin {
    if ($MutexName) {
        $Mutex = New-Object System.Threading.Mutex($false, $MutexName)
        $Mutex.WaitOne() | Out-Null
    }
}
Process {
    #//...do the stuff...//     
}
End {
    if ($MutexName) {
        $Mutex.ReleaseMutex() | Out-Null
    }
}
```

You may find odd that I chose to use `New-Object` to create the mutex instead of `[System.Threading.Mutex]::OpenExisting($MutexName)` for instance. But from testing, I find it more reliable and from what I get about this, it's more a reference to the mutex that is created rather than a mutex on its own, since it already exists. Though, I am not an expert in this field, so feel free to feedback in comments.

Now that we have our functions ready to use mutex, let's run the same script as above, modified to handle the mutex.

```powershell
Import-module C:\Workspace\GitHub\PowerShellScripts\_bamshell\Write-Log\Powershell.Logging.Utility.psm1
Initialize-Log -FilePath "C:\Temp\ParallelJobs.log" -IncludeHeader -Clear -NewMutex "MyMutex"

# This is the script block that will be called each time.
$ScriptBlock = {
    param (
        $Path
    )
    # I have to import the module containing my Write-Log functions because it's session wide loaded, and jobs are in another session.
    Import-module C:\Workspace\GitHub\PowerShellScripts\_bamshell\Write-Log\Powershell.Logging.Utility.psm1
    $ChildItem = Get-Childitem -Path $Path
    $ChildItem.FullName | Write-Log -FilePath "C:\Temp\ParallelJobs.log" -MutexName "MyMutex"
}

# I run parallel jobs on each Directory in C:\Program Files and get their content.
(Get-ChildItem 'C:\Program Files' -Directory).FullName | ForEach-Object {
    Start-Job -ScriptBlock $ScriptBlock -ArgumentList $_
}

# Job function are really well made for pipelining :)
Get-job | Wait-Job | Receive-Job
#>
Initialize-Log -CloseMutex "MyMutex"
```

This time the result is reliable. No error were thrown on the console and inside the log, I can see the results are grouped by directory.

>Note : You can also see that they are not in absolute alphabetical order, since the fastest jobs may have taken the mutex faster that others (For instance, the _Reference Assemblies_ directory that contained only one object)

```
[2018-02-10][12:10:48] - Info - C:\Program Files\BCUninstaller\UpdateHelper.exe
[2018-02-10][12:10:48] - Info - C:\Program Files\BCUninstaller\UpdateSystem.dll
[2018-02-10][12:10:48] - Info - C:\Program Files\BCUninstaller\WinUpdateHelper.exe
[2018-02-10][12:11:07] - Info - C:\Program Files\Common Files\microsoft shared
[2018-02-10][12:11:07] - Info - C:\Program Files\Common Files\Services
[2018-02-10][12:11:07] - Info - C:\Program Files\Common Files\system
[2018-02-10][12:11:15] - Info - C:\Program Files\Reference Assemblies\Microsoft
[2018-02-10][12:11:15] - Info - C:\Program Files\CONEXANT\CNXT_AUDIO_HDA
[2018-02-10][12:11:16] - Info - C:\Program Files\CONEXANT\DTSCONFIG
[2018-02-10][12:11:16] - Info - C:\Program Files\CONEXANT\Install
[2018-02-10][12:11:16] - Info - C:\Program Files\CONEXANT\MicTray
[2018-02-10][12:11:16] - Info - C:\Program Files\CONEXANT\SA3
[2018-02-10][12:11:16] - Info - C:\Program Files\CONEXANT\SSPConfig
```
Keep in mind these are simple example. I did not try to optimize mutex timeout and stuff. If you really need high performance, you may want to dig into these options though.
Another thing to take in account, is that my job performs a single call of the Write-Log function, so things can keep order per job. But if you call Write-Log multiple times per job, you will not loose any informations, but they might not be ordered per job.
For instance :

```powershell
# If instead of this call...
$ChildItem.FullName | Write-Log -FilePath "C:\Temp\ParallelJobs.log" 

# ... I had this one
$ChildItem.FullName | Foreach-Object {
    Write-Log -FilePath "C:\Temp\ParallelJobs.log" -Message $_
}
```
With the latter code, I would have called the Write-Log function _n_ times and the files wouldn't have been grouped by directory. But you wouldn't have missed any.

So you can rely on mutex to perfom parallel jobs.

## The full functions

#### Initialize-Log
```powershell
function Initialize-Log {
    [CmdletBinding(DefaultParameterSetName = "FilePath")]
    Param
    ( 
        [Parameter(Mandatory = $true, ValueFromPipeline = $true, ParameterSetName = "FilePath")]
        [ValidateNotNullOrEmpty()]
        [string]$FilePath,

        [Parameter(ParameterSetName = "FilePath")]
        [switch]$Clear,

        [Parameter(ParameterSetName = "FilePath")]
        [switch]$IncludeHeader,

        [Parameter(ParameterSetName = "FilePath")]
        [switch]$Passthru,
        
        [Parameter(ParameterSetName = "FilePath")]
        [ValidateNotNullOrEmpty()]
        [string]$NewMutex,

        [Parameter(Mandatory = $true, ParameterSetName = "CloseMutex")]
        [ValidateNotNullOrEmpty()]    
        [string]$CloseMutex
    )
    if ($PSCmdlet.ParameterSetName -eq "FilePath") {
        if (-not(Test-Path -Path $FilePath)) {
            New-Item -Path $FilePath -Force | Out-Null
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
        
        if ($NewMutex) {
            If ( -not ([System.Threading.Mutex]::TryOpenExisting($NewMutex, [ref]$null))) {
                $Mutex = New-Object System.Threading.Mutex($false, $NewMutex)
                $Mutex.WaitOne() | Out-Null
                $Mutex.ReleaseMutex()
            }    
        }

        if ($Passthru) {
            Write-Output $FilePath
        }      
    }
    else {
        if ($CloseMutex) {
            $Mutex = New-Object System.Threading.Mutex($false, $NewMutex)
            $Mutex.WaitOne() | Out-Null
            $Mutex.ReleaseMutex()
            $Mutex.Close()
            $Mutex.Dispose()
        }
    }     
}
```

#### Write-Log
```powershell
function Write-Log {
    [CmdletBinding()]
    Param
    (       
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        [ValidateScript( {
                if (-not (Test-Path $_)) {
                    throw "The file $_ does not exist. Use Initialize-Log to create it." 
                } 
                $true
            })]
        [string]$FilePath,
       
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        [string]$Message,

        [ValidateSet("Info", "Warning", "Error")]
        $Level = "Info",

        [string]$MutexName
    )
    Begin {
        if ($MutexName) {
            $Mutex = New-Object System.Threading.Mutex($false, $MutexName)
            $Mutex.WaitOne() | Out-Null
        }
    }
    Process {
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
    End {
        if ($MutexName) {
            $Mutex.ReleaseMutex() | Out-Null
        }
    }
}
```

## What's up for the next part ?
In the next and, I think, last part, we will create a `Backup-Log` function.

With all these fancy Log function, you will accumulate many logfiles that will grow over time, so it is a good thing to have a rotation cycle.

I will also take some time to show you how to wrap all of these functions in a module and take advantage of it.