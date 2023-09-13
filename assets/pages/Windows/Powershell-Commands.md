# Useful Powershell Commands
## This wiki will list some useful powershell commands

> The following scripts should be run from within Powershell ISE

<br>

This command does something
```Powershell ISE
Get-ADUser
```

## Get Disk Usage
```Powershell
$Disk = Read-Host "Please enter a Disk you want to check. For Example (E). Do not add ':': "
Get-PSDrive $Disk | Select Root,@{label='Used Space(GB)';expression={$_.Used/1gb -as [int]}},@{label='Free Space(GB)';expression={$_.Free/1gb -as [int]}} | ft -Wrap -AutoSize
