# Checking Windows for Resource Exhaustion

## Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
$MaxEvents = Read-Host "How many events would you like to see?"
Get-WinEvent -FilterHashtable @{logname = 'System'; id = 2004} -MaxEvents $MaxEvents | Format-Table -wrap
```

## Event Viewer
>Go to Event Viewer

>Go to System

>Click on Filter Current Log -> on the right hand column

>Change <All Event IDs> to be 2004

> You should now see the 'Resouce Exhaustion' events.
