# Installing an SSL Certificate onto RDWeb Apps
> This guide will take you through the process of installing an SSL Certificate onto an RDS Gateway Server and it will also highlight the process required to bind the SSL Certificate to each of the 4 Services; Redirector, Gateway, Publishing and Web Access

1. The first step required is to log onto the RDS Server
2. The second step is to copy the SSL certificate to the server, in this example, I will copy it to my desktop (C:\Users\Ash\Desktop\rds.xyz-studios.co.uk.pfx)
3. Next you will need to import the SSL Certificate. Go to Start > Type MMC and click on mmc
4. Click on File and go to 'Add/Remove Snap-in...'
5. Select Certificates > 'Computer Account', Ensure 'Local Computer' is selected and click 'Finish' and press ok.
6. Expand 'Certificates (Local Computer)'
7. Expand 'Personal'
8. Expand 'Certificates'
9. Right Click on 'Certificates' and go to 'All Tasks' > Import
10. Click next and browse to the Certificate file, in my case its C:\Users\Ash\Desktop\rds.xyz-studios.co.uk.pfx
> Remember to change the file type to 'All Files' otherwise you wont see your pfx file
11. Click Next then Enter your Password and 'mark the key exportable' if you would like to. Click Next and Finish

> At this point you should find that the SSL Certificate is installed into the store. The next thing we need to do is export the SSL as a base-64 encoded .cer 

12. Within the same SSL MMC Page, Right Click on the Newly imported SSL and go to 'All Tasks' > 'Export'

13. Click next and select 'No, do not export the private key'

14. Select 'Base-64 encoded X.509 (.CER)'

15. Click next and specify a location to save the file, In this example, I will use 'C:\Users\Ash\Desktop\rds.xyz-studios.co.uk.cer'. Click next and click finish.

>At this point we have exported the SSL certificate to a CER which we can use for the Web Broker 

16. Within the same SSL MMC Page, double click on the newly installed SSL, you should be able to identify this by the expiry date.

17. Go to 'Details' and scroll down to 'Thumbprint'. Copy the thumbprint to notepad or another document writer, as we will need this later on. 

>At this point we have the SSL Certificates thumbprint which we can use for the Web Client Broker 

> At this point remember we have exported the PFX to a CER, and we have the thumbprint. You need the CER and Thumbprint. You cannot proceed without either. 

> (Info) The following commands will need to be run within Powershell.

>Firstly we need to get the 'Connection Broker'

```Powershell
Get-RDServer -Role "RDS-CONNECTION-BROKER" | Select -Expandproperty Server 
```
My output was 'XYZ-RDSGW-01.xyz-studios.co.uk'

>The following commands you will need to change to include your 'Thumbprint'(that you copied to notepad) and 'Connection Broker' (which you should have just got)

```Powershell
Set-RDCertificate -Role RDRedirector -Thumbprint a4ab7fb719912387a92197ccdf0184dfb4796c52 -ConnectionBroker "XYZ-RDSGW-01.xyz-studios.co.uk"
Set-RDCertificate -Role RDGateway -Thumbprint a4ab7fb719912387a92197ccdf0184dfb4796c52 -ConnectionBroker "XYZ-RDSGW-01.xyz-studios.co.uk"
Set-RDCertificate -Role RDPublishing -Thumbprint a4ab7fb719912387a92197ccdf0184dfb4796c52 -ConnectionBroker "XYZ-RDSGW-01.xyz-studios.co.uk"
Set-RDCertificate -Role RDWebAccess -Thumbprint a4ab7fb719912387a92197ccdf0184dfb4796c52 -ConnectionBroker "XYZ-RDSGW-01.xyz-studios.co.uk"
Get-RDCertificate | Select Role,Subject,ExpiresOn,Thumbprint
```
My output is:
Role       : RDRedirector
Subject    : CN=rdp.xyz-studios.co.uk
ExpiresOn  : 04/28/2023 15:38:25
Thumbprint : a4ab7fb719912387a92197ccdf0184dfb4796c52

Role       : RDPublishing
Subject    : CN=rdp.xyz-studios.co.uk
ExpiresOn  : 04/28/2023 15:38:25
Thumbprint : a4ab7fb719912387a92197ccdf0184dfb4796c52

Role       : RDWebAccess
Subject    : CN=rdp.xyz-studios.co.uk
ExpiresOn  : 04/28/2023 15:38:25
Thumbprint : a4ab7fb719912387a92197ccdf0184dfb4796c52

Role       : RDGateway
Subject    : CN=rdp.xyz-studios.co.uk
ExpiresOn  : 04/28/2023 15:38:25
Thumbprint : a4ab7fb719912387a92197ccdf0184dfb4796c52

>The above should show the Roles, with the thumbprint, and they should match the thumbprint of the SSL we extracted earlier. 

>We now need to update the SSL Certificate used by the Web Client Broker. here we will need the path of the .CER that we exported earlier.

```Powershell
Import-RDWebClientBrokerCert "C:\Users\Ash\Desktop\rds.xyz-studios.co.uk.cer"
```

>The above command can be verified by running the following command in powershell

```Powershell
Get-RDWebClientBrokerCert | Select Subject,Thumbprint,NotAfter | fl
```
My Output: 
Subject    : CN=rdp.xyz-studios.co.uk
Thumbprint : a4ab7fb719912387a92197ccdf0184dfb4796c52
NotAfter   : 04/28/2023 15:38:25

>Congratulations. You should now be able to log into your RDWeb Apps and the SSL should have been updated. :thumbsup:
