# Change the Certificate used by RDP
> Within Powershell ISE run the following
```Powershell
wmic /namespace:\\root\cimv2\TerminalServices PATH Win32_TSGeneralSetting Set SSLCertificateSHA1Hash="<THUMBPRINT>"
```
  
> (Info) Note that you will need to get the Thumbprint for the SSL Certificate first.
