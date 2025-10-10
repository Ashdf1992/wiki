# Checking AD Replication

## Version 1.0
>Open Powershell or Powershell ISE

>Run the Following
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
