# WinRM HTTPS Config Runsheet
## This wiki will take you through the process of configuring WinRM with an SSL Certificate bound to the listener on the server, and certificate based authentication between the Client and the Server. This guide will essentially be in 2 parts. The first part being the Server Configuration. The second part being the client configuration. In this example. I have a Windows Server 2022 client, that needs to connect to WinRM on a Windows Server 2022 Instance. Note that the following commands are to be ran from within Powershell or Powershell ISE (Easier for the script later)

Client = Windows Server 2022 – Win-Dev-01
<br>
Server = Windows Server 2022 – Win-Dev-02

Note that these instructions can be ran on Windows Server 2012 R2 and above. Though please also note that this has only been tested on Windows Server 2022. Note that for each section, I will put into brackets which device, either Client or Server the section is targeted at

<br>

## Server Configuration
> This part of the guide is in relation to the configuration of the server, that you are connecting to, from the client. 

### Enabling WinRM With Default Configuration (Server)
-Run the following command to run through the quick configuration of WinRM on the server that you wish to configure for WinRM:
```Powershell
winrm quickconfig
```
-Press Y when prompted
<br>
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img1.png"/>

<br>

### Disabling HTTP Connections(Server)
The server is now listening for HTTP connections on port 5985. Clients cannot verify the server identity when using HTTP, so the HTTP connection method has to be disabled

-To disable the HTTP connection method carry out the following:
```Powershell
winrm delete WinRM/Config/Listener?Address=*+Transport=HTTP
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img2.png"/>

<br>

### Setting the Allowed Authentication Methods (Server)
This step is to make sure that only the desired authentication methods are enabled. The insecure ones should be disabled by default. Certificate authentication is disabled by default too, so it needs to be enabled.

-To inspect the current settings run this command:
```Powershell
winrm get WinRM/Config/Service/Auth
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img3.png"/>

To allow clients to authenticate using certificates only the certificate authentication method has to be enabled. Negotiate authentication is needed to be able to (amongst others) locally configure WinRM using the winrm command. Whether other authentication methods need to be enabled depends on your environment. For example: Kerberos authentication is used in environments with Active Directory Domain Services, so it is very likely that it must be enabled in such environments. The methods that must be disabled unless they are needed for a specific use case, are basic and credential security support provider (CredSSP). Basic authentication will send your password to the server. CredSSP will send your credentials to the server

-Disable Basic and CredSSP auth by running the following:
```Powershell
winrm set WinRM/Config/Service/Auth '@{Basic="false";Kerberos="true";Negotiate="true";Certificate="true";CredSSP="false"}'
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img4.png"/>

<br>

### Create the Server Certificate (Server)
Windows includes a cmdlet with which you can create self-signed certificates, but unfortunately certificates that can be used for WinRM authentication cannot be generated. Luckily there is a PowerShell script available that can generate usable certificates which can be downloaded using the following [Link](https://github.com/Ashdf1992/wiki/blob/main/assets/attachments/New-SelfSignedCertificateEx.ps1). Note that the file is stored within this wiki. 

-First import the function in the script file, and then create the certificate for the server using the function. The common name (CN) of the subject needs to match the name that you will use to connect to the server later. In this instance, we are connecting from ‘Win-Dev-01’ to ‘Win-Dev-02’ If you want to use the hostname of the server, you can use $env:COMPUTERNAME. In this instance, I have copied the script ‘New-SelfSignedCertificateEx.ps1’ to my user directory, you just need to ensure that the script exists within the directory, where you are running the powershell commands:
```Powershell
. .\New-SelfSignedCertificateEx.ps1
$serverCert = New-SelfSignedCertificateEx -Subject "CN=<(fully-qualified) hostname>" -EnhancedKeyUsage '1.3.6.1.5.5.7.3.1' -NotAfter "07 October 2034 16:17:33" -StoreLocation LocalMachine -IsCA $true
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img5.png"/>

> (Important Note 1)
^ Note the space in between . .

> (Important Note 2)
^ Change "CN=<(fully-qualified) hostname>". In my case, I will change it to  "CN=Win-Dev-02".

> (Important Note 3)
^ Change -NotAfter "" to include the date that you wish for the certificate to expire.

The certificate has been added to Cert:\LocalMachine\My\ (the certificates identifying the server machine) and Cert:\LocalMachine\CA\ (certificate authorities (CAs) that are trusted on the server machine), using the thumbprint as the child name.
-The thumbprint is going to be needed to verify whether the correct public key has been downloaded by clients, so display it and copy it somewhere:
```Powershell
$serverCert
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img6.png"/>

<br>

### Enabling a Secure WinRM Listener (Server)
-The final step for the Windows server is the addition of a secure WinRM listener. Execute the following command to create the listener. The hostname must match the hostname used when creating the server certificate, the thumbprint is the thumbprint of the SSL Certificate you generated above. 
> (Important Info)Note that this command needs to be ran within Command Prompt, it will not work with Powershell:
```Command Prompt
winrm create winrm/config/Listener?Address=*+Transport=HTTPS '@{Hostname="host_name";CertificateThumbprint="certificate_thumbprint"}'
```

-You may need to delete the old https listener you can do this using the following command
```Command prompt
winrm delete winrm/config/Listener?Address=*+Transport=HTTPS
```

[Powershell]

<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img7.png"/>

[Command Prompt]

<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img8.png"/>

That is it for the server configuration. We will return to the server when a local administrator for client connections has to be created

<br>
<br>

## Client Configuration
Congratulations, you made it this far. This part of the guide is in relation to the client-side configuration. Essentially, the computer you want to use to remote connect to the server

<br>

### Making Sure the Server Hostname and Server Certificate Subject Match (Client)
-It is important that the server hostname that you use matches the subject of the certificate. If they don not match the client will refuse to make a connection. If the server hostname cannot be mapped to an IP address by the DNS or WINS server on your network, you need to add a mapping to an IP address to C:\Windows\System32\drivers\etc\hosts manually. This file can only be changed by an administrator. In my case, the servers are not domain joined, so I will need to add a host entry for 10.0.0.5 – Win-Dev-02. 

<br>

### Ensuring the Client Uses Secure Authentication Methods (Client)
-Firstly, we want to ensure that the WinRM Service is running:
```Powershell
Get-Service WinRM
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img9.png"/>

-If it is not running, run the following command
```Powershell
(Get-Service WinRM).Start()
```

-Now set the authentication that the client is allowed to use:
```Powershell
winrm set WinRM/Config/Client/Auth '@{Basic="false";Kerberos="true";Negotiate="true";Certificate="true";CredSSP="false"}'
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img10.png"/>

<br>

### Install Certificate Authority Certificate of the Server onto the Client (Client)
-In this step the server certificate is retrieved, but it cannot be verified using the existing known certificate authorities. Verify the thumbprint manually to ensure that you are installing the correct server certificate as a certificate authority. After installing the certificate as a certificate authority, all certificates signed by that certificate will be accepted without user notification or confirmation. The certificate of the server can simply be downloaded by sending an HTTP request to the WinRM HTTPS endpoint on the server. The request will fail (the error is caught and ignored) because the server certificate cannot be verified, but the request will contain the unverifiable certificate afterwards. The thumbprint of the retrieved certificate is verified manually below, note that at this point, you need to have changed your host file if required to point towards the Server:
```Powershell
$webRequest = [Net.WebRequest]::Create("https://<server hostname>:5986/wsman")
try { $webRequest.GetResponse() } catch {} # Catch and ignore the certificate error
$serverCert = $webRequest.ServicePoint.Certificate
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img11.png"/>

-Output the thumbprint of the certificate and verify that it matches the thumbprint of the certificate generated in the server configuration chapter:
```Powershell
$serverCert.GetCertHashString()
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img12.png"/>

-If the server certificate is incorrect, revisit Create the Server Certificate (Server), and Enabling a Secure WinRM Listener (Server). If the server certificate is correct you can add it as a certificate authority to your certificate store. The PowerShell path of the certificate store location where the certificate will be stored is Cert:\CurrentUser\Root\. A confirmation dialog will appear for the third command:
```Powershell
$store = New-Object System.Security.Cryptography.X509Certificates.X509Store -ArgumentList "Root", "CurrentUser"
$store.Open('ReadWrite')
$store.Add($serverCert)
$store.Close()
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img13.png"/>
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img14.png"/>

<br>

### Create Certificate for the Client (Client)
-Just like the server the client needs a certificate, so the server can verify the identity of the client. Unlike on the server, the standard PowerShell cmdlet can be used to generate the client certificate. Every certificate identifies a subject. The subject name can be anything. To make identifying certificates easier, you can, for example, include the client hostname or the certificate purpose. A good value for the subject name is your Windows username or Microsoft Account email address. In this case we need to change the subject and the upn=:
```Powershell
$winClientCert = New-SelfSignedCertificate -Type Custom -Subject <subject> -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2","2.5.29.17={text}upn=<subject>") -KeyUsage DigitalSignature,KeyEncipherment -CertStoreLocation Cert:\CurrentUser\My\
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img15.png"/>

-The thumbprint is going to be needed to verify whether the correct public key has been downloaded by the server, so display it and copy it somewhere:
```Powershell
$winClientCert
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img16.png"/>

-This certificate can be used to authenticate against multiple servers. There is no need to create a client certificate for each server to be accessed using WinRM.
Export the (public part) of the certificate to a file, so it can be transferred to the server. For <fingerprint> you can use $($winClientCert.Thumbprint):
```Powershell
Export-Certificate -Cert Cert:\CurrentUser\My\<fingerprint> -FilePath <export file path>.crt
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img17.png"/>

<br>

### Install Certificate of the Client onto the Server (Server)
-Copy the exported client certificate to the server. You can use insecure methods to do this, because the certificate only contains a public key. As the name implies, having other people know this key does not pose a security risk. Note that in my case, I have simply copied the SSL Cert for the Client from C:\Win-Dev-01.crt and placed the certificate into the root of C: on the server
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img18.png"/>

-Once the client certificate is on the server, load it using:
```Powershell
$winClientCert = Get-PfxCertificate <certificate file path>
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img19.png"/>

-Output the thumbprint of the certificate and verify that it matches the thumbprint of the certificate generated above:
```Powershell
$winClientCert.GetCertHashString()
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img20.png"/>

-If the client certificate is incorrect, revisit Create 'Certificate for the Client' and 'Install Certificate of the Client onto the Server'. If the client certificate is correct you can add it as a certificate authority and a trusted person to the certificate store of the server. The reason the certificate also has to be installed as a certificate authority, is that this enables the server to verify the copy of the certificate installed as a trusted person:
```Powershell
Import-Certificate -FilePath <certificate file path> -CertStoreLocation Cert:\LocalMachine\Root
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img21.png"/>

```Powershell
Import-Certificate -FilePath <certificate file path> -CertStoreLocation Cert:\LocalMachine\TrustedPeople
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img22.png"/>

<br>

### Attach Client Certificate to User (Server)
-Now the server needs to know as which user the WinRM session must run when it is authenticated using the client certificate. You can use an existing user or create a user specifically for WinRM sessions. Add the client certificate to the WinRM client certificate store. The subject that the client will use during WinRM authentication, and the credentials of the local user to use on successful authentication, will be associated with the certificate. Note that the Subject must match the subject of the client SSL Certificate generated earlier on the client. In my case this will be ‘Win-Dev-01’. In the dialog asking for credentials, enter the credentials of the local user as which the WinRM sessions must be run:
```Powershell
New-Item -Path WSMan:\localhost\ClientCertificate -Subject '<subject>' -URI * -Issuer ((Get-PfxCertificate <certificate file path>).Thumbprint) -Credential (Get-Credential)
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img23.png"/>
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img24.png"/>

<br>

### Test Connection on the Client (Client)
-Everything should be set up correctly now. There is a cmdlet specifically for testing if everything is working as expected. By using the parameter -Authentication ClientCertificate we force the use of HTTPS:
```Powershell
Test-WSMan -ComputerName <server hostname> -Authentication ClientCertificate -CertificateThumbprint <client certificate thumbprint>
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img25.png"/>

-If the previous command was successful you can open a PowerShell session and use WinRM on the server:
```Powershell
$session = New-PSSession -ComputerName <server hostname> -CertificateThumbprint <client certificate thumbprint>
Enter-PSSession -Session $session
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img26.png"/>

-The prompt should change and should now have a prefix containing the server hostname. Execute any command you want. When you are done, exit the session using exit:
[<server hostname>]: exit

-Finally clean up the PowerShell session on the client machine:
```Powershell
Remove-PSSession -Session $session
```
<img src="https://github.com/Ashdf1992/wiki/blob/main/assets/images/WinRMHTTPSImages/img27.png"/>
