# Get AD Group Members

## Version 1 Output to console

> (Info) This version will simply Output the results to the console.

```Powershell
 $OU = "Domain Admins","Schema Admins","Enterprise Admins"
foreach ($o in $ou)
{
    echo $o
    Get-ADGroupMember $o | Select name, SamAccountName | fl

} 
```
## Version 2 Output to CSV

> (Info) This version will output the results to a CSV

```Powershell
 $OU = "Domain Admins","Schema Admins","Enterprise Admins"
foreach ($o in $ou)
{
    echo $o
    Get-ADGroupMember $o | Select name, SamAccountName | Export-CSV -Append .\User-Export.csv

} 
```
