+++
title = 'Windows Update Powershell Tricks'
date = 2022-12-12T18:08:04-08:00
draft = false
summary = 'Windows Update Powershell tips and tricks'
+++

### Consolidate Windows Update Log
```powershell
Get-WindowsUpdateLog
```
Log location at the root of C:

### Trigger Troublershooting Pack
```powershell
Get-TroubleshootingPack -Path C:\Windows\diagnostics\system\WindowsUpdate | Invoke-TroubleshootingPack
```

### Check Windows Update Settings
```powershell
Get-Item -Path Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate
```


### Remove Version Restriction
```powershell
Remove-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Name "TargetReleaseVersion" -ErrorAction SilentlyContinue
Remove-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Name "TargetReleaseVersionInfo" -ErrorAction SilentlyContinue
```
