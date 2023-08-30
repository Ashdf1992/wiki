# Check DCs and Operations Masters
Environment: CMD
``` CMD
netdom query fsmo
```


# Check AD replication
Environment: CMD
``` CMD
repadmin /showrepl
```

# Move Operation Master Roles
``` Powershell
Move-ADDirectoryServerOperationMasterRole -Identity "Enter the Name of a DC here" -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster
```
!!> The above will move the Master Roles, note however, this will need to be carried out on a DC that is not tombstoned, and is still online.!!


# Seize Operation Master Roles

``` Powershell
Move-ADDirectoryServerOperationMasterRole -Identity "Enter the Name of a DC here" -OperationMasterRole SchemaMaster, DomainNamingMaster, PDCEmulator, RIDMaster, InfrastructureMaster -Force
```
> This command will allow you to seize the Master Roles, in the event that a DC has been tombstoned and the roles are still on the Tombstoned DC. Note the -force flag.

!!> Use the above command with caution, and only use it if you are unable to move the roles. Seizing the roles should be a last resort. !!

