# Count the Number of Connections and show connections from IPs

<br>

## To list and organise connections showing the IP, based on the total number of connections run the following:

``` Bash
netstat -tn | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
```

## To list and organise connections showing the IP, based on the total number of connections, for a specific IP run the following:
> Replace 192.168.1.100 with the IP you wish to check for
``` Bash
netstat -tn | awk '{print $5}' | cut -d: -f1 | grep -c '192.168.1.100'
```
