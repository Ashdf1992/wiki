# Installation of RSAT on Windows 10 and above
Installing the RSAT (Remote Server Administration Tools for Windows 10+) tools using PowerShell. This is just a quick article, written purely as an easy reference.

> Ensure you are running Powershell as Admin

> To get a list of RSAT tools run the following within Powershell

```Powershell
Get-WindowsCapability -Name RSAT.* -Online
```
> To install all RSAT Tools run the following
```Powershell
Get-WindowsCapability -Name RSAT.* -Online | Add-WindowsCapability -Online
```

> To install just AD RSAT Tools run the following
```Powershell
Get-WindowsCapability -Name RSAT.ActiveDirectory* -Online | Add-WindowsCapability -Online
```

> To install just GPO RSAT Tools run the following
```Powershell
Get-WindowsCapability -Name RSAT.GroupPolicy* -Online | Add-WindowsCapability -Online
```
