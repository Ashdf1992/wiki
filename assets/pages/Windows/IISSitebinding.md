# Find the Website/Webapp Binding from a URL

<br>

```Powershell
# IIS Site Binding Lookup Script
# Compatible with PowerShell 5.1
# Requires: Administrator privileges and IIS PowerShell module (WebAdministration)

Import-Module WebAdministration -ErrorAction Stop

function Find-IISSiteByHostname {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory=$true)]
        [string]$Hostname
    )

    # Validate input
    if ([string]::IsNullOrWhiteSpace($Hostname)) {
        Write-Host "Error: Hostname cannot be empty." -ForegroundColor Red
        return
    }

    # Clean the hostname
    $cleanHostname = $Hostname.Trim().ToLower() -replace '^https?://', '' -replace '^www\.', '' -replace '/.*$', ''

    Write-Host "Searching IIS sites for bindings containing hostname: $cleanHostname" -ForegroundColor Cyan
    Write-Host ("=" * 80) -ForegroundColor Cyan
    Write-Host ""

    $matchingSites = @()
    $allSites = Get-Website

    foreach ($site in $allSites) {
        $bindings = Get-WebBinding -Name $site.Name

        foreach ($binding in $bindings) {
            $bindingHost = $binding.bindingInformation.Split(':')[2]
            if ($bindingHost -eq "") { $bindingHost = "*" }

            if ($bindingHost -eq $cleanHostname -or $bindingHost -eq "*") {
                $siteInfo = [PSCustomObject]@{
                    SiteName       = $site.Name
                    SiteID         = $site.Id
                    State          = $site.State
                    Protocol       = $binding.protocol
                    BindingInfo    = $binding.bindingInformation
                    HostHeader     = if ($bindingHost -eq "*") { "All Hostnames (*)" } else { $bindingHost }
                    PhysicalPath   = $site.PhysicalPath
                }
                $matchingSites += $siteInfo
            }
        }
    }

    if ($matchingSites.Count -eq 0) {
        Write-Host "No IIS sites found with a binding matching '$cleanHostname'." -ForegroundColor Yellow
        Write-Host ""
        Write-Host "Note: Wildcard (*) bindings accept any hostname on that port/IP." -ForegroundColor Gray
    }
    else {
        Write-Host "Found $($matchingSites.Count) matching binding(s):" -ForegroundColor Green
        Write-Host ""

        # Build calculated properties explicitly to avoid PS 5.1 parsing issues
        $calculatedProperties = @(
            @{ Name = "Site Name";      Expression = { $_.SiteName } }
            @{ Name = "Site ID";        Expression = { $_.SiteID } }
            @{ Name = "State";          Expression = { $_.State } }
            @{ Name = "Protocol";       Expression = { $_.Protocol.ToUpper() } }
            @{ Name = "IP:Port";        Expression = { $_.BindingInfo.Split(':')[0..1] -join ':' } }
            @{ Name = "Host Header";    Expression = { $_.HostHeader } }
            @{ Name = "Physical Path";  Expression = { $_.PhysicalPath } }
        )

        $matchingSites | Format-Table -Property $calculatedProperties -AutoSize -Wrap

        Write-Host ""
        Write-Host "Summary:" -ForegroundColor Cyan
        $matchingSites | ForEach-Object {
            Write-Host " • Site '$($_.SiteName)' (ID: $($_.SiteID)) - $($_.Protocol.ToUpper()) - $($_.BindingInfo) - Path: $($_.PhysicalPath)"
        }
    }

    Write-Host ""
    Write-Host ("=" * 80) -ForegroundColor Cyan
    Write-Host "Search complete." -ForegroundColor Cyan
}

# Main execution
try {
    $userInput = Read-Host "Enter the hostname to search for (e.g., ashtesting.com)"
    Find-IISSiteByHostname -Hostname $userInput
}
catch {
    Write-Host "An error occurred: $($_.Exception.Message)" -ForegroundColor Red
    Write-Host "Ensure you are running PowerShell as Administrator and the IIS Management Tools are installed." -ForegroundColor Red
}

if ($host.Name -ne "Windows PowerShell ISE Host") {
    Pause
}
```
