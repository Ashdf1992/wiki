# Get AD Group Members 👥

A couple of small utilities to list members of one or more Active Directory groups. Use the quick console version for immediate viewing or the CSV export for further processing.

---

## Version 1 — Output to console 🖥️

> ℹ️ This version will simply output the results to the console.

```Powershell
 $OU = "Domain Admins","Schema Admins","Enterprise Admins"
foreach ($o in $ou)
{
    echo $o
    Get-ADGroupMember $o | Select name, SamAccountName | fl

} 
```

---

## Version 2 — Output to CSV 💾

> ℹ️ This version will output the results to a CSV file.

```Powershell
 $OU = "Domain Admins","Schema Admins","Enterprise Admins"
foreach ($o in $ou)
{
    echo $o
    Get-ADGroupMember $o | Select name, SamAccountName | Export-CSV -Append .\User-Export.csv

} 
```

---

## Tips & Notes ✨
- ✅ Run with credentials that can query Active Directory.
- ✅ Use RSAT / ActiveDirectory PowerShell module on the machine running these commands.
- 📝 Adjust the group list to suit your environment.
- ⚠️ Export-CSV with -Append will create duplicates if run multiple times; remove or rename the file between runs if needed.
