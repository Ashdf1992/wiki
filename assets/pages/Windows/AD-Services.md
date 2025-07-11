# Check the current status of Active Directory Services

## Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
# List of common Active Directory-related services
$adServices = @(
    "NTDS",             # Active Directory Domain Services
    "DNS",              # DNS Server
    "kdc",              # Kerberos Key Distribution Center
    "Netlogon",         # Net Logon
    "W32Time",          # Windows Time
    "DFSR",             # Distributed File System Replication
    "ADWS"              # Active Directory Web Services
)

Write-Host "Checking Active Directory-related services..." -ForegroundColor Cyan
foreach ($serviceName in $adServices) {
    $service = Get-Service -Name $serviceName -ErrorAction SilentlyContinue
    if ($null -ne $service) {
        $statusColor = if ($service.Status -eq 'Running') { 'Green' } else { 'Red' }
        Write-Host "$($service.DisplayName) ($serviceName): $($service.Status)" -ForegroundColor $statusColor
    } else {
        Write-Host "Service '$serviceName' not found on this system." -ForegroundColor Yellow
    }
}
```
