# Checking Windows Updates on Computers Across an Active Directory environment. 

## Powershell
>Open Powershell or Powershell ISE

>Run the Following
```Powershell
# Specify the Organizational Units (OUs) where computers are located
$computerOU = "CN=Computers,DC=ENTERYOURDOMAINDETAILS,DC=ENTERYOURDOMAINDETAILS,DC=ENTERYOURDOMAINDETAILS"
$domainControllersOU = "OU=Domain Controllers,DC=ENTERYOURDOMAINDETAILS,DC=ENTERYOURDOMAINDETAILS,DC=ENTERYOURDOMAINDETAILS"
 
# Get all computers in the specified OUs
$computers = Get-ADComputer -Filter * -SearchBase $computerOU | Select-Object -ExpandProperty Name
$domainControllers = Get-ADComputer -Filter * -SearchBase $domainControllersOU | Select-Object -ExpandProperty Name
 
# Combine both lists of computers
$allComputers = $computers + $domainControllers
 
# Ensure PSWindowsUpdate module is installed
if (-not (Get-Module -ListAvailable -Name PSWindowsUpdate)) {
    Write-Host "PSWindowsUpdate module is not installed. Please install it to check for pending updates." -ForegroundColor Red
    return
}
 
# Initialize a List to store results for CSV export
$results = [System.Collections.Generic.List[Object]]::new()
 
# Define a function to check the update history for a computer using Get-HotFix
function Check-UpdateHistory {
    param (
        [string]$computer
    )
    try {
        # Query the computer for installed hotfixes (updates)
        $updateHistory = Get-HotFix -ComputerName $computer -ErrorAction Stop
        if ($updateHistory) {
            # Calculate the date 365 days ago
            $dateThreshold = (Get-Date).AddDays(-365)
            # Filter updates to only show those installed in the last 365 days
            $recentUpdates = $updateHistory | Where-Object { 
                [datetime]$_.InstalledOn -ge $dateThreshold 
            }
 
            if ($recentUpdates) {
                foreach ($update in $recentUpdates) {
                    # Create a custom object for each recent update
                    $results.Add([pscustomobject]@{
                        ComputerName  = $computer
                        UpdateType    = 'Recent Update'
                        KBNumber      = $update.HotFixID
                        InstalledOn   = $update.InstalledOn
                    })
                }
            } else {
                # Add entry if no recent updates are found
                $results.Add([pscustomobject]@{
                    ComputerName  = $computer
                    UpdateType    = 'Recent Update'
                    KBNumber      = 'None'
                    InstalledOn   = 'N/A'
                })
            }
        } else {
            # Add entry if no updates are found at all
            $results.Add([pscustomobject]@{
                ComputerName  = $computer
                UpdateType    = 'Recent Update'
                KBNumber      = 'None'
                InstalledOn   = 'N/A'
            })
        }
 
        # Now check for pending updates
        $pendingUpdates = Get-WindowsUpdate -ComputerName $computer -ErrorAction Stop
        if ($pendingUpdates) {
            foreach ($update in $pendingUpdates) {
                if ($update.KBArticleIDs) {
                    foreach ($KB in $update.KBArticleIDs) {
                        # Add pending update KB to results
                        $results.Add([pscustomobject]@{
                            ComputerName  = $computer
                            UpdateType    = 'Pending Update'
                            KBNumber      = $KB
                            InstalledOn   = 'Pending'
                        })
                    }
                }
            }
        } else {
            # Add entry if no pending updates are found
            $results.Add([pscustomobject]@{
                ComputerName  = $computer
                UpdateType    = 'Pending Update'
                KBNumber      = 'None'
                InstalledOn   = 'N/A'
            })
        }
    } catch {
        # Add entry if there was an error retrieving data
        $results.Add([pscustomobject]@{
            ComputerName  = $computer
            UpdateType    = 'Error'
            KBNumber      = 'Error'
            InstalledOn   = 'Error'
        })
    }
}
 
# Loop through each computer and check its update history
foreach ($computer in $allComputers) {
    Write-Host "Checking updates on: $computer" -ForegroundColor Cyan
    Check-UpdateHistory -computer $computer
}
 
# Export the results to a CSV file
$results | Export-Csv -Path "C:\temp\UpdateResults.csv" -NoTypeInformation
 
Write-Host "Results exported to C:\temp\UpdateResults.csv" -ForegroundColor Green
```
