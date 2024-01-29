# IIS and X-Forwarded-For Header (XFF)

> X-Forwarded-For (XFF) header is incredibly useful if you have any kind of proxy in front of your web servers.

## Configure XFF on IIS 8+
```IIS
* Start IIS Manager, then on the Connections pane on the left, click the appropriate website where you want to enable XFF logging. The Home page is then displayed in the main panel.
* From the Home page, double-click Logging.
* From the Log File section, click Select Fields.
* From the bottom left corner, click Add Field.
* In the Add Custom Field window, complete the following:
  * in Field Name, type X-Forwarded-For
  * in Source, type X-Forwarded-For
  * leave Source Type set to ‘Request Header’
  * click OK on the Add Custom Field window
  * click OK on the W3C Logging Fields window
* From the Actions pane on the right, click Apply to implement the change.
* The log files are located by default in the directory %SystemDrive%\inetpub\logs\LogFiles. IIS creates new log files and appends “_x” to the log file names to indicate that they contain custom fields.
```

[comment]: <> (https://www.loadbalancer.org/blog/iis-and-x-forwarded-for-header/)
