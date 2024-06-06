# Useful Powershell Commands
## This wiki will list some useful powershell commands

> The following scripts should be run from within Powershell ISE

<br>

This command does something
```Powershell ISE
Get-ADUser
```

<br>

## Get Disk Usage
```Powershell
$Disk = Read-Host "Please enter a Disk you want to check. For Example (E). Do not add ':': "
Get-PSDrive $Disk | Select Root,@{label='Used Space(GB)';expression={$_.Used/1gb -as [int]}},@{label='Free Space(GB)';expression={$_.Free/1gb -as [int]}} | ft -Wrap -AutoSize
```

<br>

## Get Directory Creation Date and Age
```Powershell
Get-Childitem "E:\DirectoryGoesHere" | Select-Object FullName, CreationTime,@{Name="Age";Expression={(((Get-Date) - $_.CreationTime).Days)}}
```
