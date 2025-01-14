# RDS Apps and Start Menu Issue
## The issue
Every time a user logs into the RDS server, a new set of firewall rules are created and, they are not removed when the users log off, but new ones are re-created when they login again. This appears to happen most when using User Profile Disks (UPD). You may notice at this time that when trying to open Windows Defender Firewall with Advanced Security to view the rules, the window would take a very long time to appear and end up in a “Not Responding” state.

This was resolved in Windows Server 2016 and 2019 by installing a Windows update, link for [2016](https://support.microsoft.com/en-us/help/4467684/windows-10-update-kb4467684) and [2019](https://support.microsoft.com/en-gb/help/4490481/windows-10-update-kb4490481), and manually applying a registry key.

The following powershell script will remove all of the previous entries created. In addition to this, it will also add the manual entry required within the registry to prevent the rules from being generated again in future, every time a user logs in. 

Part 1:

```powershell
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy" -Name 'DeleteUserAppContainersOnLogoff' -Value 1 -PropertyType DWord
$profiles = get-wmiobject -class win32_userprofile
cls
Write-Host "`n`n`n`n`n`n`n`n"
Write-Host "Getting Firewall Rules..."
# deleting rules with no owner would be disastrous
$Rules1 = Get-NetFirewallRule -All | 
  Where-Object {$profiles.sid -notcontains $_.owner -and $_.owner }
$Rules1Count = $Rules1.count
Write-Host "" $Rules1Count "Rules`n"

Write-Host "Getting Firewall Rules from ConfigurableServiceStore Store..."
$Rules2 = Get-NetFirewallRule -All -PolicyStore ConfigurableServiceStore | 
  Where-Object { $profiles.sid -notcontains $_.owner -and $_.owner }
$Rules2Count = $Rules2.count
Write-Host "" $Rules2Count "Rules`n"

$Total = $Rules1.count + $Rules2.count
Write-Host "Deleting" $Total "Firewall Rules:" -ForegroundColor Green

$Result = measure-command {

  $start = (Get-Date)
  $i = 0.0

  foreach($rule1 in $Rules1){

    # action
    remove-itemproperty -path "HKLM:\System\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\FirewallRules" -name $rule1.name

    # progress
    $i = $i + 1.0
    $prct = $i / $total * 100.0
    $elapsed = (Get-Date) - $start
    $totaltime = ($elapsed.TotalSeconds) / ($prct / 100.0)
    $remain = $totaltime - $elapsed.TotalSeconds
    $eta = (Get-Date).AddSeconds($remain)

    # display
    $prctnice = [math]::round($prct,2) 
    $elapsednice = $([string]::Format("{0:d2}:{1:d2}:{2:d2}", $elapsed.hours, $elapsed.minutes, $elapsed.seconds))
    $speed = $i/$elapsed.totalminutes
    $speednice = [math]::round($speed,2) 
    Write-Progress -Activity "Deleting Rules1 ETA $eta elapsed $elapsednice loops/min $speednice" -Status "$prctnice" -PercentComplete $prct -secondsremaining $remain
  }

  foreach($rule2 in $Rules2) {

    # action  
    remove-itemproperty -path "HKLM:\System\CurrentControlSet\Services\SharedAccess\Parameters\FirewallPolicy\RestrictedServices\Configurable\System" -name $rule2.name

    # progress
    $i = $i + 1.0
    $prct = $i / $total * 100.0
    $elapse = (Get-Date) - $start
    $totaltime = ($elapsed.TotalSeconds) / ($prct / 100.0)
    $remain = $totaltime - $elapsed.TotalSeconds
    $eta = (Get-Date).AddSeconds($remain)

    # display
    $prctnice = [math]::round($prct,2) 
    $elapsednice = $([string]::Format("{0:d2}:{1:d2}:{2:d2}", $elapsed.hours, $elapsed.minutes, $elapsed.seconds))
    $speed = $i/$elapsed.totalminutes
    $speednice = [math]::round($speed,2) 
    Write-Progress -Activity "Deleting Rules2 ETA $eta elapsed $elapsednice loops/min $speednice" -Status "$prctnice" -PercentComplete $prct -secondsremaining $remain
  }
}

$end = get-date
write-host end $end 
write-host eta $eta

write-host $result.minutes min $result.seconds sec
```

<br>

Part 2:


```Powershell
Remove-Item "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Notifications" -Recurse
New-Item "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Notifications"
```

<br>

## Reference
https://www.reddit.com/r/fslogix/comments/1bp447v/slow_to_do_login_fslogix_please_wait_for_the/?rdt=46676
<br>
https://msendpointmgr.com/2021/06/17/fslogix-slow-sign-in-fix/
<br>
https://www.phy2vir.com/windows-server-2016-2019-rds-server-black-screen-or-start-menu-not-working/
