# Check DCs and Operations Masters

<i> Revision: 1.0 </i>
<br>
<i>Relevent Products & Services : Active Directory, FSMO Roles</i>
<br>
<i>Environment: Powershell</i>

<br>
<br>

## This article outlines some commands that can be ran to review the current owner of the FSMO roles, it also outlines how to check for replication, and then outlines the commands required to move or seize the FSMO roles.

# Check the FSMO roles
``` Powershell
netdom query fsmo
```


# Check AD replication
>.info Always check replication prior to moving the FSMO roles
``` Powershell
repadmin /showrepl
```

# Move Operation Master Roles
``` Powershell
Move-ADDirectoryServerOperationMasterRole -Identity "Enter the Name of a DC here" -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster
```
> (Warning) The above will move the Master Roles, note however, this will need to be carried out on a DC that is not tombstoned, and is still online.


# Seize Operation Master Roles

``` Powershell
Move-ADDirectoryServerOperationMasterRole -Identity "Enter the Name of a DC here" -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster -Force
```
> (Info) This command will allow you to seize the Master Roles, in the event that a DC has been tombstoned and the roles are still on the Tombstoned DC. Note the -force flag.

> (Warning) Use the above command with caution, and only use it if you are unable to move the roles. Seizing the roles should be a last resort. !!

