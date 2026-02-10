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

---

## Version 4 AutoBan Nice GUI
``` Powershell
<#
RDP Failed Logons (Brute-Force) Check
- Compatible with PowerShell 5
- Run directly in PowerShell ISE (paste and press F5)
- No script parameters/flags needed

What it does:
1) Scans Security event ID 4625 on remote servers for the last N hours
2) Filters for RDP-relevant logon types (10 - RemoteInteractive; also includes 3 to catch NLA attempts)
3) Summarizes failed attempts by IP + Username and provides totals
4) Lists users with failed logins and their distinct source IP counts
5) Flags potential brute-force sources based on a configurable threshold

Requirements:
- You must have rights to read Security logs on the remote servers
- Security auditing for failed logons must be enabled

Tip:
- If IpAddress shows "-" or blank, it may be a local or unlogged source; those are excluded from aggregation
#>

# ===================== USER SETTINGS =====================
# 👉 Replace with your RDS servers
$Servers = @(
    hostname
)

# Lookback window in hours (e.g., last 24 hours)
$HoursBack = 24

# Mark an IP as "potential brute-force" if failed attempts from that IP (per username) >= this threshold
$BruteForceThreshold = 10

# Optional: export CSVs to Desktop (set to $true to enable)
$ExportCsv = $false
# =========================================================

# --- Setup / Validation ---
if (-not $Servers -or $Servers.Count -eq 0) {
    Write-Host "No servers specified. Please populate the `$Servers array." -ForegroundColor Yellow
    return
}

$startTime = (Get-Date).AddHours(-[int]$HoursBack)
$filter = @{
    LogName   = 'Security'
    Id        = 4625
    StartTime = $startTime
}

# Container for all events
$all = New-Object System.Collections.Generic.List[object]

Write-Host ""
Write-Host "🔐 RDP Brute-Force Check" -ForegroundColor Cyan
Write-Host ("🕒 Time Window: {0} to {1}" -f $startTime, (Get-Date)) -ForegroundColor Gray
Write-Host ("🖥️ Servers: {0}" -f ($Servers -join ", ")) -ForegroundColor Gray
Write-Host ""

foreach ($s in $Servers) {
    Write-Host ("🔎 Querying {0} ..." -f $s) -ForegroundColor Cyan
    try {
        $events = Get-WinEvent -ComputerName $s -FilterHashtable $filter -ErrorAction Stop
    }
    catch {
        Write-Warning ("Failed to read Security log from {0}: {1}" -f $s, $_.Exception.Message)
        continue
    }

    if (-not $events) {
        Write-Host ("No 4625 events found on {0} within last {1} hours." -f $s, $HoursBack) -ForegroundColor DarkGray
        continue
    }

    foreach ($e in $events) {
        # Parse XML once to reliably pull named fields
        $xml = [xml]$e.ToXml()
        $kv = @{}
        foreach ($d in $xml.Event.EventData.Data) {
            # Each Data element has a Name attrib and text content
            $kv[$d.Name] = [string]$d.'#text'
        }

        # Extract commonly used fields (defensive defaults)
        $ip   = $kv['IpAddress']
        $user = $kv['TargetUserName']
        $lt   = $kv['LogonType']

        # Filter: Skip entries without an IP or known loopbacks/placeholder
        if ([string]::IsNullOrWhiteSpace($ip) -or $ip -eq '-' -or $ip -eq '::1' -or $ip -eq '127.0.0.1') { continue }

        # Focus on RDP-related logons:
        # LogonType 10 = RemoteInteractive (typical for RDP), include 3 to catch some NLA cases observed in the field
        $isRdp = ($lt -eq '10' -or $lt -eq '3')
        if (-not $isRdp) { continue }

        $obj = [pscustomobject]@{
            Server        = $s
            TimeCreated   = $e.TimeCreated
            Username      = $user
            IpAddress     = $ip
            LogonType     = [int]($lt | ForEach-Object { if ($_ -match '^\d+$') { $_ } else { 0 } })
            FailureReason = $kv['FailureReason']
            Status        = $kv['Status']
            SubStatus     = $kv['SubStatus']
        }
        [void]$all.Add($obj)
    }
}

if ($all.Count -eq 0) {
    Write-Host ""
    Write-Host "✅ No RDP-style failed logins (4625) found in the selected window." -ForegroundColor Green
    return
}

# --- Aggregations ---

# 1) Breakdown by IP + Username
$byIpUser =
    $all |
    Group-Object IpAddress, Username |
    Sort-Object Count -Descending |
    Select-Object @{n='IP Address'; e={$_.Group[0].IpAddress}},
                  @{n='Failed Logins'; e={$_.Count}},
                  @{n='Username'; e={$_.Group[0].Username}}

# 2) Users with failed logins (with distinct IPs and top source IPs)
$byUser =
    $all |
    Group-Object Username |
    Sort-Object Count -Descending |
    Select-Object @{n='Username'; e={$_.Name}},
                  @{n='Failed Logins'; e={$_.Count}},
                  @{n='Distinct IPs'; e={ ($_.Group | Select-Object -ExpandProperty IpAddress | Sort-Object -Unique).Count }},
                  @{n='Top IPs'; e={
                        ($_.Group |
                         Group-Object IpAddress |
                         Sort-Object Count -Descending |
                         Select-Object -First 3 |
                         ForEach-Object { "$($_.Name) [$($_.Count)]" }) -join ', '
                  }}

# Potential brute-force sources (threshold per IP+Username)
$bruteCandidates = $byIpUser | Where-Object { $_.'Failed Logins' -ge [int]$BruteForceThreshold }

$total = $all.Count
$serverCount = ($Servers | Sort-Object -Unique).Count

# --- Output ---

Write-Host ""
Write-Host "🚨 Potential Brute-Force Sources (≥ $BruteForceThreshold failed attempts per IP+Username)" -ForegroundColor Yellow
if ($bruteCandidates) {
    $bruteCandidates | Format-Table -AutoSize
} else {
    Write-Host "None detected for the selected window." -ForegroundColor DarkGray
}

Write-Host ""
Write-Host "📊 Breakdown by IP Address + Username (All Servers)" -ForegroundColor Cyan
$byIpUser | Format-Table -AutoSize

Write-Host ""
Write-Host "👤 Users with Failed Logins (Count & Distinct Sources)" -ForegroundColor Cyan
$byUser | Format-Table -AutoSize

Write-Host ""
Write-Host ("🧮 Total Failed Logins: {0}   |   Servers: {1}   |   Window: Last {2} hours" -f $total, $serverCount, $HoursBack) -ForegroundColor White

# Optional export
if ($ExportCsv) {
    $stamp = Get-Date -Format 'yyyyMMdd_HHmm'
    $desktop = [Environment]::GetFolderPath('Desktop')

    $rawPath      = Join-Path $desktop ("RDP_FailedLogons_Raw_{0}.csv" -f $stamp)
    $ipUserPath   = Join-Path $desktop ("RDP_FailedLogons_ByIpUser_{0}.csv" -f $stamp)
    $usersPath    = Join-Path $desktop ("RDP_FailedLogons_ByUser_{0}.csv" -f $stamp)

    try {
        $all       | Export-Csv -NoTypeInformation -Path $rawPath
        $byIpUser  | Export-Csv -NoTypeInformation -Path $ipUserPath
        $byUser    | Export-Csv -NoTypeInformation -Path $usersPath

        Write-Host ""
        Write-Host "💾 CSVs exported to Desktop:" -ForegroundColor Green
        Write-Host " - $rawPath"
        Write-Host " - $ipUserPath"
        Write-Host " - $usersPath"
    }
    catch {
        Write-Warning ("Failed to export CSVs: {0}" -f $_.Exception.Message)
    }
}

Write-Host ""
Write-Host "✅ Completed." -ForegroundColor Green
```