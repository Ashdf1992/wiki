# Checking AD Replication

## Version 2.0 Outputs a Nice report
>Open Powershell or Powershell ISE

>Paste the following and save it to a location
```Powershell
<#
.SYNOPSIS
    Validates required Active Directory ports between domain controllers and produces a client-ready HTML report.

.DESCRIPTION
    - Tests TCP ports using a real TCP connect with timeout.
    - Tests DNS (UDP/TCP 53) by querying the target DC as a DNS server.
    - Tests NTP (UDP 123) using w32tm /stripchart (best-effort UDP validation).
    - Supports local-origin scanning or full mesh between DCs via PowerShell Remoting.
    - Generates a polished HTML report with summary and detailed results.

.PARAMETER DomainControllers
    One or more DC hostnames/FQDNs to validate.

.PARAMETER FullMesh
    When specified, runs tests from each DC to all peers (Invoke-Command/WinRM).

.PARAMETER Credential
    PSCredential for remoting to the DCs in FullMesh mode.

.PARAMETER ReportPath
    Output path to the HTML report.

.PARAMETER TcpTimeoutMs
    Timeout for TCP port tests (default 2000 ms).

.PARAMETER MaxConcurrency
    (Reserved for future use) Maximum concurrency hint. Current implementation runs sequentially for remote stability.

.NOTES
    Author: Microsoft M365 Copilot (Enterprise)
    Version: 1.1 (remoting function injection fix, colon interpolation fix)
#>

[CmdletBinding()]
param(
    [Parameter(Mandatory=$true)]
    [ValidateNotNullOrEmpty()]
    [string[]]$DomainControllers,

    [switch]$FullMesh,

    [pscredential]$Credential,

    [string]$ReportPath = (Join-Path -Path (Get-Location) -ChildPath ("DC-Port-Connectivity-Report_{0}.html" -f (Get-Date -Format 'yyyyMMdd_HHmmss'))),

    [ValidateRange(500,10000)]
    [int]$TcpTimeoutMs = 2000,

    [ValidateRange(1,128)]
    [int]$MaxConcurrency = 32
)


function ConvertTo-HtmlSafe {
    param([string]$Text)
    if ([string]::IsNullOrEmpty($Text)) { return '' }
    try {
        return [System.Net.WebUtility]::HtmlEncode($Text)
    } catch {
        # Fallback manual encoding (very conservative)
        return ($Text `
            -replace '&','&amp;' `
            -replace '<','&lt;' `
            -replace '>','&gt;' `
            -replace '"','&quot;' `
            -replace "'","&#39;")
    }
}


# ---------------------- Test Catalogue ----------------------
$TestCatalogue = @(
    [PSCustomObject]@{ Service='DNS';                  Category='Core Name Resolution'; Protocol='UDP'; Port=53;   Test='DnsQuery' }
    [PSCustomObject]@{ Service='DNS';                  Category='Core Name Resolution'; Protocol='TCP'; Port=53;   Test='Tcp'      }
    [PSCustomObject]@{ Service='Kerberos';             Category='Authentication';       Protocol='TCP'; Port=88;   Test='Tcp'      }
    [PSCustomObject]@{ Service='Kerberos PWD Change';  Category='Authentication';       Protocol='TCP'; Port=464;  Test='Tcp'      }
    [PSCustomObject]@{ Service='LDAP';                 Category='Directory';            Protocol='TCP'; Port=389;  Test='Tcp'      }
    [PSCustomObject]@{ Service='LDAPS';                Category='Directory';            Protocol='TCP'; Port=636;  Test='Tcp'      }
    [PSCustomObject]@{ Service='Global Catalog';       Category='Directory';            Protocol='TCP'; Port=3268; Test='Tcp'      }
    [PSCustomObject]@{ Service='Global Catalog (SSL)'; Category='Directory';            Protocol='TCP'; Port=3269; Test='Tcp'      }
    [PSCustomObject]@{ Service='SMB/CIFS';             Category='File/NetLogon';        Protocol='TCP'; Port=445;  Test='Tcp'      }
    [PSCustomObject]@{ Service='RPC Endpoint Mapper';  Category='RPC';                  Protocol='TCP'; Port=135;  Test='Tcp'      }
    [PSCustomObject]@{ Service='DFSR';                 Category='Replication';          Protocol='TCP'; Port=5722; Test='Tcp'      }
    [PSCustomObject]@{ Service='AD Web Services';      Category='Management';           Protocol='TCP'; Port=9389; Test='Tcp'      }
    [PSCustomObject]@{ Service='NTP/W32Time';          Category='Time';                 Protocol='UDP'; Port=123;  Test='Ntp'      }
)

# ---------------------- Local Utilities ---------------------
function Test-TcpPort {
    param(
        [Parameter(Mandatory)] [string]$ComputerName,
        [Parameter(Mandatory)] [int]$Port,
        [int]$TimeoutMs = 2000
    )
    try {
        $tcp = New-Object System.Net.Sockets.TcpClient
        $iar = $tcp.BeginConnect($ComputerName, $Port, $null, $null)
        $connected = $iar.AsyncWaitHandle.WaitOne($TimeoutMs, $false)
        if (-not $connected) {
            $tcp.Close()
            return [PSCustomObject]@{ Open=$false; LatencyMs=$null; Note='Timeout' }
        }
        $sw = [System.Diagnostics.Stopwatch]::StartNew()
        $tcp.EndConnect($iar)
        $sw.Stop()
        $lat = [int]$sw.ElapsedMilliseconds
        $tcp.Close()
        return [PSCustomObject]@{ Open=$true; LatencyMs=$lat; Note=$null }
    } catch {
        return [PSCustomObject]@{ Open=$false; LatencyMs=$null; Note=$_.Exception.Message }
    }
}

function Test-DnsViaServer {
    param(
        [Parameter(Mandatory)] [string]$DnsServer,
        [string]$QueryName = $env:COMPUTERNAME
    )
    try {
        $res = Resolve-DnsName -Server $DnsServer -Name $QueryName -Type A -ErrorAction Stop
        return [PSCustomObject]@{ Success=$true; Note='Resolve-DnsName succeeded' }
    } catch {
        return [PSCustomObject]@{ Success=$false; Note=$_.Exception.Message }
    }
}

function Test-NtpUdp {
    param(
        [Parameter(Mandatory)] [string]$Target
    )
    try {
        $output = & w32tm /stripchart /computer:$Target /samples:1 /dataonly 2>&1
        if ($LASTEXITCODE -eq 0 -and ($output -match 'samples:')) {
            return [PSCustomObject]@{ Success=$true; Note='w32tm sample received' }
        } else {
            return [PSCustomObject]@{ Success=$false; Note=($output | Select-Object -First 1) }
        }
    } catch {
        return [PSCustomObject]@{ Success=$false; Note=$_.Exception.Message }
    }
}

function Invoke-ConnectivitySuite {
    param(
        [Parameter(Mandatory)] [string]$SourceHost,
        [Parameter(Mandatory)] [string[]]$Targets,
        [Parameter(Mandatory)] $Catalogue,
        [int]$TcpTimeoutMs = 2000
    )

    $results = @()
    foreach ($t in $Targets) {
        if ($t -ieq $SourceHost) { continue }
        foreach ($test in $Catalogue) {
            $service   = $test.Service
            $category  = $test.Category
            $protocol  = $test.Protocol
            $port      = $test.Port
            $testType  = $test.Test

            $open = $false
            $lat  = $null
            $note = $null
            try {
                switch ($testType) {
                    'Tcp' {
                        $r = Test-TcpPort -ComputerName $t -Port $port -TimeoutMs $TcpTimeoutMs
                        $open = $r.Open; $lat = $r.LatencyMs; $note = $r.Note
                    }
                    'DnsQuery' {
                        $r = Test-DnsViaServer -DnsServer $t -QueryName $t
                        $open = $r.Success; $note = $r.Note
                    }
                    'Ntp' {
                        $r = Test-NtpUdp -Target $t
                        $open = $r.Success; $note = $r.Note
                    }
                }
            } catch {
                $open = $false; $note = $_.Exception.Message
            }
            $results += [PSCustomObject]@{
                Timestamp  = (Get-Date)
                Source     = $SourceHost
                Target     = $t
                Category   = $category
                Service    = $service
                Protocol   = $protocol
                Port       = $port
                IsOpen     = [bool]$open
                LatencyMs  = $lat
                Detail     = $note
            }
        }
    }
    return $results
}

# ---------------------- Execution ---------------------------
$allResults = @()

if ($FullMesh) {
    Write-Host "Running FULL MESH validation using PowerShell Remoting..." -ForegroundColor Cyan

    foreach ($source in $DomainControllers) {
        Write-Host "→ Source: $source" -ForegroundColor Yellow

        $remoteScript = {
            param($Targets,$Catalogue,$TcpTimeoutMs)

            function Test-TcpPort {
                param([string]$ComputerName,[int]$Port,[int]$TimeoutMs = 2000)
                try {
                    $tcp = New-Object System.Net.Sockets.TcpClient
                    $iar = $tcp.BeginConnect($ComputerName, $Port, $null, $null)
                    $connected = $iar.AsyncWaitHandle.WaitOne($TimeoutMs, $false)
                    if (-not $connected) {
                        $tcp.Close()
                        return [PSCustomObject]@{ Open=$false; LatencyMs=$null; Note='Timeout' }
                    }
                    $sw = [System.Diagnostics.Stopwatch]::StartNew()
                    $tcp.EndConnect($iar)
                    $sw.Stop()
                    $lat = [int]$sw.ElapsedMilliseconds
                    $tcp.Close()
                    return [PSCustomObject]@{ Open=$true; LatencyMs=$lat; Note=$null }
                } catch {
                    return [PSCustomObject]@{ Open=$false; LatencyMs=$null; Note=$_.Exception.Message }
                }
            }

            function Test-DnsViaServer {
                param([string]$DnsServer,[string]$QueryName = $env:COMPUTERNAME)
                try {
                    $res = Resolve-DnsName -Server $DnsServer -Name $QueryName -Type A -ErrorAction Stop
                    return [PSCustomObject]@{ Success=$true; Note='Resolve-DnsName succeeded' }
                } catch {
                    return [PSCustomObject]@{ Success=$false; Note=$_.Exception.Message }
                }
            }

            function Test-NtpUdp {
                param([string]$Target)
                try {
                    $output = & w32tm /stripchart /computer:$Target /samples:1 /dataonly 2>&1
                    if ($LASTEXITCODE -eq 0 -and ($output -match 'samples:')) {
                        return [PSCustomObject]@{ Success=$true; Note='w32tm sample received' }
                    } else {
                        return [PSCustomObject]@{ Success=$false; Note=($output | Select-Object -First 1) }
                    }
                } catch {
                    return [PSCustomObject]@{ Success=$false; Note=$_.Exception.Message }
                }
            }

            function Invoke-ConnectivitySuite {
                param([string]$SourceHost,[string[]]$Targets,$Catalogue,[int]$TcpTimeoutMs = 2000)
                $results = @()
                foreach ($t in $Targets) {
                    if ($t -ieq $SourceHost) { continue }
                    foreach ($test in $Catalogue) {
                        $service   = $test.Service
                        $category  = $test.Category
                        $protocol  = $test.Protocol
                        $port      = $test.Port
                        $testType  = $test.Test

                        $open = $false
                        $lat  = $null
                        $note = $null
                        try {
                            switch ($testType) {
                                'Tcp' {
                                    $r = Test-TcpPort -ComputerName $t -Port $port -TimeoutMs $TcpTimeoutMs
                                    $open = $r.Open; $lat = $r.LatencyMs; $note = $r.Note
                                }
                                'DnsQuery' {
                                    $r = Test-DnsViaServer -DnsServer $t -QueryName $t
                                    $open = $r.Success; $note = $r.Note
                                }
                                'Ntp' {
                                    $r = Test-NtpUdp -Target $t
                                    $open = $r.Success; $note = $r.Note
                                }
                            }
                        } catch {
                            $open = $false; $note = $_.Exception.Message
                        }
                        $results += [PSCustomObject]@{
                            Timestamp  = (Get-Date)
                            Source     = $SourceHost
                            Target     = $t
                            Category   = $category
                            Service    = $service
                            Protocol   = $protocol
                            Port       = $port
                            IsOpen     = [bool]$open
                            LatencyMs  = $lat
                            Detail     = $note
                        }
                    }
                }
                return $results
            }

            Invoke-ConnectivitySuite -SourceHost $env:COMPUTERNAME -Targets $Targets -Catalogue $Catalogue -TcpTimeoutMs $TcpTimeoutMs
        }

        try {
            $remoteResults = Invoke-Command -ComputerName $source -Credential $Credential -ScriptBlock $remoteScript -ArgumentList ($DomainControllers,$TestCatalogue,$TcpTimeoutMs) -ErrorAction Stop
            $allResults += $remoteResults
        } catch {
            Write-Warning ("Failed to execute on $($source): $($_.Exception.Message). Falling back to local-origin tests to its peers.")
            $allResults += Invoke-ConnectivitySuite -SourceHost $env:COMPUTERNAME -Targets $DomainControllers -Catalogue $TestCatalogue -TcpTimeoutMs $TcpTimeoutMs
        }
    }
} else {
    Write-Host "Running LOCAL-ORIGIN validation to each DC..." -ForegroundColor Cyan
    $allResults += Invoke-ConnectivitySuite -SourceHost $env:COMPUTERNAME -Targets $DomainControllers -Catalogue $TestCatalogue -TcpTimeoutMs $TcpTimeoutMs
}

# ---------------------- Report Generation -------------------
$pairSummary =
    $allResults |
    Group-Object Source,Target |
    ForEach-Object {
        $total   = $_.Group.Count
        $open    = ($_.Group | Where-Object IsOpen).Count
        $status  = if ($open -eq $total) { 'Pass' } elseif ($open -eq 0) { 'Fail' } else { 'Partial' }
        [PSCustomObject]@{
            Source = ($_.Name -split ',')[0]
            Target = ($_.Name -split ',')[1]
            TotalChecks = $total
            OpenCount   = $open
            Status      = $status
        }
    } | Sort-Object Source,Target

$style = @"
<style>
body { font-family: Segoe UI, Arial, sans-serif; margin: 24px; color: #222; }
h1 { font-size: 22px; margin-bottom: 4px; }
h2 { font-size: 18px; margin-top: 28px; }
small { color: #555; }
table { border-collapse: collapse; width: 100%; margin-top: 10px; }
th, td { border: 1px solid #e2e2e2; padding: 6px 8px; font-size: 13px; text-align: left; }
th { background: #f6f6f6; }
.badge { display:inline-block; padding:2px 8px; border-radius: 10px; font-weight:600; font-size: 12px; }
.pass { background: #e6ffed; color: #0a7a3d; border: 1px solid #b7f0c7; }
.partial { background: #fff4e5; color: #a45b00; border: 1px solid #ffd9a8; }
.fail { background: #ffecec; color: #a40000; border: 1px solid #f2b5b5; }
.ok { background:#e6ffed; }
.err { background:#ffecec; }
.footer { margin-top: 24px; color:#666; font-size:12px; }
</style>
"@

$execSummaryRows = ($pairSummary | ForEach-Object {
    $cls = switch ($_.Status) {
        'Pass'    { 'badge pass' }
        'Partial' { 'badge partial' }
        default   { 'badge fail' }
    }
    "<tr><td>$($_.Source)</td><td>$($_.Target)</td><td>$($_.OpenCount)/$($_.TotalChecks)</td><td><span class='$cls'>$($_.Status)</span></td></tr>"
}) -join [Environment]::NewLine

$detailHtml = ""
$groupedPairs = $allResults | Sort-Object Source,Target,Category,Service | Group-Object Source,Target
foreach ($pair in $groupedPairs) {
    $source = ($pair.Name -split ',')[0]
    $target = ($pair.Name -split ',')[1]
    $detailHtml += "<h2>Details: $(ConvertTo-HtmlSafe $source) → $(ConvertTo-HtmlSafe $target)</h2>`n"
    $detailHtml += "<table><thead><tr><th>Category</th><th>Service</th><th>Protocol</th><th>Port</th><th>Status</th><th>Latency (ms)</th><th>Detail</th></tr></thead><tbody>"
foreach ($rec in $pair.Group) {
    $class  = if ($rec.IsOpen) { 'ok' } else { 'err' }
    $status = if ($rec.IsOpen) { 'Open' } else { 'Closed/No Response' }
    $lat    = if ($rec.LatencyMs) { $rec.LatencyMs } else { '' }
    $det    = ConvertTo-HtmlSafe $rec.Detail

    # (Optional) defensively encode text columns too, in case device names or categories contain special chars
    $cat  = ConvertTo-HtmlSafe $rec.Category
    $svc  = ConvertTo-HtmlSafe $rec.Service
    $prot = ConvertTo-HtmlSafe $rec.Protocol

    $detailHtml += "<tr class='$class'><td>$cat</td><td>$svc</td><td>$prot</td><td>$($rec.Port)</td><td>$status</td><td>$lat</td><td>$det</td></tr>"
}
    $detailHtml += "</tbody></table>"
}

$header = "<h1>Domain Controller Port Connectivity Report</h1><small>Generated: $(Get-Date)</small>"
$execSummary = @"
<h2>Executive Summary</h2>
<table>
<thead><tr><th>Source</th><th>Target</th><th>Open/Total</th><th>Status</th></tr></thead>
<tbody>
$execSummaryRows
</tbody>
</table>
"@

$notes = @"
<div class='footer'>
<strong>Notes:</strong>
<ul>
<li>TCP checks are validated with an actual TCP connect (timeout: ${TcpTimeoutMs} ms).</li>
<li>DNS (UDP 53) is validated by issuing a DNS A query to the target DC as the server.</li>
<li>NTP (UDP 123) is validated using <code>w32tm /stripchart</code> (best-effort UDP verification).</li>
<li>RPC dynamic ports are not individually probed; Endpoint Mapper (TCP 135) and key AD services are validated.</li>
</ul>
</div>
"@

$report = @"
<html><head>
<meta charset='utf-8' />
<title>DC Port Connectivity Report</title>
$style
</head><body>
$header
$execSummary
$detailHtml
$notes
</body></html>
"@

$null = New-Item -ItemType Directory -Force -Path (Split-Path -Path $ReportPath) | Out-Null
$report | Out-File -FilePath $ReportPath -Encoding UTF8

Write-Host ""
```
> Within Powershell ISE, browse to the location where you saved the above script.
> Open a new tab in Powershell ISE when you are at the location where you have saved the above script
> Paste the following and run it.
```Powershell
.\Test-ADDCConnectivity.ps1 `
  -DomainControllers 'VM-Ashtest-UKS-01','VM-Ashtest-UKS-02' `
  -ReportPath '.\Report_FullMesh.html'
```

> For a full mesh test with WinRM Enabled run this version instead
```Powershell
 Full mesh (WinRM enabled)
.\Test-ADDCConnectivity.ps1 `
  -DomainControllers 'VM-Ashtest-UKS-01','VM-Ashtest-UKS-02' `
  -FullMesh `
  -Credential (Get-Credential) `
  -ReportPath '.\Report_FullMesh.html'

```
> The output will be a Report_FullMesh html file where you ran the script. 


## Version 1.0
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
# Ensure the Active Directory module is available
Import-Module ActiveDirectory

# Get all Domain Controllers in the domain
$domainControllers = Get-ADDomainController -Filter *

foreach ($dc in $domainControllers) {
    Write-Host "Checking replication status for: $($dc.HostName)" -ForegroundColor Cyan

    # Get replication partner metadata
    $replicationPartners = Get-ADReplicationPartnerMetadata -Target $dc.HostName -Scope Server

    foreach ($partner in $replicationPartners) {
        $lastSuccess = $partner.LastReplicationSuccess
        $partnerName = $partner.Partner

        Write-Host "  Partner: $partnerName"
        Write-Host "    Last Successful Replication: $lastSuccess"
    }

    # Check for replication failures
    $failures = Get-ADReplicationFailure -Target $dc.HostName

    if ($failures.Count -gt 0) {
        Write-Host "  Replication failures detected:" -ForegroundColor Red
        foreach ($failure in $failures) {
            Write-Host "    Partner: $($failure.Partner)"
            Write-Host "    First Failure Time: $($failure.FirstFailureTime)"
            Write-Host "    Failure Count: $($failure.FailureCount)"
            Write-Host "    Last Error: $($failure.LastErrorMessage)"
        }
    } else {
        Write-Host "  No replication failures detected." -ForegroundColor Green
    }

    Write-Host ""
}

```
