# DFS Cheatsheet

## Replication Summary for All DFS Members
```Powershell
# Get all DFS Replication Groups
$RGGroups = Get-DfsReplicationGroup

Write-Host "============================================================" -ForegroundColor Cyan
Write-Host " DFS Replication Backlog Report" -ForegroundColor Cyan
Write-Host " Generated on: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')" -ForegroundColor Cyan
Write-Host "============================================================`n" -ForegroundColor Cyan

foreach ($RG in $RGGroups) {
    Write-Host "`nReplication Group: $($RG.GroupName)" -ForegroundColor Yellow
    $RFolders = Get-DfsReplicatedFolder -GroupName $RG.GroupName

    foreach ($RF in $RFolders) {
        Write-Host "  Folder: $($RF.FolderName)" -ForegroundColor Green
        $RCons = Get-DfsrConnection -GroupName $RG.GroupName

        foreach ($RC in $RCons) {
            try {
                $date = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                [double]$count = (Get-DfsrBacklog -GroupName $RG.GroupName `
                                  -FolderName $RF.FolderName `
                                  -DestinationComputerName $RC.DestinationComputerName `
                                  -SourceComputerName $RC.SourceComputerName `
                                  -Verbose 4>&1).Message.Split(":")[2]

                $statusColor = if ($count -eq 0) { "Green" } else { "Red" }
                Write-Host ("    [{0}] Source: {1} → Destination: {2} | Backlog: {3}" -f `
                            $date, $RC.SourceComputerName, $RC.DestinationComputerName, $count) -ForegroundColor $statusColor
            }
            catch {
                Write-Host ("    [{0}] Error retrieving backlog for {1} → {2}: {3}" -f `
                            $date, $RC.SourceComputerName, $RC.DestinationComputerName, $_.Exception.Message) -ForegroundColor Red
            }
        }
    }
}

Write-Host "`n============================================================" -ForegroundColor Cyan
Write-Host " Report Complete" -ForegroundColor Cyan
Write-Host "============================================================" -ForegroundColor Cyan
```

## Check Replication Backlog (including file names), and Check Replication State
```Powershell
# Define source and destination computers
$sourceComputer = "ASH-WEB-01"
$destinationComputer = "ASH-WEB-02"
$replicatedFolder = "IIS-Content"

Write-Host "============================================================" -ForegroundColor Cyan
Write-Host " DFS Replication Health Check" -ForegroundColor Cyan
Write-Host "============================================================`n" -ForegroundColor Cyan

# Check DFS Replication Backlog
Write-Host "Checking DFS Replication Backlog between $sourceComputer and $destinationComputer for folder '$replicatedFolder'..." -ForegroundColor Yellow
try {
    $backlog = Get-DfsrBacklog -SourceComputerName $sourceComputer -DestinationComputerName $destinationComputer -GroupName $replicatedFolder -FolderName $replicatedFolder
    if ($backlog.Count -eq 0) {
        Write-Host "`n✅ No backlog detected. Replication is up to date." -ForegroundColor Green
    } else {
        Write-Host "`n⚠ Backlog detected: $($backlog.Count) files pending replication." -ForegroundColor Red
        $backlog | Select-Object @{Name="File Name";Expression={$_.FileName}},
                                   @{Name="Size (KB)";Expression={[math]::Round($_.FileSize/1KB,2)}},
                                   @{Name="Last Updated";Expression={$_.UpdateTime}} |
                  Format-Table -AutoSize
    }
} catch {
    Write-Host "`n❌ Error retrieving backlog: $($_.Exception.Message)" -ForegroundColor Red
}

# Check DFS Replication State
Write-Host "`nChecking DFS Replication State..." -ForegroundColor Yellow
try {
    $state = Get-DfsrState
    if ($state) {
        Write-Host "`nCurrent replication activity:" -ForegroundColor Cyan
        $state | Select-Object @{Name="File Name";Expression={$_.FileName}},
                                 @{Name="Path";Expression={$_.FilePath}},
                                 @{Name="Status";Expression={$_.UpdateStatus}} |
                Format-Table -AutoSize
    } else {
        Write-Host "`n✅ No replication activity detected at this time." -ForegroundColor Green
    }
} catch {
    Write-Host "`n❌ Error retrieving replication state: $($_.Exception.Message)" -ForegroundColor Red
}

Write-Host "`n============================================================" -ForegroundColor Cyan
Write-Host " Check Complete" -ForegroundColor Cyan
Write-Host "============================================================" -ForegroundColor Cyan
```

## Pre-seed Data
Running from ASH-WEB-01, copying data to ASH-WEB-02, excluding DFSR Private
``` Powershell
# Define source and destination paths
$Source      = "D:\IIS-Content\"
$Destination = "\\ASH-WEB-02\d$\IIS-Content\"
 
# Robocopy options:
# /MIR      - Mirror source to destination (includes subdirectories)
# /COPYALL  - Copy all file info (data, attributes, timestamps, security, owner, audit)
# /R:3      - Retry 3 times on failure
# /W:5      - Wait 5 seconds between retries
# /XD       - Exclude directory (DfsrPrivate)
# /LOG      - Optional: log output to a file
# /MT:16    - Multi-threaded copy (16 threads for speed)
 
Robocopy $Source $Destination /TEE /V /MIR /COPYALL /R:3 /W:5 /XD "DfsrPrivate" /MT:16 /LOG:"C:\Robocopy_Preseed.log"
```

## Clearing the DFS Replication Database

Occasionally, you may need to reset the DFS Replication (DFSR) database on a volume to resolve replication issues. Follow these steps carefully:

IMPORTANT NOTES:
<br>
-Ensure you have a valid backup of your data before proceeding.
<br>
-This process will force DFSR to rebuild its database, which may temporarily increase replication traffic.
<br>
-Run all commands in an elevated PowerShell session (Administrator).

Steps:
<br>
-Stop the DFS Replication Service
This prevents DFSR from accessing the database during the reset.
``` Powershell
Stop-Service DFSR
```
<br>

-Navigate to the root of the drive where the DFSR database resides (e.g., C:).
``` Powershell
Set-Location C:\
```
<br>

-Create an empty folder to use as a source for the mirror operation.
``` Powershell
New-Item -ItemType Directory -Path "C:\empty" -Force
```
<br>

-Mirror the empty folder into the DFSR database folder using robocopy.
This effectively clears the DFSR database contents.
```Powershell
robocopy "C:\empty" "C:\System Volume Information\DFSR" /MIR
```
/MIR = Mirror source to destination (delete extra files in destination).

<br>

-Restart the DFS Replication Service
This will trigger DFSR to rebuild its database.
``` Powershell
Start-Service DFSR
```

<br>

### ✅ Script (All Steps Combined)
IMPORTANT NOTES:
<br>
-Edit the following values accordingly:
<br>
$emptyPath = "C:\empty"
<br>
$dbPath = "C:\System Volume Information\DFSR"

``` Powershell
# Clear DFSR Database on C: Drive
# IMPORTANT: Run as Administrator and ensure backups exist before proceeding.

Write-Host "============================================================" -ForegroundColor Cyan
Write-Host " Clearing DFS Replication Database (C:) " -ForegroundColor Cyan
Write-Host "============================================================`n" -ForegroundColor Cyan

try {
    Write-Host "Stopping DFS Replication Service..." -ForegroundColor Yellow
    Stop-Service DFSR -ErrorAction Stop
    Write-Host "✅ DFS Replication Service stopped." -ForegroundColor Green

    Write-Host "`nPreparing empty folder for mirror operation..." -ForegroundColor Yellow
    $emptyPath = "C:\empty"
    if (-not (Test-Path $emptyPath)) {
        New-Item -ItemType Directory -Path $emptyPath -Force | Out-Null
    }
    Write-Host "✅ Empty folder ready at $emptyPath." -ForegroundColor Green
    Write-Host "`nClearing DFSR database using robocopy..." -ForegroundColor Yellow
    $dbPath = "C:\System Volume Information\DFSR"
    robocopy $emptyPath $dbPath /MIR | Out-Null
    Write-Host "✅ DFSR database cleared." -ForegroundColor Green

    Write-Host "`nRestarting DFS Replication Service..." -ForegroundColor Yellow
    Start-Service DFSR -ErrorAction Stop
    Write-Host "✅ DFS Replication Service restarted successfully." -ForegroundColor Green
}
catch {
    Write-Host "`n❌ An error occurred: $($_.Exception.Message)" -ForegroundColor Red
}

Write-Host "`n============================================================" -ForegroundColor Cyan
Write-Host " Operation Complete" -ForegroundColor Cyan
Write-Host "============================================================" -ForegroundColor Cyan
```
