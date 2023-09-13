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

## Mini Report - 1 VM
```Powershell
Get-VM | Select Name
$VM = Read-Host "Enter a VM from the list Above to Check. For Example 'Ash-VM-DC-01' "
Clear
Write-Host "CPU:"
Get-VM $VM | Get-VMProcessor | Select VMName,Count
Write-Host ""
Write-Host "Memory Allocation:"
Get-VM $VM | Select Name,@{label='Memory Assigned(MB)';expression={$_.memoryassigned/1mb -as [int]}}
Write-Host ""
Write-Host "Disk Allocation:"
Get-VM $VM | Select-Object VMId | Get-VHD | select path,@{label='Size(GB)';expression={$_.size/1gb -as [int]}}
