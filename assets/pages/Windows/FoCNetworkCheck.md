# Failover Cluster Inter-Network Connectivity Checker

### This Powershell script will check inter-network connectivity from the cluster node it is ran on, to other cluster nodes referenced within '$destclusternode', this value will need to be altered

> Revision: 1.0
> 
> Products and/or Services: Failover Cluster, SQL, Active Directory, Firewall Ports
> 
> Environment: Powershell

> Note that this script needs to be ran from a Cluster Node, and as a Domain Admin. 

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
#$Rand_alloc_high_ports = Get-NetTCPConnection -RemoteAddress $destclusternodeIP | Where-Object { $_.LocalPort -ge "49152"} | Select -ExpandProperty Localport
#$Rand_alloc_high_ports = Invoke-Command -ComputerName $destclusternode -ScriptBlock {Get-NetTCPConnection | Where-Object { $_.LocalPort -ge "49152"} | Select -ExpandProperty LocalPort
#Get-Process -Id (Get-NetTCPConnection -State Listen | Where-Object { $_.LocalPort -ge "49152"}).OwningProcess | Where-Object {$_.ProcessName -eq "clussvc"} | Select -ExpandProperty Id
#Get-NetTCPConnection | where-object { ($_.OwningProcess -eq ) -and ($_.LocalPort -ge "49152")} | Select -ExpandProperty LocalPort
$processIDcommand = {(Get-Process -Id (Get-NetTCPConnection -State Listen | Where-Object { $_.LocalPort -ge "49152"}).OwningProcess | Where-Object {$_.ProcessName -eq "clussvc"})}
$Rand_alloc_high_portsbuilder1 = Invoke-Command -ComputerName $destclusternode -ScriptBlock $processIDcommand
[string]$process = $Rand_alloc_high_portsbuilder1.Id.ToString()
$portscommand = {Get-NetTCPConnection | where-object { ($_.OwningProcess -eq $using:process) -and ($_.LocalPort -ge "49152")} | Select -ExpandProperty LocalPort -Unique}
$Rand_alloc_high_portsbuilder2 = Invoke-Command -ComputerName $destclusternode -ScriptBlock $portscommand
$ports = $Standardclusterports + $Rand_alloc_high_portsbuilder2
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
