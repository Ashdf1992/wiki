# Get Site Bindings for all sites using https

<br>

```Powershell
$sites = Get-IISSite
foreach ($s in $sites) {Get-IISSiteBinding $s -Protocol https -WarningAction Ignore}
```
