# Checking AD Replication

## Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell

<#
.SYNOPSIS
    Domain Controller Health Check Script

.DESCRIPTION
    Performs a comprehensive health check of all Domain Controllers in the domain.
    Checks include:
    - Domain Controller availability
    - Replication status
    - FSMO role holders
    - AD Services status (NTDS, DNS, Netlogon, W32Time)
    - SYSVOL and NETLOGON share availability
    - Time synchronization status
    - Event log errors
#>

Import-Module ActiveDirectory

function Write-Section {
    param([string]$Title)
    Write-Host "`n=== $Title ===" -ForegroundColor Cyan
}

function Write-Status {
    param([string]$Message, [bool]$Success)
    if ($Success) {
        Write-Host "$Message" -ForegroundColor Green
    } else {
        Write-Host "$Message" -ForegroundColor Red
    }
}

# Get all Domain Controllers
$DCs = Get-ADDomainController -Filter *

Write-Section "1) - Domain Controller Availability"
foreach ($dc in $DCs) {
    try {
        $ping = Test-Connection -ComputerName $dc.HostName -Count 2 -Quiet -ErrorAction Stop
        Write-Status "$($dc.HostName): Online" $true
    } catch {
        Write-Status "$($dc.HostName): Offline or Unreachable" $false
    }
}

Write-Section "2) - Replication Status"
# Get all Domain Controllers in the domain
$domainControllers = Get-ADDomainController -Filter *

foreach ($dc in $domainControllers) {
    Write-Host "Checking replication status for: $($dc.HostName)" -ForegroundColor Cyan

    # Get replication partner metadata
    $replicationPartners = Get-ADReplicationPartnerMetadata -Target $dc.HostName -Scope Server

    foreach ($partner in $replicationPartners) {
        $lastSuccess = $partner.LastReplicationSuccess
        $partnerName = $partner.Partner

        Write-Host "  Partner: $partnerName"
        Write-Host "    Last Successful Replication: $lastSuccess"
    }

    # Check for replication failures
    $failures = Get-ADReplicationFailure -Target $dc.HostName

    if ($failures.Count -gt 0) {
        Write-Host "  Replication failures detected:" -ForegroundColor Red
        foreach ($failure in $failures) {
            Write-Host "    Partner: $($failure.Partner)"
            Write-Host "    First Failure Time: $($failure.FirstFailureTime)"
            Write-Host "    Failure Count: $($failure.FailureCount)"
            Write-Host "    Last Error: $($failure.LastErrorMessage)"
        }
    } else {
        Write-Host "  No replication failures detected." -ForegroundColor Green
    }

    Write-Host ""
}

Write-Section "3) - FSMO Role Holders"
Write-Host "`nRetrieving current FSMO role holders..." -ForegroundColor Cyan

try {
    # Get domain and forest context
    $domain = Get-ADDomain
    $forest = Get-ADForest

    # Collect FSMO role holders
    $fsmoRoles = @(
        [PSCustomObject]@{ Role = "Schema Master";            Holder = $forest.SchemaMaster;            Scope = "Forest: $forest" },
        [PSCustomObject]@{ Role = "Domain Naming Master";     Holder = $forest.DomainNamingMaster;      Scope = "Forest: $forest" },
        [PSCustomObject]@{ Role = "PDC Emulator";             Holder = $domain.PDCEmulator;             Scope = "Domain: $domain" },
        [PSCustomObject]@{ Role = "RID Master";               Holder = $domain.RIDMaster;               Scope = "Domain: $domain" },
        [PSCustomObject]@{ Role = "Infrastructure Master";    Holder = $domain.InfrastructureMaster;    Scope = "Domain: $domain" }
    )

    # Display domain and forest info
    Write-Host "`nDomain:  $($domain.DNSRoot)" -ForegroundColor Green
    Write-Host "Forest:  $($forest.RootDomain)" -ForegroundColor Green

    # Display FSMO roles in a table
    Write-Host "`nFSMO Role Assignments:`n" -ForegroundColor Green
    $fsmoRoles | Format-Table Role, Holder, Scope -AutoSize
}
catch {
    Write-Error "An error occurred while retrieving FSMO roles: $_"
}

Write-Section "4) - Active Directory Services Status"
foreach ($dc in $DCs) {
    Write-Host "`nChecking services on $($dc.HostName)..." -ForegroundColor Magenta
    Invoke-Command -ComputerName $dc.HostName -ScriptBlock {
        $services = @("NTDS", "DNS", "Netlogon", "W32Time")
        foreach ($svc in $services) {
            $status = Get-Service -Name $svc -ErrorAction SilentlyContinue
            if ($status) {
                $color = if ($status.Status -eq 'Running') { 'Green' } else { 'Red' }
                Write-Host "$svc : $($status.Status)" -ForegroundColor $color
            } else {
                Write-Host "$svc : Not Found" -ForegroundColor Red
            }
        }
    }
}

Write-Section "5) - SYSVOL and NETLOGON Share Availability"
foreach ($dc in $DCs) {
    try {
        $shares = Get-WmiObject -Class Win32_Share -ComputerName $dc.HostName
        $sysvol = $shares | Where-Object { $_.Name -eq "SYSVOL" }
        $netlogon = $shares | Where-Object { $_.Name -eq "NETLOGON" }
        Write-Status "$($dc.HostName): SYSVOL Share Present" ($null -ne $sysvol)
        Write-Status "$($dc.HostName): NETLOGON Share Present" ($null -ne $netlogon)
    } catch {
        Write-Status "$($dc.HostName): Unable to retrieve shares" $false
    }
}

Write-Section "6) - Time Synchronization Status"
foreach ($dc in $DCs) {
    try {
        $timeSource = Invoke-Command -ComputerName $dc.HostName -ScriptBlock {
            w32tm /query /status | Select-String "Source"
        }
        Write-Host "$($dc.HostName): $timeSource" -ForegroundColor Green
    } catch {
        Write-Status "$($dc.HostName): Unable to retrieve time source" $false
    }
}

Write-Section "7) - Recent Event Log Errors (Directory Service)"
foreach ($dc in $DCs) {
    Write-Host "`nChecking Directory Service event logs on $($dc.HostName)..." -ForegroundColor Magenta
    try {
        $errors = Get-WinEvent -ComputerName $dc.HostName -LogName "Directory Service" -ErrorAction Stop |
                  Where-Object { $_.LevelDisplayName -eq "Error" -and $_.TimeCreated -gt (Get-Date).AddDays(-1) }
        if ($errors.Count -eq 0) {
            Write-Host "No recent errors found." -ForegroundColor Green
        } else {
            Write-Host "$($errors.Count) error(s) found in the last 24 hours:" -ForegroundColor Red
            $errors | ForEach-Object { Write-Host "$($_.TimeCreated): $($_.Message)" -ForegroundColor Red }
        }
    } catch {
        Write-Status "$($dc.HostName): Unable to retrieve event logs" $false
    }
}

```
