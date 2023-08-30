# How to install Microsoft Store Apps without using the store
<br>

## This article explains the process of how to install Microsoft Store Apps without using the Microsoft Store. In the event access is restricted for users

<br>

### Find the URL of the App
So the first step is to find the URL of the app in the online Microsoft Store. You don’t need the actual store for this, you can just use your browser to [Open The Store](https://apps.microsoft.com/store/apps). If you have found the app that you want to install, just copy the URL from the address bar.

<br>

The URL for the Microsoft Remote Desktop App is:
```
https://apps.microsoft.com/store/detail/microsoft-remote-desktop/9WZDNCRFJ3PS
NOTE: If the URL contains '?activetab=pivot:overviewtab' simply remove it from the URL
```

<br>

### Generate Microsoft Store link
We need to convert the link to the actual Microsoft Store items. To do this we will use the website https://store.rg-adguard.net.

Paste the URL and make sure you change the option RP to Retail
![storebypass-1.png](/storebypass-1.png)

<br>

### Download the appxBundle
After you clicked on the checked mark it will find all the related apps. Most of the time the results start with .Net Frameworks that are required for the app, but we can skip them. Somewhere in the middle, you will find the appxBundles for the Microsoft Remote Desktop App.
![storebypass-2.png](/storebypass-2.png)

<br>

Make sure you select the latest version, ignore the date column, just check the version number. Also, make sure you select the appxBundle and not the eappxBundle. The latter is for Xbox.

To download the appxbundle, copy the link and paste it into a new browser tab. Just click on the link itself doesn’t always work, but opening it in a new tab seems to do the trick.

<br>

### Use PowerShell to install the appxBundle
The last step is to install the Microsoft Remote Desktop App with PowerShell.

``` Powershell
Add-AppxPackage -Path "c:\temp\Microsoft.RemoteDesktop_2022.812.1727.0_neutral___8wekyb3d8bbwe.AppxBundle
```

<br>

> Microsoft Remote Desktop App should now be installed without the need for the store.
{.is-success}
