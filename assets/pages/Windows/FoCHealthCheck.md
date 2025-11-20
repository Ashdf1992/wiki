# Failover Cluster Health Check

<i> Revision: 1.0 </i>
<br>
<i>Relevent Products & Services : Failover Cluster, SQL, Active Directory, Firewall Ports</i>
<br>
<i>Environment: Powershell</i>

<br>
<br>

### This Powershell script will check health of a Failover Cluster on Windows.

> Note that this script needs to be ran from a Cluster Node, and as a Domain Admin. 

```Powershell
<#
.SYNOPSIS
  Non-disruptive WSFC health check with JSON and two HTML reports.

.DESCRIPTION
  Performs read-only checks on cluster state, nodes, networks, resources, CSVs,
  quorum, witness, and recent events (Fix A provider probing).
  Outputs:
   - HealthSummary.json (machine-readable)
   - HealthSummary.html  (overview)
   - HealthDetails.html  (detailed view rendered from JSON)

.PARAMETER Cluster
  Cluster name or FQDN. Defaults to local cluster context.

.PARAMETER OutputPath
  Folder for JSON and HTML outputs.

.PARAMETER IncludeClusterLog
  Collects cluster log (read-only).

.PARAMETER MaxEventHours
  Lookback window for event checks.

.NOTES
  Requires FailoverClusters module. Script is non-disruptive (read-only).
#>

[CmdletBinding()]
param(
    [Parameter(Position=0)]
    [string]$Cluster,
    [Parameter()]
    [string]$OutputPath = (Join-Path -Path $PWD -ChildPath ("ClusterHealth_" + (Get-Date -Format "yyyyMMdd_HHmmss"))),
    [Parameter()]
    [switch]$IncludeClusterLog,
    [Parameter()]
    [int]$MaxEventHours = 24
)

begin {
    if (-not (Get-Module -ListAvailable -Name FailoverClusters)) {
        Write-Error "FailoverClusters module not found. Install the feature or run on a node with the management tools."
        break
    }
    Import-Module FailoverClusters -ErrorAction Stop
    New-Item -ItemType Directory -Force -Path $OutputPath | Out-Null

    $summary = [ordered]@{
        Timestamp       = (Get-Date).ToString("o")
        ClusterName     = $null
        OverallState    = "Unknown"
        Findings        = @()
        Nodes           = @()
        Networks        = @()
        Resources       = @()
        ResourceGroups  = @()
        CSVs            = @()
        Witness         = @{}
        Quorum          = @{}
        DynamicQuorum   = @{}
        RecentEvents    = @()
        Recommendations = @()
    }

    function Add-Finding {
        param([ValidateSet('Info','Warning','Error')][string]$Level,[string]$Message)
        $summary.Findings += [ordered]@{ Level=$Level; Message=$Message }
        switch ($Level) {
            'Error'   { $script:hasError = $true }
            'Warning' { $script:hasWarning = $true }
        }
    }

    $hasError   = $false
    $hasWarning = $false
}

process {
    try {
        # Cluster context (read-only)
        $cl = if ($Cluster) { Get-Cluster -Name $Cluster -ErrorAction Stop } else { Get-Cluster -ErrorAction Stop }
        $summary.ClusterName = $cl.Name

        # Quorum & Witness
        $quorum = Get-ClusterQuorum -Cluster $cl
        $summary.Quorum = [ordered]@{
            QuorumType   = $quorum.QuorumType
            Witness      = $quorum.QuorumResource
        }
        $summary.Witness = [ordered]@{
            Present      = [bool]$quorum.QuorumResource
            ResourceName = $quorum.QuorumResource
        }

        # Dynamic quorum/resiliency flags
        $summary.DynamicQuorum = [ordered]@{
            DynamicQuorumEnabled      = $cl.DynamicQuorum
            LowerQuorumPriorityNodeId = $cl.LowerQuorumPriorityNodeId
            WitnessDynamicWeight      = $cl.WitnessDynamicWeight
            QuarantineDuration        = $cl.QuarantineDuration
        }

        # Nodes
        $nodes = Get-ClusterNode -Cluster $cl | Sort-Object Name
        foreach ($n in $nodes) {
            $nodeInfo = [ordered]@{
                Name          = $n.Name
                State         = $n.State
                DrainStatus   = $n.DrainStatus
                IsPaused      = ($n.State -eq 'Paused')
                DynamicWeight = $n.DynamicWeight
            }
            $summary.Nodes += $nodeInfo

            if ($n.State -ne 'Up') { Add-Finding -Level 'Error'   -Message "Node '$($n.Name)' state is '$($n.State)'." }
            if ($n.DrainStatus -in @('Draining','Drained')) { Add-Finding -Level 'Warning' -Message "Node '$($n.Name)' has DrainStatus '$($n.DrainStatus)'." }
        }

        # Networks
        $nets = Get-ClusterNetwork -Cluster $cl | Sort-Object Name
        foreach ($net in $nets) {
            $netInfo = [ordered]@{
                Name    = $net.Name
                Role    = $net.Role
                Address = $net.Address
                Metric  = $net.Metric
                State   = $net.State
            }
            $summary.Networks += $netInfo
            if ($net.State -ne 'Up') { Add-Finding -Level 'Error' -Message "Network '$($net.Name)' state is '$($net.State)'." }
        }

        # Groups & Resources
        $groups = Get-ClusterGroup -Cluster $cl | Sort-Object Name
        foreach ($g in $groups) {
            $ownerNodesRaw = $g | Get-ClusterOwnerNode
            $preferredOwners = $ownerNodesRaw | ForEach-Object {
                if ($_.PSObject.Properties.Match('Name').Count -gt 0) { $_.Name }
                elseif ($_.PSObject.Properties.Match('NodeName').Count -gt 0) { $_.NodeName }
                else { $_.ToString() }
            }

            $grpInfo = [ordered]@{
                Name            = $g.Name
                State           = $g.State
                OwnerNode       = $g.OwnerNode
                PreferredOwners = $preferredOwners
            }
            $summary.ResourceGroups += $grpInfo

            if ($g.Name -eq 'Available Storage' -and $g.State -eq 'Offline') {
                Add-Finding -Level 'Warning' -Message "Group 'Available Storage' is Offline (commonly normal when no unused disks)."
            } elseif ($g.State -notin @('Online','PartialOnline')) {
                Add-Finding -Level 'Error' -Message "Group '$($g.Name)' state is '$($g.State)'."
            }

            $resources = Get-ClusterResource -Cluster $cl | Where-Object { $_.OwnerGroup -eq $g } | Sort-Object Name
            foreach ($r in $resources) {
                $resInfo = [ordered]@{
                    Name         = $r.Name
                    ResourceType = $r.ResourceType
                    State        = $r.State
                    OwnerGroup   = $r.OwnerGroup
                    OwnerNode    = $r.OwnerNode
                    IsCritical   = $r.IsCritical
                }
                $summary.Resources += $resInfo

                if ($r.State -ne 'Online' -and $r.ResourceType -ne 'Cluster IP Address') {
                    Add-Finding -Level 'Error' -Message "Resource '$($r.Name)' in group '$($g.Name)' state is '$($r.State)'."
                }
            }
        }

        # CSVs
        $csvs = Get-ClusterSharedVolume -Cluster $cl -ErrorAction SilentlyContinue
        foreach ($csv in ($csvs | Sort-Object Name)) {
            $csvInfo = [ordered]@{
                Name         = $csv.Name
                OwnerNode    = $csv.OwnerNode
                FriendlyName = $csv.SharedVolumeInfo.FriendlyVolumeName
                State        = $csv.State
                RedirectedIO = $csv.SharedVolumeInfo.RedirectedAccess
            }
            $summary.CSVs += $csvInfo
            if ($csv.State -ne 'Online') { Add-Finding -Level 'Error' -Message "CSV '$($csv.Name)' state is '$($csv.State)'." }
            if ($csv.SharedVolumeInfo.RedirectedAccess) { Add-Finding -Level 'Warning' -Message "CSV '$($csv.Name)' is in Redirected I/O mode." }
        }

        # === Fix A: Safe event checks ===
        $cutoff = (Get-Date).AddHours(-1 * $MaxEventHours)
        $requestedProviders = @('Microsoft-Windows-FailoverClustering','Disk','Ntfs','Microsoft-Windows-NetworkProfile')

        foreach ($n in $nodes) {
            try {
                $providersToQuery = @()
                foreach ($p in $requestedProviders) {
                    try {
                        $probe = Get-WinEvent -ComputerName $n.Name -FilterHashtable @{
                            LogName      = 'System'
                            ProviderName = $p
                            StartTime    = $cutoff
                        } -MaxEvents 1 -ErrorAction Stop
                        if ($probe) { $providersToQuery += $p }
                    } catch { }
                }

                $events = if ($providersToQuery.Count -gt 0) {
                    Get-WinEvent -ComputerName $n.Name -FilterHashtable @{
                        LogName      = 'System'
                        ProviderName = $providersToQuery
                        StartTime    = $cutoff
                    } -ErrorAction Stop | Select-Object TimeCreated, ProviderName, Id, LevelDisplayName, Message
                } else {
                    Get-WinEvent -ComputerName $n.Name -FilterHashtable @{
                        LogName   = 'System'
                        StartTime = $cutoff
                        Level     = 2
                    } -ErrorAction Stop | Select-Object TimeCreated, ProviderName, Id, LevelDisplayName, Message
                }

                $summary.RecentEvents += [ordered]@{
                    Node   = $n.Name
                    Count  = ($events | Measure-Object).Count
                    Sample = ($events | Select-Object -First 10)
                }

                $errIds = $events | Where-Object { $_.LevelDisplayName -eq 'Error' -or $_.Id -in (1205,1069,1135,1137,1146) }
                if ($errIds) {
                    Add-Finding -Level 'Warning' -Message "Node '$($n.Name)' has $($errIds.Count) recent cluster/storage/network errors."
                }
            } catch {
                Add-Finding -Level 'Warning' -Message "Could not read System log on node '$($n.Name)': $($_.Exception.Message)"
            }
        }

        # Recommendations
        if ($summary.Networks | Where-Object { $_.Role -eq 'Cluster' -and $_.Metric -gt 1000 }) {
            $summary.Recommendations += "Review cluster network metrics; ensure heartbeat network has lowest metric."
        }
        if ($summary.CSVs | Where-Object { $_.RedirectedIO }) {
            $summary.Recommendations += "Investigate CSV redirected I/O; check storage path/SMB and node maintenance state."
        }

        # Overall state
        if     ($hasError)   { $summary.OverallState = 'Degraded' }
        elseif ($hasWarning) { $summary.OverallState = 'Warning'  }
        else                 { $summary.OverallState = 'Healthy'  }

        # === Outputs ===
        $jsonPath = Join-Path $OutputPath 'HealthSummary.json'
        $summary | ConvertTo-Json -Depth 6 | Out-File -FilePath $jsonPath -Encoding UTF8

        # Optional: Cluster log
        if ($IncludeClusterLog) {
            try { Get-ClusterLog -Cluster $cl -UseLocalTime -Destination $OutputPath | Out-Null } catch { }
        }

        # ---- HTML Overview ----
        $htmlPath = Join-Path $OutputPath 'HealthSummary.html'
        $badgeColor = switch ($summary.OverallState) {
            'Healthy'  { '#2e7d32' }
            'Warning'  { '#ed6c02' }
            'Degraded' { '#c62828' }
            default    { '#555555' }
        }

        $overviewHtml = @"
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8"/>
<title>Cluster Health Overview</title>
<style>
body { font-family: Segoe UI, Arial; margin: 24px; color: #202124; }
.badge { display:inline-block; padding:6px 10px; border-radius:16px; color:#fff; font-weight:600; }
table { border-collapse: collapse; width: 100%; margin-top:10px; }
th, td { border: 1px solid #e5e7eb; padding: 8px; text-align: left; font-size: 14px; }
th { background-color: #f9fafb; }
li.error { color:#c62828; }
li.warning { color:#ed6c02; }
</style>
</head>
<body>
<h1>Cluster Health Overview</h1>
<p><strong>Cluster:</strong> $($summary.ClusterName)</p>
<p><strong>Generated:</strong> $($summary.Timestamp)</p>
<p><strong>Overall State:</strong> <span class="badge" style="background:$badgeColor">$($summary.OverallState)</span></p>
<p><a href="HealthDetails.html">View Detailed Report</a></p>
<h2>Nodes</h2>
<table><tr><th>Name</th><th>State</th><th>DrainStatus</th></tr>
"@
        foreach ($n in $summary.Nodes) {
            $overviewHtml += "<tr><td>$($n.Name)</td><td>$($n.State)</td><td>$($n.DrainStatus)</td></tr>"
        }
        $overviewHtml += "</table></body></html>"
        $overviewHtml | Out-File -FilePath $htmlPath -Encoding UTF8

        # ---- HTML Details ----
        $detailsPath = Join-Path $OutputPath 'HealthDetails.html'
        $jsonObj = Get-Content -Path $jsonPath -Raw | ConvertFrom-Json

        function Encode-ForHtml {
            param([string]$Text)
            if ($null -eq $Text) { return '' }
            return ($Text -replace '&','&amp;' -replace '<','&lt;' -replace '>','&gt;' -replace '"','&quot;' -replace "'","&#39;")
        }

        $detailsHtml = @"
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8"/>
<title>Cluster Health Details</title>
<style>
body { font-family: Segoe UI, Arial; margin: 24px; color: #202124; }
details summary { cursor: pointer; font-weight:600; padding:6px; background:#f3f4f6; border-radius:4px; margin-bottom:6px; }
details[open] summary { background:#e5e7eb; }
table { border-collapse: collapse; width: 100%; margin-top:10px; }
th, td { border: 1px solid #e5e7eb; padding: 8px; text-align: left; font-size: 14px; }
th { background-color: #f9fafb; }
.badge-ok { background:#6b7280; color:#fff; padding:2px 8px; border-radius:12px; }
.badge-warn { background:#ed6c02; color:#fff; padding:2px 8px; border-radius:12px; }
.badge-err { background:#c62828; color:#fff; padding:2px 8px; border-radius:12px; }
</style>
</head>
<body>
<h1>Cluster Health Details</h1>
<h2>Recent Events (last $MaxEventHours h)</h2>
"@
        foreach ($ev in $jsonObj.RecentEvents) {
            $detailsHtml += "<details><summary><strong>Node:</strong> $($ev.Node) — <em>$($ev.Count) events</em></summary>`n"
            $detailsHtml += "<table><tr><th>Time</th><th>Provider</th><th>ID</th><th>Level</th><th>Message</th></tr>`n"
            foreach ($e in $ev.Sample) {
                $levelBadge = switch ($e.LevelDisplayName) {
                    'Error'   { "<span class='badge-err'>Error</span>" }
                    'Warning' { "<span class='badge-warn'>Warning</span>" }
                    default   { "<span class='badge-ok'>$($e.LevelDisplayName)</span>" }
                }
                $msgShort = if ($e.Message.Length -gt 80) { ($e.Message.Substring(0,80) + '...') } else { $e.Message }
                $tooltip = Encode-ForHtml -Text $e.Message
                $detailsHtml += "<tr><td>$($e.TimeCreated)</td><td>$($e.ProviderName)</td><td>$($e.Id)</td><td>$levelBadge</td><td title='$tooltip'>$msgShort</td></tr>`n"
            }
            $detailsHtml += "</table></details>`n"
        }
        $detailsHtml += "</body></html>"
        $detailsHtml | Out-File -FilePath $detailsPath -Encoding UTF8

        Write-Host "Reports generated:"
        Write-Host "JSON   : $jsonPath"
        Write-Host "HTML   : $htmlPath"
        Write-Host "Detail : $detailsPath"

    } catch {
        Write-Error ("Health check failed: {0}" -f $_.Exception.Message)
    }
}
```
