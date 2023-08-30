# Clearing IIS Log Files
```Powershell
$LogPath = "C:\inetpub\logs\LogFiles"
$maxDaystoKeep = -30
$outputPath = "c:\inetpub\logs\LogFiles\Cleanup_Old_logs.log"
$itemsToDelete = dir $LogPath -Recurse -File *.log | Where LastWriteTime -lt ((get-date).AddDays($maxDaystoKeep)) 

if ($itemsToDelete.Count -gt 0)
{
    ForEach ($item in $itemsToDelete)
    {
        "$($item.BaseName) is older than $((get-date).AddDays($maxDaystoKeep)) and will be deleted" | Add-Content $outputPath
        Get-item $item.FullName | Remove-Item -Verbose
    }
}
ELSE
{
    "No items to be deleted today $($(Get-Date).DateTime)"  | Add-Content $outputPath
}
```
