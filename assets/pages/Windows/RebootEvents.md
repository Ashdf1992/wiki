# Get all events prior to a reboot. 

## V2
>Run the following
```Powershell
<#
.SYNOPSIS
  Reports Error & Warning events around the last reboot with accurate anchors:
  - BEFORE: 30 minutes prior to Event ID 6008 (Unexpected Shutdown), if present;
            otherwise 30 minutes prior to the preceding 6006 (Event Log stop) or the 6005 itself.
  - AFTER : 30 minutes after the first Event ID 6005 (Event Log start) following the reboot.

.DESIGN
  - Unplanned reboot path: 6008 (shutdown) -> next 6005 (start).
  - Planned/clean path   : last 6005 (start) and the nearest preceding 6006 (stop) if available.

.NOTES
  - Requires PowerShell 5.1+ (or PowerShell 7+) on Windows (uses Get-WinEvent).
  - Default logs: System, Application, Security. Add more if you need them.
  - You can change the time period by altering '[int]$WindowMinutes = 30,', within the 'param' section.
  - You can change the events that come back by editing 'Level = 0,1,2,3,4 # 0=LogAlways, 1=Critical, 2=Error, 3=Warning, 4=Informational, 5=Verbose' within the 'Get-WindowEvents' function.
#>

param(
    [int]$WindowMinutes = 30,
    [string[]]$LogsToCheck = @('System','Application','Security')
)

function Get-LastRebootContext {
    # Try the unplanned path first (6008)
    $last6008 = Get-WinEvent -FilterHashtable @{ LogName='System'; Id=6008 } -MaxEvents 1 -ErrorAction SilentlyContinue

    if ($last6008) {
        $shutdownEvent = $last6008
        # Find the first 6005 (Event Log service start) AFTER the unexpected shutdown
        $bootEvent = Get-WinEvent -FilterHashtable @{
            LogName   = 'System'
            Id        = 6005
            StartTime = $shutdownEvent.TimeCreated
        } -MaxEvents 1 -ErrorAction SilentlyContinue

        if (-not $bootEvent) {
            # Fallback: if logs were truncated, at least return the latest 6005
            $bootEvent = Get-WinEvent -FilterHashtable @{ LogName='System'; Id=6005 } -MaxEvents 1 -ErrorAction SilentlyContinue
        }

        return [pscustomobject]@{
            Mode          = 'Unexpected'   # 6008 seen
            ShutdownEvent = $shutdownEvent # 6008
            BootEvent     = $bootEvent     # 6005 (post-reboot)
        }
    }

    # Planned/clean path (no 6008): use last 6005; find preceding 6006 if present
    $last6005 = Get-WinEvent -FilterHashtable @{ LogName='System'; Id=6005 } -MaxEvents 1 -ErrorAction SilentlyContinue
    if ($last6005) {
        $prev6006 = Get-WinEvent -FilterHashtable @{
            LogName = 'System'
            Id      = 6006
            EndTime = $last6005.TimeCreated
        } -MaxEvents 1 -ErrorAction SilentlyContinue

        return [pscustomobject]@{
            Mode          = 'PlannedOrUnknown' # no 6008
            ShutdownEvent = $prev6006          # may be $null
            BootEvent     = $last6005
        }
    }

    # Nothing useful found
    return $null
}

function Get-WindowEvents {
    param(
        [datetime]$Start,
        [datetime]$End,
        [string[]]$Logs
    )
    foreach ($log in $Logs) {
        Get-WinEvent -FilterHashtable @{
            LogName   = $log
            Level     = 0,1,2,3,4          # 0=LogAlways, 1=Critical, 2=Error, 3=Warning, 4=Informational, 5=Verbose
            StartTime = $Start
            EndTime   = $End
        } -ErrorAction SilentlyContinue |
        Select-Object TimeCreated, Id, LevelDisplayName, ProviderName, Message, LogName
    }
}

# --- Build reboot context
$ctx = Get-LastRebootContext
if (-not $ctx) {
    Write-Host "No suitable reboot events (6008/6006/6005) were found in the System log." -ForegroundColor Yellow
    exit 1
}

# BEFORE: anchor to the shutdown moment
#   - Prefer 6008.TimeCreated for unplanned (accurate reboot trigger)
#   - Else 6006.TimeCreated if available
#   - Else 6005.TimeCreated (best-effort fallback)
$beforePivot =
    if     ($ctx.ShutdownEvent) { $ctx.ShutdownEvent.TimeCreated }
    elseif ($ctx.BootEvent)     { $ctx.BootEvent.TimeCreated }
    else                        { Get-Date }

# AFTER: anchor to the post-boot start (6005)
$afterPivot =
    if ($ctx.BootEvent) { $ctx.BootEvent.TimeCreated }
    else                { $beforePivot }  # extreme fallback

$beforeStart = $beforePivot.AddMinutes(-$WindowMinutes)
$beforeEnd   = $beforePivot
$afterStart  = $afterPivot
$afterEnd    = $afterPivot.AddMinutes($WindowMinutes)

# --- Header
Write-Host ""
Write-Host "Reboot context: $($ctx.Mode)" -ForegroundColor Cyan
if ($ctx.ShutdownEvent) {
    $sid = $ctx.ShutdownEvent.Id
    Write-Host ("Shutdown anchor  ({0}): {1}" -f $sid, $ctx.ShutdownEvent.TimeCreated)
}
if ($ctx.BootEvent) {
    Write-Host ("Boot/start anchor (6005): {0}" -f $ctx.BootEvent.TimeCreated)
}
Write-Host ("Window Minutes: {0}" -f $WindowMinutes)

# --- Collect and print
$beforeResults = Get-WindowEvents -Start $beforeStart -End $beforeEnd -Logs $LogsToCheck
$afterResults  = Get-WindowEvents -Start $afterStart  -End $afterEnd  -Logs $LogsToCheck

Write-Host "`n===== Events BEFORE reboot (from $beforeStart to $beforeEnd) =====" -ForegroundColor Yellow
if ($beforeResults) {
    $beforeResults | Sort-Object TimeCreated | Format-Table -Wrap -AutoSize
} else {
    Write-Host "No Error/Warning events found in this window." -ForegroundColor Green
}

Write-Host "`n===== Events AFTER reboot (from $afterStart to $afterEnd) =====" -ForegroundColor Yellow
if ($afterResults) {
    $afterResults | Sort-Object TimeCreated | Format-Table -Wrap -AutoSize
} else {
    Write-Host "No Error/Warning events found in this window." -ForegroundColor Green
}

# --- Optional CSV exports:
 $dir = "C:\Temp"; if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Path $dir }
 $beforeResults | Export-Csv "C:\Temp\BeforeReboot.csv" -NoTypeInformation
 $afterResults  | Export-Csv "C:\Temp\AfterReboot.csv"  -NoTypeInformation

```

## V1
>Run the Following
```Powershell
function Get-RebootReport {
    [CmdletBinding()]
    param (
        # Computer(s) to report on reboot history.
        [Parameter(Position=0)]
        [string[]]
        $ComputerName=$env:COMPUTERNAME,
        
        [switch]$SkipCheck
    )
    
    begin {
    }
    
    process {
        foreach ($Computer in $ComputerName) {
            if (-not ( Test-Connection $Computer -Count 2 -Quiet -ea 0 ) -and !$SkipCheck){
                Write-Warning "The device named $Computer is not responding to an Echo request!"
            }else{
        
            Get-WinEvent -ComputerName $Computer -FilterHashtable @{
                logname='System'; id=1074,6008} |
                    ForEach-Object {
                        $rv = New-Object PSObject |
                            Select-Object Date,
                                        User,
                                        Action,
                                        Process,
                                        Reason,
                                        ReasonCode,
                                        Comment
                        $rv.Date = $_.TimeCreated
                        $rv.User = $_.Properties[6].Value
                        $rv.Process = $_.Properties[0].Value -replace '^.*\\' -replace '\s.*$'
                        $rv.Action = $_.Properties[4].Value
                        $rv.Reason = $_.Properties[2].Value
                        $rv.ReasonCode = $_.Properties[3].Value
                        $rv.Comment = $_.Properties[5].Value
                        $rv
                    }
            }
        }
    }
    
    end {
    }
}
$csvPath="$($env:temp)\temp.csv";$LogNames=@('System','Application');$LastRebootTime = (Get-RebootReport)[0].Date;$Events=@();$LogNames|%{$FilterHashtable = @{LogName=$_;EndTime=($LastRebootTime.AddMinutes(1));StartTime=($LastRebootTime.AddMinutes(-5));};Get-WinEvent -FilterHashtable $FilterHashtable|%{$Events += $_}};$Events| sort TimeCreated |select LogName, TimeCreated, ProviderName, Message | export-csv $csvPath -notype; ii $csvPath
```
https://raw.githubusercontent.com/tonypags/PsWinAdmin/master/Public/Get-RebootReport.ps1
