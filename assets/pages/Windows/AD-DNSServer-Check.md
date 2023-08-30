# DNS Check for AD Environment
Environment: CMD
``` Powershell
### Checks all Servers on a domain (Not including Clusters and AVG's), and outputs the DNS clients server
$Computers_Online = @()
$computers = (Get-ADComputer -Filter * | where-object {$_.name -notlike "*DC*" -and $_.name -NotLike "*CLS*" -and $_.name -notlike "*AVG*"}).name
foreach ($Server in $Computers) 
{
    $OnlineCheck = test-connection $Server -count 1 -quiet
        if ($OnlineCheck -eq "True") 
        {
            $Computers_Online += "$Server"
        } 
        else 
        {
            Write-Warning "$Server is not responding to ping"
        }
}

### you may need to change the serveraddress property -like parameter depending on the first octect

Foreach ($Server in $Computers_Online) 
{
    invoke-command -computername $server -scriptblock 
    {
        echo $env:computername Get-DnsClientServerAddress | where -Property ServerAddresses -like "*172*" | Format-List ServerAddresses
    }
}

```
