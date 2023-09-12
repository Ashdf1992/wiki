# Microsoft Exchange Cheat Sheet

## Get the Agent logs
```Powershell
$StartDate = Read-Host "Please enter a Start Date and time using the following format dd/mm/yyy hh:mm: "
$EndDate = Read-Host "Please enter a Start Date and time using the following format dd/mm/yyy hh:mm: "
$Recipient = Read-Host "Please enter a Recipient: "
get-agentlog -StartDate "$StartDate" -EndDate "$EndDate" | where {$_.Recipients -eq "$Recipient"}
```

## Get 'Message Tracking Logs'
```Powershell
$StartDate = Read-Host "Please enter a Start Date and time using the following format dd/mm/yyy hh:mm: "
$EndDate = Read-Host "Please enter a Start Date and time using the following format dd/mm/yyy hh:mm: "
$Recipient = Read-Host "Please enter a Recipient: "
Get-MailboxServer | Get-MessageTrackingLog -Start "$StartDate" -End "$EndDate" -Recipient "$Recipient"
```

## Get Whitespace
```Powershell
$Sort = Read-Host "Sort by 'Name' or 'AvailableNewMailboxSpace': "
Get-MailboxDatabase -Status | sort $Sort | select name,@{Name='DB Size (Gb)';Expression={$_.DatabaseSize.ToGb()}},@{Name='Available New Mbx Space Gb)';Expression={$_.AvailableNewMailboxSpace.ToGb()}} | ft
```

## Move a Mailbox
```Powershell
$MailboxtoMove = Read-Host "Enter the Mailbox you want to move: (eg ash@xyz.co.uk)"
$TargetDB = Read-Host "What Database do you want to move the Mailbox to: (eg XYZ-DB1)"
New-MoveRequest "$MailboxtoMove" -TargetDatabase "$TargetDB" -Priority Emergency -BadItemLimit 20
```

## Check a Mailbox Move
```Powershell
$MailboxtoCheck = Read-Host "Enter the Mailbox you want to move: (eg ash@xyz.co.uk)"
Get-MoveRequest "$MailboxtoCheck" | Select *
```

## Change the Priority of a Move Request
```Powershell
Write-Host "Low"
Write-Host "Normal"
Write-Host "High"
Write-Host "Emergency"
$MailboxtoChange = Read-Host "Enter the Mailbox you want to modify: (eg ash@xyz.co.uk)"
$Priority = Read-Host "What priority do you want to change the move request to?: "
Set-MoveRequest "$MailboxtoChange" -Priority "$Priority"
```

## Check Move Request Statistics
```Powershell
$MailboxtoCheck = Read-Host "Enter the Mailbox you want to check: (eg ash@xyz.co.uk)"
Get-MoveRequestStatistics "$MailboxtoCheck" 
```

## Check Move Request Statistics (Verbose)
```Powershell
$MailboxtoCheck = Read-Host "Enter the Mailbox you want to check: (eg ash@xyz.co.uk)"
Get-MoveRequestStatistics "$MailboxtoCheck" | fl
```

## Check All Move Requests
```Powershell
Get-MoveRequest | Get-MoveRequestStatistics | Select DisplayName, StatusDetail, TotalMailboxSize, PercentComplete, SourceDatabase | ft
```

## Watch All Move Requests
```Powershell
while (1) {Get-MoveRequest | Get-MoveRequestStatistics | Select DisplayName, StatusDetail, TotalMailboxSize, PercentComplete, SourceDatabase | ft; sleep 5} > Watch Equivalent
```

## Watch a specific Move Request
```Powershell
$MailboxtoWatch = Read-Host "Enter the Mailbox you want to watch: (eg ash@xyz.co.uk)"
while (1) {Get-MoveRequestStatistics "$MailboxtoWatch" | Select DisplayName,StatusDetail,LastUpdateTimestamp,BytesTransferred,TotalMailboxSize,ItemsTransferred,TotalMailboxItemCount,PercentComplete | fl; sleep 5} > Watch Equivalent
```

## View Soft Deleted Mailboxes Per Database
```Powershell
$DBtoCheck = Read-Host "Enter the Database that you want to view Soft Deleted mailboxes for: (eg XYZ-DB15)"
Get-MailboxStatistics -Database "$DBtoCheck" | where {$_.DisconnectReason -eq "SoftDeleted"} | Select DisplayName,Database,MailboxGuid,DisconnectDate,DisconnectReason
```

## Remove a Soft Deleted Mailbox
```Powershell
$DatabasetoRemoveFrom = Read-Host "Enter the Database that you want to remove the Soft Deleted Mailbox from: (eg XYZ-DB15)"
$MailboxtoRemove = Read-Host "Enter the Mailbox GUID of the Mailbox that you wish to remove: (eg a12bcdd3-4e56-7fa8-b912-345f1cd6e78f)"
Remove-StoreMailbox -Database "$DatabasetoRemoveFrom" -Identity "$MailboxtoRemove" -MailboxState SoftDeleted
```

## Remove all completed Move Requests
```Powershell
Get-MoveRequest -MoveStatus Completed | Remove-MoveRequest
```

## Remove specific completed Move Request
```Powershell
$MailboxtoRemove = Read-Host "Enter the Completed Move request that you wish to remove: (eg ash@xyz.co.uk)"
Remove-MoveRequest $MailboxtoRemove
```
