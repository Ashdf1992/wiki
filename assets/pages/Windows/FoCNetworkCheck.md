# Failover Cluster Inter-Network Connectivity Checker

### This Powershell script will check inter-network connectivity from the cluster node it is ran on, to other cluster nodes referenced within '$destclusternode', this value will need to be altered

```Powershell
Write-Output "This script is being ran from:" $env:computername
Write-Output ""
Write-Output ""
Write-Output "Here is a list of Cluster Nodes: "
Get-ClusterNode | Select -ExpandProperty Name
Write-Output ""
Write-Output ""
$destclusternode = Read-Host "Enter the 'Name' of the 'Cluster Node' listed above you wish to check "
$destclusternodeIP = Resolve-DnsName $destclusternode | Select -ExpandProperty IPAddress
$Standardclusterports = "3343","445","135"
$Rand_alloc_high_ports = Get-NetTCPConnection -RemoteAddress $destclusternodeIP | Where-Object { $_.LocalPort -ge "49152"} | Select -ExpandProperty LocalPort
$ports = $Standardports + $Rand_alloc_high_ports
foreach ($d in $destclusternode)
{
    Write-Output $destclusternode
    Write-Output $destclusternodeIP    
    foreach ($p in $ports)
        {
            Test-NetConnection $destclusternode -Port $p | Select RemotePort,TcpTestSucceeded
        }
}
```
