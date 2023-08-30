# Get AD Group Members

# Tabs {.tabset}
## Version 1 Output to console

> This version will simply Output the results to the console.
{.is-info}

```Powershell
 $OU = "Domain Admins","Schema Admins","Enterprise Admins"
foreach ($o in $ou)
{
    echo $o
    Get-ADGroupMember $o | Select name, SamAccountName | fl

} 
```
## Version 2 Output to CSV

> This version will output the results to a CSV
{.is-info}

```Powershell
 $OU = "Domain Admins","Schema Admins","Enterprise Admins"
foreach ($o in $ou)
{
    echo $o
    Get-ADGroupMember $o | Select name, SamAccountName | Export-CSV -Append .\User-Export.csv

} 
```
