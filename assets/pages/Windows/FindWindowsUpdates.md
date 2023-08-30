# Find what Updates were installed
# Tabs {.tabset}
## Option 1 - Basic Info
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
Get-WinEvent -FilterHashtable @{logname = 'System'; id = 17,19} -MaxEvents 20 | Format-Table -wrap
```
> You should now be able to see some basic information as to what updates were installed.
{.is-success}

## Option 2 - Verbose Info
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
Get-WinEvent -FilterHashtable @{logname = 'System'; id = 17,19,22,27,28,43} -MaxEvents 20 | Format-Table -wrap
```
> You should now be able to see some basic information as to what updates were installed, with more information.
{.is-success}
