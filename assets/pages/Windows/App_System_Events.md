# Retrieve Critical, Error, and Warning Events for a specified time

## Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
# Prompt for start time in hh:mm format
$StartTime = Read-Host "Enter start time (hh:mm, 24-hour format)"
$Duration = Read-Host "Enter duration to check for before and after the specified time ((60), mm, minute format)"


# Validate input
if ($StartTime -notmatch '^\d{2}:\d{2}$') {
    Write-Host "Invalid time format. Please use hh:mm (e.g., 09:30)." -ForegroundColor Red
    return
}

# Convert to DateTime (today's date)
$Date = Get-Date -Format 'yyyy-MM-dd'
$SpecifiedTime = [datetime]::ParseExact("$Date $StartTime", 'yyyy-MM-dd HH:mm', $null)

# Calculate time window (±30 minutes)
$StartDateTime = $SpecifiedTime.AddMinutes(-$Duration)
$EndDateTime   = $SpecifiedTime.AddMinutes($Duration)

# Define event levels and logs
$Levels = @(1, 2, 3)  # 1=Critical, 2=Error, 3=Warning

Write-Host "`nRetrieving events between $StartDateTime and $EndDateTime..." -ForegroundColor Cyan

# --- Application Log ---
Write-Host "`n=== Application Log Events ===" -ForegroundColor Yellow
$AppResults = Get-WinEvent -FilterHashtable @{
    LogName   = 'Application'
    Level     = $Levels
    StartTime = $StartDateTime
    EndTime   = $EndDateTime
} | Select-Object TimeCreated, Id, LevelDisplayName, ProviderName, Message

$AppResults | Format-Table -AutoSize -Wrap

# --- System Log ---
Write-Host "`n=== System Log Events ===" -ForegroundColor Yellow
$SysResults = Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    Level     = $Levels
    StartTime = $StartDateTime
    EndTime   = $EndDateTime
} | Select-Object TimeCreated, Id, LevelDisplayName, ProviderName, Message

$SysResults | Format-Table -AutoSize -Wrap

# Optional: Export both to CSV
# $AppResults | Export-Csv -Path "$env:USERPROFILE\Desktop\ApplicationEvents.csv" -NoTypeInformation
# $SysResults | Export-Csv -Path "$env:USERPROFILE\Desktop\SystemEvents.csv" -NoTypeInformation
```
