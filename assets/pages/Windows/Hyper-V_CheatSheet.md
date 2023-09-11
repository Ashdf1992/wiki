# Clearing IIS Log Files

## Get-VM & Memory Usage
```Powershell
Get-VM | Select Name,@{label='Memory Assigned(MB)';expression={$_.memoryassigned/1mb -as [int]}} | out-gridview
```

## Get-VM & CPU count
```Powershell
Get-VM | Get-VMProcessor | Select VMName,Count | Out-GridView
```

## Get VM & HDD Size
```Powershell
Get-VM | Select-Object VMId | Get-VHD | select path,@{label='Size(GB)';expression={$_.size/1gb -as [int]}}
```
