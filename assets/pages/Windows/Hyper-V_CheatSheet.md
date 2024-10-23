# Hyper-V Cheatsheet

## Get-VM & Memory Usage
```Powershell
Get-VM | Select Name,@{label='Memory Assigned(MB)';expression={$_.memoryassigned/1gb -as [int]}} | out-gridview
```

## Get-VM & CPU count
```Powershell
Get-VM | Get-VMProcessor | Select VMName,Count | Out-GridView
```

## Get VM & HDD Size
```Powershell
Get-VM | Select-Object VMId | Get-VHD | select path,@{label='Size(GB)';expression={$_.size/1gb -as [int]}}
```

## Get Shared Cluster Volumes Used and Free space in GB
```Powershell
Get-ClusterSharedVolume | ForEach-Object {[PSCustomObject]@{VolumeName = $_.Name; UsedSpace = $_.SharedVolumeInfo.Partition.UsedSpace/1gb; FreeSpace = $_.SharedVolumeInfo.Partition.FreeSpace/1gb;}} | ft -wrap -autosize
```

## Mini Report - 1 VM
```Powershell
Get-VM | Select Name
$VM = Read-Host "Enter a VM from the list Above to Check. Just copy and paste. "
Clear
Write-Host "CPU:"
Get-VM $VM | Get-VMProcessor | Select VMName,@{label='vCPU Core Count';expression={$_.Count}}
Write-Host ""
Write-Host "Memory Allocation:"
Get-VM $VM | Select Name,@{label='Memory Assigned(MB)';expression={$_.memoryassigned/1gb -as [int]}}
Write-Host ""
Write-Host "Disk Allocation:"
Get-VM $VM | Select-Object VMId | Get-VHD | select path,@{label='Size(GB)';expression={$_.size/1gb -as [int]}}
Write-Host "Complete!"
```

## Mini Report - All VMs
```Powershell
$VMs = Get-VM | Select -ExpandProperty Name
foreach ($v in $VMs)
{
    Write-Host "VM Name: $v"
    Write-Host "vCPU Info:"
    Get-VM $v | Get-VMProcessor | Select VMName,@{label='vCPU Core Count';expression={$_.Count}} | ft
    Write-Host "Memory Info:"
    Get-VM $v | Select Name,@{label='Memory Assigned(MB)';expression={$_.memoryassigned/1gb -as [int]}} | ft
    Write-Host "Disk Allocation:"
    Get-VM $v | Select-Object VMId | Get-VHD | select path,@{label='Size(GB)';expression={$_.size/1gb -as [int]}} | ft
    Write-Host ""


}
Write-Host "Report Complete!"
```
