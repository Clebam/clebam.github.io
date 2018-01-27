---
layout: post
title:  "Why PowerShell ?"
categories: PowerShell
tags:  beginner
author: Clebam
---
* content
{:toc}

<!-- A short phrase of preview with a short text -->
#### _Many good reasons to enter PowerShell world_

There are many reasons to start with PowerShell. On a given day, you will be cross-roads between repeating 27 times the same task and automating it. On another day, you will need to repeat it 2500 times. Here is you chance, you will have to go through scripting. And PowerShell is the answer.

![Image](/img/posh840x420.png){:width="360"} <!--Standard width: 300px-->

<!--End_Preview-->

### From VBScript...

Back in the days, when I started as a Sysadmin Junior, I learned a bit of VBScript. The purpose of my first script was to copy a file from a share to another. It was a pretty easy task to perform and after some research, I found how to do this. This script worked until a few weeks ago when I decided to move it to PowerShell.
I made many scripts with VBScript, not thousands, but quite a few; from copying files to reset each computer from the AD spooler service remotely. But here's the trick... I can't remember any of the commands I used. Even for the most easy task, like copying a file. I'd say its _ObjFSO.Copy_ or something like that but I would have to google it again

### ... to PowerShell
And here is the first strength of PowerShell. It uses a _Verb-Noun_ convention that makes it easy to remember commands - or to be precise _cmdlets_.
If I say `Get-ComputerInfo` or `Copy-Item`, you instantly have an idea of what the cmdlet deals with.

Cmdlets are meant to be easily understood and remembered. If they are built correctly, they should follow the rules of [approved verbs](https://msdn.microsoft.com/en-us/library/ms714428(v=vs.85).aspx){:target="_blank"}.

Why is this so strong ? Well, in a Windows sysadmin career, you may have got the job done without scripting that much. Maybe copying and modifying some scripts after some googling was enough. But the tendancy is changing, and it's changing fast. New fancy names like _DevOps_ shows that the IT jobs tends to be more or less Jack-of-All-Trade. I don't say it's a good thing, but it's happening. And PowerShell is easy. Not simple, it can be pretty complex. But given a few tries and following a few basic tutorials, you will have a good start to handle automated tasks.

### How PowerShell magic happens ?

Another good great functionnality of ~~PlumberShell~~ PowerShell is piping. It allows you to chain actions from a cmdlet to another, taking the output of a cmdlet and sending it as the input of the next one down the pipe.
Here is an example:
```powershell
Get-ChildItem -Path C:\Temp | Where-Object Name -match "New" | Remove-Item
```
The first cmdlet lists the files of C:\Temp and it outputs an object containing all files to the next cmdlet `Where-Object`. Then `Where-Object` works a bit like the SQL _Where_ and filter the files with the property _Name_ that matches the string _"New"_. So it outputs all the files with _"New"_ in it to the next cmdlet. It then becomes the input of `Remove-Item` cmdlet which erases the files.
So in a single line, you filtered files with a given pattern and erased them. Again, easy to understand, easy to remember.

One last thing, is that PowerShell is pretty much self-documented and there is a few cmdlet that will help you in your PowerShell life:
```powershell
Get-Help -Name Get-ChildItem # Returns a Comment Base Help about a given cmdlet
Get-Command # Lists the available commands
$MyFile = Get-ChildItem -Path C:\Temp\MyFile.txt
$MyFile | Get-Member # Lists al the properties and method of a given object
```
### You can start today
I recommand you to give a try with PowerShell right now. Today is the good day. Go to your Windows start menu, search for **PowerShell ISE** and try a `Get-ChildItem`. Press F5 and your PowerShell journey begins...