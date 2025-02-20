# Find events 5 minutes before and 1 minute after the last reboot that occurred. 

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
                logname='System'; id=1074} |
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
