# Network Change Checker

## Powershell
### Network Change Checker
>Open Powershell or Powershell ISE
>
>Paste the following and run
``` Powershell
$LookBackHours = 72
$OutputFolder  = "C:\Temp\Network_Change_Check"

# Keywords/phrases to search for in event messages
$KeywordPatterns = @(
    'disconnect',
    'disconnected',
    'not connected',
    'media disconnected',
    'network cable unplugged',
    'link down',
    'link is down',
    'link has been disconnected',
    'network connectivity',
    'lost connectivity',
    'limited connectivity',
    'no network',
    'no internet',
    'dhcp',
    'lease',
    'address changed',
    'ip address',
    'default gateway',
    'dns server',
    'network profile',
    'adapter',
    'nic'
)

# Provider name patterns commonly associated with network changes
$ProviderPatterns = @(
    'Tcpip',
    'Tcpip6',
    'Microsoft-Windows-TCPIP',
    'Microsoft-Windows-NlaSvc',
    'Microsoft-Windows-NetworkProfile',
    'Dhcp-Client',
    'NetBT',
    'e1',
    'igb',
    'bnx',
    'b57',
    'mlx',
    'ndis',
    'Intel',
    'Broadcom',
    'Mellanox',
    'Realtek'
)

# Event logs to inspect (script will only query logs that exist)
$LogsToCheck = @(
    'System',
    'Application',
    'Microsoft-Windows-NetworkProfile/Operational',
    'Microsoft-Windows-NlaSvc/Operational',
    'Microsoft-Windows-Dhcp-Client/Admin',
    'Microsoft-Windows-Dhcp-Client/Operational',
    'Microsoft-Windows-TCPIP/Operational',
    'Microsoft-Windows-WLAN-AutoConfig/Operational'
)

# -----------------------------
# Derived values
# -----------------------------
$ComputerName = $env:COMPUTERNAME
$StartTime    = (Get-Date).AddHours(-$LookBackHours)
$EndTime      = Get-Date
$TimeStamp    = Get-Date -Format "yyyyMMdd_HHmmss"

# Create output folder
if (-not (Test-Path $OutputFolder)) {
    New-Item -Path $OutputFolder -ItemType Directory -Force | Out-Null
}

# Output files
$AdaptersCsv   = Join-Path $OutputFolder "$ComputerName`_NetworkAdapters_$TimeStamp.csv"
$EventsCsv     = Join-Path $OutputFolder "$ComputerName`_NetworkEvents_$TimeStamp.csv"
$SummaryCsv    = Join-Path $OutputFolder "$ComputerName`_NetworkEventSummary_$TimeStamp.csv"
$SummaryTxt    = Join-Path $OutputFolder "$ComputerName`_NetworkCheckSummary_$TimeStamp.txt"

# -----------------------------
# Helper functions
# -----------------------------
function Write-Section {
    param([string]$Title)

    Write-Host ""
    Write-Host ("=" * 100) -ForegroundColor DarkCyan
    Write-Host (" {0}" -f $Title) -ForegroundColor Cyan
    Write-Host ("=" * 100) -ForegroundColor DarkCyan
}

function Get-SafeEventMessage {
    param([Parameter(Mandatory=$true)]$Event)

    $message = $null

    try {
        $message = $Event.FormatDescription()
    } catch {
        try {
            $message = $Event.Message
        } catch {
            $message = $null
        }
    }

    if ([string]::IsNullOrWhiteSpace($message)) {
        $message = "<Unable to render event message>"
    }

    return ($message -replace "`r`n", " | ")
}

function Convert-ToEventObject {
    param(
        [Parameter(Mandatory=$true)]
        [object[]]$Events
    )

    $list = @()

    foreach ($evt in $Events) {
        $msg = Get-SafeEventMessage -Event $evt

        $list += [pscustomobject]@{
            Server       = $ComputerName
            TimeCreated  = $evt.TimeCreated
            LogName      = $evt.LogName
            ProviderName = $evt.ProviderName
            Id           = $evt.Id
            Level        = $evt.LevelDisplayName
            MachineName  = $evt.MachineName
            RecordId     = $evt.RecordId
            ProcessId    = $evt.ProcessId
            ThreadId     = $evt.ThreadId
            Message      = $msg
        }
    }

    return $list
}

function Test-AnyRegexMatch {
    param(
        [string]$InputText,
        [string[]]$Patterns
    )

    if ([string]::IsNullOrWhiteSpace($InputText)) {
        return $false
    }

    foreach ($pattern in $Patterns) {
        if ($InputText -match $pattern) {
            return $true
        }
    }

    return $false
}

# -----------------------------
# Start
# -----------------------------
Write-Section "Network Change / Disconnect Investigation"
Write-Host "Server       : $ComputerName" -ForegroundColor Yellow
Write-Host "Lookback     : $LookBackHours hours" -ForegroundColor Yellow
Write-Host "Start Time   : $StartTime" -ForegroundColor Yellow
Write-Host "End Time     : $EndTime" -ForegroundColor Yellow
Write-Host "Output Folder: $OutputFolder" -ForegroundColor Yellow

# -----------------------------
# Current adapter status
# -----------------------------
Write-Section "Current Network Adapter Status"

$adapterResults = @()

if (Get-Command Get-NetAdapter -ErrorAction SilentlyContinue) {
    $netAdapters = Get-NetAdapter -ErrorAction SilentlyContinue | Sort-Object Name

    foreach ($adapter in $netAdapters) {
        $ipInfo = $null
        $ipv4   = $null
        $gateway = $null
        $dns    = $null

        if (Get-Command Get-NetIPConfiguration -ErrorAction SilentlyContinue) {
            try {
                $ipInfo = Get-NetIPConfiguration -InterfaceIndex $adapter.ifIndex -ErrorAction SilentlyContinue
            } catch {
                $ipInfo = $null
            }
        }

        if ($ipInfo) {
            if ($ipInfo.IPv4Address) {
                $ipv4 = ($ipInfo.IPv4Address | ForEach-Object { $_.IPv4Address }) -join ', '
            }
            if ($ipInfo.IPv4DefaultGateway) {
                $gateway = ($ipInfo.IPv4DefaultGateway | ForEach-Object { $_.NextHop }) -join ', '
            }
            if ($ipInfo.DNSServer) {
                $dns = ($ipInfo.DNSServer.ServerAddresses) -join ', '
            }
        }

        $adapterResults += [pscustomobject]@{
            Name                 = $adapter.Name
            InterfaceDescription = $adapter.InterfaceDescription
            Status               = $adapter.Status
            LinkSpeed            = $adapter.LinkSpeed
            MacAddress           = $adapter.MacAddress
            InterfaceIndex       = $adapter.ifIndex
            IPv4Address          = $ipv4
            DefaultGateway       = $gateway
            DnsServers           = $dns
        }
    }
}
else {
    # Fallback for older environments
    $wmiAdapters = Get-WmiObject Win32_NetworkAdapter -Filter "PhysicalAdapter = True" | Sort-Object NetConnectionID

    foreach ($adapter in $wmiAdapters) {
        $adapterResults += [pscustomobject]@{
            Name                 = $adapter.NetConnectionID
            InterfaceDescription = $adapter.Name
            Status               = $adapter.NetConnectionStatus
            LinkSpeed            = $adapter.Speed
            MacAddress           = $adapter.MACAddress
            InterfaceIndex       = $adapter.InterfaceIndex
            IPv4Address          = $null
            DefaultGateway       = $null
            DnsServers           = $null
        }
    }
}

if ($adapterResults.Count -gt 0) {
    $adapterResults | Export-Csv -Path $AdaptersCsv -NoTypeInformation -Encoding UTF8
    $adapterResults | Format-Table -AutoSize
}
else {
    Write-Host "No network adapters were returned." -ForegroundColor Yellow
}

# -----------------------------
# Determine which logs exist
# -----------------------------
Write-Section "Checking available event logs"

$existingLogs = @()
foreach ($log in $LogsToCheck) {
    try {
        $logInfo = Get-WinEvent -ListLog $log -ErrorAction Stop
        if ($logInfo) {
            $existingLogs += $log
            Write-Host ("[FOUND]   {0}" -f $log) -ForegroundColor Green
        }
    } catch {
        Write-Host ("[MISSING] {0}" -f $log) -ForegroundColor DarkYellow
    }
}

# -----------------------------
# Collect matching events
# -----------------------------
Write-Section "Collecting network-related events"

$matchedRawEvents = @()

foreach ($logName in $existingLogs) {
    Write-Host ("Scanning log: {0}" -f $logName) -ForegroundColor Gray

    try {
        $events = Get-WinEvent -FilterHashtable @{
            LogName   = $logName
            StartTime = $StartTime
            EndTime   = $EndTime
        } -ErrorAction SilentlyContinue

        foreach ($evt in $events) {
            $provider = $evt.ProviderName
            $message  = Get-SafeEventMessage -Event $evt

            $providerMatch = Test-AnyRegexMatch -InputText $provider -Patterns $ProviderPatterns
            $messageMatch  = Test-AnyRegexMatch -InputText $message -Patterns $KeywordPatterns

            # Keep events that are clearly network-related by provider or message
            if ($providerMatch -or $messageMatch) {
                $matchedRawEvents += $evt
            }
        }
    } catch {
        Write-Host ("Failed to scan log: {0}" -f $logName) -ForegroundColor Red
    }
}

# Convert and de-duplicate
$eventResults = @()
if ($matchedRawEvents.Count -gt 0) {
    $eventResults = Convert-ToEventObject -Events $matchedRawEvents |
        Sort-Object LogName, RecordId -Unique |
        Sort-Object TimeCreated
}

# -----------------------------
# Build summary
# -----------------------------
$summaryResults = @()

if ($eventResults.Count -gt 0) {
    $grouped = $eventResults | Group-Object ProviderName, Id | Sort-Object Count -Descending

    foreach ($g in $grouped) {
        $parts = $g.Name -split ', '
        $providerName = $parts[0]
        $eventId = $parts[1]

        $summaryResults += [pscustomobject]@{
            ProviderName = $providerName
            EventId      = $eventId
            Count        = $g.Count
        }
    }
}

# -----------------------------
# Export results
# -----------------------------
if ($eventResults.Count -gt 0) {
    $eventResults | Export-Csv -Path $EventsCsv -NoTypeInformation -Encoding UTF8
}

if ($summaryResults.Count -gt 0) {
    $summaryResults | Export-Csv -Path $SummaryCsv -NoTypeInformation -Encoding UTF8
}

# -----------------------------
# Console output
# -----------------------------
Write-Section "Summary"

Write-Host ("Total matching events found: {0}" -f $eventResults.Count) -ForegroundColor Yellow
Write-Host ("Adapter report            : {0}" -f $AdaptersCsv) -ForegroundColor Yellow
Write-Host ("Event report              : {0}" -f $EventsCsv) -ForegroundColor Yellow
Write-Host ("Summary report            : {0}" -f $SummaryCsv) -ForegroundColor Yellow

if ($summaryResults.Count -gt 0) {
    $summaryResults | Format-Table -AutoSize
}
else {
    Write-Host "No matching network-related events were found in the selected time window." -ForegroundColor Yellow
}

Write-Section "Most Recent Matching Events"

if ($eventResults.Count -gt 0) {
    $eventResults |
        Sort-Object TimeCreated -Descending |
        Select-Object -First 50 TimeCreated, LogName, ProviderName, Id, Level, Message |
        Format-Table -Wrap -AutoSize
}
else {
    Write-Host "No events to display." -ForegroundColor Yellow
}

# -----------------------------
# Save text summary
# -----------------------------
$txtLines = @()
$txtLines += "Network Change / Disconnect Investigation Summary"
$txtLines += "Server       : $ComputerName"
$txtLines += "Lookback     : $LookBackHours hours"
$txtLines += "Start Time   : $StartTime"
$txtLines += "End Time     : $EndTime"
$txtLines += ""
$txtLines += "Output Files:"
$txtLines += " - Adapters: $AdaptersCsv"
$txtLines += " - Events  : $EventsCsv"
$txtLines += " - Summary : $SummaryCsv"
$txtLines += ""
$txtLines += ("Total matching events: {0}" -f $eventResults.Count)
$txtLines += ""

if ($summaryResults.Count -gt 0) {
    $txtLines += "Top Providers / Event IDs:"
    $txtLines += ($summaryResults | Out-String)
} else {
    $txtLines += "No matching network-related events were found."
}

$txtLines | Set-Content -Path $SummaryTxt -Encoding UTF8

Write-Section "Completed"
Write-Host ("Text summary : {0}" -f $SummaryTxt) -ForegroundColor Green
Write-Host "Done." -ForegroundColor Green
```
