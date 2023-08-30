# Output IPs connecting on Specific port every 30 seconds
> Within Powershell ISE run the following
{.is-info}
```Powershell
while(1)
{
   Get-NetTCPConnection -LocalPort 443 | Select RemoteAddress | Export-csv "C:\WebRequests.csv" -Append
   start-sleep -seconds 30
}
```
> (Info) Note that you can change LocalPort to be whatever port you require. The output will be to a CSV file declared within the Export-CSV section
