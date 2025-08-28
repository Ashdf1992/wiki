# 🏢 Check DCs and Operations Masters

*Revision: 1.2*  
*Relevant Products & Services: Active Directory, FSMO Roles*  
*Environment: PowerShell*

---

## 📋 Overview

This article outlines commands to:
- Review the current owner of FSMO roles
- Check for replication health
- Move or seize FSMO roles

---

## 🔍 Check FSMO Roles (Enhanced)

``` Powershell
# Enhanced FSMO Role Display Script

# Ensure the Active Directory module is available
Import-Module ActiveDirectory -ErrorAction Stop

Write-Host "`nRetrieving current FSMO role holders..." -ForegroundColor Cyan

try {
    # Get domain and forest context
    $domain = Get-ADDomain
    $forest = Get-ADForest

    # Collect FSMO role holders
    $fsmoRoles = @(
        [PSCustomObject]@{ Role = "Schema Master";            Holder = $forest.SchemaMaster;            Scope = "Forest: $forest" },
        [PSCustomObject]@{ Role = "Domain Naming Master";     Holder = $forest.DomainNamingMaster;      Scope = "Forest: $forest" },
        [PSCustomObject]@{ Role = "PDC Emulator";             Holder = $domain.PDCEmulator;             Scope = "Domain: $domain" },
        [PSCustomObject]@{ Role = "RID Master";               Holder = $domain.RIDMaster;               Scope = "Domain: $domain" },
        [PSCustomObject]@{ Role = "Infrastructure Master";    Holder = $domain.InfrastructureMaster;    Scope = "Domain: $domain" }
    )

    # Display domain and forest info
    Write-Host "`nDomain:  $($domain.DNSRoot)" -ForegroundColor Green
    Write-Host "Forest:  $($forest.RootDomain)" -ForegroundColor Green

    # Display FSMO roles in a table
    Write-Host "`nFSMO Role Assignments:`n" -ForegroundColor Green
    $fsmoRoles | Format-Table Role, Holder, Scope -AutoSize
}
catch {
    Write-Error "An error occurred while retrieving FSMO roles: $_"
}


```

---

## 🕹️ Check FSMO Roles (Legacy)
``` Powershell
netdom query fsmo
```

---

## 🔄 Check AD Replication
> (Warning) Always check replication prior to moving the FSMO roles. Ensure that all of the servers have good replication health, unless one of your Domain Controllers is offline. 
``` Powershell
repadmin /showrepl
```

---

## 🚚 Move Operation Master Roles
``` Powershell
Move-ADDirectoryServerOperationMasterRole -Identity "Enter the Name of a DC here" -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster
```

---

## 🛑 Seize Operation Master Roles
> **ℹ️ Info:**  
> Use this command to seize Master Roles if a DC is tombstoned and roles are still assigned to it.  
> **⚠️ Warning:**  
> Use with caution and only if you cannot move the roles. Seizing should be a last resort!
``` Powershell
Move-ADDirectoryServerOperationMasterRole -Identity "Enter the Name of a DC here" -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster -Force
```

