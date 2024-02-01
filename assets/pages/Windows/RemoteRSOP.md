# Remote RSOP from a Domain Environment

### This Powershell script will generate a Remote RSOP report and store it within C:\rsop-report for each of the servers declared within '$servers'. The script needs to be ran as a Domain Admin.

```Powershell
Import-Module GroupPolicy
$folderPath = "C:\rsop-report"
if (!(Test-Path $folderPath -PathType Container)) {
    New-Item -ItemType Directory -Force -Path $folderPath
}
$servers = "a.xyz.local","b.xyz.local","c.xyz.local","d.xyz.local","e.xyz.local","f.xyz.local"
$user = whoami
foreach ($s in $servers)
{
    Get-GPResultantSetOfPolicy -Computer $s -user $user -ReportType html -Path $folderPath\$s.html
}
```
