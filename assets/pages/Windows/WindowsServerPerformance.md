# Windows Server — Performance Monitoring & Tuning

This page documents how to monitor, analyze, and optimize Windows Server performance. It outlines key performance metrics (CPU, memory, disk I/O, network), recommended counters to collect, common bottlenecks, diagnostic and tracing tools (PerfMon, Resource Monitor, Windows Performance Recorder, Sysinternals), baseline and trending workflows, and practical tuning and troubleshooting guidance for on‑premises and cloud deployments.

```Powershell
# Windows Server General Health Check Script
# Compatible with PowerShell 5.1
# Requires: Administrator privileges
# Modules: No external modules required (uses built-in cmdlets)
# Purpose: Provides a professional, client-friendly overview of Windows Server health
#          including performance, recent errors, Windows Update status, disk space, and key services.

# Check for Administrator privileges
if (-not ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    Write-Host "ERROR: This script must be run as Administrator." -ForegroundColor Red
    Pause
    exit
}

Write-Host "Windows Server General Health Check" -ForegroundColor Cyan
Write-Host ("=" * 80) -ForegroundColor Cyan
Write-Host "Server: $env:COMPUTERNAME   Date: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')" -ForegroundColor Gray
Write-Host ""

# Section 1: System Overview
Write-Host "1. System Overview" -ForegroundColor Yellow
Write-Host ("-" * 50) -ForegroundColor Yellow

$os = Get-CimInstance Win32_OperatingSystem
$uptime = (Get-Date) - $os.LastBootUpTime
$cpu = Get-CimInstance Win32_Processor | Measure-Object -Property LoadPercentage -Average
$memTotal = [math]::Round($os.TotalVisibleMemorySize / 1MB, 2)
$memFree = [math]::Round($os.FreePhysicalMemory / 1MB, 2)
$memUsedPct = [math]::Round((($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / $os.TotalVisibleMemorySize) * 100, 1)

Write-Host "   OS              : $($os.Caption) $($os.OSArchitecture)"
Write-Host "   Uptime          : $($uptime.Days) days, $($uptime.Hours) hours"
Write-Host "   CPU Usage       : $($cpu.Average)%"
Write-Host "   Memory Usage    : $memUsedPct% ($($memTotal - $memFree) GB used of $memTotal GB)"
Write-Host ""

# Section 2: Disk Space
Write-Host "2. Disk Space Usage" -ForegroundColor Yellow
Write-Host ("-" * 50) -ForegroundColor Yellow

Get-WmiObject Win32_LogicalDisk -Filter "DriveType=3" | Select-Object `
    DeviceID,
    @{Name="Size (GB)";Expression={[math]::Round($_.Size/1GB,2)}},
    @{Name="Free (GB)";Expression={[math]::Round($_.FreeSpace/1GB,2)}},
    @{Name="Free %";Expression={[math]::Round(($_.FreeSpace/$_.Size)*100,1)}} |
    Format-Table -AutoSize

Write-Host ""

# Section 3: Recent Critical Events (Last 24 hours)
Write-Host "3. Recent Critical/System Errors (Last 24 hours)" -ForegroundColor Yellow
Write-Host ("-" * 50) -ForegroundColor Yellow

$criticalEvents = Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2; StartTime=(Get-Date).AddHours(-24)} -ErrorAction SilentlyContinue | 
    Select-Object -First 10 TimeCreated, Id, ProviderName, Message

if ($criticalEvents) {
    $criticalEvents | Format-Table TimeCreated, Id, ProviderName -Wrap
    Write-Host "   Showing up to 10 most recent critical/warning events." -ForegroundColor Gray
} else {
    Write-Host "   No critical or error events found in the last 24 hours." -ForegroundColor Green
}
Write-Host ""

# Section 4: Windows Update Status
Write-Host "4. Windows Update Status" -ForegroundColor Yellow
Write-Host ("-" * 50) -ForegroundColor Yellow

try {
    $updateSession = New-Object -ComObject Microsoft.Update.Session
    $updateSearcher = $updateSession.CreateUpdateSearcher()
    $historyCount = $updateSearcher.GetTotalHistoryCount()
    $recentUpdates = $updateSearcher.QueryHistory(1, [math]::Min(10, $historyCount)) | 
        Select-Object Title, Date, ResultCode, @{Name="Result";Expression={
            switch ($_.ResultCode) {
                2 { "Succeeded" }
                3 { "SucceededWithErrors" }
                4 { "Failed" }
                5 { "Aborted" }
                default { "Unknown" }
            }
        }}

    if ($recentUpdates) {
        $lastUpdate = ($recentUpdates | Sort-Object Date -Descending | Select-Object -First 1).Date
        Write-Host "   Last successful update : $lastUpdate"
        Write-Host ""
        $recentUpdates | Sort-Object Date -Descending | Format-Table Date, Result, Title -Wrap
        Write-Host "   Showing up to 10 most recent updates." -ForegroundColor Gray
    } else {
        Write-Host "   No update history found." -ForegroundColor Yellow
    }
} catch {
    Write-Host "   Unable to retrieve Windows Update history (WUA service may be disabled or COM error)." -ForegroundColor Yellow
}
Write-Host ""

# Section 5: Key Performance Counters (sampled over ~6 seconds)
Write-Host "5. Current Performance Counters (Averages)" -ForegroundColor Yellow
Write-Host ("-" * 50) -ForegroundColor Yellow

try {
    $samples = Get-Counter -Counter `
        "\Processor(_Total)\% Processor Time",
        "\Memory\Available MBytes",
        "\Memory\% Committed Bytes In Use",
        "\PhysicalDisk(_Total)\% Disk Time",
        "\PhysicalDisk(_Total)\Avg. Disk Queue Length",
        "\Network Interface(*)\Bytes Total/sec" -SampleInterval 2 -MaxSamples 3 -ErrorAction Stop

    $avgCounters = $samples.CounterSamples | Group-Object Path | ForEach-Object {
        $avg = ($_.Group | Measure-Object -Property CookedValue -Average).Average
        [PSCustomObject]@{
            Counter = ($_.Name -replace '^\\', '' -replace '\\', ' > ')
            Value   = if ($_.Name -like "*\%*") { "{0:N1}%" -f $avg }
                      elseif ($_.Name -like "*Available MBytes*") { "{0:N0} MB" -f $avg }
                      elseif ($_.Name -like "*Bytes Total/sec*") { "{0:N0} B/sec" -f $avg }
                      else { "{0:N2}" -f $avg }
        }
    }
    $avgCounters | Format-Table -AutoSize Counter, Value
} catch {
    Write-Host "   Warning: Unable to retrieve performance counters." -ForegroundColor Yellow
}
Write-Host ""

# Section 6: Health Alerts
Write-Host "6. Health Alerts" -ForegroundColor Yellow
Write-Host ("-" * 50) -ForegroundColor Yellow

$alerts = @()

# High CPU
if ($cpu.Average -gt 80) {
    $alerts += "High CPU usage ($($cpu.Average)%)`n" +
              "   Description: Sustained high CPU can indicate heavy workload, poorly optimized processes, or potential malware.`n" +
              "   Resolution Ideas:`n" +
              "     - Open Task Manager or Resource Monitor to identify top processes.`n" +
              "     - Check for unexpected applications or services consuming CPU.`n" +
              "     - Review scheduled tasks and startup items.`n" +
              "     - Consider adding CPU resources or optimizing workloads."
}

# High Memory
if ($memUsedPct -gt 85) {
    $alerts += "High memory usage ($memUsedPct%)`n" +
              "   Description: Low available memory can cause paging to disk and degrade performance.`n" +
              "   Resolution Ideas:`n" +
              "     - Identify memory-intensive processes in Task Manager.`n" +
              "     - Check for memory leaks in long-running services.`n" +
              "     - Increase physical RAM if feasible.`n" +
              "     - Restart services or server during maintenance window if needed."
}

# Low Disk Space
Get-WmiObject Win32_LogicalDisk -Filter "DriveType=3" | ForEach-Object {
    $freePct = [math]::Round(($_.FreeSpace / $_.Size) * 100, 1)
    if ($freePct -lt 15) {
        $drive = $_.DeviceID
        $alerts += "Low disk space on $drive ($freePct% free)`n" +
                  "   Description: Drives with less than 15% free space risk failure for logs, updates, or temp files.`n" +
                  "   Resolution Ideas:`n" +
                  "     - Clean up temporary files (Disk Cleanup tool).`n" +
                  "     - Move large logs or data to another volume.`n" +
                  "     - Extend volume if possible (Disk Management).`n" +
                  "     - Review old backups or unused applications."
    }
}

# Critical Events
if ($criticalEvents) {
    $alerts += "Critical or error events detected in System log (last 24 hours)`n" +
              "   Description: Recent critical events may indicate hardware issues, driver problems, or service failures.`n" +
              "   Resolution Ideas:`n" +
              "     - Review the events listed above in detail using Event Viewer.`n" +
              "     - Search event IDs on Microsoft Docs or support sites.`n" +
              "     - Check hardware health (if disk/controller events).`n" +
              "     - Update drivers/firmware if related to specific components."
}

# No Recent Updates (more than 30 days)
if ($recentUpdates) {
    $lastSuccess = $recentUpdates | Where-Object Result -eq "Succeeded" | Sort-Object Date -Descending | Select-Object -First 1
    if ($lastSuccess) {
        $daysSince = ((Get-Date) - $lastSuccess.Date).Days
        if ($daysSince -gt 30) {
            $alerts += "No successful Windows Updates in over 30 days (last: $($lastSuccess.Date))`n" +
                      "   Description: Out-of-date servers are vulnerable to security issues and bugs.`n" +
                      "   Resolution Ideas:`n" +
                      "     - Run Windows Update manually.`n" +
                      "     - Check WSUS/Group Policy if updates are managed centrally.`n" +
                      "     - Review update history for failures and resolve errors.`n" +
                      "     - Schedule regular maintenance windows for patching."
        }
    }
}

# High Disk Queue
$diskQueue = $samples.CounterSamples | Where-Object { $_.Path -like "*Avg. Disk Queue Length*" }
if ($diskQueue) {
    $queueAvg = ($diskQueue | Measure-Object CookedValue -Average).Average
    if ($queueAvg -gt 2) {
        $alerts += "High average disk queue length ($([math]::Round($queueAvg,2)))`n" +
                  "   Description: Prolonged disk queue indicates I/O bottleneck — disk subsystem cannot keep up with requests.`n" +
                  "   Resolution Ideas:`n" +
                  "     - Check for disk-intensive processes (Resource Monitor > Disk tab).`n" +
                  "     - Defragment if HDD; check alignment/health if SSD.`n" +
                  "     - Consider faster storage (SSD upgrade) or RAID reconfiguration.`n" +
                  "     - Move heavy I/O workloads (e.g., databases, logs) to dedicated drives."
    }
}

if ($alerts.Count -eq 0) {
    Write-Host "   No critical health issues detected at this time." -ForegroundColor Green
}
else {
    Write-Host "   The following health issues have been detected:" -ForegroundColor Red
    Write-Host ""
    foreach ($alert in $alerts) {
        $lines = $alert -split "`n"
        foreach ($line in $lines) {
            if ($line.Trim().StartsWith("Description:") -or $line.Trim().StartsWith("Resolution Ideas:")) {
                Write-Host "   $($line.Trim())" -ForegroundColor Yellow
            }
            elseif ($line.Trim().StartsWith("- ")) {
                Write-Host "     $($line.Trim())" -ForegroundColor Gray
            }
            else {
                Write-Host "   • $line" -ForegroundColor Red
            }
        }
        Write-Host ""
    }

    Write-Host "   General Recommendations:" -ForegroundColor Cyan
    Write-Host "     - Regularly review Event Logs using Event Viewer."
    Write-Host "     - Implement monitoring (e.g., SCOM, PRTG, or Azure Monitor)."
    Write-Host "     - Schedule monthly maintenance for updates and cleanup."
    Write-Host "     - Keep drivers and firmware up to date via vendor tools (Dell iDRAC, HP iLO, etc.)."
}

Write-Host ""
Write-Host ("=" * 80) -ForegroundColor Cyan
Write-Host "Health check complete." -ForegroundColor Cyan
Write-Host "This is a snapshot - ongoing monitoring is recommended for production servers." -ForegroundColor Gray

# Pause at end
if ($host.Name -ne "Windows PowerShell ISE Host") {
    Pause
}
```