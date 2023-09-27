# Finding the cause of a reboot

## Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
#Based on <http://gallery.technet.microsoft.com/scriptcenter/Get-PendingReboot-Query-bdb79542>
function Test-PendingReboot {
    if (Get-ChildItem "HKLM:\Software\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending" -EA Ignore) 
        { 
            $ComponentStatus = "Component Based Servicing Reboot required"
            return $ComponentStatus
        }
    if (Get-Item "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired" -EA Ignore) 
        { 
            $WindowsUpdateStatus = "Windows Update Reboot is pending"
            return $WindowsUpdateStatus 
        }
    if (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager" -Name PendingFileRenameOperations -EA Ignore) 
        { 
            $WindowsGenericReboot = "A generic reboot is required"
            return $true 
        }
    
    try { 
            $util = [wmiclass]"\\.\root\ccm\clientsdk:CCM_ClientUtilities"
            $status = $util.DetermineIfRebootPending()
            if (($status -ne $null) -and $status.RebootPending) 
                {
                    return $true
                }
        }
    catch 
        { 
        }

    return $false
}

Test-PendingReboot
```
