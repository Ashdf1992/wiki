# RDS Long Login Times/Black Screen Fix

<br>

> IMPORTANT: This guide is meant to be followed in order. If a step resolves the issue, stop at that step and do not apply any of the additional steps.
{.is-danger}

<br>

## 1) Windows Updates
> First thing is first. Apply the latest Windows Updates and reboot the server. If this does not resolve the issue, continue to 'Registry Entries'
{.is-info}

<br>

## 2) Registry Entries

> Apply the following registry entries via Powershell.
{.is-info}

> Remember that altering the registry requires a reboot of Windows for the changes to take effect so reboot after making alterations to the registry.
{.is-warning}

```Powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\" -Name 'DelayedDesktopSwitchTimeout' -Value 30000 -PropertyType DWord
New-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\" -Name 'FirstLogonTimeout' -Value 30000 -PropertyType DWord
New-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\" -Name 'AppReadinessPreShellTimeoutMs' -Value 30000 -PropertyType DWord
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy" -Name 'DeleteUserAppContainersOnLogoff' -Value 1 -PropertyType DWord
```


> If this hasnt helped to resolve the issue, proceed to 'Reset the StateRepository Database'. 
{.is-info}

<br>

## 3) Reset the StateRepository Database
> There are 2 Powershell Scripts you can run a version that keeps a transcript and provides more information, or a basic version. Ensure that you run one or the other, as an Administrator via Powershell, and also ensure that you reboot after applying one of them. 
{.is-info}



## Tabs {.tabset}
### Version 1 Standard - With Transcript

> Standard - With Transcript
{.is-info}

```Powershell
Start-Transcript C:\TEMP\last_StateRepositoryResetLog.txt

Write-Verbose "Resetting StateRepository Databases for AppX / App Readiness" -Verbose
Write-Verbose "Attempting to stop StateRepository service" -Verbose

$retry = 3
DO{
    Get-Service -Name StateRepository | Stop-Service -Force
    Start-Sleep 1
    Get-Service -Name AppReadiness | Stop-Service -Force
    Start-Sleep 1
    $retry--
} While (
    ((Get-Service -Name StateRepository).Status -ne "Stopped") -and ($retry -gt 0)
)

if ((Get-Service -Name StateRepository).Status -eq "Stopped") {
    Write-Verbose "StateRepository service is stopped" -Verbose
    Write-Verbose "Attempting to delete old StateRepository database files..." -Verbose
    del C:\ProgramData\Microsoft\Windows\AppRepository\StateRepository*
    Write-Verbose "Done. Starting StateRepository Service" -Verbose
    Get-Service -Name StateRepository | Start-Service
} else {
    Write-Verbose "StateRepository service failed to stop! Terminating." -Verbose
}
Stop-Transcript
```
> If Windows Updates, the registry entries above, and the State Repository Database reset do not fix the issue, its unlikely that the issue has yet been resolved by Microsoft. If these havent resolved your issue proceed to the 'Workaround'
{.is-info}

### Version 2 Basic

> Basic
{.is-info}

```Powershell
Get-Service -Name StateRepository | Stop-Service -Force
Get-Service -Name AppReadiness | Stop-Service -Force
del C:\ProgramData\Microsoft\Windows\AppRepository\StateRepository*
Get-Service -Name StateRepository | Start-Service
```


> If Windows Updates, the registry entries above, and the State Repository Database reset do not fix the issue, its unlikely that the issue has yet been resolved by Microsoft. If these havent resolved your issue proceed to the 'Workaround'
{.is-info}

<br>

## 4) Workaround

> The work around should only be deployed as a last resort if the above 3 steps have failed.
{.is-warning}


> The workaround at present is to completely disable the 'App Readiness' service. Disabling the server, appears to restore login times back to their optimal level, however, it has been known to cause issues with logging into Microsoft Office Applications, Edge, and it has also been known to cause issues applying Windows Updates. 
{.is-info}

> To stop the service run the following Powershell cmdlet
{.is-info}

```Powershell
Stop-Service -Name "AppReadiness"
```




