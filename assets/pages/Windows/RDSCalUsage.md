This Wiki will highlight the process required to check for licenses on an RDS Farm. 

> (Info) Select whether or not you know which server is used for RDS Licensing below

## Already know Licensing Server
> (Info) Firstly, you will need to log into your licensing server. Once logged in carry out the below steps. 
```Explorer
Go to Windows Administrative Tools
Go to Remote Desktop Services (this shows as a folder)
Double Click on 'Remote Desktop Licensing Manager'
Go to All Servers > 'Server Name'
```
>Here you should be able to see all of the licenses that are available. Should you wish to create a report, continue with the following instructions
```Explorer
Right Click on 'Reports' 
Select 'Create Report'
Select CAL Usage
Within the 'Report Pane' right click on the newly created report
Select 'Save As' 
Save it to a location of your choice. 
Open with Excel.
```
> You should now be able to see a report of all of the CALs that are available on the server, and you will also see what Computers or Users they have been assigned to.

## Not sure what server is used for licensing
> (Info) Firstly you will want to log into either one of your Session Host Servers, or your Gateway Server. 

Then carry out the following steps to identify your licensing server
```Explorer
Go to Windows Administrative Tools
Go to Remote Desktop Services (this is a folder)
Go to Remote Desktop Licensing Diagnoser. 
```
> (Info) In the bottom section you should see 'Remote Desktop Services License Server Information'. You should then see the licensing server.

> (Info) Log into the server that is used for licensing. Then carry out the below steps.

```Explorer
Go to Windows Administrative Tools
Go to Remote Desktop Services (this shows as a folder)
Double Click on 'Remote Desktop Licensing Manager'
Go to All Servers > 'Server Name'
```
>Here you should be able to see all of the licenses that are available. Should you wish to create a report, continue with the following instructions
```Explorer
Right Click on 'Reports' 
Select 'Create Report'
Select CAL Usage
Within the 'Report Pane' right click on the newly created report
Select 'Save As' 
Save it to a location of your choice. 
Open with Excel.
```
> You should now be able to see a report of all of the CALs that are available on the server, and you will also see what Computers or Users they have been assigned to.
