# Server Manager - Online - data retrieval failures

## If you are getting errors within Server Manager showing 'Online - data retrieval failures' carry out the following steps within Administrative CMD or Powershell

<br>

Environment: CMD
``` CMD
cd C:\Windows\System32\wbem\AutoRecover
for /f %s in ('dir /b *.mof *.mfl') do mofcomp %s
```
Reference:
https://community.spiceworks.com/topic/2290529-win-svr-2016-vm-error-in-server-manager-online-data-retrieval-failures
