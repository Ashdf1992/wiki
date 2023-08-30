# Finding the cause of a reboot

## Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
Get-WinEvent -FilterHashtable @{logname = 'System'; id = 1074,6008} -MaxEvents 2 | Format-Table -wrap
```
> You should now be able to see the cause of a reboot.

## Event Viewer
>Go to Event Viewer

>Go to System

>Click on Filter Current Log -> on the right hand column

>Change <All Event IDs> to be 1076,41,6008,1074

> You should now be able to see the cause of a reboot.
