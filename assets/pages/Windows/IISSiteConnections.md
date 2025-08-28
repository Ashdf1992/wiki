# Checking the total number of connections to sites within IIS

## V1 - Connections referencing each log file
### Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
$basePath = "D:\inetpub\logs"
$cutoffDate = (Get-Date).AddDays(-30)
$totalConnections = 0
$report = @()

# Recursively find files created in the last 30 days
Get-ChildItem -Path $basePath -Recurse -File -Include *.log | Where-Object { $_.CreationTime -ge $cutoffDate } | ForEach-Object {
    $file = $_.FullName
    $connectionCount = (Get-Content $file | Select-String "- 200").Count
    $totalConnections += $connectionCount

    $report += [PSCustomObject]@{
        FileName = $file
        Connections = $connectionCount
    }
}

# Output the report
Write-Host "===== Connection Report for Files Created in the Last 30 Days ====="
$report | Format-Table -AutoSize

Write-Host "`nTotal Connections Found: $totalConnections"
```
>Example output
```Powershell
FileName                                                    Connections
--------                                                    -----------
D:\inetpub\logs\LogFiles\W3SVC1\u_ex250730_x.log            21
```

## V2 - Connections referencing each log file and showing the 'Site Name'
### Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
Import-Module WebAdministration

$basePath = "D:\inetpub\logs"
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

        Write-Host "File: $file"
        Write-Host "Connections: $connectionCount"
        Write-Host "Site: $siteName"
        Write-Host "----------------------------------------"
    } else {
        Write-Host "No W3SVC ID found in path: $file"
    }
}

# Output summary
Write-Host "`n===== Site Connection Summary ====="
```
>Example output
```Powershell
File: D:\inetpub\logs\LogFiles\W3SVC1\u_ex250730_x.log
Connections: 21
Site: testsite
----------------------------------------
```

## V3 - A summary showing all sites, and their cumulative number of connections
### Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
Import-Module WebAdministration

$basePath = "D:\inetpub\logs"
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
Write-Host "`n===== Site Connection Summary ====="
$siteConnections.GetEnumerator() | Sort-Object Value -Descending | ForEach-Object {
    Write-Host "$($_.Key): $($_.Value) connections"
}

Write-Host "`nTotal Connections Found: $totalConnections"
```
>Example output
```Powershell
===== Site Connection Summary =====
TestSite: 21 connections
Total Connections Found: 21
```
