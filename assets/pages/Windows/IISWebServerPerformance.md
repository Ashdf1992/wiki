# IIS Web Server Performance Checks

This page describes the performance-related items to validate on an IIS web server. It explains which metrics, configuration settings, and diagnostics should be checked to ensure the site(s) remain responsive and resource-efficient.

```Powershell
# IIS Performance Health Check Script
# Compatible with PowerShell 5.1
# Requires: Administrator privileges
# Modules: No external modules required (uses built-in Get-Counter and WMI/CIM)
# Purpose: Provides a professional, client-friendly overview of current IIS server performance
#          and highlights potential performance issues.

# Elevate if not already Administrator (optional self-elevation check)
if (-not ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    Write-Host "ERROR: This script must be run as Administrator." -ForegroundColor Red
    Pause
    exit
}

Write-Host "IIS Performance Health Check" -ForegroundColor Cyan
Write-Host ("=" * 80) -ForegroundColor Cyan
Write-Host "Date: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')" -ForegroundColor Gray
Write-Host ""

# Section 1: System Overview
Write-Host "1. System Overview" -ForegroundColor Yellow
Write-Host ("-" * 40) -ForegroundColor Yellow

$os = Get-CimInstance Win32_OperatingSystem
$cpu = Get-CimInstance Win32_Processor | Measure-Object -Property LoadPercentage -Average
$memTotal = [math]::Round($os.TotalVisibleMemorySize / 1MB, 2)
$memFree = [math]::Round($os.FreePhysicalMemory / 1MB, 2)
$memUsedPct = [math]::Round((($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / $os.TotalVisibleMemorySize) * 100, 1)

Write-Host "   OS          : $($os.Caption) $($os.OSArchitecture)"
Write-Host "   CPU Usage   : $($cpu.Average)%"
Write-Host "   Memory      : $memUsedPct% used ($($memTotal - $memFree) GB / $memTotal GB)"
Write-Host ""

# Section 2: Key Performance Counters (sampled over 5 seconds)
Write-Host "2. Current IIS & System Performance Counters" -ForegroundColor Yellow
Write-Host ("-" * 40) -ForegroundColor Yellow

try {
    $counters = Get-Counter -Counter `
        "\Processor(_Total)\% Processor Time",
        "\Memory\% Committed Bytes In Use",
        "\Memory\Available MBytes",
        "\ASP.NET Applications(__Total__)\Requests/Sec",
        "\ASP.NET Applications(__Total__)\Request Execution Time",
        "\ASP.NET Applications(__Total__)\Requests In Application Queue",
        "\Web Service(_Total)\Current Connections",
        "\Web Service(_Total)\Bytes Sent/sec",
        "\Web Service(_Total)\Bytes Received/sec",
        "\Web Service(_Total)\Connection Attempts/sec" -SampleInterval 2 -MaxSamples 3 -ErrorAction Stop

    $avg = $counters.CounterSamples | Group-Object Path | ForEach-Object {
        $avgValue = ($_.Group | Measure-Object -Property CookedValue -Average).Average
        [PSCustomObject]@{
            Counter = $_.Name.TrimStart('\').Replace('\',' > ')
            Value   = switch ($_.Name) {
                {$_ -like "*\%*"} { "{0:N1}%" -f $avgValue }
                {$_ -like "*Bytes*"} { "{0:N0}" -f $avgValue }
                {$_ -like "*Available MBytes*"} { "{0:N0} MB" -f $avgValue }
                default { "{0:N2}" -f $avgValue }
            }
        }
    }

    $avg | Format-Table -AutoSize Counter, Value
}
catch {
    Write-Host "   Warning: Could not retrieve performance counters (Perf counters may be disabled or permissions issue)." -ForegroundColor Yellow
}
Write-Host ""

# Section 3: Performance Issue Alerts
Write-Host "3. Performance Issue Alerts" -ForegroundColor Yellow
Write-Host ("-" * 40) -ForegroundColor Yellow

$alerts = @()

# CPU
if ($cpu.Average -gt 80) { $alerts += "High CPU usage ($($cpu.Average)%) - Investigate running processes and application pools." }

# Memory
if ($memUsedPct -gt 85) { $alerts += "High memory usage ($memUsedPct%) - Consider adding RAM or checking for memory leaks." }

# ASP.NET Queue (if available)
$queueCounter = $counters.CounterSamples | Where-Object { $_.Path -like "*Requests In Application Queue*" }
if ($queueCounter -and ($queueCounter | Measure-Object -Average CookedValue).Average -gt 10) {
    $queueAvg = ($queueCounter | Measure-Object -Average CookedValue).Average
    $alerts += "High ASP.NET request queue ($([math]::Round($queueAvg,1))) - Application pools may be overloaded or slow."
}

# Current Connections
$connCounter = $counters.CounterSamples | Where-Object { $_.Path -like "*Current Connections*" }
if ($connCounter -and ($connCounter | Measure-Object -Average CookedValue).Average -gt 1000) {
    $connAvg = ($connCounter | Measure-Object -Average CookedValue).Average
    $alerts += "High current connections ($([math]::Round($connAvg,0))) - Possible traffic spike or slow responses."
}

# Request Execution Time (if available)
$execTime = $counters.CounterSamples | Where-Object { $_.Path -like "*Request Execution Time*" }
if ($execTime -and ($execTime | Measure-Object -Average CookedValue).Average -gt 2000) {
    $execAvg = ($execTime | Measure-Object -Average CookedValue).Average
    $alerts += "Slow ASP.NET request execution time ($([math]::Round($execAvg/1000,2)) sec avg) - Review application code or database performance."
}

if ($alerts.Count -eq 0) {
    Write-Host "   No critical performance issues detected at this time." -ForegroundColor Green
}
else {
    foreach ($alert in $alerts) {
        Write-Host "   • $alert" -ForegroundColor Red
    }
    Write-Host ""
    Write-Host "   Recommendations:" -ForegroundColor Cyan
    Write-Host "     - Review IIS Application Pools for recycling settings and worker processes."
    Write-Host "     - Check IIS logs (%SystemDrive%\inetpub\logs\LogFiles) for slow requests."
    Write-Host "     - Use Failed Request Tracing (if enabled) for detailed diagnostics."
    Write-Host "     - Consider Resource Monitor or Performance Monitor for deeper analysis."
}

Write-Host ""
Write-Host ("=" * 80) -ForegroundColor Cyan
Write-Host "Health check complete." -ForegroundColor Cyan
Write-Host "This is a snapshot - performance should be monitored over time for trends." -ForegroundColor Gray

# Pause at end
if ($host.Name -ne "Windows PowerShell ISE Host") {
    Pause
}
```