# Check DCs and Operations Masters

<i> Revision: 1.2 </i>
<br>
<i>Relevent Products & Services : Active Directory, FSMO Roles</i>
<br>
<i>Environment: Powershell</i>

<br>
<br>

## This article outlines some commands that can be ran to review the current owner of the FSMO roles, it also outlines how to check for replication, and then outlines the commands required to move or seize the FSMO roles.

# Check the FSMO roles v2
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

<br>

# Check the FSMO roles v1 (legacy)
``` Powershell
netdom query fsmo
```

<br>

# Check AD replication
> (Warning) Always check replication prior to moving the FSMO roles. Ensure that all of the servers have good replication health, unless one of your Domain Controllers is offline. 
``` Powershell
repadmin /showrepl
```

<br>

# Move Operation Master Roles
``` Powershell
Move-ADDirectoryServerOperationMasterRole -Identity "Enter the Name of a DC here" -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster
```

<br>

# Seize Operation Master Roles
> (Info) This command will allow you to seize the Master Roles, in the event that a DC has been tombstoned and the roles are still on the Tombstoned DC. Note the -force flag.
> (Warning) Use the following command with caution, and only use it if you are unable to move the roles. Seizing the roles should be a last resort. !!
``` Powershell
Move-ADDirectoryServerOperationMasterRole -Identity "Enter the Name of a DC here" -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster -Force
```

