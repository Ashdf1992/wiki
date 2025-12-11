# SQL Server Performance Checks (Windows)

This page documents a concise checklist to assess and tune SQL Server performance on Windows. Use these checks to quickly identify configuration issues, resource bottlenecks, and query-related problems.

```Powershell
# SQL Server Performance Health Check Script
# Compatible with PowerShell 5.1
# Requires: Administrator privileges (recommended)
# Modules: SqlServer module will be installed automatically if missing
# Purpose: Provides a professional, client-friendly overview of SQL Server instance performance
#          and highlights potential issues. Works with default and named instances.

# Check for Administrator (recommended for full access to performance data)
if (-not ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    Write-Host "WARNING: Running without Administrator privileges may limit access to some performance data." -ForegroundColor Yellow
}

Write-Host "SQL Server Performance Health Check" -ForegroundColor Cyan
Write-Host ("=" * 80) -ForegroundColor Cyan
Write-Host "Date: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')" -ForegroundColor Gray
Write-Host ""

# Check for SqlServer module and install automatically if missing
if (-not (Get-Module -ListAvailable -Name SqlServer)) {
    Write-Host "SqlServer PowerShell module not found. Installing automatically..." -ForegroundColor Yellow
    
    try {
        # Check for NuGet provider (required for module installation)
        if (-not (Get-PackageProvider -Name NuGet -ErrorAction SilentlyContinue)) {
            Write-Host "   Installing NuGet package provider..." -ForegroundColor Cyan
            Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force -Scope CurrentUser | Out-Null
        }

        # Set PSGallery to Trusted if not already
        if ((Get-PSRepository -Name PSGallery).InstallationPolicy -ne 'Trusted') {
            Set-PSRepository -Name PSGallery -InstallationPolicy Trusted
        }

        Write-Host "   Downloading and installing SqlServer module (this may take 1-2 minutes)..." -ForegroundColor Cyan
        Install-Module -Name SqlServer -Scope CurrentUser -Force -AllowClobber -ErrorAction Stop
        Write-Host "   SqlServer module installed successfully." -ForegroundColor Green
    }
    catch {
        Write-Host "ERROR: Failed to install SqlServer module automatically." -ForegroundColor Red
        Write-Host "Details: $($_.Exception.Message)" -ForegroundColor Red
        Write-Host "Please install it manually by running:" -ForegroundColor Red
        Write-Host "   Install-Module -Name SqlServer -Scope CurrentUser -Force" -ForegroundColor Red
        Pause
        exit
    }
}

# Import the module now that it's available
Import-Module SqlServer -ErrorAction Stop

# Prompt for SQL Instance (supports default MSSQLSERVER or named instances)
$instanceName = Read-Host "Enter SQL Server instance name (leave blank for default/local instance, or enter SERVER\INSTANCE)"

if ([string]::IsNullOrWhiteSpace($instanceName)) {
    $instanceName = "."
    $serverName = "$env:COMPUTERNAME (Default Instance)"
}
else {
    $serverName = $instanceName
}

Write-Host "Connecting to SQL Server instance: $serverName" -ForegroundColor Cyan
Write-Host ""

try {
    $server = New-Object Microsoft.SqlServer.Management.Smo.Server($instanceName)
    $server.ConnectionContext.ConnectTimeout = 10
    $server.ConnectionContext.Connect()
}
catch {
    Write-Host "ERROR: Could not connect to SQL Server instance '$instanceName'." -ForegroundColor Red
    Write-Host "Details: $($_.Exception.Message)" -ForegroundColor Red
    Write-Host "Ensure SQL Server is running, you have permissions, and Browser service is started (for named instances)." -ForegroundColor Red
    Pause
    exit
}

# Section 1: Instance Overview
Write-Host "1. SQL Server Instance Overview" -ForegroundColor Yellow
Write-Host ("-" * 50) -ForegroundColor Yellow

Write-Host "   Instance Name       : $($server.Name)"
Write-Host "   Edition             : $($server.Edition)"
Write-Host "   Version             : $($server.VersionString)"
Write-Host "   Product Level       : $($server.ProductLevel)"
Write-Host "   Uptime              : $($server.Uptime) minutes"
Write-Host "   Physical Memory     : $([math]::Round($server.PhysicalMemory / 1024, 2)) GB"
Write-Host "   Max Server Memory   : $([math]::Round($server.Configuration.MaxServerMemory.ConfigValue / 1024, 2)) GB"
Write-Host "   Min Server Memory   : $([math]::Round($server.Configuration.MinServerMemory.ConfigValue / 1024, 2)) GB"
Write-Host ""

# Section 2: Key Performance Counters (sampled over ~6 seconds)
Write-Host "2. Current Performance Counters (Averages)" -ForegroundColor Yellow
Write-Host ("-" * 50) -ForegroundColor Yellow

$counterList = @(
    "\SQLServer:SQL Statistics\Batch Requests/sec",
    "\SQLServer:SQL Statistics\SQL Compilations/sec",
    "\SQLServer:SQL Statistics\SQL Re-Compilations/sec",
    "\SQLServer:Buffer Manager\Page Life Expectancy",
    "\SQLServer:Buffer Manager\Buffer cache hit ratio",
    "\SQLServer:Buffer Manager\Lazy writes/sec",
    "\SQLServer:Memory Manager\Total Server Memory (KB)",
    "\SQLServer:Memory Manager\Target Server Memory (KB)",
    "\SQLServer:Locks(_Total)\Average Wait Time (ms)",
    "\SQLServer:Locks(_Total)\Lock Waits/sec",
    "\SQLServer:General Statistics\User Connections",
    "\SQLServer:Access Methods\Page Splits/sec",
    "\Processor(_Total)\% Processor Time"
)

try {
    $samples = Get-Counter -Counter $counterList -SampleInterval 2 -MaxSamples 3 -ErrorAction Stop
    $avgCounters = $samples.CounterSamples | Group-Object Path | ForEach-Object {
        $avg = ($_.Group | Measure-Object -Property CookedValue -Average).Average
        [PSCustomObject]@{
            Counter = ($_.Name -replace '\\SQLServer:', '' -replace '\(_Total\)', '' -replace '\\', ' > ').Trim()
            Value   = switch -Wildcard ($_.Name) {
                "*Page Life Expectancy*" { "{0:N0} sec" -f $avg }
                "*hit ratio*"            { "{0:N1}%" -f $avg }
                "*Memory*"               { "{0:N0} MB" -f ($avg / 1024) }
                "*\%*"                   { "{0:N1}%" -f $avg }
                default                  { "{0:N1}" -f $avg }
            }
        }
    }
    $avgCounters | Sort-Object Counter | Format-Table -AutoSize Counter, Value
}
catch {
    Write-Host "   Warning: Unable to retrieve some performance counters." -ForegroundColor Yellow
}
Write-Host ""

# Section 3: Performance Issue Alerts
Write-Host "3. Performance Issue Alerts" -ForegroundColor Yellow
Write-Host ("-" * 50) -ForegroundColor Yellow

$alerts = @()

# CPU
$cpuCounter = $samples.CounterSamples | Where-Object { $_.Path -like "*Processor Time*" }
if ($cpuCounter) {
    $cpuAvg = ($cpuCounter | Measure-Object CookedValue -Average).Average
    if ($cpuAvg -gt 80) { 
        $alerts += "High CPU usage ($([math]::Round($cpuAvg,1))%)`n" +
                  "   Description: Sustained CPU above 80% indicates the server is under heavy load, often from poorly optimized queries, missing indexes, or excessive compilations/re-compilations.`n" +
                  "   Resolution Ideas:`n" +
                  "     - Identify top CPU-consuming queries using DMVs: SELECT * FROM sys.dm_exec_query_stats ORDER BY total_worker_time DESC`n" +
                  "     - Run sp_WhoIsActive or use Extended Events to capture currently running queries.`n" +
                  "     - Add missing indexes (review sys.dm_db_missing_index_details).`n" +
                  "     - Update statistics on frequently queried tables.`n" +
                  "     - Consider query tuning or refactoring inefficient plans."
    }
}

# Page Life Expectancy (PLE)
$pleCounter = $samples.CounterSamples | Where-Object { $_.Path -like "*Page Life Expectancy*" }
if ($pleCounter) {
    $pleAvg = ($pleCounter | Measure-Object CookedValue -Average).Average
    if ($pleAvg -lt 300) { 
        $alerts += "Low Page Life Expectancy ($([math]::Round($pleAvg,0)) sec)`n" +
                  "   Description: PLE below 300 seconds suggests memory pressure; data pages are being flushed from the buffer pool too quickly, leading to more physical I/O.`n" +
                  "   Resolution Ideas:`n" +
                  "     - Increase physical RAM if possible.`n" +
                  "     - Raise 'Max Server Memory' configuration to allow SQL more memory (ensure OS has enough left).`n" +
                  "     - Identify memory-intensive queries via DMVs (sys.dm_exec_query_stats).`n" +
                  "     - Optimize large scans or reduce unnecessary data retrieval in queries.`n" +
                  "     - Check for external memory consumers (e.g., antivirus, other services)."
    }
}

# Buffer Cache Hit Ratio
$bchrCounter = $samples.CounterSamples | Where-Object { $_.Path -like "*Buffer cache hit ratio*" }
if ($bchrCounter) {
    $bchrAvg = ($bchrCounter | Measure-Object CookedValue -Average).Average
    if ($bchrAvg -lt 95) { 
        $alerts += "Low Buffer Cache Hit Ratio ($([math]::Round($bchrAvg,1))%)`n" +
                  "   Description: Below 95% means SQL is reading frequently from disk instead of memory, indicating potential memory pressure or inefficient queries.`n" +
                  "   Resolution Ideas:`n" +
                  "     - Similar to low PLE: Add RAM or increase Max Server Memory.`n" +
                  "     - Tune queries to reduce I/O (add indexes, avoid table scans).`n" +
                  "     - Monitor 'Buffer Manager: Page reads/sec' for confirmation of disk activity.`n" +
                  "     - Clear plan cache cautiously only if ad-hoc queries are bloating it."
    }
}

# High Page Splits
$splitsCounter = $samples.CounterSamples | Where-Object { $_.Path -like "*Page Splits/sec*" }
if ($splitsCounter) {
    $splitsAvg = ($splitsCounter | Measure-Object CookedValue -Average).Average
    if ($splitsAvg -gt 100) { 
        $alerts += "High Page Splits/sec ($([math]::Round($splitsAvg,0)))`n" +
                  "   Description: Excessive page splits occur during inserts/updates on clustered indexes with poor fill factors or sequential keys, causing fragmentation and performance degradation.`n" +
                  "   Resolution Ideas:`n" +
                  "     - Lower fill factor on indexes (e.g., 70-90%) for tables with frequent inserts.`n" +
                  "     - Rebuild/reorganize fragmented indexes (sys.dm_db_index_physical_stats).`n" +
                  "     - Consider using GUIDs carefully or switch to sequential keys if appropriate.`n" +
                  "     - Implement regular index maintenance jobs."
    }
}

# Lock Waits
$lockWaits = $samples.CounterSamples | Where-Object { $_.Path -like "*Lock Waits/sec*" }
if ($lockWaits) {
    $waitsAvg = ($lockWaits | Measure-Object CookedValue -Average).Average
    if ($waitsAvg -gt 10) { 
        $alerts += "High Lock Waits/sec ($([math]::Round($waitsAvg,1)))`n" +
                  "   Description: Frequent lock waits indicate blocking; sessions are waiting for others to release locks, often due to long-running transactions or poor isolation levels.`n" +
                  "   Resolution Ideas:`n" +
                  "     - Use sp_WhoIsActive to identify blocking chains.`n" +
                  "     - Enable Read Committed Snapshot Isolation (RCSI) to reduce reader/writer blocking.`n" +
                  "     - Shorten transactions and avoid unnecessary long-running ones.`n" +
                  "     - Review application code for efficient locking hints.`n" +
                  "     - Capture blocking with Extended Events or Blocked Process Report."
    }
}

# User Connections
$connections = $samples.CounterSamples | Where-Object { $_.Path -like "*User Connections*" }
if ($connections) {
    $connAvg = ($connections | Measure-Object CookedValue -Average).Average
    if ($connAvg -gt 500) { 
        $alerts += "High User Connections ($([math]::Round($connAvg,0)))`n" +
                  "   Description: A large number of active connections can exhaust resources, especially if connection pooling is not used properly in applications.`n" +
                  "   Resolution Ideas:`n" +
                  "     - Ensure applications use connection pooling (most .NET/IIS apps do by default).`n" +
                  "     - Audit connection strings and close connections promptly.`n" +
                  "     - Increase 'Max Pool Size' if needed, but investigate leaks first.`n" +
                  "     - Use sys.dm_exec_sessions to identify idle connections or specific applications."
    }
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
    Write-Host "     - Run Brent Ozar's sp_Blitz for a comprehensive health check." 
    Write-Host "     - Use sp_WhoIsActive for real-time session monitoring."
    Write-Host "     - Review wait statistics: SELECT * FROM sys.dm_os_wait_stats ORDER BY wait_time_ms DESC"
    Write-Host "     - Check index fragmentation and implement regular maintenance."
    Write-Host "     - Monitor SQL Server Error Log and consider baseline comparisons."
}

Write-Host ""
Write-Host ("=" * 80) -ForegroundColor Cyan
Write-Host "Health check complete." -ForegroundColor Cyan
Write-Host "This is a snapshot - SQL Server performance should be monitored over time." -ForegroundColor Gray

# Pause at end
if ($host.Name -ne "Windows PowerShell ISE Host") {
    Pause
}
```