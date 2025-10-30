# DNS Check for AD Environment

Environment: PowerShell

## Purpose 📝
Scan domain servers (excluding domain controllers, clusters and AV gateways) and report DNS client server addresses that match a given pattern (for example a 172.* DNS subnet).

## Requirements ⚠️
- Run with an account that can query Active Directory and remote computers.
- RSAT / ActiveDirectory PowerShell module installed on the machine running the script.
- WinRM enabled on target servers for Invoke-Command (or run locally).

## Quick usage ▶️
- Edit the -like pattern in the script to match the DNS subnet you want to check (examples: `'*172*'`, `'*10.*'`, `'*192.168*'`).
- Run from a workstation with RSAT and network access to target servers.
- Expect output rows with ComputerName and ServerAddresses for matches.

## Script 🛠️
```powershell
# Checks all servers in the domain (excluding DC*, CLS*, AVG*)
$Computers_Online = @()
$computers = (Get-ADComputer -Filter * |
    Where-Object { $_.Name -notlike '*DC*' -and $_.Name -notlike '*CLS*' -and $_.Name -notlike '*AVG*' }).Name

foreach ($Server in $computers) {
    if (Test-Connection -ComputerName $Server -Count 1 -Quiet) {
        $Computers_Online += $Server
    } else {
        Write-Warning "$Server is not responding to ping"
    }
}

# Change the pattern below depending on the DNS subnet you want to detect
$dnsPattern = '*172*'

foreach ($Server in $Computers_Online) {
    Invoke-Command -ComputerName $Server -ScriptBlock {
        $cn = $env:COMPUTERNAME
        Get-DnsClientServerAddress |
            Where-Object { $_.ServerAddresses -like $using:dnsPattern } |
            Select-Object @{Name='ComputerName';Expression={$cn}}, ServerAddresses
    } -ErrorAction SilentlyContinue
}
```

## Notes & tips 💡
- The script uses `$using:dnsPattern` so you only set the pattern once outside the remote scriptblock.
- If WinRM is not available, run the DNS checks locally on each server or use an alternative remoting method.
- Adjust the name filters (`DC*`, `CLS*`, `AVG*`) to match your environment naming conventions.
- Consider exporting results to CSV for easier analysis (e.g., pipe output to `Export-Csv`).
