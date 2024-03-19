+++
title = 'RMM Powershell Script Building'
date = 2023-01-01T18:18:25-08:00
draft = false
summary = 'Setup Windows for Powershell Script testing and building for RMM'
+++

The idea behind this post is to document a way to setup a build environment
for building Powershell scripts running via RMM.

### Setup

- Create a 'scripts' folder at the root of C:\
- Download [PS Tools](https://download.sysinternals.com/files/PSTools.zip)
- Place the file PsExec.exe in the 'scripts' folder you created.
- Create your scripts in the same folder.
- Install VSCode with the Powershell extension.
- Verify you're testing your scripts with Powershell 5.1.

### Getting Started

RMM Systems such as TacticalRMM use the System account to run their scripts. 
In order to run under a user context you must use a built-in method with your 
RMM or use the RunAsUser Powershell module.

- Open a Powershell window as Admin.
- Navigate to the root of C:\ and enter the scripts folder.
- Launch a new Powershell window as System using PsExec.
```powershell
.\PsExec -i -s "powershell.exe"
```
- In the new window navigate to our C:\scripts directory.
- Verify we are running as System
```powershell
whoami
```
Output should look like the following.
```powershell
nt authority\system
```
- We are now ready to start scripting.

All scripts and testing should be done from the Powershell window running in 
the system context.
