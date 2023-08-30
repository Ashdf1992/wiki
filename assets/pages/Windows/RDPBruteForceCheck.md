# RDP Brute Force Check
## This wiki will highlight how to check for failed RDP logon attempts to your Windows Server

> The following scripts should be run from within Powershell ISE

## Version 1 Entire List
> (Info) This version will simply provide you the number of failed login attempts over the past 24 hours.

Load up Powershell ISE and then paste the below 2 commands
```Powershell ISE
$failLogs = Get-EventLog -LogName Security -InstanceId 4625 -After ((Get-Date).AddDays(-1)) | Select-Object TimeGenerated, Index, InstanceId, @{n='Username';e={$_.ReplacementStrings[5]}}
$failLogs.count
```
---

## Version 2 IPs
> (Info) This version will show the IPs that have failed to log in more than 5 times within the past 24 hours.

Load up Powershell ISE and run the first command
```Powershell ISE
$badRDPlogons = Get-EventLog -LogName Security -After ((Get-Date).AddDays(-1)) -InstanceId 4625 | Select-Object @{n='IpAddress';e={$_.ReplacementStrings[-2]} }
```
Then run the next 2 commands
```Powershell ISE
$getip = $badRDPlogons | group-object -property IpAddress | where {$_.Count -gt 5} | Select -ExpandProperty Name
$getip
```
---

## Version 3 AutoBan
> (Info) This version will automatically ban the IP Addresses that have failed to login using RDP, in the last 24 hours.

The first step only needs to be run once, and the IP Address 1.1.1.1 is just a template to create the rule.
Load up Powershell ISE and within the Powershell Section run the following
```Powershell ISE
New-NetFirewallRule -DisplayName "RDP_Brute_Force_Guard" –RemoteAddress 1.1.1.1 -Direction Inbound -Protocol TCP –LocalPort 3389 -Action Block
```
Then get a list of IPs that have failed to login 5 times within the past 24 hours, using the following commands within Powershell ISE:
```Powershell ISE
$failLogs = Get-EventLog -LogName Security -After ((Get-Date).AddDays(-1)) -InstanceId 4625 | Select-Object @{n='IpAddress';e={$_.ReplacementStrings[-2]} }
$failLogs2 = $failLogs | group-object -property IpAddress | where {$_.Count -gt 5} | Select -Property Name

```
Then we want to get a list of current IPs within the IP Block Rule, and add the new list of IPs to the rule, this can be done by running the following 
```Powershell ISE
$current_ips = (Get-NetFirewallRule -DisplayName "RDP_Brute_Force_Guard" | Get-NetFirewallAddressFilter ).RemoteAddress
foreach ($ip in $failLogs)
{
$current_ips += $ip.name
}
```
The next step is to apply the new IPs to the ruleset within Windows Firewall.
```Powershell ISE
Set-NetFirewallRule -DisplayName "RDP_Brute_Force_Guard" -RemoteAddress $current_ips
```
> (Info) The above can be added as a task that is triggered when the event 4625 is triggered in event viewer. Which will automatically Ban IP Addresses.
