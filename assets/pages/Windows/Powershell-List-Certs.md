# Get a list of SSL Certificates using Powershell
```Powershell
Get-ChildItem -Path Cert:LocalMachine\MY | Select FriendlyName,Thumbprint
```

```Powershell
Get-ChildItem  -Path Cert:\LocalMachine\MY | Where-Object {$_.Subject -Match "mail"} | Select-Object FriendlyName, Thumbprint, Subject, NotBefore, NotAfter
```

```Powershell
Get-ChildItem -Path Cert:LocalMachine\MY | Select-Object FriendlyName, Thumbprint, Subject, NotBefore, NotAfter
```

```Powershell
Get-ChildItem  -Path Cert:\LocalMachine\MY | Where-Object {$_.Thumbprint -Match "F97AC1202A9F70BFEC6FE777080BADBAA923DAA8"} | Select-Object FriendlyName, Thumbprint, Subject, NotBefore, NotAfter
```
