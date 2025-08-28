# Checking the total number of connections to sites within IIS


## Notes:
You can change $cutoffDate = (Get-Date).AddDays(-30) to specify the number of days you want to check for connections. By default its set to 30 days in the past. 
<br>
<br>
You can change the http response code by altering this line. $connectionCount = (Get-Content $file | Select-String "- 200").Count. By default its set to '- 200', this can be changed to '- 500' or '- 301', or any other http response code. 


## V1 - Connections referencing each log file
### Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
$basePath = "D:\inetpub\Logs"
$cutoffDate = (Get-Date).AddDays(-30)
$totalConnections = 0
$report = @()

# Header
Write-Host "`n===== Connection Report for Files Created in the Last 30 Days =====`n" -ForegroundColor Cyan

# Recursively find .log files created in the last 30 days
Get-ChildItem -Path $basePath -Recurse -File -Include *.log | Where-Object { $_.CreationTime -ge $cutoffDate } | ForEach-Object {
    $file = $_.FullName
    $connectionCount = (Get-Content $file | Select-String "- 200").Count
    $totalConnections += $connectionCount

    $report += [PSCustomObject]@{
        "Log File"     = $file
        "Connections"  = $connectionCount
        "Created On"   = $_.CreationTime.ToString("yyyy-MM-dd HH:mm")
    }
}

# Output formatted table
$report | Sort-Object Connections -Descending | Format-Table -AutoSize

# Summary
Write-Host "`n===============================================================" -ForegroundColor DarkGray
Write-Host "Total Connections Found Across All Files: $totalConnections" -ForegroundColor Green
Write-Host "===============================================================`n" -ForegroundColor DarkGray
```
>Example output
```Powershell
FileName                                                    Connections
--------                                                    -----------
D:\inetpub\logs\LogFiles\W3SVC1\u_ex250730_x.log            21
===============================================================
Total Connections Found Across All Files: 21
===============================================================
```

## V2 - Connections referencing each log file and showing the 'Site Name'
### Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
Import-Module WebAdministration

$basePath = "D:\inetpub\Logs"
$cutoffDate = (Get-Date).AddDays(-30)
$totalConnections = 0
$siteConnections = @{}

# Get mapping of W3SVC IDs to site names
$siteMap = @{}
Get-ChildItem IIS:\Sites | ForEach-Object {
    $id = $_.Id
    $name = $_.Name
    $siteMap["W3SVC$id"] = $name
}

# Recursively find .log files created in the last 30 days
Get-ChildItem -Path $basePath -Recurse -File -Include *.log | Where-Object { $_.CreationTime -ge $cutoffDate } | ForEach-Object {
    $file = $_.FullName
    $connectionCount = (Get-Content $file | Select-String "- 200 ").Count
    $totalConnections += $connectionCount

    # Extract W3SVC ID from path
    if ($file -match "W3SVC\d+") {
        $svcId = $matches[0]
        $siteName = $siteMap[$svcId]
        if (-not $siteName) { $siteName = $svcId }

        if ($siteConnections.ContainsKey($siteName)) {
            $siteConnections[$siteName] += $connectionCount
        } else {
            $siteConnections[$siteName] = $connectionCount
        }

        Write-Host ""
        Write-Host "╔══════════════════════════════════════════════════════════════╗" -ForegroundColor Cyan
        Write-Host ("║ Site: {0,-55} ║" -f $siteName) -ForegroundColor Cyan
        Write-Host ("║ File: {0,-55} ║" -f $file) -ForegroundColor Cyan
        Write-Host ("║ Connections: {0,-47} ║" -f $connectionCount) -ForegroundColor Cyan
        Write-Host "╚══════════════════════════════════════════════════════════════╝" -ForegroundColor Cyan
    } else {
        Write-Host ""
        Write-Host "⚠ No W3SVC ID found in path: $file" -ForegroundColor DarkYellow
    }
}

# Output summary
Write-Host "`n═══════════════════════════════════════════════════════════════" -ForegroundColor Green
Write-Host "               Site Connection Summary" -ForegroundColor Green
Write-Host "═══════════════════════════════════════════════════════════════" -ForegroundColor Green

$siteConnections.GetEnumerator() | Sort-Object Value -Descending | ForEach-Object {
    Write-Host ("{0,-40} {1,10}" -f $_.Key, $_.Value) -ForegroundColor Yellow
}

Write-Host "`n═══════════════════════════════════════════════════════════════" -ForegroundColor DarkGray
Write-Host ("{0,-40} {1,10}" -f "Total Connections", $totalConnections) -ForegroundColor Green
Write-Host "═══════════════════════════════════════════════════════════════`n" -ForegroundColor DarkGray
```
>Example output
```Powershell
╔══════════════════════════════════════════════════════════════╗
║ Site: testsite                                               ║
║ File: D:\inetpub\Logs\LogFiles\W3SVC1\u_ex250730_x.log       ║
║ Connections: 21                                              ║
╚══════════════════════════════════════════════════════════════╝
═══════════════════════════════════════════════════════════════
               Site Connection Summary
═══════════════════════════════════════════════════════════════
testsite                                         21
═══════════════════════════════════════════════════════════════
Total Connections                                21
═══════════════════════════════════════════════════════════════
```

## V3 - A summary showing all sites, and their cumulative number of connections
### Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
Import-Module WebAdministration

$basePath = "D:\inetpub\localuser"
$cutoffDate = (Get-Date).AddDays(-30)
$totalConnections = 0
$siteConnections = @{}

# Get mapping of W3SVC IDs to site names
$siteMap = @{}
Get-ChildItem IIS:\Sites | ForEach-Object {
    $id = $_.Id
    $name = $_.Name
    $siteMap["W3SVC$id"] = $name
}

# Recursively find .log files created in the last 30 days
Get-ChildItem -Path $basePath -Recurse -File -Include *.log | Where-Object { $_.CreationTime -ge $cutoffDate } | ForEach-Object {
    $file = $_.FullName
    $connectionCount = (Get-Content $file | Select-String "- 200 ").Count
    $totalConnections += $connectionCount

    # Extract W3SVC ID from path
    if ($file -match "W3SVC\d+") {
        $svcId = $matches[0]
        $siteName = $siteMap[$svcId]
        if (-not $siteName) { $siteName = $svcId }

        if ($siteConnections.ContainsKey($siteName)) {
            $siteConnections[$siteName] += $connectionCount
        } else {
            $siteConnections[$siteName] = $connectionCount
        }
    }
}

# Output summary
Write-Host "`n═══════════════════════════════════════════════════════════════" -ForegroundColor Cyan
Write-Host "                    Site Connection Summary" -ForegroundColor Cyan
Write-Host "═══════════════════════════════════════════════════════════════" -ForegroundColor Cyan

$siteConnections.GetEnumerator() | Sort-Object Value -Descending | ForEach-Object {
    Write-Host ("{0,-40} {1,10}" -f $_.Key, $_.Value) -ForegroundColor Yellow
}

Write-Host "`n═══════════════════════════════════════════════════════════════" -ForegroundColor DarkGray
Write-Host ("{0,-40} {1,10}" -f "Total Connections Found", $totalConnections) -ForegroundColor Green
Write-Host "═══════════════════════════════════════════════════════════════`n" -ForegroundColor DarkGray

```
>Example output
```Powershell
═══════════════════════════════════════════════════════════════
                    Site Connection Summary
═══════════════════════════════════════════════════════════════
TestSite: 21 connections
═══════════════════════════════════════════════════════════════
Total Connections Found                          21
═══════════════════════════════════════════════════════════════
```
