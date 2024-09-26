# How to use Get-NetTCPConnection instead of Netstat

[comment]: <> (https://adamtheautomator.com/netstat-port/)

This tutorial isn’t going to cover all of the parameters that come with the Get-NetTCPConnection cmdlet. If you’re curious, run Get-Help Get-NetTCPConnection -Detailed to discover more examples.

---

## Only see listening ports
```Powershell
Get-NetTCPConnection -State Listen
```
![1.Image to go here.jpg](/images/1.Image to go here.jpg)


<br>

## Get the Owning Process
```Powershell
Get-Process -Id 692
```

<br>

## Only see listening and established ports
```Powershell
Get-NetTCPConnection -State Listen,Established
```

<br>

## Check for 443 (https) as the Remote Port
Here you can see the web servers that you are connected to by querying the remote port 443
```Powershell
Get-NetTCPConnection -RemotePort 443
```

<br>

## Get the Owning Process for a specific Local Port
```Powershell
$localport = Read-Host "Enter the Local Port Number you wish to get the owning process for: "
Get-NetTCPConnection -LocalPort $localport -State Listen | select Local*,Remote*,State,@{Name="Process ID";Expression={(Get-Process -Id $_.OwningProcess).Id}},@{Name="Process";Expression={(Get-Process -Id $_.OwningProcess).ProcessName}}
```
