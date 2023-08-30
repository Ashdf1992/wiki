# MSSQL Stuff

<br>

## Installing SSL Certificate on SQL
> The first step is to install the SSL Cerficate using MMC

> You will then want to edit the following registry key to reflect the new thumbprint of the imported SSL Certificate:
HKLM\SOFTWARE\Microsoft\Microsoft SQL Server\\'YourSQLServerInstance'\MSSQLServer\SuperSocketNetLib\Certificate 

> Once imported, and the registry key has been changed you will simply need to restart the SQL Services on the SQL Server. 

> Note that you can get the thumbprint of the existing certificate in use by running the following powershell command

```Powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\<YourSQLServerInstance>\MSSQLServer\SuperSocketNetLib" | Select-Object Certificate | fl
```
> Note that you can get a list of installed certificates by running the following within powershell

```Powershell
cd Cert:
Get-ChildItem -path cert:\LocalMachine\My
```

>Alternatively if you want to view an SSL Certificate and you know the thumbprint you can use the following command

```Powershell
cd Cert:
Get-ChildItem -path “Thumbprint” -recurse
```

> To see more detailed information in relation to a particulae Certificate within powershell run the following command

```Powershell
Get-ChildItem -path “Thumbprint” -recurse | Select * | fl
```
