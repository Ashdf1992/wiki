# VMWare PVSCSI Controller Boot Issue Fix


> Select from the Tab's if you have already rebooted or not, and then follow the instructions carefully.
{.is-info}

<br>

# Tabs {.tabset}
## Not Rebooted

> These steps are required if you have any servers that have already installed "VMware, Inc. – SCSIAdapter 1.3.18.0" AND not rebooted yet
{.is-info}

You can rollback before the reboot performing the following:
```Windows Explorer
Go to Computer Management
Go to Device Manager
Scroll down and expand Storage Controllers
Right Click on VMware PVSCSI Controller and go to Properties
Go to the Driver tab
Select Rollback driver
```
After carrying out the above steps that are relavent to the situation, you should now be able to access Windows again, and reboot without any of the previous issues. 

## Rebooted

> These steps are required if you have any servers that have already installed "VMware, Inc. – SCSIAdapter 1.3.18.0" AND the server has been rebooted, causing it to go into Recovery
{.is-info}

If the server has already been rebooted, and is having issues booting into Windows and the server is constantly trying to load up 'Automatic/System Recovery' carry out the following steps:
```WinPE
Hit F8 at boot to access the boot menu
Select 'Disable Driver Signature Enforcement' -> Windows should now at the very least boot
Go to Computer Management
Go to Device Manager
Scroll down and expand Storage Controllers
Right Click on VMware PVSCSI Controller and go to Properties
Go to the Driver tab
Select Rollback driver
```
After carrying out the above steps that are relavent to the situation, you should now be able to access Windows again, and reboot without any of the previous issues. 



