# Check the current status of Active Directory Services 🔍🛠️

Quick utility to verify common Active Directory–related Windows services on a server (Domain Controller or management host).

---

## Prerequisites ✅
- Run with an account that can query local services (start PowerShell as Administrator if needed).  
- Best run on a Domain Controller or a machine that manages DCs.

---

## How to run ▶️
1. Open PowerShell or PowerShell ISE.  
2. Paste and run the script below.  
3. Review the colored output: green = Running, red = Not running, yellow = Not found.

---

## Script 🧾
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

---

## Output — What to expect 📤
- Each listed service prints its display name, service name, and current status.  
- Green indicates the service is running; red indicates stopped/other; yellow means the service is not installed on the host.

---

## Tips & Notes 💡
- If a critical service (e.g., NTDS, DNS, Netlogon) is not running, investigate event logs and service dependencies.  
- You can adapt the $adServices array to include additional services specific to your environment.  
- To capture the output to a file, redirect PowerShell output, e.g.:  
  PowerShell.exe -File .\Check-AD-Services.ps1 > AD-Services-Status.txt
