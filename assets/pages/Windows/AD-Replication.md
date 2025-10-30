# Checking AD Replication 🔁🕵️‍♂️

## Version 1.0
> Open PowerShell or PowerShell ISE ▶️

> Run the following script to check replication status across domain controllers. The script queries partner metadata and replication failures, then prints a concise summary for each DC.

```Powershell
# Ensure the Active Directory module is available
Import-Module ActiveDirectory

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

```

## Purpose 📝
Quickly audit AD replication health across all domain controllers and surface any recent failures.

## Output format 📤
- Console output per DC with partner names and last successful replication times.
- If failures exist, details include partner, first failure time, failure count and last error.

## Requirements ⚠️
- Run with an account that can query Active Directory.
- RSAT / ActiveDirectory PowerShell module installed.

## Tips & Best Practices 💡
- Run from a management workstation or DC with necessary privileges.
- Consider redirecting output to a file for historical tracking:
  - Example: run the script and pipe or capture output to a dated TXT/CSV.
- Investigate failures promptly — replication issues can lead to inconsistent AD data.
