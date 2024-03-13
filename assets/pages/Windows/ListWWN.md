# List WWNs for Disks, and their size in GB

>Run the Following
```Powershell
Get-PhysicalDisk | fl uniqueid,DeviceID,FriendlyName,@{Name="GB";Expression={$_.size/1GB}}
```
