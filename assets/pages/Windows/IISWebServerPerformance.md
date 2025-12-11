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
if ($cpu.Average -gt 80) { 
    $alerts += "High CPU usage ($($cpu.Average)%)`n" +
              "   Description: Sustained CPU above 80% indicates the server is under heavy processing load, often from inefficient .NET code, large responses, or high traffic volumes.`n" +
              "   Resolution Ideas:`n" +
              "     - Use Performance Monitor or Resource Monitor to identify top CPU-consuming w3wp.exe processes (application pools).`n" +
              "     - Enable Failed Request Tracing in IIS to capture slow requests (> certain threshold).`n" +
              "     - Review application code for inefficiencies (e.g., loops, large data processing).`n" +
              "     - Consider scaling out (add more servers) or optimizing queries if backend database is involved.`n" +
              "     - Recycle overloaded application pools manually or adjust recycling settings."
}

# Memory
if ($memUsedPct -gt 85) { 
    $alerts += "High memory usage ($memUsedPct%)`n" +
              "   Description: Memory usage above 85% can lead to paging, reduced performance, and potential application pool crashes.`n" +
              "   Resolution Ideas:`n" +
              "     - Check for memory leaks in .NET applications (use dotMemory or DebugDiag).`n" +
              "     - Review large session state usage or caching strategies.`n" +
              "     - Increase physical RAM if possible.`n" +
              "     - Limit memory per application pool (Rapid-Fail Protection settings).`n" +
              "     - Recycle application pools on memory thresholds."
}

# ASP.NET Queue (if available)
$queueCounter = $counters.CounterSamples | Where-Object { $_.Path -like "*Requests In Application Queue*" }
if ($queueCounter -and ($queueCounter | Measure-Object -Average CookedValue).Average -gt 10) {
    $queueAvg = ($queueCounter | Measure-Object -Average CookedValue).Average
    $alerts += "High ASP.NET request queue ($([math]::Round($queueAvg,1)))`n" +
              "   Description: Requests are backing up in the queue, meaning worker processes are saturated and cannot handle incoming traffic quickly enough.`n" +
              "   Resolution Ideas:`n" +
              "     - Increase the number of worker processes (web garden) or queue length limits in machine.config.`n" +
              "     - Identify slow requests via IIS logs or Failed Request Tracing.`n" +
              "     - Optimize slow backend calls (database, web services).`n" +
              "     - Scale up CPU/RAM or scale out to additional servers.`n" +
              "     - Check application pool CPU throttling settings."
}

# Current Connections
$connCounter = $counters.CounterSamples | Where-Object { $_.Path -like "*Current Connections*" }
if ($connCounter -and ($connCounter | Measure-Object -Average CookedValue).Average -gt 1000) {
    $connAvg = ($connCounter | Measure-Object -Average CookedValue).Average
    $alerts += "High current connections ($([math]::Round($connAvg,0)))`n" +
              "   Description: A large number of simultaneous connections can exhaust resources, especially with slow clients or keep-alive issues.`n" +
              "   Resolution Ideas:`n" +
              "     - Review connection timeouts and keep-alive settings in IIS.`n" +
              "     - Check for traffic spikes or potential DoS (review IIS logs for source IPs).`n" +
              "     - Implement load balancing if not already in place.`n" +
              "     - Optimize client-side code to close connections promptly.`n" +
              "     - Consider HTTP/2 or compression to reduce connection overhead."
}

# Request Execution Time (if available)
$execTime = $counters.CounterSamples | Where-Object { $_.Path -like "*Request Execution Time*" }
if ($execTime -and ($execTime | Measure-Object -Average CookedValue).Average -gt 2000) {
    $execAvg = ($execTime | Measure-Object -Average CookedValue).Average
    $alerts += "Slow ASP.NET request execution time ($([math]::Round($execAvg/1000,2)) sec avg)`n" +
              "   Description: Average request processing time is high, leading to poor user experience and potential queuing.`n" +
              "   Resolution Ideas:`n" +
              "     - Enable and review Failed Request Tracing for requests exceeding thresholds.`n" +
              "     - Profile application code (Visual Studio profiler or dotTrace).`n" +
              "     - Optimize database queries or external service calls.`n" +
              "     - Enable output caching or compression in IIS.`n" +
              "     - Check for blocking I/O operations in code."
}

if ($alerts.Count -eq 0) {
    Write-Host "   No critical performance issues detected at this time." -ForegroundColor Green
}
else {
    Write-Host "   The following performance issues have been detected:" -ForegroundColor Red
    Write-Host ""
    foreach ($alert in $alerts) {
        $lines = $alert -split "`n"
        foreach ($line in $lines) {
            if ($line.Trim().StartsWith("Description:") -or $line.Trim().StartsWith("Resolution Ideas:")) {
                Write-Host "   $($line.Trim())" -ForegroundColor Yellow
            }
            elseif ($line.Trim().StartsWith("- ") -or $line.Trim().StartsWith("  - ")) {
                Write-Host "     $($line.Trim())" -ForegroundColor Gray
            }
            else {
                Write-Host "   • $line" -ForegroundColor Red
            }
        }
        Write-Host ""
    }
    Write-Host "   General Recommendations:" -ForegroundColor Cyan
    Write-Host "     - Review IIS logs (%SystemDrive%\inetpub\logs\LogFiles) for patterns in slow/error requests."
    Write-Host "     - Use tools like Failed Request Tracing, Performance Monitor, or Application Insights for deeper analysis."
    Write-Host "     - Consider regular application pool recycling schedules."
    Write-Host "     - Test under load using tools like JMeter or Azure Load Testing."
    Write-Host "     - Ensure Windows/IIS and .NET Framework are fully patched."
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